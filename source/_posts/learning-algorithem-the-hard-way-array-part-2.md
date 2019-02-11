title: Learning algorithem the hard way array (part 2)
date: 2019-02-07 22:09:22
categories: programming
tags:

- algorithms
- data structure
- array

---

# What is Array？

---

数组（Array）是一种线性表数据结构。它用一组连续的内存空间来存储一组具有相同数据类型的数据。

<!--more-->

上述定义当中有有几个较为关键的概念：

## 线性表 （Linear List）

线性表是指数据排成一个线型的结构。每个线性表上的数据最多只有前后两个方向。

除了数组之外，链表、队列、栈也是线性表结构。

![linear-list](https://www.zhuxiaodong.net/static/images/linear-list.jpg)

而与其对应的概念是非线性表，例如二叉树、图、堆等。

![non-linear-list](https://www.zhuxiaodong.net/static/images/non-linear-list.jpg)

## 连续的内存空间和相同的数据类型

数组的随机访问（根据索引下标访问）时间复杂度为 O(1) 。其随机访问的实现原理是：

``` shell
a[i]_address = base_address + i * data_type_size
```

## 为什么大部分编程语言的数组下标是从 0 开始设计，而不是从 1 开始？

我们来思考另外一个问题，为什么大部分编程语言的数组索引下标是从 0 开始，而不是从 1 开始？如果从 1 开始的话，上述的公式必须转化为：

``` shell
a[i]_address = base_address + ((i - 1) * data_type_size)
```

对比从 0 开始的方案，需要多做一次减法运算。因此大部分编程语言都选择从 0 开始作为数组的初始下标。

# 低效的插入和删除操作

---

## 插入操作

假设数组的长度为 n ，我们需要将新元素 e 插入到数组中的第 k 个位置，那我们需要将后续的 n - k 个元素顺序地往后进行移动，因此在这种场景下插入操作的时间复杂度为 O(n) 。

但是有一种特殊场景下：数组仅仅作为一种容器去存储数据，而不用关心数组中存储元素的顺序，能够将数组插入操作的算法复杂度提升为 O(1) 。

举个例子，假设数据 a[10] 中存储了 5 个元素： a、b、c、d、e 。我们将元素 x 插入到第三个位置，只需要将原有的第三个元素 c 移动到最后，再将第三个元素替换为 x ，具体的实现代码如下：

```c
a[5] = c;
a[2] = x;
```

![array-insert-o-1](https://www.zhuxiaodong.net/static/images/array-insert-o-1.jpg)

## 删除操作

与插入操作类似，如果我们要删除第 k 个位置的数据，为了内存的连续性，也需要搬移数据，因此数组的删除操作时间复杂度为 O(n) 。

有一种数组删除操作的优化方案是，不需要每一次删除操作时都移动数据，而是先将被删除的索引位置记录下来，等到数组容量不足时，再一次性地从内存当中移除数据，并移动剩余元素的位置。其核心思想与 JVM 的标记清除内存回收算法非常相似。

# 数组与容器类型的对比

---

许多编程语言中都提供了高级的容器类型，例如 Java 的 ArrayList ， C++ 的 Vector 等，数组与这些高级容器类型相比，有哪些适合的应用场景了？

## 适合使用容器类型的场景

高级容器类型支持动态扩容，适合存储元素的大小无法事先确定的场景。例如 ArrayList 会在存储空间不足时，自动扩容 1.5 倍。**需要注意的是，我们应该养成设置 ArrayList 初始容量的习惯，**因为这样能够节省掉很多次内存分配空间和数据搬移的操作。

```java
//假设从 db 当中获取用户的数据为 10000条
//指定初始容量为 10000 会比不指定容量获得更好的性能
ArrayList<User> list = new ArrayList<User>(10000);
for (int i = 0; i < db.getUserList(); i++) {
    list.add(db[i]);
}
```

## 适合使用数组的场景

ArrayList 无法存储基本类型：int 、long 等，只能存储包装类型： Integer 、Long ，而 Boxing 和 UnBoxing 会有一定的性能损耗，如果特别关注性能，就可以使用数组。

# Java implemention

---

初步实现了类似于 `ArrayList` 的代码。[ref here](https://github.com/LeonZhu1981/algorithm-practice/blob/master/java/array/src/main/java/net/zhuxiaodong/algorithm/practice/array/MyGenericArrayList.java)

JDK 1.8 ArrayList 的实现代码。[ref here](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/tip/src/share/classes/java/util/ArrayList.java)