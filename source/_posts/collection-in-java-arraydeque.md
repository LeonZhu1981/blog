title: Java 集合类详解-ArrayDeque
date: 2018-03-29 08:58:31
categories: programming
tags:
- java
- collection
- ArrayDeque
- algorithms
- data structure
- stack
- queue
- deque
---

# 概述
---

LinkedList 实现了队列接口 Queue 和双端队列接口 Deque ，Java 容器类中还有一个双端队列的实现类 ArrayDeque ，它是基于数组实现的。一般来说，由于需要移动元素，数组的插入和删除效率会非常低，但 ArrayDeque 的效率却非常高，本章我们就来讨论一下它具体是如何实现的。

<!--more-->

ArrayDeque 的构造方法如下：

```
public ArrayDeque()
public ArrayDeque(int numElements)
public ArrayDeque(Collection<? extends E> c)
```

numElements 表示元素个数，初始分配的空间会至少容纳这么多元素，但空间不是正好 numElements 这么大。 ArrayDeque 实现了 Deque 接口，同 LinkedList 一样，它的队列长度也是没有限制的， Deque 扩展了 Queue ，有队列的所有方法，还可以看作栈，有栈的基本方法 push/pop/peek ，此外还包括操作两端的方法如 addFirst/removeLast 等，具体的使用用法与 LinkedList 类似，我们就不再做过多的讲解了。

# 实现原理
---

ArrayDeque 内部主要有如下的实例变量：

```
private transient E[] elements;
private transient int head;
private transient int tail;
```

elements 就是存储元素的数组。 ArrayDeque 的高效来源于 head 和 tail 这两个变量，它们使得物理上简单的从头到尾的数组变为了一个逻辑上循环的数组，避免了在头尾操作时的移动操作。

## 循环数组

对于一般数组，比如 arr ，第一个元素为 arr[0] ，最后一个为 arr[arr.length - 1] 。但对于 ArrayDeque 中的数组，它是一个逻辑上的循环数组，所谓循环是指元素到数组尾之后可以接着从数组头开始，数组的长度、第一个和最后一个元素都与 head 和 tail 这两个变量有关，其具体实现为：

* 如果 head 和 tail 相同，则数组为空，长度为 0 。

![arraydeque-01](https://www.zhuxiaodong.net/static/images/arraydeque-01.png)

* 如果 tail 大于 head ，则第一个元素为 elements[head] ，最后一个为 elements[tail - 1] ，长度为 tail-head ，元素索引从 head 到 tail - 1 。

![arraydeque-02](https://www.zhuxiaodong.net/static/images/arraydeque-02.png)

* 如果 tail 小于 head ，且为0，则第一个元素为 elements[head] ，最后一个为 elements[elements.length - 1]，元素索引从 head 到 elements.length - 1。

![arraydeque-03](https://www.zhuxiaodong.net/static/images/arraydeque-03.png)

* 如果 tail 小于 head ，且大于0，则会形成循环，第一个元素为 elements[head] ，最后一个是 elements[tail - 1]，元素索引从 head 到 elements.length - 1，然后再从 0 到 tail - 1 。

![arraydeque-04](https://www.zhuxiaodong.net/static/images/arraydeque-04.png)

## 构造函数

```
public ArrayDeque() {
  elements = (E[]) new Object[16];
}

public ArrayDeque(int numElements) {
  allocateElements(numElements);
}
```

无参数的构造函数分配了一个长度为 16 的数组。

有参数的构造函数调用了 `allocateElements(numElements)` 方法，其核心逻辑为：

* 如果 numElements 小于 8，其分配的数组长度就是 8 。
* 如果 numElements 大于等于 8 ，分配的实际长度是基于 numElements 计算出的最接近的一个 2^n 数。例如 numElements 为 10 ，则实际分配的长度为 2^4 = 16 ，如果 numElements 为 32 ，则实际分配长度为： 2^6 = 64 。

**为什么要分配的实际长度必须要大于 numElements ？**

因为循环数组必须至少留出一个空余的位置， tail 变量指向下一个空位，为了容纳 numElements 个元素，至少需要 numElements + 1 个位置。

除了上述两个构造函数之外， ArrayDeque 还有一个构造方法：

```
public ArrayDeque(Collection<? extends E> c) {
  allocateElements(c.size());
  addAll(c);
}
```

同样调用 allocateElements 分配数组，随后调用了 addAll ，而 addAll 只是循环调用了 add 方法。

## 从尾部添加元素 add

```
public boolean add(E e) {
  addLast(e);
  return true;
}

public void addLast(E e) {
  if(e == null)
      throw new NullPointerException();
  elements[tail] = e;
  if((tail = (tail + 1) & (elements.length - 1)) == head)
      doubleCapacity();
}
```

首先将元素添加到 tail 处，然后 tail 指向下一个位置，即 `tail = (tail + 1) & (elements.length - 1)`，如果与 head 相同，说明队列满了，会调用 doubleCapacity 扩展数组。

这里重点解释一下 `tail = (tail + 1) & (elements.length - 1)` 代码的原理：

tai + 1 与 elements.length - 1 的 **位与** 操作就可以得到下一个元素的位置，原因是 elements.length 是 2 的冥次方，而 elements.length - 1 的后几位全都是 1 。例如，`elements.length - 1 = 7` ，二进制为 0111 。

```
-1 & 7 = 7
8 & 7 = 0
```

上面两个 **位与** 操作都能够正确找出循环数组中下一个元素的位置。这种位操作是循环数组中一种常见的操作，效率非常高。

**为什么分配的实际长度是基于 numElements 计算出的最接近的一个 2^n 数 ？**

前面我们在讲解构造函数时，我们提到了上面的这个规则，但是没有讲解其原因，学习到这里我们应该明白了，为了支持上述的高效的 **位与** 操作从而得到下一个元素的位置，因此必须要确保 elements.length 是 2 的冥次方。


doubleCapacity() 方法将数组扩大为两倍，分配一个长度翻倍的新数组 a ，将 head 右边的元素复制到新数组开头处，再复制左边的元素到新数组中，最后重新设置 head 和 tail ，head 设为 0 ， tail 设为 n ：

```
private void doubleCapacity() {
  assert head == tail;
  int p = head;
  int n = elements.length;
  int r = n - p; //number of elements to the right of p
  int newCapacity = n << 1;
  if(newCapacity < 0)
      throw new IllegalStateException("Sorry, deque too big");
  Object[] a = new Object[newCapacity];
  System.arraycopy(elements, p, a, 0, r);
  System.arraycopy(elements, 0, a, r, p);
  elements = (E[])a;
  head = 0;
  tail = n;
}
```

我们来看一个例子，假设原长度为 8 ， head 和 tail 为 4 ，现在开始扩大数组，扩大前后的结构如下所示：

![arraydeque-05](https://www.zhuxiaodong.net/static/images/arraydeque-05.png)

## 从头部添加元素 addFirst

```
public void addFirst(E e) {
  if(e == null)
      throw new NullPointerException();
  elements[head = (head - 1) & (elements.length - 1)] = e;
  if(head == tail)
      doubleCapacity();
}
```

在头部添加，要先让 head 指向前一个位置，然后再赋值给 head 所在位置。 head 的前一个位置是`(head - 1) & (elements.length - 1)`。刚开始 head 为 0 ，如果 elements.length 为 8 ，则结果为 **位与** 操作为 7 。

比如如下的代码执行：

```
Deque<String> queue = new ArrayDeque<>(7);
queue.addFirst("a");
queue.addFirst("b");
```

执行完了之后其内部结构为：

![arraydeque-06](https://www.zhuxiaodong.net/static/images/arraydeque-06.png)

## 从头部删除元素 removeFirst

```
public E removeFirst() {
  E x = pollFirst();
  if(x == null)
      throw new NoSuchElementException();
  return x;
}

public E pollFirst() {
  int h = head;
  E result = elements[h]; //Element is null if deque empty
  if(result == null)
      return null;
  elements[h] = null;     //Must null out slot
  head = (h + 1) & (elements.length - 1);
  return result;
}
```

将原头部位置置为 null ，然后 head 置为下一个位置，下一个位置为 `(h + 1) & (elements.length - 1)` 。

## 查看长度 size

```
public int size() {
  return (tail - head) & (elements.length - 1);
}
```

## 检查元素是否存在 contains

```
public boolean contains(Object o) {
  if(o == null)
      return false;
  int mask = elements.length - 1;
  int i = head;
  E x;
  while( (x = elements[i]) != null) {
      if(o.equals(x))
          return true;
      i = (i + 1) & mask;
  }
  return false;
}
```

从 head 开始遍历并进行对比，循环过程中没有使用 tail ，而是到元素为 null 就结束了，这是因为在 ArrayDeque 中，有效元素不允许为 null 。

# ArrayDeque 的复杂度分析以及应用场景
---

ArrayDeque 实现了双端队列，内部使用动态扩展的循环数组实现，通过 head 和 tail 变量维护数组的开始和结尾，数组长度为2的幂次方，使用高效的位操作进行各种判断，以及对 head 和 tail 进行维护。其特点为：

* 在两端添加、删除元素的效率很高，但是由于支持动态扩展，会产生额外的内存分配以及数组复制开销，因此，添加 N 个元素的复杂度为 O(N) 。
* 根据元素内容查找效率比较低，为 O(N) 。
* 与 LinkedList 不同，没有索引位置的概念，不能根据索引位置进行操作；由于没有索引位置的概念，同样也无法支持从指定的位置 insert or remove 一个元素。

ArrayDeque 和 LinkedList 都实现了Deque接口，应该用哪一个呢？如果只需要 Deque 接口，从两端进行操作，一般而言，ArrayDeque 效率更高一些，应该被优先使用；如果同时需要根据索引位置进行操作，或者经常需要在中间进行插入和删除，则应该选择 LinkedList 。
