---
title: Java线程池总结
date: 2017-07-19 17:50:41
tags:
- JUC
- ThreadPoolExecutor
---

## 引言

`java.util.concurrent.ThreadPoolExecutor`是JDK提供的一个线程池实现，在实际项目中是最 常用的JUC组件之一。通过合理使用线程池，可以复用已创建的线程，这不仅能够降低创建和销毁线程的整体消耗，还能够更快速地响应任务，并且通过线程池统一分配、管理、监控，也可以避免无限制地创建线程而造成的系统故障。

<!-- more -->

## 线程池构造

通常情况下，我们会选择使用`java.util.concurrent.Executors`提供的几个工厂方法来创建线程池：

```java
	//创建一个可以按需创建线程的线程池
	public static ExecutorService newCachedThreadPool() {
		return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
	}
	//创建一个固定线程数的线程池
	public static ExecutorService newFixedThreadPool(int nThreads) {
		return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
	}
	//创建一个只包含单个线程的线程池
	public static ExecutorService newSingleThreadExecutor() {
		return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
	}
```

除此之外，如果对于`ThreadPoolExecutor`较为熟悉的话，更加推荐的方式是，通过选定合适的配置参数，直接使用构造器来创建一个线程池：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

构造器中每个参数的含义解释如下：

- **corePoolSize**：线程池的基本大小。线程池会保持这个数量的线程，即使其中包含空闲线程。当线程池的线程数量少于此数量，那么提交新任务会创建一个新线程(*即使有空闲线程*)。
- **maximumPoolSize**：线程池允许创建的最大线程数量。
- **keepAliveTime & unit**：当线程池的线程数量超过基本大小时，多余的线程能够维持空闲的时间。
- **workQueue**：阻塞队列，用于转移或保存已提交的任务。
- **threadFactory**：线程工厂，用于创建新线程的工厂类。
- **handler(RejectExecutionHandler)**：饱和策略。当提交新任务时，如果碰到线程池关闭或者线程池的线程数达到了最大线程数且阻塞队列也达到最大容量，那么就会触发饱和策略。


## 合二为一的CTL

`ThreadPoolExecutor`声明了一个原子整型变量的属性`ctl`用于表达主要的控制状态：

```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

这是一个二合一的属性，之所以这么说，是因为它实际包装了两个概念：`workerCount`和`runState`。

所谓的`workerCount`是指已经允许运行并且未被允许停止的工作线程数量，由于`ctl`只有32位，但又要表达两个概念，因此将低位的29位用于`workerCount`，这限制了工作线程数量最多只能为2<sup>29</sup> -1(*后期这个限制如果变成问题，按照Doug Lea的说法会考虑将ctl变更成`AtomicLong`类型*)。

`runState`由`ctl`的高3位来表示，主要提供对于整个线程池的生命周期的控制，目前有五种状态：

- **RUNNING**：接受新任务，并且会处理已排队任务。
- **SHUTDOWN**：不接受新任务，但是会处理已排队任务。
- **STOP**：不接受新任务，也不处理已排队任务，并且会中断正在执行的任务。
- **TIDYING**：所有的任务都已经终止，工作者线程数量为零，那么会变迁到TIDYING状态，并将运行`terminated()`方法。
- **TERMINATED**：`terminated()`方法调用已经完成。

从数值角度看，`runState`会随着状态的变迁表现出**单调递增**，整体变迁过程如图所示：

![线程池状态变迁图](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/ThreadPoolExecutor_state_transition.png)

## 任务队列

线程池中的任务队列，主要是用于暂存待执行的任务，本质上它是阻塞队列。一般情况下，如果提交新任务时发现线程池的工作线程数量已经超过**corePoolSize**，那么首先会尝试将新任务入队。当然根据实际任务队列类型的不同，这个行为可能会有不同的表现，通常会有以下三种常用的策略：

- **同步移交**：一般的默认选择是使用`SynchronousQueue`，比如`Executors.newCachedThreadPool`就使用了这种策略。当新任务到达时，并不会进行排队行为，而是将其直接移交给工作线程，如果此时没有空闲线程，会尝试进行创建，否则会触发饱和策略。这就意味着使用这种类型的阻塞队列往往需要配合一个非常大甚至是无穷大的**maximumPoolSize**线程池定义。但是这种情况下，如果任务的处理速度跟不上任务达到速度的话，可能导致线程越来越多，进而引发系统问题。采用这种策略的好处是更加高效，因为任务直接移交给执行线程，并且可以避免某些相关依赖的任务可能引发的线程饥饿死锁。
- **无界队列**：最常见的例子是使用一个未定义容量的`LinkedBlockingQueue`，当新任务到达时，如果发现所有的基本线程都有任务在执行，那么会将任务入队，而考虑到队列是无界的，那么**maximumPoolSize**会被忽略，整个线程池不会创建超过**corePoolSize**定义的线程数，所以任务只能等待已有的基本线程空闲下来进行处理。因此，如果任务到达速度超过其处理速度，那么工作队列会无限制的增大。
- **有界队列**：一种更为稳妥的策略是使用类似`ArrayBlockingQueue`这样的有界阻塞队列，如果配合定义一个合理的**maximumPoolSize**，能够有效的避免资源衰竭的情况，但是这两个参数之间的调节可能并不是那么好把握，这中间需要较好的进行权衡。定义较大的队列容量和较小的线程数量，有助于减少CPU使用率、系统资源以及上下文切换开销，但是付出的代价可能是较低的吞吐量。使用较小的队列和较大的线程数量，可以使CPU保持忙碌，但可能遭遇不可接受的调度开销，其也会降低吞吐量。

除此之外，如果你的任务有优先级之分，那么可以考虑使用 `PriorityBlockingQueue`这样的优先队列，它能够使任务不按照提交的顺序来执行，然而需要警惕的是，在一定条件下，也极有可能引起低优先级任务永远无法执行的情况。

## 饱和策略

当我们使用`ThreadPoolExecutor.execute`提交新任务时，如果碰到线程池已经关闭或者线程池饱和(*线程数达到最大允许的线程数量并且工作队列也达到最大容量*)的情况，那么就会触发饱和策略的执行。

接口定义如下：

```java
public interface RejectedExecutionHandler {
    void rejectedExecution(Runnable r, ThreadPoolExecutor executor);
}
```

JDK已经预定义了`AbortPolicy`、`CallerRunsPolicy`、`DiscardPolicy`、`DiscardOldestPolicy`四种饱和策略。如果没有特别指定，默认情况下会启用`AbortPolicy`，其会抛出`RejectedExecutionException`的运行时异常。`DiscardPolicy`则会悄悄忽略新任务，并且什么事也不做。`DiscardOldestPolicy`会抛弃原本下一个将要执行的任务，并且尝试重新提交新任务(*使用这种策略需要注意的是，如果任务队列是优先队列的话，那么被抛弃的会是优先级最高的任务*)。`CallerRunsPolicy`实现了一种调节机制，它会让任务提交者线程自身执行新任务，避免再重新提交任务，从而使得线程池中的工作者线程可以有时间来处理任务。

如果以上几种策略都不满足实际需求，那么你可以通过实现`RejectedExecutionHandler`接口来进行定制化，并且在创建线程池时，配置你自定义的饱和策略。

## 任务提交

线程池最常被使用的方法就是`execute`，它接受一个新任务，并尝试执行，大致的处理流程如下所述：

1. 判断当前工作线程数量是否小于**corePoolSize**，如果是则会创建一个新线程来处理任务，否则进入下一步判断。
2. 判断任务队列是否未满，如果未满则直接将任务入队等待后续执行，否则进行下一步判断。
3. 判断当前工作线程数量是否小于**maximumPoolSize**，如果是则会创建一个新线程来处理任务，否则(*线程池已经达到饱和*)将会触发饱和策略。

下图展示了整个处理过程：

![任务提交流程](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/ThreadPoolExecutor_task_submit.png)

当然上面这个过程是较为粗粒度的描述，如果你查看实际的源码，那么会发现其实在这中间还包含了不少细节考虑，下面以源码注释的方式来进行说明：

```java
public void execute(Runnable command) {
	if (command == null)
		throw new NullPointerException();
	int c = ctl.get();
	//判断当前工作线程是否小于基本线程数，如果是则尝试创建工作线程，否则进入后续判断
	if (workerCountOf(c) < corePoolSize) {
		//在创建新线程之前，会检查线程池状态，并且重新确认工作线程是否小于基本线程
		//如果添加成功则直接返回，否则继续后续判断
		if (addWorker(command, true))
			return;
		c = ctl.get();
	}
	//判定线程池是否还处于运行状态，如果满足，则尝试往队列添加新任务
	if (isRunning(c) && workQueue.offer(command)) {
		//考虑到其他线程对线程池的操作，需要重新对线程池状态进行检测
		int recheck = ctl.get();
		//如果发现此时线程池已经不在运行状态，那么会尝试将任务从队列删除
		//这是由于线程关闭之后，不能再接受新任务的提交
		if (! isRunning(recheck) && remove(command))
			//如果线程池已不在运行状态并且任务删除成功，那么触发饱和策略处理此任务
			reject(command);
		//此时可能情况：1.线程池还在运行状态 2.线程池未在运行状态，但往队列删除任务失败
		//不管是情况1还是情况2，队列此时都不为空，需要判断工作线程是否为零这种可能性
		else if (workerCountOf(recheck) == 0)
			//确认工作线程为零，而此时队列里还有待执行任务，那么尝试创建线程处理队列中的任务
			//请注意，如果线程池状态已经越过SHUTDOWN，则不会考虑创建新线程
			addWorker(null, false);
	}
	//如果线程池已不在运行状态或者添加队列失败(队列已满)，那么尝试创建工作线程
	else if (!addWorker(command, false))
		//此时判定为线程池已关闭或者线程池是饱和状态，需要触发饱和策略
		reject(command);
}
```

## 线程池关闭

`ThreadPoolExecutor`提供了`shutdown`和`shutdownNow`两个方法用于关闭线程池。在上面讲述线程池状态时，提到过调用`shutdown`方法会让线程池的状态变迁到**SHUTDOWN**，而调用`shutdownNow`会让线程池状态直接变迁到**STOP**。 另外这两个方法之间，差异比较大的一点是，调用`shutdown`方法只会阻止任务继续提交，但是队列内的任务依然会执行完成，然而调用`shutdownNow`不仅会阻止任务提交，连队列内的任务也不会执行，甚至还会中断已开始的任务。

结合`shutdown`方法的源码，再了解一下大致过程：

```java
public void shutdown() {
	final ReentrantLock mainLock = this.mainLock;
  	//使用全局锁进行锁定 
	mainLock.lock();
	try {
		checkShutdownAccess();
		//修改线程池状态为SHUTDOWN
		advanceRunState(SHUTDOWN);
		//中断空闲的工作线程
		interruptIdleWorkers();
		//提供的扩展方法，供子类扩展使用
		onShutdown(); // hook for ScheduledThreadPoolExecutor
	} finally {
		//释放全局锁
		mainLock.unlock();
	}
	//尝试终结线程池
	tryTerminate();
}
```

接着结合`shutdownNow`方法的源码，了解和`shutdownNow`方法的区别：

```java
public List<Runnable> shutdownNow() {
	List<Runnable> tasks;
	final ReentrantLock mainLock = this.mainLock;
	//使用全局锁进行锁定 
	mainLock.lock();
	try {
		checkShutdownAccess();
		//修改线程池状态为STOP
		advanceRunState(STOP);
		//尝试中断所有的工作线程
		interruptWorkers();
		//将任务队列排空
		tasks = drainQueue();
	} finally {
		//释放全局锁
		mainLock.unlock();
	}
	//尝试终结线程池
	tryTerminate();
	return tasks;
}
```

## 线程数配置

对于计算密集型任务，在拥有N<sub>cpu</sub>个处理器的系统上，可以考虑将线程池大小设为N<sub>cpu</sub>+1。对于IO密集型任务，则可以考虑将线程池设置为2 * N<sub>cpu</sub>。

在Java并发编程实战一书中，还提出了对于一般的情况可以使用的公式：

 N<sub>threads</sub> = N<sub>cpu</sub> * U<sub>cpu</sub> * ( 1 + W / C)

其中CPU数目可以通过以下方法获取:

```java
int N_CPUS = Runtime.getRuntime().availableProcessors();
```

U<sub>cpu</sub>表示CPU利用率，可能需要在某个基准负载下进行监控。

W/C 是等待时间与计算时间的比值，可能借助分析或监控工具进行粗略估算。



## 参考

[Java并发编程实战](https://book.douban.com/subject/10484692/)

[ThreadPoolExecutor Javadocs](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ThreadPoolExecutor.html)