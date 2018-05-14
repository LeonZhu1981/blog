title:  Java 集合类详解-HashSet
date: 2018-03-30 14:52:13
categories: programming
tags:
- java
- collection
- HashSet
- algorithms
- data structure
---

# 概述
---

在上一篇文章中，我们介绍了 HashMap ，其中 keySet 和 entrySet 返回的是 Set 接口。本篇文章中，我们重点介绍 Set 接口的另外一个重要的实现类 HashSet 。

<!--more-->

Set 表示的是`没有重复元素`、且`不保证顺序`的容器接口，它扩展了 Collection ，但没有定义任何新的方法，不过，对于其中的一些方法，它有自己的规范。其接口定义如下：

```
public interface Set<E> extends Collection<E> {
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    /* 迭代遍历时，不要求元素之间有特别的顺序
    HashSet的实现就是没有顺序，但有的Set实现可能会有特定的顺序，比如TreeSet */
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);
    /* 添加元素, 如果集合中已经存在相同元素了，则不会改变集合，直接返回false，
    只有不存在时，才会添加，并返回true */
    boolean add(E e);
    boolean remove(Object o);
    boolean containsAll(Collection<?> c);
    // 重复的元素不添加，不重复的添加，如果集合有变化，返回true，没变化返回false
    boolean addAll(Collection<? extends E> c);
    boolean retainAll(Collection<?> c);
    boolean removeAll(Collection<?> c);
    void clear();
    boolean equals(Object o);
    int hashCode();
}
```

与 HashMap 类似，HashSet 的构造方法有：

```
public HashSet()
public HashSet(int initialCapacity)
public HashSet(int initialCapacity, float loadFactor)
public HashSet(Collection<? extends E> c)
```

其中 initialCapacity 和 loadFactor 与 HashMap 保持一致。

下面用一个简单的示例来进行说明：

```
import java.util.Set;
import java.util.HashSet;
import java.util.Arrays;

public class HashSetTester {
  public static void main(String [] args) {
    Set<String> set = new HashSet<String>();
    set.add("hello");
    set.add("golang");
    set.add("world");
    set.addAll(Arrays.asList(new String[]{"hello","java"}));
    for(String s : set) {
        System.out.print(s+" ");
    }
  }
}

// Output
hello golang java world
```

与 HashMap 类似， HashSet 要求元素重写 Object 类的 hashCode 和 equals 方法，并且判断两个对象是否相等，必须 equals 和 hashCode 都相同才行。让我们来看下面这个简单的示例。

```
import java.util.Set;
import java.util.HashSet;
import java.util.Arrays;

public class OverrideTester {
  public static void main(String [] args) {
    Set<Spec> set = new HashSet<Spec>();
    set.add(new Spec("M","red"));
    set.add(new Spec("M","red"));
    System.out.println(set);
  }

  private static class Spec {
    String size;
    String color;
    public Spec(String size, String color) {
        this.size = size;
        this.color = color;
    }
    @Override
    public String toString() {
        return "[size=" + size + ", color=" + color + "]";
    }
  }
}
```

Spec 包括了 size 和 color 两个字段，虽然我们在向 HashSet 添加了 2 个 Spec 对象的 size 和 color 都相同，但是你会看到执行结果为：

```
[[size=M, color=red], [size=M, color=red]]
```

显然，Spec 对象才向 HashSet 当中添加时，没有达到过滤掉重复键值对的目的。正确的做法是重写 Object 类的 hashCode 和 equals 方法 。

```
import java.util.Set;
import java.util.HashSet;
import java.util.Arrays;

public class OverrideTester {
  public static void main(String [] args) {
    Set<Spec> set = new HashSet<Spec>();
    set.add(new Spec("M","red"));
    set.add(new Spec("M","red"));
    System.out.println(set);
  }

  private static class Spec {
    String size;
    String color;
    public Spec(String size, String color) {
        this.size = size;
        this.color = color;
    }
    @Override
    public String toString() {
        return "[size=" + size + ", color=" + color + "]";
    }

    @Override
  	public boolean equals(Object o) {
  	    if (this == o) return true;
  	    if (o == null || getClass() != o.getClass()) return false;

  	    Spec spec = (Spec) o;

  	    if (size != null ? !size.equals(spec.size) : spec.size != null) return false;
  	    return color != null ? color.equals(spec.color) : spec.color == null;
  	}

  	@Override
  	public int hashCode() {
  	    int result = size != null ? size.hashCode() : 0;
  	    result = 31 * result + (color != null ? color.hashCode() : 0);
  	    return result;
  	}
  }
}
```

输出结果为：

```
[[size=M, color=red]]
```

tip: 现代的 IDE 能够非常智能的生成类的重写 hashCode 和 equals 方法，例如 IntelliJ IDEA ，具体方式使用快捷键 `cmd + N` ，在弹出的窗口上选择 `equals() and hashCode()` 。

可以参考如下图示：
![generate-equals-hashcode](https://www.zhuxiaodong.net/static/images/generate-equals-hashcode.png)

## HashSet 的应用场景

* 排重：如果对排重后的元素没有顺序要求，则 HashSet 可以方便地用于排除重复数据。
* 保存特殊值：Set 可以用于保存各种特殊值，程序处理用户请求或数据记录时，根据是否为特殊值判断是否进行特殊处理，比如保存IP地址的黑名单或白名单。
* 集合运算：使用 Set 可以方便地进行数学集合中的运算，如交集、并集等运算，这些运算有一些很现实的意义。比如，用户标签计算，每个用户都有一些标签，两个用户的标签交集就表示他们的共同特征，交集大小除以并集大小可以表示他们的相似程度。

# 实现原理
---

## 内部构造

HashSet 内部是用 HashMap 实现的，维护了一个 HashMap 实例变量：

```
private transient HashMap<E,Object> map;
```

虽然内部维护的是一个 HashMap 的结构，但是 HashSet 没有键值对，而只有键，值都是固定的值，值的定义为：

```
private static final Object PRESENT = new Object();
```

## 构造函数

HashSet 的构造方法，主要就是调用了 HashMap 的构造方法：

```
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
```

而接受 Collection 参数的构造方法有一些区别：

```
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
```

其中 `c.size()/.75f` 用于计算 initialCapacity ，而 0.75f 为 loadFactor 的默认值。

## 添加元素 add

```
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

调用 map 的 put 方法，元素 e 用于键，值就是固定值 PRESENT ， put 返回 null 表示原来没有对应的键，添加成功了。 HashMap 中一个键只会保存一份，所以重复添加 HashMap 不会变化。

## 检查是否包含元素 contains

```
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

## 删除元素 remove

```
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

调用 map 的 remove 方法，返回值为 PRESENT 表示原来有对应的键且删除成功了。

## 迭代器

```
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

就是返回 map 的 keySet 的迭代器。

# 总结
---

HashSet 有如下 3 个最大的特点：
* 没有重复元素。
* 可以高效地添加、删除元素、判断元素是否存在，效率都为O(1) 。
* 没有顺序。

HashSet 可以方便高效地实现去重、集合运算等功能。如果要保持添加的顺序，可以使用 HashSet 的一个子类 LinkedHashSet 。 Set 还有一个重要的实现类 TreeSet ，它可以排序。我们将在后续的文章中介绍。
