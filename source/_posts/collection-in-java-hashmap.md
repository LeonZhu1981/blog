title: Java 集合类详解-HashMap
date: 2018-03-29 17:50:30
categories:
tags:
- java
- collection
- HashMap
- algorithms
- data structure
- Map
- Set
---

# Map & Set 概述
---

通过前面几个章节的介绍，我们学习了 ArrayList ， LinkedList ，ArrayDeque ，这几个列表和队列的集合类型有一个共同的缺点：查找元素（ 这里的查找不是指根据索引位置来访问元素 ）的效率较低，时间复杂度都基本为： O(N) 。从本篇文章开始，我们会介绍各种 Map 和 Set ，其查找效率会远远高于列表和队列。 Map 与 Set 都是接口类型，有多种不同的实现：

HashMap 、 HashSet 、 TreeMap 、 TreeSet 、 LinkedHashMap 、 LinkedHashSet 、 EnumMap 、 EnumSet

我们先从最为常用的 HaspMap 开始。

<!--more-->

# Map 接口
---

Map 由 Key 和 Value 组成。一个 Key 会映射到一个 Value， Map 按照 Key 存储和访问值，且 Key 不能重复，给同一个键重复设值会覆盖原来的值。

典型的业务应用场景包括：

* 一个词典应用，键可以为单词，值可以为单词信息类，包括含义、发音、例句等。
* 统计和记录一本书中所有单词出现的次数，可以以单词为键，以出现次数为值。
* 管理配置文件中的配置项，配置项是典型的键值对。
* 根据身份证号查询人员信息，身份证号为键，人员信息为值。

> 我们前面介绍的数组、 ArrayList 、 LinkedList 可以视为一种特殊的 Map ，键为索引，值为对象。

Map 在 Java 7 中的定义为：

```
// K和V是类型参数，分别表示键(Key)和值(Value)的类型
public interface Map<K,V> {
    V put(K key, V value); //保存键值对，如果原来有这个key，则覆盖并返回原来的值
    V get(Object key); //根据键获取值, 如果没找到，返回null
    V remove(Object key); //根据键删除键值对, 返回key原来的值，如果不存在，返回null
    int size(); //查看Map中键值对的个数
    boolean isEmpty(); //是否为空
    boolean containsKey(Object key); //查看是否包含某个键
    boolean containsValue(Object value); //查看是否包含某个值
    void putAll(Map<? extends K, ? extends V> m); //保存m中的所有键值对到当前Map
    void clear(); //清空Map中所有键值对
    Set<K> keySet(); //获取Map中键的集合
    Collection<V> values(); //获取Map中所有值的集合
    Set<Map.Entry<K, V>> entrySet(); //获取Map中的所有键值对

    //嵌套接口，表示一条键值对
    interface Entry<K,V> {
        K getKey(); //获取键值对的键
        V getValue(); //获取键值对的值
        V setValue(V value); //设置键值对的值
    }
}
```

Set 是一个接口，表示的是数学中的集合概念，即没有重复的元素集合。Java 7中的Set定义为：

```
public interface Set<E> extends Collection<E> {
}
```

它扩展了Collection，但没有定义任何新的方法，不过，它要求所有实现者都必须确保 Set 的语义约束，即不能有重复元素。

Map 中的键是没有重复的，keySet()、 values() 、 entrySet() 有一个共同的特点，它们返回的都是视图，不是复制的值，基于返回值的修改会直接修改 Map 自身。例如： `map.keySet().clear();` 会删除所有键值对 。

# HashMap
---

## 应用示例

我们通过一个实际的示例是说明 HashMap 的使用方法，下面这段代码用于统计出， 随机产生一个 0 ~ 3 之间的数字 1000 次，每一个数字产生了多少次，以测试 Random 类的随机分布性。

```
import java.util.Map;
import java.util.HashMap;
import java.util.Random;

public class RandomCounter {
  public static void main(String []args) {
      Random rnd = new Random();
      Map<Integer, Integer> countMap = new HashMap<>();

      for (int i = 0; i < 1000; i++) {
          int num = rnd.nextInt(4);
          Integer count = countMap.get(num);

          if (count == null) {
              countMap.put(num, 1);
          } else {
              countMap.put(num, ++count);
          }
      }

      for (Map.Entry<Integer, Integer> kv : countMap.entrySet()) {
          System.out.println(kv.getKey() + "," + kv.getValue());
      }
  }
}
```

其运行结果为：

```
0,245
1,249
2,267
3,239
```

## 实现原理

HashMap 的构造函数包括：

```
public HashMap(int initialCapacity)
public HashMap(int initialCapacity, float loadFactor)
public HashMap(Map<? extends K, ? extends V> m)
```

而其主要的实例变量为：

```
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;
transient int size;
int threshold;
final float loadFactor;

static class Entry<K,V> implements Map.Entry<K,V> {
  final K key;
  V value;
  Entry<K,V> next;
  int hash;
  Entry(int h, K k, V v, Entry<K,V> n) {
      value = v;
      next = n;
      key = k;
      hash = h;
  }
}
```

HashMap 中最重要的实例变量是一个 Entry 类型的数组，被称为哈希表或哈希桶。而 Entry 类型当中又定义了 hash 实例变量，表示 key 的 hash 值。直接存储 hash 值是为了在比较的时候提升效率。

HashMap 中另外一个重要的实例变量为 table ，其初始值为：EMPTY_TABLE ，其实就是一个空数组。

```
static final Entry<?,?>[] EMPTY_TABLE = {};
```

### HashMap 如何进行扩容

当添加键值对后，table 就不是空表了，它会随着键值对的添加进行扩容，扩容的策略类似于 ArrayList 。添加第一个元素时，默认分配的大小为 16 ，不过，并不是当 size 大于 16 时再进行扩容，而是与 `threshold` 实例变量有关。

`threshold` 表示阈值，当键值对个数 size 大于等于 threshold 时会进行扩容。threshold 是怎么算出来的呢？一般而言，threshold = table.length * loadFactor 。 比如，如果 table.length 为 16 ，loadFactor 为 0.75，则 threshold 为 12 。loadFactor 是负载因子，表示整体上 table 被占用的程度，是一个浮点数，默认为 0.75 ，可以通过构造方法进行修改。

### 默认的构造方法

```
public HashMap() {
  this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}
```

DEFAULT_INITIAL_CAPACITY 为 16 ， DEFAULT_LOAD_FACTOR 为 0.75。

### 保存键值对 put

```
public V put(K key, V value) {
  if(table == EMPTY_TABLE) {
      inflateTable(threshold);
  }
  if(key == null)
      return putForNullKey(value);
  int hash = hash(key);
  int i = indexFor(hash, table.length);
  for(Entry<K,V> e = table[i]; e != null; e = e.next) {
      Object k;
      if(e.hash == hash && ((k = e.key) == key || key.equals(k))) {
          V oldValue = e.value;
          e.value = value;
          e.recordAccess(this);
          return oldValue;
      }
  }
  modCount++;
  addEntry(hash, key, value, i);
  return null;
}
```

如果是首次调用，会执行 `inflateTable(threshold)` 方法来给 table 分配实际的空间：

```
private void inflateTable(int toSize) {
  //Find a power of 2 >= toSize
  int capacity = roundUpToPowerOf2(toSize);
  threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
  table = new Entry[capacity];
}
```

默认情况下， capacity 的值为 16 ， threshold 会变为 12 ， table 会分配一个长度为 16 的 Entry 数组。接下来，检查 key 是否为 null ，如果是，调用 putForNullKey 单独处理，我们暂时忽略这种情况。在 key 不为 null 的情况下，下一步调用 hash 方法计算 key 的 hash 值。

```
final int hash(Object k) {
  int h = 0;
  h ^= k.hashCode();
  h ^= (h >>> 20) ^ (h >>> 12);
  return h ^ (h >>> 7) ^ (h >>> 4);
}
```

其大概的算法是，先获取传入对象的本身的 hash 值，在此基础上再根据 hash 值做位运算，再次做位运算的目的是为了确保哈希值的随机性和均匀性。

重新计算了 hash 值之后，再调用 indexFor 方法，计算应该将这个键值对放到 table 的哪个位置，其具体的代码为：

```
static int indexFor(int h, int length) {
  return h & (length-1);
}
```

HashMap 中，length 为 2 的幂次方，h & (length - 1) 等同于取模运算 h % length 。

找到了保存位置 i ，table[i] 指向一个单向链表。接下来，就是在这个链表中逐个查找是否已经有这个键了，遍历代码为：

```
for (Entry<K,V> e = table[i]; e != null; e = e.next)
```

而比较的的逻辑是，先比较其 hash 值，再使用 equals 方法判断对象是否相等。

```
if(e.hash == hash && ((k = e.key) == key || key.equals(k)))
```

> 需要注意的是，这里有一个性能优化的小技巧，如下的代码中先比较了 hash 值， 再使用 equals 方法进行比较。原因是因为 hash 值为整型类型，比较的性能一般要比 equals 方法高很多。

如果比较之后发现 hash 值相等，直接修改 Entry 中的 value 为新的值，并返回 old value 的值。

如果比较之后 hash 值不相等，则先执行 modCount++ ，其逻辑与前面提到的一致，用于记录修改次数，方便在迭代当中检查是否有结构性的变化，然后再调用 addEntry 方法在给定的位置添加一条，代码为：

```
void addEntry(int hash, K key, V value, int bucketIndex) {
  if((size >= threshold) && (null != table[bucketIndex])) {
      resize(2 * table.length);
      hash = (null != key) ? hash(key) : 0;
      bucketIndex = indexFor(hash, table.length);
  }
  createEntry(hash, key, value, bucketIndex);
}
```

如果空间足够，则不需要 resize ，会直接调用 createEntry 方法进行添加，其核心逻辑为：新建一个 Entry 对象，插入单向链表的头部，并增加 size 。

```
void createEntry(int hash, K key, V value, int bucketIndex) {
  Entry<K,V> e = table[bucketIndex];
  table[bucketIndex] = new Entry<>(hash, key, value, e);
  size++;
}
```

如果空间不够，即 size 已经要超过阈值 threshold 了，并且对应的 table 位置已经插入过对象了，具体的检查代码为：

```
if((size >= threshold) && (null != table[bucketIndex]))
```

接下来会按照 2 倍的容量去调用 resize 方法对 table 进行扩容：

```
void resize(int newCapacity) {
  Entry[] oldTable = table;
  int oldCapacity = oldTable.length;
  Entry[] newTable = new Entry[newCapacity];
  transfer(newTable, initHashSeedAsNeeded(newCapacity));
  table = newTable;
  threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

其中的 transfer 方法会将原来的键值对 copy 过来，然后更新内部的 table 变量和 threshold 的值：

```
void transfer(Entry[] newTable, boolean rehash) {
  int newCapacity = newTable.length;
  for(Entry<K,V> e : table) {
      while(null != e) {
          Entry<K,V> next = e.next;
          if(rehash) {
              e.hash = null == e.key ? 0 : hash(e.key);
          }
          int i = indexFor(e.hash, newCapacity);
          e.next = newTable[i];
          newTable[i] = e;
          e = next;
      }
  }
}
```

#### put 执行过程的简单总结

上述的代码描述可能会比较复杂，如果进一步的进行简化之后，其步骤可以描述为：

* 计算键的哈希值。
* 根据哈希值得到保存位置（取模）。
* 插到对应位置的链表头部或更新已有值。
* 根据需要扩展 table 大小。

#### 图例

假设有如下的代码再创建一个 HashMap 之后，调用其 put 方法添加元素：

```
Map<String,Integer> countMap = new HashMap<>();
countMap.put("hello", 1);
countMap.put("world", 3);
countMap.put("position", 4);
```

首先，第一行代码创建了一个 HashMap 实例，其内存结构为

![hashmap-put-01](https://www.zhuxiaodong.net/static/images/hashmap-put-01.png)

接下来第二行的代码为添加 key 为 "hello" 的键值对，其中对字符串 "hello" 调用 hash 方法的值为：96207088 ， 然后再将该值取模 16 的结果为：0 .

我们可以来验证一下上述计算过程：

```
public class HashTester {
  public static void main(String [] args) {
      int initialCapacity = 16;
      int hashCode = hash("hello");
      int modValue = hashCode & (initialCapacity - 1); // 等价于 hashCode % 16
      System.out.println(hashCode);
      System.out.println(modValue);
  }

  final static int hash(Object k) {
      int h = 0;
      h ^= k.hashCode();
      h ^= (h >>> 20) ^ (h >>> 12);
      return h ^ (h >>> 7) ^ (h >>> 4);
  }
}

// Output
96207088
0
```

由于计算出的值为 0 ，因此，会将该键值对插入到 table[0]，并指向链表的第一个位置（头部），其内存结构为：

![hashmap-put-02](https://www.zhuxiaodong.net/static/images/hashmap-put-02.png)

第二行的代码为添加 key 为 "world" 的键值对，对字符串 "world" 调用 hash 方法的值为： 111207038 ， 然后再将该值取模 16 的结果为：14 ，其内存结构为：

![hashmap-put-03](https://www.zhuxiaodong.net/static/images/hashmap-put-03.png)

第三行的代码为添加 key 为 "position" 的键值对，对字符串 "position" 调用 hash 方法的值为： 771782464 ， 然后再将该值取模 16 的结果为：0 。由于 table[0] 已经存放了之前 "hello" 键值对的值，因此先将 table[0] 替换为 "position" 键值对，即链表头部的位置，再将 "position" 键值对的 next 节点指向 "hello" 键值对，其内存结构为：

![hashmap-put-04](https://www.zhuxiaodong.net/static/images/hashmap-put-04.png)

> 上述键值对计算 hash 并取模之后，如果其值重复，被称为 **哈希冲突（拉链法）** 。HashMap 采用了一种较为常见的解决冲突的方法：**链接地址法**，即上述第三张图例所示。此外，上面的代码主要基于 JDK 1.7 ，完全采用单链表来存储处理哈希冲，而在 JDK 1.8 中则采用了一种混合模式，对于链表长度大于8的，会转换为红黑树存储。关于 JDK 1.8 当中的实现，可以参考[这篇文章](https://tech.meituan.com/java-hashmap.html)

### 查找方法 get contains

get 方法能够根据 key 获取对应的值，代码如下：

```
public V get(Object key) {
    if(key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);
    return null == entry ? null : entry.getValue();
}
```

由于 HashMap 支持 key 为 null 值，当 key 为 null 时，会放在 table[0] ，调用 getForNullKey() 获取值；如果 key 不为 null ，则调用 getEntry() 获取键值对节点 entry ，然后调用节点的 getValue() 方法获取值。其中 getEntry() 方法的代码为：

```
final Entry<K,V> getEntry(Object key) {
    if(size == 0) {
        return null;
    }
    int hash = (key == null) ? 0 : hash(key);
    for(Entry<K,V> e = table[indexFor(hash, table.length)]; e != null; e = e.next) {
        Object k;
        if(e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

其代码逻辑简单总结为：

* 计算键的 hash 值：`int hash = (key == null) ? 0 : hash(key);` 。
* 根据 hash 找到 table 中对应链表： `table[indexFor(hash, table.length)];` 。
* 在链表中遍历查找： `for(Entry<K,V> e = table[indexFor(hash, table.length)]; e != null; e = e.next)` 。
* 先根据链表中的已存在元素的 hash 值进行比较，再调用 equals 方法进行比较： `if(e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))`

而 containsKey 方法的逻辑与 get 方法是类似的：

```
public boolean containsKey(Object key) {
    return getEntry(key) != null;
}
```

HashMap 可以方便高效地按照键进行操作，但如果要根据值进行操作，则需要遍历。以下是 containsValue 方法的代码，其主要逻辑是，如果要查找的值不为null，从table的第一个链表开始，从上到下，从左到右逐个节点进行访问，通过 equals 方法比较值，直到找到为止。

```
public boolean containsValue(Object value) {
    if(value == null)
        return containsNullValue();
    Entry[] tab = table;
    for(int i = 0; i < tab.length ; i++)
        for(Entry e = tab[i] ; e != null ; e = e.next)
            if(value.equals(e.value))
                return true;
    return false;
}
```

### 删除 remove

根据键删除键值对的代码为：

```
public V remove(Object key) {
    Entry<K,V> e = removeEntryForKey(key);
    return(e == null ? null : e.value);
}

final Entry<K,V> removeEntryForKey(Object key) {
    if(size == 0) {
        return null;
    }
    int hash = (key == null) ? 0 : hash(key);
    int i = indexFor(hash, table.length);
    Entry<K,V> prev = table[i];
    Entry<K,V> e = prev;
    while(e != null) {
        Entry<K,V> next = e.next;
        Object k;
        if(e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            modCount++;
            size--;
            if(prev == e)
                table[i] = next;
            else
                prev.next = next;
            e.recordRemoval(this);
            return e;
        }
        prev = e;
        e = next;
    }
    return e;
}
```

* 计算 hash ，根据 hash 找到对应 table 的位置 i 。
* 遍历 table[i] ，查找待删节点，使用变量 prev 指向前一个节点，next 指向后一个节点，e指向当前节点。
* 判断是否找到，依然是先比较 hash 值， hash 值相同时再用 equals 方法比较。
* 删除的逻辑就是让长度减小，然后让待删节点的前后节点链起来，如果待删节点是第一个节点，则让 table[i] 直接指向后一个节点。其具体的代码为：

```
size--;
if(prev == e)
    table[i] = next;
else
    prev.next = next;
```

* 最后调用的 e.recordRemoval(this) 方法；在 HashMap 中代码为空，主要是为了 HashMap 的子类扩展使用。

### 实现原理总结

* HashMap 内部维护了一个哈希表，即为实例变量数组 table ，每个元素 table[i] 指向一个单向链表，根据键存取值，用键算出 hash 值，然后取模运算之后得到数组中索引的位置 index , 最后将操作 table[index] 指向一个单向链表。

* 存取的时候依据键的 hash 值，只在对应的链表中操作，不会访问别的链表，在对应链表操作时也是先比较 hash 值，如果相同再用 equals 方法比较。

> 从 HashMap 进行比较的逻辑我们可以得出，**为什么我们再重写 Object 类的 getHashCode 或 equals 方法的任意一个时，要求必须两者都同时重写实现。** 可以参考[这篇文章](https://www.cnblogs.com/Qian123/p/5703507.html)

* JDK 1.8 中对 HashMap 做了大量的优化，其中比较关键的是在哈希冲突严重的情况下，即大量元素映射到同一个链表的情况下（具体是至少8个元素，且总的键值对个数至少是64），会将该链表转换为一个红黑树，以提高查询的效率。

![java8-hashmap](https://www.zhuxiaodong.net/static/images/java8-hashmap.png)


# HashMap 总结
---

## HashMap 的特点：

* 根据键保存和获取值的效率都很高，为 O(1) ，每个单向链表往往只有一个或少数几个节点，根据 hash 值就可以直接快速定位。
* HashMap 中的键值对没有顺序，因为 hash 值是随机的。
* 扩容是一个特别耗性能的操作，所以在使用 HashMap 的时候，初始化的时候给一个大致的数值，避免进行频繁的扩容。
* 负载因子是可以修改的，也可以大于1，但是建议不要轻易修改，除非情况非常特殊。
* HashMap 是线程不安全的，不要在并发的环境中同时操作 HashMap ，建议使用 ConcurrentHashMap 。
> 另外还有一个 Hashtable 类虽然通过 synchronized

* JDK 1.8 中引入红黑树大大优化了 HashMap 的性能。

## HashMap 的使用场景：

* HashMap适合需要经常根据键存取值，而且不需要保持其顺序性的场景。
* 如果需要保持集合的顺序性，可以使用 HashMap 的一个子类： LinkedHashMap 。此外， TreeMap 也可以进行排序。
