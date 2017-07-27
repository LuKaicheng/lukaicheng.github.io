---
title: 并发容器－理解ConcurrentHashMap(JDK7版本)
date: 2017-07-27 11:54:50
tags:
- JUC
- ConcurrentHashMap
---

## 引言

`HashMap`是程序开发中经常被使用到的一个哈希表实现，然而在多线程情况下，使用`HashMap`却可能引发程序异常()，这是因为它被设计为非线程安全。如果想要确保在多线程场景下的正确性，那么早些时候可能除了使用`sychronized`或者`HashTable`之外，并没有太好的选择。但是当1.5版本引入JUC包之后，在你的工具库里就多了一个更好的选择：`ConcurrentHashMap`。 

<!-- more -->

## 整体概述

作为一个哈希表实现，`ConcurrentHashMap`解决冲突的方法采用的是和`HashMap`一样的链址法(*数组+链表*)：不同hash值的项位于数组的不同索引位置，相同hash的项在数组同一个索引位置通过链表串联起来。同时为了满足多线程安全且提供比`HashTable`更好的并发性能，它引入了分段锁，使得锁的粒度更小，以支持更多的线程可以同时对其进行读写操作。其整体的数据结构如下图所示：

![整体数据结构视图](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/ConcurrentHashMap_Jdk7.png)

其中`HashEntry`代表了实际存储的元素，声明了键值属性以及一个指向冲突节点的next属性：

```java
static final class HashEntry<K,V> {
	final int hash;
	final K key;
	volatile V value;
	volatile HashEntry<K,V> next;
	//省略部分代码
}
```

`Segment`是整个类的核心，它是`ReentrantLock`的子类，一个实例代表了一个分段锁，它的内部声明了其负责管理的集合子集`table`。

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
	//实际管理的HashEntry数组
	transient volatile HashEntry<K,V>[] table;
	//HashEntry数组实际包含的元素数量
	transient int count;
	//HashEntry修改次数
	transient int modCount;
	//元素临届值，当超过这个值需要进行rehash
  	transient int threshold;
	//省略部分代码
}

```

`ConcurrentHashMap`在类中声明了`Segment`数组用来分段管理整个集合：

```java
//ConcurrentHashMap声明分段锁集合
final Segment<K,V>[] segments;
```

## Segment的惰性初始化

为了减少空间的占用，`Segment`分段锁数组被设计成惰性初始化，除了在构造阶段会创建起始位置的`Segment`分段锁，剩下的分段锁根据需要才进行创建。(*个人猜测应该是考虑到`Segment`对象里包含了`HashEntry`数组，后者会占用不少空间*)

查看构造器可以看到，尽管创建了`Segment`分段锁数组，但是其中只有s0被真正赋值：

```java
@SuppressWarnings("unchecked")
public ConcurrentHashMap(int initialCapacity,
						float loadFactor, int concurrencyLevel) {
	//省略部分代码
	Segment<K,V> s0 =
		new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
						(HashEntry<K,V>[])new HashEntry[cap]);
	Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
	UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
	this.segments = ss;
}
```

剩余分段锁的创建会发生在方法`ensureSegment`中，主要是往`ConcurrentHashMap`添加数据的时候触发(*另外，方法`size`和`containsValue`如果碰上需要加锁尝试的情况，也会触发此方法*)：

```java
    @SuppressWarnings("unchecked")
    private Segment<K,V> ensureSegment(int k) {
        final Segment<K,V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K,V> seg;
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
            Segment<K,V> proto = ss[0]; // use segment 0 as prototype
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);
            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                == null) { // recheck
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                        break;
                }
            }
        }
        return seg;
    }
```

创建`Segment`对象并没有使用锁，而是基于volatile+CAS的方式：首先根据索引计算要新创建的`Segment`对象的偏移位置，通过`UNSAFE.getObjectVolatile`方法(*具有volatile读的内存语义*)来获取此位置的最新值，如果返回值为空，则判定还未初始化，那么以构造器阶段创建的`Segment`对象为原型，创建`HashEntry`数组，再构造新的`Segment`对象，最后用CAS的方式尝试赋值。另外需要注意的是，在构建新的`Segment`过程中之所以需要重新检测，是因为整个过程中并没有加锁，不保证原子性，其他线程也同样有机会抢修改此偏移位置的值，所以需要持续检测最新值。

## get方法分析

 同`HashMap`相比，`ConcurrentHashMap`的读操作需要先经历`Segment`定位的过程，而在`Segment`内部的查找行为与`HashMap`类似，另外，对于哈希算法的考虑也有所不同。

### 元素均匀分布的考虑

众所周知，实现一个哈希表最重要的一点是**让所有的元素尽可能均匀分布**，由于`ConcurrentHashMap`引入了`Segment`数组，那么不仅需要满足元素在`Segment`数组中能够均匀分布，而且在每个`Segment`内部的 `HashEntry`数组中也同样需要如此，所以我们看到`ConcurrentHashMap`在计算哈希值时，采用了特别的哈希算法，其计算出来的哈希值同时考虑了元素在`Segment`和`HashEntry`的均匀分布：

```java
    private int hash(Object k) {
        int h = hashSeed;

        if ((0 != h) && (k instanceof String)) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // Spread bits to regularize both segment and index locations,
        // using variant of single-word Wang/Jenkins hash.
        h += (h <<  15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h <<   3);
        h ^= (h >>>  6);
        h += (h <<   2) + (h << 14);
        return h ^ (h >>> 16);
    }
```

 (*这个算法过于复杂，有兴趣可以找相关资料进一步进行了解*)

### Segment的定位

由于在实际运行中，定位`Segment`是很多方法必须经过的一个步骤，因此为了尽可能优化性能，约定了`Segment`数组的容量只能是2的倍数，这样一来就可以利用位操作来取代原本需要的取模操作。

(*当y是2的幂时，x % y = x & (y - 1)，这个思路其实和取掩码的方式一样，不过我没找到严格的数学证明*)

```java
int sshift = 0;
int ssize = 1;
while (ssize < concurrencyLevel) {
	++sshift;
 	ssize <<= 1;
}
this.segmentShift = 32 - sshift;
this.segmentMask = ssize - 1;
```

在实际初始化阶段，首先会计算出`Segment`数组的容量：`ssize`，这个值最终会是一个大于或等于所传入并行级别的数当中最小的2的倍数。而另外一个变量`sshift`表示实际需要多少位来表示`Segment`数组的容量，两者之间的关系：**ssize = 1 << sshift**。

借助这两个临时变量，我们就可以设定全局变量`segmentShift`和`segmentMask`的值：

```java
//段掩码，用于参与与运算以快速定位在Segment数组中的索引位置
final int segmentMask;
//段偏移量，用于指示哈希值中不需要参与定位的低位位数
final int segmentShift;
```

那么实际定位的时候是就可以快速的使用位运算来获取具体的索引，这里需要注意的是由哈希算法计算出来的哈希值，最终会排除`segmentShift`数量的低位，只使用剩余的高位来参与定位：

```java
//计算分段锁的索引
int j = (hash >>> segmentShift) & segmentMask;
```

### HashEntry的定位

同`Segment`数组类似，`HashEntry`数组容量也必须是2的倍数，其会根据指定的初始容量和`Segment`数组的大小，来确定初始情况下每个`Segment`中包含的`HashEntry`数组大小：

```java
int c = initialCapacity / ssize;
if (c * ssize < initialCapacity)
	++c;
int cap = MIN_SEGMENT_TABLE_CAPACITY;
while (cap < c)
	cap <<= 1;
```

最后，结合全部来全面分析一下`get`方法(*`containsKey`方法也是类似思路*)：

1. 首先，计算出传入键的哈希值，并根据位运算计算出可能负责管理此键的`Segment`分段锁索引。
2. 接着，需要注意的是这里利用`UNSAFE.getObjectVolatile`方法提供`volatile`读的内存语义来实现获取最新的`Segment`值，并判定其是否为空。
3. 然后，如果获取到的分段锁不为空，进一步再利用位运算定位在`HashEntry`数组上的索引，和步骤2一样的方式获取最新值，并判定其是否匹配。
4. 最后可以看到，如果步骤3第一次尝试失败，那么还会去检测是否有冲突链，再循着冲突链查看是否有匹配元素，否则返回null。

```java
    public V get(Object key) {
        Segment<K,V> s; // manually integrate access methods to reduce overhead
        HashEntry<K,V>[] tab;
        int h = hash(key);
        long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
        if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
            (tab = s.table) != null) {
            for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                     (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
                 e != null; e = e.next) {
                K k;
                if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                    return e.value;
            }
        }
        return null;
    }
```

## put方法分析

`put`方法可能是除了`get`方法之外另一个出镜率比较高的方法，`ConcurrentHashMap`对外公开的`put`方法首先会定位`Segment`，然后确认是否需要进行`Segment`的初始化行为，这两个过程在上文已经分析过了，在此不再复述，最后我们看到实际的方法功能是由`Segment`内部的`put`方法来实现：

```java
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);
        int j = (hash >>> segmentShift) & segmentMask;
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  in ensureSegment
            s = ensureSegment(j);
        return s.put(key, hash, value, false);
    }
```

那么就让我们来详细了解一下`Segment`内部的`put`方法，以下是整体流程图：

![put方法处理流程](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/ConcurrentHashMap_Jdk7_put.png)

从图中可以看出，整个方法由锁确保了在并发情况下的正确性，主要的业务逻辑其实比较简单：根据传入元素的哈希值，判断是否已经存在相同键值的元素，如果没有的话，那么就会新建一个`HashEntry`对象插入(*值得注意的是，这个新节点会成为链条的起始节点*)，否则根据参数`onlyIfAbsent`来确定是否更新旧值。整个流程图中，值得我们关注的有两个方法：一个是锁获取方法`scanAndLockForPut`，另一个扩容方法`rehash`。

### scanAndLockForPut

`scanAndLockForPut`方法的逻辑同`scanAndLock`方法非常相似，两者都是先以非阻塞的方式尝试获取锁，当超过有限次数之外，会进入阻塞方式的锁请求。在这个过程当中，两者都会对此节点可能的位置进行定位(*根据文档的注释，貌似是一种**代码预热**行为，因为在获取锁之后，接下来的动作也将是对节点的定位*)。此外，如果节点定位成功之后发现首节点发生变化的话，那么又会重新进行定位(*这其中`entryForHash`方法内部确保了总是能及时看见最新值，由`UNSAFE.getObjectVolatile`提供友情支持*)。而`scanAndLockForPut`多出来的一个功能点是，它会根据实际情况推测，在一定的条件下预先创建新节点(*熟悉hadoop的同学一定不陌生，专门有关于**推测执行**的配置*)。以下是完整的源代码：

```java
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; // negative while locating node
            while (!tryLock()) {
                HashEntry<K,V> f; // to recheck first below
                if (retries < 0) {
                    if (e == null) {
                        if (node == null) // speculatively create node
                            node = new HashEntry<K,V>(hash, key, value, null);
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }
```

### 扩容

扩容的基本思路比较简单：创建一个新数组，容量是当前数组的2倍，然后将旧数组的项经过重新散列赋值到新数组，最终将`Segment`内部的`table`属性指向新数组。不过有一点需要注意，由于扩容采用的是倍数2，那么节点经过重新计算的索引值，要么和原先相同，要么新索引＝原索引＋原数组容量(*美团的[这篇文章](https://tech.meituan.com/java-hashmap.html)对这个问题说明的比较清楚*)。利用这一点，在碰到冲突链的时候，会首先寻找冲突链中第一个其所有后续节点新索引值和其保持一致的节点，然后直接把这部分链赋值到新数组，这样避免不必要的节点创建，下面我将通过源码和图片结合的方式进行一下说明。

我们假设冲突链有5个节点：A、B、C、D、E，其中A和C新索引值是2保持不变，B、D、E新索引值是18：

![初始状态](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/chm_rehash_step1.png)

扩容过程如果发现冲突链，那么为了减少不必要的节点创建，首先会循着冲突链寻找可复用的节点：

```java
HashEntry<K,V> lastRun = e;
int lastIdx = idx;
for (HashEntry<K,V> last = next; last != null; last = last.next) {
	int k = last.hash & sizeMask;
	if (k != lastIdx) {
		lastIdx = k;
		lastRun = last;
	}
}
```

上面这段循环结束之后，`lastIdx`为18，`lastRun`指向D，这意味着D节点及其后续所有节点都有相同的索引值：

![循环处理](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/chm_rehash_step2.png)

找到可复用的链之后，先将其赋值到新数组，然后再处理原先冲突链的前半部分，这里值得注意的是插入到新数组之后，节点的顺序会同原先在旧数组的顺序相反：

```java
newTable[lastIdx] = lastRun;
// Clone remaining nodes
for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
	V v = p.value;
	int h = p.hash;
	int k = h & sizeMask;
	HashEntry<K,V> n = newTable[k];
	newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
}
```

最终结果如下图所示：

![迁移完成](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/chm_rehash_step3.png)

## 先乐观后悲观的size

为了避免热点域问题，`ConcurrentHashMap` 并没有定义一个统一的`size`属性，而是在每个`Segment`对象里定义了`size`属性，这样一来使得获取整个集合的元素总数不是那么的容易。出于性能以及准确性的权衡，实际中采用的是先乐观后悲观的理念：首先尝试三次在不加锁的情况下，对所有`Segment`的`size`和`modCount`进行累加，如果连续两次`modCount`的累加结果一致，则认定统计的`size`总和有效，否则所有的`Segment`分段锁都进行加锁操作，再进行统计。(*方法`containsValue`也采用了类似的策略*)

结合源代码，再好好理解一下整个过程：

```java
    static final int RETRIES_BEFORE_LOCK = 2;

    public int size() {
        // Try a few times to get accurate count. On failure due to
        // continuous async changes in table, resort to locking.
        final Segment<K,V>[] segments = this.segments;
        int size;
        boolean overflow; // true if size overflows 32 bits
        long sum;         // sum of modCounts
        long last = 0L;   // previous sum
        int retries = -1; // first iteration isn't retry
        try {
            for (;;) {
                if (retries++ == RETRIES_BEFORE_LOCK) {
                    for (int j = 0; j < segments.length; ++j)
                        ensureSegment(j).lock(); // force creation
                }
                sum = 0L;
                size = 0;
                overflow = false;
                for (int j = 0; j < segments.length; ++j) {
                    Segment<K,V> seg = segmentAt(segments, j);
                    if (seg != null) {
                        sum += seg.modCount;
                        int c = seg.count;
                        if (c < 0 || (size += c) < 0)
                            overflow = true;
                    }
                }
                if (sum == last)
                    break;
                last = sum;
            }
        } finally {
            if (retries > RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    segmentAt(segments, j).unlock();
            }
        }
        return overflow ? Integer.MAX_VALUE : size;
    }
```

顺便提一下，查看相关资料的时候，看见有人提到为何去掉了`size`属性的`volatile`修饰符。我根据文档注释猜测，应该是出于性能方面的考虑，因为对于`size`属性的修改一般都是在加锁的情况下(*方法`put`和方法`remove`都会先加锁*)，因此对于写`size`的情况下即使去掉`volatile`的修饰也没有影响，关键在于读取`size`的场景往往不会加锁(*方法`isEmpty`和方法`size`*)，然而内存可见性的保证又是必须的，那么我们回头看看上面的代码解决这个场景的思路：

```java
Segment<K,V> seg = segmentAt(segments, j);
```

`Segment`对象的获取是通过`segmentAt`方法，那么再深入看一下此方法：

```java
    @SuppressWarnings("unchecked")
    static final <K,V> Segment<K,V> segmentAt(Segment<K,V>[] ss, int j) {
        long u = (j << SSHIFT) + SBASE;
        return ss == null ? null :
            (Segment<K,V>) UNSAFE.getObjectVolatile(ss, u);
    }
```

最终可以明显看到，在读取具体的`Segment`使用了`UNSAFE.getObjectVolatile`来确保`volatile`读语义。

## 总结

本文主要分析了`ConcurrentHashMap`基本原理和的几个常用方法，可以看到除了使用分段锁的概念对锁粒度进行拆分，在几个方法内部还包含了不少的优化之处(*代码预热、推测执行、先乐观后悲观等*)，其中的思想有不少值得我们学习体会和借鉴。另外，JDK从7升级到8之后，除了带来FP、Stream等一系列新概念之外，`ConcurrentHashMap`也进行了翻天覆地的重构，新的实现方式已经抛弃了目前的分段锁概念，关于对新的`ConcurrentHashMap`的学习将留待后期有时间再继续总结。

## 参考

[Java并发编程实战](https://book.douban.com/subject/10484692/)

[Java并发编程的艺术](https://book.douban.com/subject/26591326/)

[Unsafe](https://github.com/openjdk-mirror/jdk7u-jdk/blob/master/src/share/classes/sun/misc/Unsafe.java)

[Java 8系列之重新认识HashMap](https://tech.meituan.com/java-hashmap.html)

