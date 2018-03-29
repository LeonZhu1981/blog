title: Java 集合类详解-ArrayList
date: 2018-03-27 15:58:31
categories: programming
tags:
- java
- collection
- ArrayList
- algorithms
- data structure
- iterator
- iterable
---

# 如何扩容？
---

## add() 方法
在调用 add 方法时，会调用 `ensureCapacityInternal(size + 1);` 方法

```
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
```

<!--more-->

上述代码会首先判断内部数组 elementData 是否为空，如果为空的话，则至少分配 DEFAULT_CAPACITY （初始值为10） 的大小，接下来会调用 `ensureExplicitCapacity(minCapacity);` 方法：

```
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

其中 modCount 表示内部的修改次数。

如果需要的长度大于了当前数组的长度，则调用 `grow()` 方法：

```
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 右移一位相当于除以2.因此 newCapacity 相当于 oldCapacity 的 1.5 倍
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 如果扩展1.5倍还是小于minCapacity，就扩展为minCapacity
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 如果扩展之后的长度大于
    // "private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;"   
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

```
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

## remove() 方法

```
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    // 计算要移动的元素个数
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);

    //将size减1，同时释放引用以便原对象被垃圾回收
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

它也增加了modCount，然后计算要移动的元素个数，从index往后的元素都往前移动一位，实际调用System.arraycopy方法移动元素。elementData[--size]=null；这行代码将size减1，同时将最后一个位置设为null，便于JVM进行垃圾回收。

## 迭代器

迭代器的标准写法为：

```
Iterator<Integer> it = intList.iterator();
while(it.hasNext()) {
    System.out.println(it.next());
}
```

其语法糖为 `foreach`：

```
for(Integer a : intList) {
    System.out.println(a);
}
```

## 迭代器接口

ArrayList实现了Iterable接口，Iterable表示可迭代，Java 7中的定义为

```
public interface Iterable<T> {
    Iterator<T> iterator();

    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }

    default Spliterator<T> spliterator() {
        return Spliterators.spliteratorUnknownSize(iterator(), 0);
    }
}
```

> NOTE: default 方法是 java 8 中新增加的一种为接口添加默认实现方法的扩展方式。

它返回一个实现了Iterator接口的对象

```
public interface Iterator<E> {
    boolean hasNext();

    E next();

    default void remove() {
        throw new UnsupportedOperationException("remove");
    }

    default void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        while (hasNext())
            action.accept(next());
    }
}
```

hasNext（）判断是否还有元素未访问，next（）返回下一个元素，remove（）删除最后返回的元素. 只要对象实现了Iterable接口，就可以使用foreach语法，编译器会转换为调用Iterable和Iterator接口的方法。

* Iterable表示对象可以被迭代，它有一个方法iterator()，返回Iterator对象，实际通过Iterator接口的方法进行遍历；
* 如果对象实现了Iterable，就可以使用foreach语法；
* 类可以不实现Iterable，也可以创建Iterator对象。

# 如何正确地在ArrayList中删除某一个元素
---

下面的这段代码会抛出 `ConcurrentModificationException` ，原因是因为迭代器内部会维护一些索引位置相关的数据，要求在迭代过程中，容器不能发生结构性变化，否则这些索引位置就失效了。所谓结构性变化就是添加、插入和删除元素，只是修改元素内容不算结构性变化。

```
public void remove(ArrayList<Integer> list) {
    for(Integer a : list) {
        if(a<=100) {
            list.remove(a);
        }
    }
}
```

正确地方法是使用 Iterator 的 remove 方法

```
public static void remove(ArrayList<Integer> list) {
    Iterator<Integer> it = list.iterator();
    while(it.hasNext()) {
        if(it.next() <= 100) {
            it.remove();
        }
    }
}
```

## 迭代器 remove 方法的实现原理

ArrayList 中 iterator 方法的实现为：

```
public Iterator<E> iterator() {
    return new Itr();
}
```

新建了一个Itr对象，Itr是一个成员内部类，实现了Iterator接口，声明为：

```
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;
}
```

其中比较重要的是 `expectedModCount` 表示期望的修改次数，其值被初始化为 ArrayList 中的当前修改次数 `modCount` 。由于发生结构性变化的时候modCount都会增加，而每次迭代器操作的时候都会检查expectedModCount是否与modCount相同，这样就能检测出结构性变化。

**为什么通过迭代器去删除元素时，必须调用 next() 方法？**

```
public static void removeAll(ArrayList<Integer> list){
    Iterator<Integer> it = list.iterator();
    while(it.hasNext()){
        it.remove();
    }
}
```

上面这段代码，实际运行时会抛出 `java.lang.IllegalStateException` 异常，正确的写法是：

```
public static void removeAll(ArrayList<Integer> list) {
    Iterator<Integer> it = list.iterator();
    while(it.hasNext()) {
        // 首先调用一次 next 方法
        it.next()
        it.remove();
    }
}
```

要想知道原因，我们就需要看看 `next()` 的定义

```
public E next() {
    checkForComodification();
    int i = cursor;
    if(i >= size)
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if(i >= elementData.length)
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}

final void checkForComodification() {
    if(modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

next前面部分主要就是在检查是否发生了结构性变化，如果没有变化，就更新cursor和lastRet的值，以保持其语义，然后返回对应的元素。

我们再来看看 `remove()` 方法

```
public void remove() {
    if(lastRet < 0)
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

总结一下，如果不调用 `next()` 就直接调用 `remove()` 方法，会导致 cursor 和 lastRet 的指向位置不正确。

# 一些 ArrayList 性能相关的总结
---

## public ArrayList(int initialCapacity)

如果能够在业务场景能够确定 ArrayList 容量大小的情况下，可以使用该构造函数指定初始化的容量大小，这样能够尽量的减少 ArrayList 由于自动扩容造成的额外的内存分配和数组 Copy 。

## public void ensureCapacity(int minCapacity)

```
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```

它可以确保数组的大小至少为minCapacity，如果不够，会进行扩容。如果已经预知ArrayList需要比较大的容量，调用这个方法可以减少ArrayList内部分配和扩展的次数。

## public void trimToSize()

```
/**
 * Trims the capacity of this <tt>ArrayList</tt> instance to be the
 * list's current size.  An application can use this operation to minimize
 * the storage of an <tt>ArrayList</tt> instance.
 */
public void trimToSize() {
    modCount++;
    if (size < elementData.length) {
        elementData = (size == 0)
          ? EMPTY_ELEMENTDATA
          : Arrays.copyOf(elementData, size);
    }
}
```

使用该方法，能将 ArrayList 的内部数组容量减少至实际的存储空间大小，从而达到节约内存的目的。

# ArrayList 的复杂度分析以及应用场景
---

* 可以随机访问，按照索引位置进行访问效率很高，用算法描述中的术语，效率是O(1)，简单说就是可以一步到位。
* 除非数组已排序，否则按照内容查找元素效率比较低，具体是O(N)，N为数组内容长度，也就是说，性能与数组长度成正比。
* 添加元素的效率还可以，重新分配和复制数组的开销被平摊了，具体来说，添加N个元素的效率为O(N)。
* 插入和删除元素的效率比较低，因为需要移动元素，具体为O(N)。
