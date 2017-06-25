---
title: 延迟任务实践总结
date: 2017-06-25 15:41:52
tags:
- DelayQueue
- Redis
---

## 背景

在实际开发项目过程中，可能会碰到类似这样的需求：期望一个任务不是马上执行，而是可以在给定的时间段之后才执行，通常这被称为**延迟任务**。解决这个问题的方案已经有不少，最直接的是采用定时任务的方式扫描数据库，或者考虑使用时间轮算法，或者基于RabbitMQ来实现延迟队列。最近我在项目中就碰到了同样的问题：当新内容添加到系统时，会带有一个过期时间，一旦到达此过期时间，那么需要自动下架该内容。对于此问题，在实际项目演变过程中我采用了与上述不同的两种方式来解决。

<!-- more -->

## 基于DelayQueue实现延迟队列

由于项目最开始服务器资源较为紧张，初步预估先采用单实例部署，故最初想到的方案是使用JDK自带的`java.util.concurrent.DelayQueue`来实现。`DelayQueue`实质上是一个**无界的带有优先级的阻塞队列**，其包含的元素必须实现`java.util.concurrent.Delayed`接口(*实际由内部的`PriorityQueue`持有元素*)：

```java
public interface Delayed extends Comparable<Delayed> {

	long getDelay(TimeUnit unit);
}
```

结合实际问题，创建出具体的`DelayTask`类，它包含过期时间和任务唯一标识，其中，过期时间会参与到方法`compareTo`和`getDelay`的实现，而任务唯一标识除了用于表达任务的唯一性，还用于后续获取该任务的相关信息，从而进行实际的业务处理。

```java
public class DelayedTask implements Delayed {
    //任务唯一标识
    private String uniqueIdentity;
    //过期时间
    private long expireTime;

    public DelayedTask(String uniqueIdentity, long expireTime) {
        this.uniqueIdentity = uniqueIdentity;
        this.expireTime = expireTime;
    }

    //在插入优先队列时被调用，以决定元素具体的插入位置
    @Override
    public int compareTo(Delayed o) {
        if (this.expireTime < ((DelayedTask) o).expireTime)
            return -1;
        else if (this.expireTime > ((DelayedTask) o).expireTime)
            return 1;
        else
            return 0;
    }

    //当前时间减去过期时间，假如小于等于零，则表示该任务已过去
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert(expireTime - System.currentTimeMillis(), TimeUnit.MILLISECONDS);
    }

    public String getUniqueIdentity() {
        return uniqueIdentity;
    }

}
```

接下来基于生产者-消费者模式，使用Spring的`@Service`注解来定义一个延迟队列的单例，这样可以方便我们在多个生产者和消费者之间共享同一个延迟队列。这个类里我们定义了一个任务入队方法`enqueue`和任务出队`dequeue`方法，其中后者实际使用`DelayQueue.take`获取超时任务，如果队列暂时没有任务超时将会导致线程阻塞，因此实际消费者必须在单独线程中获取任务以免影响主线程。

```java
@Service
public class ContentDelayQueue {

    //延迟队列
    private DelayQueue<DelayedTask> queue = new DelayQueue<DelayedTask>();

    //将需要延迟处理的对象放入队列
    public void enqueue(DelayedTask task) {
        queue.offer(task);
    }

    //返回将要超时的对象，如果暂时没有，则会进行阻塞，因此建议在一个独立的线程中进行处理
    public DelayedTask dequeue() throws InterruptedException {
        return queue.take();
    }
}
```

最后还有一点需要注意，使用这种方式实现的延迟队列，当程序运行时数据存在于内存之中，一旦程序停止，整个队列的数据就会消失，因此需要有一个初始化行为保证在程序重启之后可以从数据库获取信息恢复原来的延迟队列。如果使用了Spring的话，可以考虑实现`InitializingBean`接口，把初始化行为放在`afterPropertiesSet`方法里实现。

## 基于Redis实现延迟队列

尽管采用`DelayQueue`可以实现延迟任务的处理，然而如果需要部署多个程序实例，不经过修改是无法满足需求的，但是如果要修改可能并不是一件简单的事情。于是随着项目推进，分布式环境下实现延迟队列这个问题摆在了我的面前。由于之前没有引入RabbitMQ，现有组件只有Redis和Kafka，经过研究，最终决定利用Redis实现分布式环境下的的延迟队列方案。

### 设计思路

参考网上已有的方案([Delayd tasks](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-2-delayed-tasks/))，并结合项目实际情况，确定了最终的总体设计思路，如下所述：

1. 以任务过期时间毫秒数作为分值，将延迟任务添加到Redis Sorted Set，从而使得任务可以根据过期时间的先后顺序进行排序。
2. 利用Redis Sorted Set的`ZRANGEBYSCORE`命令，获取分值在零至当前时间之间的任务(即已经到期的延迟任务)，接着将其插入到Redis Set。
3. 利用Redis Set的`SPOP`命令可以获取并移除Set上的一个到期的延迟任务，处理完业务逻辑之后，再使用Redis Sorted Set的`ZREM`命令移除原始的延迟任务。

### 代码实现

对于延迟任务的定义，依然沿用上一节的思路，只不过这次不需要实现`Delayed`接口：

```java
public class DelayedTask {

    //任务唯一标识
    private String uniqueIdentity;
    //过期时间毫秒数
    private long expireTime;

    public DelayedTask(String uniqueIdentity, long expireTime) {
        this.uniqueIdentity = uniqueIdentity;
        this.expireTime = expireTime;
    }

    public String getUniqueIdentity() {
        return uniqueIdentity;
    }

    public long getExpireTime() {
        return expireTime;
    }

    public void setExpireTime(long expireTime) {
        this.expireTime = expireTime;
    }

}
```

首先实现步骤1，将任务添加到Redis有序集合。由于项目使用的Redis是集群方式配置，因此这里连接Redis的API使用的是Jedis库的`JedisCluster`，注意分值是过期时间，对应内容是延迟任务对象格式化成JSON之后的字符串。

```java
@Service
public class DelayedTaskProducer implements InitializingBean {

    private final static Logger LOGGER = LoggerFactory.getLogger(DelayedTaskProducer.class);

    private static final String KEY_SORTED_SET = "SortedSetForDelayedTask";

    private static final String KEY_SET = "SetForDelayedTask";

    @Autowired
    private JedisCluster client;
    //用于将对象格式化成JSON字符串
    private final ObjectMapper mapper = new ObjectMapper();

    public void addDelayedTask(String uniqueIdentitfy, long expireTime) throws Exception {
        //构建延迟任务
        DelayedTask task = new DelayedTask(uniqueIdentitfy, expireTime);
        //以过期时间为分值，将延迟任务添加到Redis有序集合
        client.zadd(KEY_SORTED_SET, expireTime, mapper.writeValueAsString(task));
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Thread t1 = new Thread(new TransferWorker(), "DelayTaskTransferThread");
        t1.start();
        LOGGER.info("DelayTaskTransferThread Start");
        Thread t2 = new Thread(new FinishWorker(), "DelayTaskFinishThread");
        t2.start();
        LOGGER.info("DelayTaskFinishThread Start");
    }
}
```

接着是步骤2的实现代码，在实际项目实现中，我定义了一个内部类`TransferWorker`，它实现了`Runnable`接口，可以单独运行在一个线程中。主要工作就是根据`zrangeByScore`命令获取已经到期的延迟任务，并将它们转移到另外一个Redis集合中。

```java
    private class TransferWorker implements Runnable {

        @Override
        public void run() {
            while (true) {
                try {
                    //从Redis Sorted Set获取已经过去的延迟任务
                    Set<String> set = client.zrangeByScore(KEY_SORTED_SET, 0,
                            System.currentTimeMillis());
                    if (set.isEmpty()) {
                        //可以做成配置
                        TimeUnit.SECONDS.sleep(30);
                        continue;
                    }
                    String[] members = new String[set.size()];
                    set.toArray(members);
                    //将过去的延迟任务添加到Redis Set
                    client.sadd(KEY_SET, members);
                } catch (Exception e) {
                    LOGGER.error(e.getMessage(), e);
                }
            }
        }
    }
```

然后是步骤3的代码实现，同步骤2的做法一样，我也将`FinishWorker`定义为内部类，并实现了`Runnable`接口。它会从Redis集合中获取延迟任务，并将其移除，随后根据任务信息进行业务处理，最终如果一切顺利，那么会把原始任务从Redis有序集合中移除。

```java
    private class FinishWorker implements Runnable {

        @Override
        public void run() {
            while (true) {
                try {
                    //从Redis Set获取一个延迟任务，并将其从Set中移除
                    String any = client.spop(KEY_SET);
                    if (any == null || any.trim().isEmpty()) {
                        //可以做成配置
                        TimeUnit.SECONDS.sleep(10);
                        continue;
                    }
                    DelayedTask task = mapper.readValue(any, DelayedTask.class);
                    //省略业务处理代码...
                    
                    //将原始的延迟任务从Redis Sorted Set中移除
                    client.zrem(KEY_SORTED_SET, any);
                } catch (Exception e) {
                    LOGGER.error(e.getMessage(), e);
                }
            }
        }

    }
```

最后，我让`DelayedTaskProducer`实现`InitializingBean`接口，这样在初始化时，可以启动`TransferWork`和`FinishWorker`这两个线程任务。当然也可以将`TransferWork`和`FinishWorkder`提取出来，声明成非内部类，并在另外的地方启动它们。另外，由于这仅仅只是示例，所以在代码里只各启动了一个线程，然而在实际情况下，需要根据任务量来最终确定两种任务启动的线程数。

```java
    @Override
    public void afterPropertiesSet() throws Exception {
        Thread t1 = new Thread(new TransferWorker(), "DelayTaskTransferThread");
        t1.start();
        LOGGER.info("DelayTaskTransferThread Start");
        Thread t2 = new Thread(new FinishWorker(), "DelayTaskFinishThread");
        t2.start();
        LOGGER.info("DelayTaskFinishThread Start");
    }
```

###  背后考虑

最后再来总结一下整个设计过程中的一些考虑：

- 最重要的一点是业务处理过程需要满足**幂等性**。
- 对程序异常有一定容忍，允许重试。
- 由于将移除有序集合的原始任务放在最后，因此即使前面的步骤发生异常，也可以通过重新加载有序集合的原始任务进行重试。
- 使用Set作为中转的原因是期望同一个任务只被一个线程消费，用`SPOP`命令可以实现。当然List也能满足这一点，但最终使用Set的原因是为了避免步骤2中多个程序实例的线程插入相同的任务。

##  参考

[Delayd tasks](https://redislabs.com/ebook/part-2-core-concepts/chapter-6-application-components-in-redis/6-4-task-queues/6-4-2-delayed-tasks/)