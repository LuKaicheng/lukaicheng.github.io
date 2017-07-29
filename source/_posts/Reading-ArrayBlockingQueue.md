---
title: 阻塞队列--ArrayBlockingQueue解读分析
date: 2017-07-29 10:55:27
tags:
- JUC
- BlockingQueue
- Java collections framework
---

## 引言

阻塞队列是一种十分有用的数据结构，其非常适合作为生产者－消费者模式当中的中间容器，这是因为与普通的队列相比，它可以在适当的时候通过阻塞生产者或者消费者线程，在一定程度上调节两者之间速率不匹配的问题，起到缓冲的作用。

## 阻塞队列方法简介

阻塞队列接口的全限定名为` java.util.concurrent.BlockingQueue`，其继承了`java.util.Queue`接口，声明如下：

```java
public interface BlockingQueue<E> extends Queue<E> {
	//添加元素，如果空间足够就马上插入，否则抛出IllegalStateException，更推荐使用offer
	boolean add(E e);
	//添加元素，功能同add，但是在空间不足时只会返回false，不会抛异常
	boolean offer(E e);
	//添加元素，如果队列已满则会阻塞线程进行等待
	void put(E e) throws InterruptedException;
	//添加元素，如果队列已满则会尝试限时等待，但一旦超时则会放弃操作
	boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;
	//获取并删除队列头元素，如果队列为空则会进行阻塞等待
	E take() throws InterruptedException;
	//获取并删除队列头元素，如果队列为空则会尝试限时等待，一旦超时则放弃操作
	E poll(long timeout, TimeUnit unit) throws InterruptedException;
	//返回队列还可以接受多少个无阻塞插入的元素
	int remainingCapacity();
	//从队列中移除指定元素
	boolean remove(Object o);
	//判断队列中是否包含指定元素
	public boolean contains(Object o);
  	//将队列中的所有元素移除并加入到传入的集合
	int drainTo(Collection<? super E> c);
	//移除队列中最多指定数量的元素并将其加入到传入的集合
	int drainTo(Collection<? super E> c, int maxElements);
  
}
```

它提供了好几种不同的元素入队和元素出队方法，当然，在实际工程中，`offer`、`put`、`take`、`poll`方法可能使用较多，下面我将以`ArrayBlockingQueue`这个较为常用的阻塞队列实现为例，分析一下实现阻塞的基本思路，其中主要分析以上几个常用方法。

<!--  more  -->

## ArrayBlockingQueue简介

`ArrayBlockingQueue`是一个FIFO(先进先出)的阻塞队列，从名字中就不难猜到，其底层是由一个数组来存储队列元素，另外它声明了两个指针作用的索引变量，用来指示下一个可插入的位置以及下一个元素的位置，当两者递增到预设的容量上限时，会被重置为0，从而可以循环利用已有的数组空间。因此需要注意一点的是，`ArrayBlockingQueue`在初始化设定容量之后，就无法再改变其容量大小，而且它不会进行隐式地扩容。

```java
//队列元素
final Object[] items;
//下一个元素获取的位置
int takeIndex;
//下一个元素可插入的位置
int putIndex;
//队列中的实际元素数量
int count;
```

为了实现线程安全，`ArrayBlockingQueue`内部声明了`ReentrantLock`用于守护对其所有的访问。而它本身之所以能实现阻塞的功能，正是得益于锁上的`notEmpty`和`notFull`这两个条件。其中，`notEmpyt`用于表达队列不为空，`notFull`用于表达队列未满：

```java
//主锁，维护所有的访问
final ReentrantLock lock;
//队列不为空的条件
private final Condition notEmpty;
//队列未满的条件
private final Condition notFull;
```

在初始化过程中，除了创建实际存储的元素之外，锁和条件也将被初始化，其中锁可以指定是公平锁还是非公平锁：

```java
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

## 阻塞入队实现分析

首先来分析一下无超时时间的阻塞入队方法`put`：首先使用主锁进行可中断的锁定，接着判断队列中实际的元素数量是否已经达到数组容量上限，如果判定为真，那么将会触发`notFull.await`方法，这个方法会导致线程放弃持有的锁转入等待状态(*WAITING*)，直到有另一个线程在`notFull`条件上触发`signal`或`signalAll`才有可能重新唤醒此线程继续操作。当然如果判断出此时队列未满，那么就会进行元素入队(`enqueue`)。

(*这里需要意识到的一点是，由于`ArrayBlockingQueue`在元素入列和元素出队都是使用同一把锁进行锁定，所以如果一个线程拿到锁进行插入，那么其他线程不管是插入元素还是弹出元素均需要等待此线程释放锁才能继续操作*)

```java
    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

`enqueue`的整个逻辑也十分简单直接：首先根据之前提到的`putIndex` 索引，将新元素赋值到数组中。接着递增`putIndex`和`count`，其中如果发现递增的`putIndex`达到了数组容量，那么会将其重置为0，以便循环利用已空闲的数组位。最后也是很关键的一点，由于外部插入元素的方法进行了锁定，那么可以保证入队之后，队列至少会包含新插入的这个元素，即意味着队列不为空，因此这时候需要触发`notEmpty.signal`，来唤醒一个等待满足此条件的线程：

```java
    private void enqueue(E x) {
        final Object[] items = this.items;
        items[putIndex] = x;
        if (++putIndex == items.length)
            putIndex = 0;
        count++;
        notEmpty.signal();
    }
```

## 阻塞出队实现分析

阻塞出队的过程与入队的过程极为相似：首先是使用主锁进行可中断锁定，接着判断当前队列是否为空，如果判定成立，那么将会触发`notEmpty.await`，使得当前操作的线程在这一条件上等待，否则就会进行实际的出入操作：

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

`dequeue`的实现并不复杂：首先根据`takeIndex`找到需要出队的元素，接着递增`takeIndex`指向下一个元素，此时同`putIndex`一样，为了循环利用数组，如果判定达到了数组容量上限，那么会被重置为0。`itrs.elementDequeued`方法和迭代器有关，当出队元素时会触发此方法，由于本文主要关注阻塞实现，因为不做过多分析。最后同样也是最关键的，由于外部锁定保证了只有一个线程进行出队操作，那么此时弹出一个元素之后，队列肯定未满，因此需要触发`notFull.signal`来唤醒等待此条件满足的线程。

```java
    private E dequeue() {
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];
        items[takeIndex] = null;
        if (++takeIndex == items.length)
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();
        return x;
    }
```

## 限时阻塞实现分析

了解了阻塞入队和阻塞出入的实现，接下来再来看一下限时等待的实现：

```java
    public boolean offer(E e, long timeout, TimeUnit unit)
        throws InterruptedException {
        checkNotNull(e);
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length) {
                if (nanos <= 0)
                    return false;
                nanos = notFull.awaitNanos(nanos);
            }
            enqueue(e);
            return true;
        } finally {
            lock.unlock();
        }
    }

    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;
                nanos = notEmpty.awaitNanos(nanos);
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

可以看见不论是限时阻塞入队还是限时阻塞出队，其主体流程和不限时的版本高度相似，不同之处在于，限时版本调用的是`Condition.awaitNanos`方法，这个方法使得线程放弃当前持有的锁转入超时等待状态(*TIMED_WAITING*)，如果返回值小于等于0，则表明已经超时，对于这种情况，可以看见两个方法都添加了相应判断逻辑，如果已经超时，方法会直接返回而不进行后续的入队或出队操作。

## 总结

 `ArrayBlockingQueue`通过使用全局锁的方法，来确保任何时间点只有一个线程能进行入队和出队操作，从而保证线程安全(*当然这种实现方式非常影响吞吐量*)。而之所以没有采用`synchronized`，我猜想应该是由于`Object`提供的通知机制(*wait、notify、notifyAll*)无法在一个对象锁上支持多个条件谓词的原因。然而，在`ArrayBlockingQueue`的设计上，它需要有一种机制能够支持不同操作等待不同条件，因此其使用了`ReentrantLock`作为全局锁，从而可以创建多个`Condition`来满足这种需求。其实，这种思路也可以借鉴在我们平时的项目中，当程序需要等待一个条件满足才能进行后续操作的话，可以考虑采用类似的方案来实现。

