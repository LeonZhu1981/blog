title: Java 集合类详解-TreeMap
date: 2018-03-30 16:51:19
categories: programming
tags:
- java
- collection
- TreeMap
- algorithms
- data structure
---

# 二叉查找树/平衡二叉树/红黑树
---

由于接下来我们要学习的几种集合类型： TreeMap 、 TreeSet 、 LinkedHashMap 、 LinkedHashSet 、 EnumMap 、 EnumSet 都会基于一些基本的数据结构，例如：二叉查找树、AVL、红黑树等，因此在学习之外，我们有必须要再复习一下这些基础的数据结构知识。

<!--more-->

关于二叉查找树和平衡二叉树的概念，我们已经在之前学习 MariaDB 的索引结构时研究过了，可以参考[这篇文章](https://www.zhuxiaodong.net/2018/learning-mariadb-mysql-index-algorithm-part-8/) 。

## 红黑树

我们都知道，二叉查找树存在的主要问题是，数在插入的时候会导致树倾斜，不同的插入顺序会导致树的高度不一样，而树的高度直接的影响了树的查找效率。理想的高度是 logN ，最坏的情况是所有的节点都在一条斜线上，这样的树的高度为 N 。

基于二叉查找树存在的问题，一种新的树——平衡二叉查找树( Balanced )产生了。平衡树在插入和删除的时候，会通过旋转操作将高度保持在 logN 。其中两款具有代表性的平衡树分别为 AVL 树和红黑树。AVL 树由于实现比较复杂，而且插入和删除性能差，在实际环境下的应用不如红黑树广泛。

红黑树（ Red-Black Tree ）的实际应用非常广泛，比如 Linux 内核中的完全公平调度器、高精度计时器、 ext3 文件系统等等，各种语言的函数库如 Java 的 TreeMap和 TreeSet ， C++ STL 的 map 、 multimap 、 multiset 等。

红黑树有如下5个特点：

* 每个节点要么是红色，要么是黑色；
* 根节点永远是黑色的；
* 所有的叶节点都是是黑色的（注意这里说叶子节点其实是上图中的 NIL 节点）；
* 每个红色节点的两个子节点一定都是黑色；
* 从任一节点到其子树中每个叶子节点的路径都包含相同数量的黑色节点；

![red-black-tree](https://www.zhuxiaodong.net/static/images/red-black-tree.png)

限于篇幅有限，更多的内容请参考：
https://juejin.im/entry/58371f13a22b9d006882902d
https://tech.meituan.com/redblack-tree.html

## 总结

* 二叉查找树保持了元素的顺序，而且是一种综合效率很高的数据结构，基本的保存、删除、查找的效率都为 O(h) ，h 为树的高度。在树平衡的情况下，h为 log 2 (n) ，n 为节点数。比如，如果 n 为 1024，则 log 2 (n) 为 10 。

* 基于二叉查找树所存在的树不平衡的问题，可以使用 AVL 树和红黑树来解决，红黑树的实际应用更为广泛

* 与哈希表一样，树也是计算机程序中一种重要的数据结构和思维方式。如果不需要顺序，优先考虑使用哈希，否则，优先考虑使用树。

# TreeMap
---

## 概述
---

TreeMap 作为 Map 接口的一个实现类，实现了存储的键值对是按照键有顺序的，其实现基础是二叉查找树。除了实现 Map 接口之外，因为有序， TreeMap 还实现了更多的接口和方法。

TreeMap 有两个基本构造方法：

```
public TreeMap()
public TreeMap(Comparator<? super K> comparator)
```

第一个为默认构造方法，如果使用默认构造方法，要求 Map 中的键实现 Comparable 接口， TreeMap 内部进行各种比较时会调用键的 Comparable 接口中的 compareTo 方法。
第二个接受一个比较器对象 comparator ，如果 comparator 不为null，在 TreeMap 内部进行比较时会调用这个 comparator 的 compare 方法，而不再调用键的 compareTo 方法，也不再要求键实现 Comparable 接口。

**TreeMap 是按键而不是按值有序，无论哪一种，都是对键而非值进行比较。**

来我们来看一个实际的示例：

```
import java.util.Map;
import java.util.TreeMap;
import java.util.Map.Entry;

public class TreeMapTester {
  public static void main(String [] args) {
    Map<String, String> map  = new TreeMap<>();
    map.put("a", "abstract");
    map.put("c", "call");
    map.put("b", "basic");
    map.put("T", "tree");
    for (Entry<String,String> kv : map.entrySet()) {
        System.out.print(kv.getKey()+"="+kv.getValue()+" ");
    }
  }
}

// Output
T=tree a=abstract b=basic c=call
```

根据输出我们可以看到，键 T 排在了最前面，这是由于大写字母的 ASCII 码都排在了小写字母 ASCII 码前面。如果希望忽略大小写的话，可以在 TreeMap 的构造函数传入 `String.CASE_INSENSITIVE_ORDER` 这个忽略大小写的 `Comparator` 对象。

```
Map<String, String> map  = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);

// Output
a=abstract b=basic c=call T=tree
```

如果需要 order by desc 的话，可以参考如下的代码：

```
Map<String, String> map  = new TreeMap<>(Collections.reverseOrder());

// Output
c=call b=basic a=abstract T=tree
```

如果需要逆序且忽略大小写的话，可以调整为：

```
Map<String, String> map  = new TreeMap<>(Collections.reverseOrder(String.CASE_INSENSITIVE_ORDER));

// Output
T=tree c=call b=basic a=abstract
```

TreeMap 还有一个特性，就是自动会对键的比较结果进行排重，即使键实际上不同，但只要比较结果相同，它们就会被认为相同，键只会保存一份：

```
Map<String, String> map  = new TreeMap<>(String.CASE_INSENSITIVE_ORDER);
map.put("T", "tree");
map.put("t", "try");
for(Entry<String,String> kv : map.entrySet()) {
    System.out.print(kv.getKey()+"="+kv.getValue()+" ");
}

//Output
T=try
```

再来看另外一个将 TreeMap 当中，日期进行排序的例子：

```
import java.util.Map;
import java.util.TreeMap;
import java.util.Map.Entry;
import java.util.Collections;
import java.util.Comparator;
import java.text.SimpleDateFormat;
import java.text.ParseException;

public class TreeMapTester {
  public static void main(String [] args) {
    Map<String, Integer> map  = new TreeMap<>(new Comparator<String>(){
    	SimpleDateFormat sdf = new SimpleDateFormat("yyyy-mm-dd");

    	@Override
    	public int compare(String o1, String o2) {
    	   try {
    	     return sdf.parse(o1).compareTo(sdf.parse(o2));
    	   } catch (ParseException) {
    	     return 0;
    	   }
    	}
    });
    map.put("2016-7-3", 100);   
    map.put("2016-7-10", 120);
    map.put("2016-7-5", 90);
    for(Entry<String,Integer> kv : map.entrySet()){
      System.out.println(kv.getKey()+","+kv.getValue()+" ");
    }
  }
}

// Output
2016-7-3,100
2016-7-5,90
2016-7-10,120
```

与 HashMap 相比：相同的是，它们都实现了 Map 接口，都可以按 Map 进行操作。不同的是，迭代时， TreeMap 按键有序，为了实现有序，它要求要么键实现 Comparable 接口，要么创建 TreeMap 时传递一个 Comparator 对象。


## 实现原理

TreeMap 内部是用红黑树实现的，我们将在下面主要讨论基于 JDK 1.7 的实现方案：

```
private final Comparator<? super K> comparator;
private transient Entry<K,V> root = null;
private transient int size = 0;
```

comparator 表示一个比较器，用于排序时使用，通过构造函数中传递。 size 为当前键值对个数。 root 指向树的根节点，从根节点可以访问到每个节点，节点的类型为 Entry 。根据如下 Entry 代码的定义，每个节点除了键（ key ）和值（ value ）之外，还包括 left ， right ， parent ，对于根节点， 父节点为 null ，对于叶子节点，孩子节点都为 null ，还有一个成员 color 表示颜色， TreeMap 是用红黑树实现的，每个节点都有一个颜色，非黑即红。

```
static final class Entry<K,V> implements Map.Entry<K,V> {
    K key;
    V value;
    Entry<K,V> left = null;
    Entry<K,V> right = null;
    Entry<K,V> parent;
    boolean color = BLACK;
    Entry(K key, V value, Entry<K,V> parent) {
        this.key = key;
        this.value = value;
        this.parent = parent;
    }
}
```

### 保存键值对 put

如下是 put 方法其中一部分的代码：

```
public V put(K key, V value) {
    Entry<K,V> t = root;
    if(t == null) {
        compare(key, key); // type (and possibly null) check
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }
    //…
```

当添加第一个节点时， root 为 null ，执行的就是这段代码，主要就是新建一个节点，设置 root 指向它， size 设置为 1 ， modCount++ 的含义与之前章节介绍的类似，用于迭代过程中检测结构性的变化。

下面主要研究一下 compare 方法的代码。首先会判断 comparator 类型不匹配或者为 null ， compare 方法就会抛出异常。如果不是第一次添加，会执行后面的代码，添加的关键步骤是查询父节点。

```
final int compare(Object k1, Object k2) {
    return comparator==null ? ((Comparable<? super K>)k1).compareTo((K)k2)
        : comparator.compare((K)k1, (K)k2);
}
```

如果不是第一次添加，会执行后面的代码，添加的关键步骤是寻找父节点。寻找父节点根据是否设置了 comparator 来区分了两种具体的情况：

```
int cmp;
Entry<K,V> parent;
//split comparator and comparable paths
Comparator<? super K> cpr = comparator;
if(cpr != null) {
    do {
        parent = t;
        cmp = cpr.compare(key, t.key);
        if(cmp < 0)
            t = t.left;
        else if(cmp > 0)
            t = t.right;
        else
            return t.setValue(value);
    } while (t != null);
} else {
    if(key == null)
        throw new NullPointerException();
    Comparable<? super K> k = (Comparable<? super K>) key;
    do {
        parent = t;
        cmp = k.compareTo(t.key);
        if(cmp < 0)
            t = t.left;
        else if(cmp > 0)
            t = t.right;
        else
            return t.setValue(value);
    } while(t != null);
}
```

找到父节点之后新建一个节点，根据新的键与父节点的比较结果，插入作为左孩子或右孩子，并增加 size 和 modCount ：

```
Entry<K,V> e = new Entry<>(key, value, parent);
if(cmp < 0)
    parent.left = e;
else
    parent.right = e;
fixAfterInsertion(e);
size++;
modCount++;
```

总结一下，put 方法的基本思路就是：循环比较找到父节点，并插入作为其左孩子或右孩子，然后调整保持树的大致平衡。

### 根据键获取值 get

```
public V get(Object key) {
    Entry<K,V> p = getEntry(key);
    return(p==null ? null : p.value);
}

final Entry<K,V> getEntry(Object key) {
    // Offload comparator-based version for sake of performance
    if(comparator != null)
        return getEntryUsingComparator(key);
    if(key == null)
        throw new NullPointerException();
    Comparable<? super K> k = (Comparable<? super K>) key;
    Entry<K,V> p = root;
    while(p != null) {
        int cmp = k.compareTo(p.key);
        if(cmp < 0)
            p = p.left;
        else if(cmp > 0)
            p = p.right;
        else
            return p;
    }
    return null;
}
```

如果 comparator 不为空，调用单独的方法 getEntryUsingComparator ，否则，假定 key 实现了 Comparable 接口，使用接口的 compareTo 方法进行比较，找的逻辑也很简单，从根开始找，小于往左边找，大于往右边找，直到找到为止，如果没找到，返回 null 。 getEntryUsingComparator 方法的逻辑类似，就不赘述了。

### 查看是否包含某个值 containsValue

TreeMap 可以高效地按键进行查找，但如果要根据值进行查找，则需要遍历：

```
public boolean containsValue(Object value) {
    for(Entry<K,V> e = getFirstEntry(); e != null; e = successor(e))
        if(valEquals(value, e.value))
            return true;
    return false;
}

final Entry<K,V> getFirstEntry() {
    Entry<K,V> p = root;
    if(p != null)
        while (p.left != null)
            p = p.left;
    return p;
}
```

主体就是一个循环遍历， getFirstEntry 方法返回第一个节点， successor 方法返回给定节点的后继节点， valEquals 就是比较值，从第一个节点开始，逐个进行比较，直到找到为止，如果循环结束也没找到则返回 false ， getFirstEntry 的查找方法为，第一个节点就是最左边的节点。

successor 的算法就是查找后继节点：

```
static <K,V> TreeMap.Entry<K,V> successor(Entry<K,V> t) {
    if(t == null)
        return null;
    else if(t.right != null) {
        Entry<K,V> p = t.right;
        while (p.left != null)
            p = p.left;
        return p;
    } else {
        Entry<K,V> p = t.parent;
        Entry<K,V> ch = t;
        while(p != null && ch == p.right) {
            ch = p;
            p = p.parent;
        }
        return p;
    }
}
```

* 如果有右孩子（ t.right != null ），则后继节点为右子树中最小的节点。
* 如果没有右孩子，后继节点为某祖先节点，从当前节点往上找，如果它是父节点的右孩子，则继续找父节点，直到它不是右孩子或父节点为空，第一个非右孩子节点的父亲节点就是后继节点，如果父节点为空，则后继节点为 null 。

### 根据键删除键值对 remove

```
public V remove(Object key) {
    Entry<K,V> p = getEntry(key);
    if(p == null)
        return null;
    V oldValue = p.value;
    deleteEntry(p);
    return oldValue;
}
```

删除节点有三种情况：

* 叶子节点：这个容易处理，直接修改父节点对应引用置 null 即可
* 只有一个孩子：就是在父亲节点和孩子节点直接建立链接。
* 有两个孩子：先找到后继节点，找到后，替换当前节点的内容为后继节点，然后再删除后继节点，因为这个后继节点一定没有左孩子，所以就将两个孩子的情况转换为了前面两种情况。

deleteEntry 方法的代码比较长，需要拆分开来讲解。首先是处理的就是两个孩子的情况， s 为后继，当前节点 p 的 key 和 value 设置为了 s 的 key 和 value ，然后将待删节点 p 指向了 s ，这样就转换为了一个孩子或叶子节点的情况。

```
private void deleteEntry(Entry<K,V> p) {
    modCount++;
    size--;
    //If strictly internal, copy successor's element to p and then make p
    //point to successor.
    if(p.left != null && p.right != null) {
        Entry<K,V> s = successor(p);
        p.key = s.key;
        p.value = s.value;
        p = s;
    } //p has 2 children
```

p 为待删节点， replacement 为要替换 p 的孩子节点，主体代码就是在 p 的父节点 p.parent 和 replacement 之间建立链接，以替换 p.parent 和 p 原来的链接，如果 p.parent 为 null ，则修改 root 以指向新的根。 fixAfterDeletion 重新平衡树。

```
//Start fixup at replacement node, if it exists.
Entry<K,V> replacement = (p.left != null ? p.left : p.right);
if(replacement != null) {
    //Link replacement to parent
    replacement.parent = p.parent;
    if(p.parent == null)
        root = replacement;
    else if(p == p.parent.left)
        p.parent.left  = replacement;
    else
        p.parent.right = replacement;
    // Null out links so they are OK to use by fixAfterDeletion.
    p.left = p.right = p.parent = null;
    // Fix replacement
    if(p.color == BLACK)
        fixAfterDeletion(replacement);
} else if (p.parent == null) { // return if we are the only node.
```



最后来看叶子节点的情况：一种是删除最后一个节点，修改root为null；另一种是根据待删节点是父节点的左孩子还是右孩子，相应的设置孩子节点为null。

```
} else if(p.parent == null) { // return if we are the only node.
    root = null;
} else { //  No children. Use self as phantom replacement and unlink.
    if(p.color == BLACK)
        fixAfterDeletion(p);
    if(p.parent != null) {
        if(p == p.parent.left)
            p.parent.left = null;
        else if(p == p.parent.right)
            p.parent.right = null;
        p.parent = null;
    }
}
```

## TreeMap 总结

TreeMap 的用法和实现原理，与 HashMap 相比， TreeMap 同样实现了 Map 接口，但内部使用红黑树实现。红黑树是统计效率比较高的大致平衡的排序二叉树，这决定了它有如下特点：

* 按键有序， TreeMap 同样实现了 SortedMap 和N avigableMap 接口，可以方便地根据键的顺序进行查找，如第一个、最后一个、某一范围的键、邻近键等。
* 为了按键有序， TreeMap 要求键实现 Comparable 接口或通过构造函数提供一个 Comparator 对象。
* 根据键保存、查找、删除的效率比较高，为 O(h)， h 为树的高度，在树平衡的情况下， h 为log2(N)， N 为节点数。

**TreeMap 和 HashMap 的应用场景**
如果不要求排序，优先考虑 HashMap ，否则，优先考虑 TreeMap 。
