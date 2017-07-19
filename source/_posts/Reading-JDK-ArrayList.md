---
title: JDK源码阅读总结--ArrayList
date: 2016-11-17 15:26:10
tags:
- Java collections framework
---
## 1 简介

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

ArrayList是List接口的一种可变长的数组实现，支持随机访问，它允许添加包括**null**在内的所有元素。另外，ArrayList的实现并不是线程安全的，如果有多个线程访问，且其中至少一个会涉及**结构性修改**，最好采用并发控制策略，或者采用*Collections.synchronizedList*进行包装，防止意外的非同步访问。以下所述基于JDK1.7.0_79源码。

<!--more-->

## 2 基本属性

### 2.1 elementData
**elementData**属性是实际存储列表元素的数组缓存区。

```java
private transient Object[] elementData;
```
尽管从声明上看它是数组类型，然而它的长度却并不固定，这里的意思并不说是可以在数组创建之后再改变它的大小，而是指当进行容量评估时，如果发现需要扩容或者收缩，那么会重新创建一个数组并伴随一次拷贝，然后将**elementData**指向新数组，从而达到扩容或者收缩的目的。

### 2.2 size
**size**属性就是列表实际所包含的元素个数。

```java
private int size;
```
这是因为一般情况下，**elementData**数组并不会被完全占满，所以无法用数组长度等价于元素个数，而又需要有一种机制能够快速反馈元素个数，因此设置了**size**属性来缓存这个信息。

```java
public int size(){
    return size;
}

public boolean isEmpty(){
    return size == 0;
}
```
**size()**方法和**isEmpty()**方法就是通过它才保证调用只需花费常数时间。
## 3 容量调整

### 3.1 扩容
在新版本JDK中，ArrayList不仅像之前一样拥有私有的动态扩容方法，同时还对外提供了公有方法供调用者主动进行扩容。
```java
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != EMPTY_ELEMENTDATA) ? 0 : DEFAULT_CAPACITY;
    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}

private void ensureCapacityInternal(int minCapacity) {
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY,minCapacity);
    }
    ensureExplicitCapacity(minCapacity);
}
```
如上所示，**ensureCapacity()**是供外部主动扩容调用的方法，**ensureCapacityInternal()**是供内部扩容调用的方法。这两个方法都是先预估出所需的最小容量，进而通过调用私有方法**ensureExplicitCapacity()**来保证这个容量需求得以满足。接下来看一下这个方法的实现。
```java
private void ensureExplicitCapacity(int minCapacity){
    modCount++;
    if(minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

private void grow(int minCapacity){
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);//扩容到1.5倍
    if(newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if(newCapacity - MAX_ARRAY_SIZE > 0) //overflow处理
	    newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData,newCapacity);
}

private static int hugeCapacity(int minCapacity){
    if(minCapacity < 0)
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? 
        Integer.MAX_VALUE : 
        MAX_ARRAY_SIZE; 
}
```
**ensureExplicitCapacity()**方法会首先判定所需的最小容量是否超过当前数组长度，只有在肯定的情况下，才会调用**grow()**方法。观察此方法代码，首先默认将原先的数组长度扩充到1.5倍，接着将新容量与最小容量比较，如果发现新容量小于最小容量，那么将会把新容量赋值为最小容量。而后则是对容量扩充到1.5倍时可能发生的int值溢出的情况，进行一些处理。最后才是真正的进行新数组创建，旧数组元素拷贝的过程。
### 3.2 收缩
由于列表在增加元素的时候会进行隐式扩容，从而导致底层数组容量往往超过实际所包含的元素，如果碰到资源敏感的场景下，那么可以使用**trimToSize()**方法进行收缩，使得底层数组容量和实际元素个数持平。
```java
public void trimToSize(){
    modCount++;
    if(size < elementData.length){
        elementData = Arrays.copyOf(elementData,size);
    }
}
```
## 4 异常检测

### 4.1 rangeCheck
针对索引范围的检查，ArrayList提供了两个版本的方法：**rangeCheck()**和**rangeCheckForAdd()**
```java
private void rangeCheck(int index){
    if(index >= size)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```
**rangeCheck()**主要是针对*get()、set()、remove()*方法进行索引检测，注意到上面代码并没有对index为负的情况进行检测，这是因为紧跟本方法调用的是对底层数组的访问，而后者会直接检测索引为负的情况，但另外一方面由于隐式扩容的缘故，底层数组包含的实际元素个数往往小于数组长度，因此针对索引超出的情况只能在本方法里判定。
```java
private void rangeCheckForAdd(int index){
    if(index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```
顾名思义，**rangeCheckForAdd()**方法在运行时会被*add()、addAll()*所调用。相比**rangeCheck()**方法，由于后续不是紧跟对底层数组的访问，因此增加了对于索引为负的情况的判定，同时因为允许元素可以末端添加，去掉了index = size这种情况。

### 4.2 modCount
**modCount**是ArrayList从父类AbstractList从继承过来的属性，这个属性用于记录列表经历的结构性修改的次数，主要被迭代器([iterator](#61-iterator)和[listIterator](#62-listiterator))使用。
```java
protected transient int modCount = 0;
```
迭代器借助于它可以提供fast-fail行为，即当迭代器进行迭代时，如果检测它的值被意外修改，可以抛出ConcurrentModificationException异常。
```java
final void checkForComodification() {
    if (modCount != expectedModCount) //与预期不符合，抛出异常
         throw new ConcurrentModificationException();
}
```
## 5 增删改查

### 5.1 set & get

```java
public E set(int index, E element){
	rangeCheck(index);
  	E oldValue = elementData(index);
  	elementData[index] = element;
  	return oldValue;
}

public E get(int index){
  	rangeCheck(index);
  	return elementData(index);
} 

E elementData(int index){
  	reurn (E)elementData[index];
}
```

众所周知，ArrayList的**set()**和**get()**方法均是常数消耗的操作，这是由于底层元素存储依赖于数组，因而在元素获取时可以直接利用数组随机访问的能力，这一点看上面的源码正好可以印证。

### 5.2 add

在一般情况下，从算法分析的角度上说ArrayList的元素添加操作时间复杂度是O(n)，这是因为在当前元素添加元素，往往需要将原先在当前位置的元素以及后续元素，往后移动一位。

```java
public void add(int index,E element){
	rangeCheckForAdd(index);
  	ensureCapacityInternal(size+1);
  	System.arraycopy(elementData,index,elementData,index+1,size-index);
  	elementData[index] = element;
  	size++;
}
```

不过，针对只在末尾添加元素的场景，ArrayList做了特别处理，不需要移动元素位置，直接快速赋值

```java
public boolean add(E e){
  	ensureCapacityInternal(size+1);
  	elementData[size++] = e;
  	return true;
}
```

但是别忘记，不管是一般情况还是特殊情况，最开始都会进行容量检测，这里可能又是一次元素拷贝的开销，因此在容量预知的情况下，应该在ArrayList实例化的时候指定容量。

### 5.3 remove

```java
public E remove(int index){
	rangeCheck(index);
	modCount++;
	E oldValue = elementData(index);//elementData方法在5.1代码示例给出
  	int numMoved = size - index - 1;//计算需要移动的元素个数
 	if(numMoved > 0)
    	System.arraycopy(elementData,index+1,elementData,index,numMoved);  
 	elementData[--size] = null;//实际元素个数-1且将原先末尾置空便于GC 
    return oldValue;  
}
```

同**add()**方法类似，**remove()**则是将被删除元素的后续元素前移了一位，达到删除元素的目的，可以看到，同样也是需要一次数组元素的拷贝。另外，还注意到modCount值发生了改变，这是因为**remove()**是一种结构性的修改。

## 6 迭代器

### <span id="61-iterator">6.1 iterator</span>
ArrayList声明了内部类**Itr**实现了Iterator接口,通过方法**iterator()**对外提供该迭代器。
```java
public Iterator<E> iterator() {
    return new Itr();
}
private class Itr implements Iterator<E>{}
```
**Itr**在内部定义了三个属性：**cursor**用于表示下一个元素的索引，默认初始为0；**lastRet**用于表示上一个返回元素的索引，初始为-1；**expectedModCount**用于表示期望的修改次数，初始为迭代器实例化时外部列表的modCount值。对于**hasNext()**方法，只要判定**cursor**是否已经和当前列表的**size**相同，即可判定是否还存在下一个元素。对于**next()**方法，由于已经知道需要返回的元素的索引(cursor)，那么可以直接通过数组索引访问获取元素，当然对于索引的校验以及列表的结构修改检测也是必须的，且同时需要更新cursor和lastRet。下面主要详细看下**remove()**方法。
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);//后续元素前移 & modCount++
        cursor = lastRet;//元素前移，当前位置的元素的索引发生变化
        lastRet = -1;//只有在next重新调用之后，才能再次调用remove
        expectedModCount = modCount;//由于modCount变化，需要重新赋值
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
**Remove()**方法首先对lastRet进行了检验，结合后续代码中对于lastRet赋值为-1的情况和**next()**方法实现情况，可以知道**remove()**方法只有在**next()**方法被调用一次才能调用并且不能连续连续调用**remove()**方法。接着是上面已经提过的modCount检测。然后才是真正的remove操作，它是通过调用外部对象的remove方法来实现移除最近返回的元素。但是这里要注意的是，调用这个方法，会有两个**"副作用"**：一，被移除元素的后续元素都会前移一位；二，modCount将会+1。因此在最后的代码中，才会看见cursor和expectedModCount被重新赋值，这实质上是为了抵消这两个副作用的影响。

### <span id="62-listiterator">6.2 listIterator</span>
除了一般版本的迭代器，ArrayList还提供了listIterator，与一般版本相比，不仅可以往后进行迭代，还可以向前迭代。外部可以通过**listIterator()**方法获取该迭代器，该方法返回**ListItr**内部类的一个实例，**ListItr**继承关系如下所示。
```java
public ListIterator<E> listIterator() {
    return new ListItr(0);
}
private class ListItr extends Itr implements ListIterator<E> {}
```
**ListItr**继承了**Itr**，在*hasNext()、next()、remove()*这几个方法的实现上，并未改变其父类中的行为，仅仅是对*hasPrevious()、previous()、nextIndex()、previousIndex()、set()、add()*方法进行了实现。**nextIndex()**和**previousIndex()**两个方法都可以直接借助cursor属性进行返回。**hasPrevious()**方法同**hasNext()**实现类似，只不过这次是需要判定cursor是否同0相同,即可判断出是否还存在上一个元素。同样，对于**previous()**方法可参考**next()**，只不过需要返回的是数据中cursor-1位置的数据。**set()**方法则是简单的调用了外部列表的**set()**方法。最后主要来讲一下**add()**方法的实现。
```java
public void add(E e) {
    checkForComodification();

    try {
        int i = cursor;
        ArrayList.this.add(i, e);//当前位置元素以及之后元素后移 & modCount++
        cursor = i + 1;//重新赋值当前游标的值
        lastRet = -1;//保证只有在next或者previous之后，才能重新调用set方法
        expectedModCount = modCount;//由于modCount变化，需要重新赋值
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

**Add()**方法首先照例进行列表结构性修改检测，接着实际上调用了外部ArrayList对象的add方法来进行元素添加，类似的，这个调用也会产生两个**"副作用"**：一、原有当前位置以及之后的元素被后移一位；二、modCount将会+1。因此，在后续代码中可以看到对cursor进行了+1操作，同时将新的modCount值赋予expectedModCount。

## 7 子列表

ArrayList对外暴露了**subList()**方法来提供子列表的功能，但是子列表并不是重新创建的列表对象，它仅仅只是一个*视图*。**SubList**内部实际保存了父列表的对象引用，并定义了**parentOffset**和**offset**两个偏移量。其中**parentOffset**表示子列表相对于父列表的偏移量，**offset**表示子列表相对于根列表的偏移量。在实际调用时，子列表的*get()、set()*方法会通过使用**offset**来快速进行索引，而*add()、remove()*方法则会借助父对象引用和**parentOffset**来调用父类相应方法实现子列表的相应功能。

![offset说明](https://raw.githubusercontent.com/LuKaicheng/lukaicheng.github.io/hexo/source/images/subList.png)

由于子列表和父列表实质上是共用同一份底层数据，那么对于子列表进行的元素添加、删除操作，父列表上也同样会得到体现，然而从反方向看却需要警惕，如果对父列表进行元素添加、删除等涉及结构性修改的操作，会导致已生成的子列表变成不可用(这是因为modCount机制导致)。

## 8 序列化

从ArrayList的类声明中可以知道其实现了**Serializable**接口，能够序列化和反序列化，然而细心的可能已经发现**elementData**属性却是被transient修饰，但是实际序列化过程中，元素数据却并没有丢失。其实，这是因为**elementData**数组实际长度往往大于实际元素个数，如果直接采用默认序列化方式，那么其实除了会序列化实际元素之外，还会去序列化数组的剩余空间，并且在反序列化的时候会浪费额外空间来存储。所以，通过实现了**writeObject()**和**readObject**，ArrayList实现了自定义的序列化和反序列化过程。

```java
private void writeObject(java.io.ObjectOutputStream s)
	throws java.io.IOException{
        int expectedModCount = modCount;
        s.defaultWriteObject();
        s.writeInt(size);
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);//根据实际size序列化数据
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }

private void readObject(java.io.ObjectInputStream s)
	throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        s.defaultReadObject();
        s.readInt();
        if (size > 0) {
            ensureCapacityInternal(size);
            Object[] a = elementData;
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }
```

从实际的源码不难看出，在实际序列化过程中，通过size来确定实际元素并逐个序列化，从而避免序列化不必要的元素，这不仅效率更高，且在反序列化重新构建ArrayList时可以给**elementData**设置合适的容量，避免空间浪费。

## 参考

[ArrayList API](https://docs.oracle.com/javase/7/docs/api/java/util/ArrayList.html)

[why-does-arraylistrangecheck-not-check-if-the-index-is-negative](http://stackoverflow.com/questions/38950203/why-does-arraylistrangecheck-not-check-if-the-index-is-negative)

[why-does-arraylist-use-transient-storage](http://stackoverflow.com/questions/9848129/why-does-arraylist-use-transient-storage)