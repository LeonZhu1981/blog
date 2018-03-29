title: Java 集合类详解-LinkedList
date: 2018-03-28 19:58:31
categories: programming
tags:
- java
- collection
- LinkedList
- algorithms
- data structure
- stack
- queue
- deque
---

# 概述
---

ArrayList随机访问效率很高，但插入和删除性能比较低；LinkedList同样实现了List接口，它的特点与ArrayList几乎正好相反。

<!--more-->

LinkedList 的构造方法与 ArrayList 比较类似：

```
public LinkedList()
public LinkedList(Collection<? extends E> c)

// 使用方式
List<String> list = new LinkedList<>();
List<String> list2 = new LinkedList<>(Arrays.asList(new String[]{"a","b","c"}));
```

LinkedList与ArrayList一样，同样实现了List接口，而List接口扩展了Collection接口，Collection又扩展了Iterable接口，因此其接口方法与ArrayList类似。

# Queue 与 Stack 的实现
---

## Queue

LinkedList 实现了队列数据结构的 Queue 接口，其特点就是先进先出，在队列的尾部添加元素，从头部删除元素。Queue 接口的定义为：

```
public interface Queue<E> extends Collection<E> {
    boolean add(E e);

    boolean offer(E e);

    E remove();

    E poll();

    E element();

    E peek();
}
```

* add()/offer() ： 在尾部添加元素
* element()/peek() ：查看头部元素，返回头部元素，但不改变队列
* remove()/poll() ：删除头部元素，返回头部元素，并从队列中删除该元素

Queue 的每种操作都有两种形式，其区别在于，对于特殊情况的处理不同。特殊情况是指某些场景下队列的为空，或者长度有限制并且队列被占满了的情况。LinkedList的实现中，队列长度没有限制，但别的Queue的实现可能有。在队列为空时，element和remove会抛出异常NoSuchElementException，而peek和poll返回特殊值null；在队列为满时，add会抛出异常IllegalStateException，而offer只是返回false。

LinkedList 被当中做 Queue 的使用方式为：

```
import java.util.Queue;
import java.util.LinkedList;

public class Queue{
     public static void main(String []args) {
        Queue<String> queue = new LinkedList<>();
        queue.offer("a");
        queue.offer("b");
        queue.offer("c");
        while(queue.peek()!=null) {
            System.out.println(queue.poll());
        }
     }
}

// Output
a
b
c
```

## Stack

栈也是一种常用的数据结构，与队列相反，它的特点是先进后出、后进先出，类似于一个储物箱，放的时候是一件件往上放，拿的时候则只能从上面开始拿。Java中没有单独的栈接口，栈相关方法包括在了表示双端队列的接口Deque中，主要有三个方法。

```
void push(E e);
E pop();
E peek();
```

* push表示入栈，在头部添加元素，栈的空间可能是有限的，如果栈满了，push会抛出异常IllegalStateException。
* pop表示出栈，返回头部元素，并且从栈中删除，如果栈为空，会抛出异常NoSuchElementException。
* peek查看栈头部元素，不修改栈，如果栈为空，返回null。

LinkedList 作为 Stack 的使用方式为：

```
import java.util.Deque;
import java.util.LinkedList;

public class Stack {
     public static void main(String []args) {
        Deque<String> stack = new LinkedList<>();
        stack.push("a");
        stack.push("b");
        stack.push("c");
        while(stack.peek()!=null) {
            System.out.println(stack.pop());
        }
    }
}

//Output
c
b
a
```

**Stack 类**

除了上面我们介绍的 Deque 接口可以当成栈使用之外，在 Java 当中还有另外一个类 Stack<E> ，同样可以模拟栈操作。

```
public class Stack<E> extends Vector<E> { }
```

它与 Deque 接口的一个最大的区别是 Stack 类当中的 pop/peek 通过 synchronized 实现了线程安全。不需要线程安全的情况下，推荐使用LinkedList或ArrayDeque。

```
public E push(E item) {
    addElement(item);
    return item;
}

public synchronized E pop() {
    E       obj;
    int     len = size();

    obj = peek();
    removeElementAt(len - 1);

    return obj;
}

public synchronized E peek() {
    int     len = size();

    if (len == 0)
        throw new EmptyStackException();
    return elementAt(len - 1);
}
```

## Deque 接口

栈和队列都是在两端进行操作，栈只操作头部，队列两端都操作，但尾部只添加、头部只查看和删除。有一个更为通用的操作两端的接口Deque。Deque扩展了Queue，包括了栈的操作方法，此外，它还有如下更为明确的操作两端的方法：

```
void addFirst(E e);
void addLast(E e);
E getFirst();
E getLast();
boolean offerFirst(E e);
boolean offerLast(E e);
E peekFirst();
E peekLast();
E pollFirst();
E pollLast();
E removeFirst();
E removeLast();
```

xxxFirst操作头部，xxxLast操作尾部。与队列类似，每种操作有两种形式，区别也是在队列为空或满时处理不同。为空时，getXXX/removeXXX会抛出异常，而peekXXX/pollXXX会返回null。队列满时，addXXX会抛出异常，offerXXX只是返回false。

总结一下：栈和队列只是双端队列的特殊情况，它们的方法都可以使用双端队列的方法替代，不过，使用不同的名称和方法，概念上更为清晰。

# LinkedList 的实现原理
---

ArrayList内部是数组，元素在内存是连续存放的，但LinkedList不是。LinkedList其实就是链表的数据结构实现，确切地说，它的内部实现是双向链表，每个元素在内存都是单独存放的，元素之间通过链接连在一起。

为了表示链接关系，需要一个 Node 的结构。Node 包括实际的元素，但同时有两个链接，分别指向前一个节点（前驱）和后一个节点（后继）。Node 是一个内部类，具体定义为：

```
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

Node类表示节点，item指向实际的元素，next指向后一个节点，prev指向前一个节点。

LinkedList内部组成就是如下三个实例变量：

```
transient int size = 0;
transient Node<E> first;
transient Node<E> last;
```

size表示链表长度，默认为0，first指向头节点，last指向尾节点，初始值都为null。

## add() 方法

```
public boolean add(E e) {
    linkLast(e);
    return true;
}

void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if(l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

其基本步骤如下：

* 创建一个新的节点newNode。l和last指向原来的尾节点，如果原来链表为空，则为null。 `final Node<E> newNode = new Node<>(l, e, null);`
* 修改尾节点last，指向新的最后节点newNode。 `last = newNode;`
* 修改前节点的后向链接，如果原来链表为空，则让头节点指向新节点，否则让前一个节点的next指向新节点。

```
if(l == null)
    first = newNode;
else
    l.next = newNode;
```

* 增加链表大小。`size++`
* 增加 modCount 的大小。`modCount++` ，目的与ArrayList是一样的，记录修改次数，便于迭代中间检测结构性变化。

接下来我们通过一些图例来说明上述的整个过程。

```
List<String> list = new LinkedList<String>();
list.add("a");
list.add("b");
```

第一行代码执行完了之后，其内部结构如图所示：

![linkedlist-add-01](https://www.zhuxiaodong.net/static/images/linkedlist-add-01.png)

在添加 a 元素之后，内部结构为：

![linkedlist-add-02](https://www.zhuxiaodong.net/static/images/linkedlist-add-02.png)

在添加 b 元素之后，内部结构为：

![linkedlist-add-03](https://www.zhuxiaodong.net/static/images/linkedlist-add-03.png)

可以看出，与 ArrayList 不同，LinkedList 的内存是按需分配的，不需要预先分配多余的内存，添加元素只需分配新元素的空间，然后调整前驱和后续节点的指针指向对象即可。

## 根据索引访问元素 get

```
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

checkElementIndex 检查索引位置的有效性，如果无效，则抛出异常：

```
private void checkElementIndex(int index) {
    if(!isElementIndex(index))
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}

private boolean isElementIndex(int index) {
    return index >= 0 && index < size;
}
```

如果index有效，则调用node方法查找对应的节点，其item属性就指向实际元素内容。size >> 1 等于 size/2 ，如果索引位置在前半部分（ index <（size>>1）），则从头节点开始查找，否则，从尾节点开始查找。

```
Node<E> node(int index) {
  if(index < (size >> 1)) {
      Node<E> x = first;
      for(int i = 0; i < index; i++)
          x = x.next;
      return x;
  } else {
      Node<E> x = last;
      for(int i = size - 1; i > index; i--)
          x = x.prev;
      return x;
  }
}
```

因此，LinkedList 通过索引查找的方式与 ArrayList 明显不同， ArrayList 中数组元素连续存放，可以根据索引直接定位，而在 LinkedList 中，则必须从头或尾顺着链接查找，效率比较低。

## 根据内容查找元素 indexOf

```
public int indexOf(Object o) {
  int index = 0;
  if(o == null) {
      for(Node<E> x = first; x != null; x = x.next) {
          if(x.item == null)
              return index;
          index++;
      }
  } else {
      for(Node<E> x = first; x != null; x = x.next) {
          if(o.equals(x.item))
              return index;
          index++;
      }
  }
  return -1;
}
```

其原理是从头节点顺着链接往后找，如果要找的是 null ，则找第一个 item 为 null 的节点，否则使用 equals 方法进行比较。总体而言，其查找的效率也不高，时间复杂度为 O(N) 。

## 在指定的位置插入元素 insert

```
public void add(int index, E element) {
  checkPositionIndex(index);
  if(index == size)
      linkLast(element);
  else
      linkBefore(element, node(index));
}
```

如果 index 为 LinkedList 的大小 ，就直接将元素添加到最后面，否则就插入到 index 对应节点的前面，调用方法为 linkBefore ：

```
void linkBefore(E e, Node<E> succ) {
  final Node<E> pred = succ.prev;
  final Node<E> newNode = new Node<>(pred, e, succ);
  succ.prev = newNode;
  if(pred == null)
      first = newNode;
  else
      pred.next = newNode;
  size++;
  modCount++;
}
```

参数 succ 表示后继节点，变量 pred 表示前驱节点，目标节点将会在 pred 和 succ 中间插入 ，其具体步骤是：

* 新建一个节点 newNode ，前驱为 pred ，后继为 succ 。 `Node<E> newNode = new Node<>(pred, e, succ);`
* 让后继的前驱指向新节点。 `succ.prev = newNode;`
* 让前驱的后继指向新节点，如果前驱为空，那么修改头节点指向新节点。

```
if (pred == null)
  first = newNode;
else
  pred.next = newNode;
```

还有一点需要注意的是 `linkBefore(element, node(index))` 这里传入的第二个参数，和前面我们讲解 `根据索引访问元素 get` 一致，由于需要先查找到被插入元素的位置，而这里查找的时间复杂度为 O(N/2) ，因此这也直接影响了 LinkedList 在中间任意位置插入节点的性能。

* 增加长度

我们还是通过具体的图示来看是如何操作的。

```
List<String> list = new LinkedList<String>();
list.add("a");
list.add("b");
```

在上述代码的基础上，我们在索引位置 1 添加一个元素 c 。

```
list.add(1, "c");
```

其内存结构为：

![linkedlist-insert-01](https://www.zhuxiaodong.net/static/images/linkedlist-insert-01.png)

可以看出，在中间插入元素，LinkedList 只需按需分配内存，修改前驱和后继节点的链接， 但由于需要先根据索引查找到原有元素的位置（参考上文对 node(int index) 方法的峰分析），因此其时间复杂度为 O(N/2) ，而 ArrayList 则可能需要分配很多额外空间，且移动所有后续元素。


## 在指定的位置删除元素 remove

```
public E remove(int index) {
    checkElementIndex(index);
    return unlink(node(index));
}
```

通过 node 方法找到节点后，调用了 unlink 方法。

```
E unlink(Node<E> x) {
  final E element = x.item;
  final Node<E> next = x.next;
  final Node<E> prev = x.prev;
  if(prev == null) {
      first = next;
  } else {
      prev.next = next;
      x.prev = null;
  }
  if(next == null) {
      last = prev;
  } else {
      next.prev = prev;
      x.next = null;
  }
  x.item = null;
  size--;
  modCount++;
  return element;
}
```

删除 x 节点，基本思路就是让 x 的前驱和后继直接链接起来，next 是 x 的后继， prev 是 x 的前驱，具体分为两步：
* 让 x 的前驱的后继指向 x 的后继。如果 x 没有前驱，说明删除的是头节点，则修改头节点指向x的后继。
* 让 x 的后继的前驱指向 x 的前驱。如果 x 没有后继，说明删除的是尾节点，则修改尾节点指向x的前驱。

可以看到，LinkedList 删除一个节点的复杂度为 O(N) ，其效率很高。

# LinkedList 的复杂度分析以及应用场景
---

LinkedList 是一个 List ，但也实现了 Deque 接口，可以作为队列、栈和双端队列使用。实现原理上， LinkedList 内部是一个双向链表，并维护了长度、头节点和尾节点，其特性为：
* 按需分配空间，不需要预先分配很多空间。
* 不可以随机访问，按照索引位置访问效率比较低，必须从头或尾顺着链接找，复杂度为 O(N/2) 。
> 之所以为 O(N/2)，是因为会首先判断索引位置是否在中间，来决定是头节点进行遍历，还是从尾节点进行遍历。

* 不管列表是否已排序，只要是按照内容查找元素，效率都比较低，必须逐个比较，复杂度为 O(N)。
* 在两端添加、删除元素的复杂度为 O(1)。
* 在中间插入、删除元素，要先定位，效率比较低，复杂度为 O(N/2)，但修改本身的效率很高，复杂度为 O(1)。

理解了 LinkedList 和 ArrayList 的特点，就能比较容易地进行选择了，如果列表长度未知，添加、删除操作比较多，尤其经常从两端进行操作，而按照索引位置访问相对比较少，则 LinkedList 是比较理想的选择。
