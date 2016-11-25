---
title: JDK源码阅读--ArrayList
date: 2016-11-17 15:26:10
tags:
---
## 1 简介

ArrayList是List接口的一种可增长数组实现。
以下基于JDK1.7.0_79。

## 2 基本属性

### 2.1 elementData
**elementData**属性是实际存储列表元素的数组缓存区。

```java
private transient Object[] elementData;
```
尽管从声明上看它是数组类型，然而它的长度却并不固定，这里的意思并不说是可以在数组创建之后再改变它的大小，而是指当进行容量评估时，如果发现需要扩容或者收缩时，那么会重新创建一个数组并伴随一次拷贝，然后将**elementData**指向新数组，从而达到扩容或者收缩的目的。

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
**ensureExplicitCapacity()**方法会首先判定所需的最小容量是否超过当前数组长度，只有在肯定的情况下，才会调用**grow()**方法。观察10-11行代码，JDK会默认将原先的数组长度扩充到1.5倍，接着12-13行代码，新容量会同最小容量比较，如果新容量小于最小容量，那么将会把新容量赋值为最小容量。而代码14-15行，则是考虑到了容量扩充到1.5倍时可能发生的int值溢出并进行相应处理。最后16行才是真正的进行新数组创建，旧数组元素拷贝的过程。
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
**rangeCheck()**主要是针对get()、set()、remove()方法进行索引检测，注意到上面代码并没有对index为负的情况进行检测，这是因为紧跟本方法调用的是对底层数组的访问，而后者会直接检测索引为负的情况，但另外一方面由于隐式扩容的缘故，底层数组包含的实际元素个数往往小于数组长度，因此针对索引超出的情况只能在本方法里判定。
```java
private void rangeCheckForAdd(int index){
    if(index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```
顾名思义，**rangeCheckForAdd()**方法在运行时会被add()、addAll()所调用。相比**rangeCheck()**方法，由于后续不是紧跟对底层数组的访问，因此增加了对于索引为负的情况的判定，同时因为允许元素可以末端添加，去掉了index = size这种情况。

### 4.2 modCount
**modCount**是ArrayList从父类AbstractList从继承过来的属性，这个属性用于记录列表经历的结构性修改的次数，主要被迭代器([iterator](#51-iterator)和[listIterator](#52-listiterator))使用。
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
## 5 迭代器

### <span id="51-iterator">5.1 iterator</span>
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
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
第7行调用外部对应的remove方法，可不仅仅只是删除了上一个返回的元素，还做了另外两件事情：一、将被删除元素之后的元素统一进行了前移；二、将modCount进行了+1操作。因此，可以看到代码第8行，cursor的值被赋予上一次返回的索引位置，这是因为下个元素被前移的缘故。第9行将lastRet重新赋值为-1，结合代码第2、3行，保证只有再次调用next之后，才能重新调用remove。最后第10行，将expetedModCount重新赋值为列表的modCount，这是由于上述第二个副作用的缘由，如果不重新赋值，将无法通过下一次的checkForComodification检测。
### <span id="52-listiterator">5.2 listIterator</span>
除了一般版本的迭代器，ArrayList还提供了listIterator，与一般版本相比，不仅可以往后进行迭代，还可以向前迭代。外部可以通过**listIterator()**方法获取该迭代器，该方法返回**ListItr**内部类的一个实例，**ListItr**继承关系如下所示。
```java
public ListIterator<E> listIterator() {
    return new ListItr(0);
}
private class ListItr extends Itr implements ListIterator<E> {}
```
由于**List**继承了**Itr**，在**hasNext()、next()、remove()**这几个方法的实现上，并未改变父类中的行为，仅仅实现了**hasPrevious()、previous()、nextIndex()、previousIndex()、set()、add()**。**nextIndex()**和**previousIndex()**由于cursor属性的存在，实现比较简单。**hasPrevious()**方法同next实现类似，只不过这次是需要判定cursor是否同0相同。

**未完待续...**

## 参考

[why-does-arraylistrangecheck-not-check-if-the-index-is-negative](http://stackoverflow.com/questions/38950203/why-does-arraylistrangecheck-not-check-if-the-index-is-negative)