title: 深入理解强引用/软引用/弱引用/虚引用
date: 2017-07-10 11:19:35
categories: programming
tags:
- java
- memory management
- GC
- Reference
- ReferenceQueue
---

# 强引用（Strong Reference）：
---

强引用不会被GC回收，并且在java.lang.ref里也没有实际的对应类型，平时工作接触的最多的就是强引用。Object obj = new Object();这里的obj引用便是一个强引用。如果一个对象具有强引用，那就类似于必不可少的生活用品，垃圾回收器绝不会回收它。当内存空间不足时，Java虚拟机宁愿抛出OutOfMemoryError错误，使程序异常终止，也不会靠随意回收具有强引用的对象来解决内存不足问题。

<!--more-->

让我们来看一段关于强引用的代码示例：

```
public class ClassStrong {

    public static class Referred {
        @Override
        protected void finalize() throws Throwable {
            System.out.println("Referred对象被垃圾收集");
        }
    }

    public static void collect() throws InterruptedException {
        System.out.println("开始垃圾收集...");
        System.gc();
        System.out.println("结束垃圾收集...");
        Thread.sleep(2000);
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("创建一个强引用--->");

        // 这是一个强引用
        // 如果没有引用这个对象就会被垃圾收集
        Referred strong = new Referred();

        // 开始垃圾收集
        ClassStrong.collect();

        System.out.println("删除引用--->");
        // 这个对象将要被垃圾收集
        strong = null;
        ClassStrong.collect();

        System.out.println("Done");
    }
}
```

得到的输出结果为：

```
java -XX:+PrintGC ClassStrong

创建一个强引用--->
开始垃圾收集...
[GC (System.gc())  3932K->488K(251392K), 0.0006913 secs]
[Full GC (System.gc())  488K->332K(251392K), 0.0034508 secs]
结束垃圾收集...
删除引用--->
开始垃圾收集...
[GC (System.gc())  1643K->428K(251392K), 0.0006725 secs]
[Full GC (System.gc())  428K->321K(251392K), 0.0060193 secs]
结束垃圾收集...
Referred对象被垃圾收集
Done
```

这个例子说明了强引用时候GC不会回收对象，只有这个对象没有强引用了，GC才会进行回收并调用finalize方法。

# java.lang.ref package
---

java.lang.ref包下，包含了如下的类型：

![java-lang-ref-package](https://www.zhuxiaodong.net/static/images/java-lang-ref-package.png)

可以看到SoftReference，WeakReference，PhantomReference，FinalReference都实现了抽象类Reference。JVM在执行GC时，会对这四种类型的采用不同的回收策略。

# Reference VS ReferenceQueue
---

Reference抽象类有2个构造函数，其中有一个构造函数可以传入ReferenceQueue参数；另外有两个比较重要的字段：referent（存储真正引用的对象）；queue（与之对应的ReferenceQueue）

Reference的完整代码可以参考[这里](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/lang/ref/Reference.java#Reference)。

ReferenceQueue的完整代码可以参考[这里](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/lang/ref/ReferenceQueue.java#ReferenceQueue)。

```
public abstract class Reference<T> {

	private T referent;         /* Treated specially by GC */

    volatile ReferenceQueue<? super T> queue;

    /* When active:   NULL
     *     pending:   this
     *    Enqueued:   next reference in queue (or this if last)
     *    Inactive:   this
     */
    @SuppressWarnings("rawtypes")
    Reference next;

    /* When active:   next element in a discovered reference list maintained by GC (or this if last)
     *     pending:   next element in the pending list (or null if last)
     *   otherwise:   NULL
     */
    transient private Reference<T> discovered;  /* used by VM */

    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    private static Reference<Object> pending = null;

	/* -- Constructors -- */

    Reference(T referent) {
        this(referent, null);
    }

    Reference(T referent, ReferenceQueue<? super T> queue) {
        this.referent = referent;
        this.queue = (queue == null) ? ReferenceQueue.NULL : queue;
    }
}
```

ReferenceQueue本身提供队列的功能，有入队（enqueue）和出队（poll，remove，其中remove阻塞等待提取队列元素）。ReferenceQueue对象本身保存了一个Reference类型的head节点，Reference封装了next字段，这样就是可以组成一个单向链表。同时ReferenceQueue提供了两个静态字段NULL，ENQUEUED。这两个字段的主要功能：NULL是当我们构造Reference实例时queue传入null时，会默认使用NULL，这样在enqueue时判断queue是否为NULL,如果为NULL直接返回，入队失败。ENQUEUED的作用是防止重复入队，reference后会把其queue字段赋值为ENQUEUED,当再次入队时会直接返回失败。

```
public class ReferenceQueue<T> {

    /**
     * Constructs a new reference-object queue.
     */
    public ReferenceQueue() { }

    private static class Null<S> extends ReferenceQueue<S> {
        boolean enqueue(Reference<? extends S> r) {
            return false;
        }
    }

    static ReferenceQueue<Object> NULL = new Null<>();
    static ReferenceQueue<Object> ENQUEUED = new Null<>();

    static private class Lock { };
    private Lock lock = new Lock();
    private volatile Reference<? extends T> head = null;
    private long queueLength = 0;

    boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
        synchronized (lock) {
            // Check that since getting the lock this reference hasn't already been
            // enqueued (and even then removed)
            ReferenceQueue<?> queue = r.queue;
            if ((queue == NULL) || (queue == ENQUEUED)) {
                return false;
            }
            assert queue == this;
            r.queue = ENQUEUED;
            r.next = (head == null) ? r : head;
            head = r;
            queueLength++;
            if (r instanceof FinalReference) {
                sun.misc.VM.addFinalRefCount(1);
            }
            lock.notifyAll();
            return true;
        }
    }
}
```

## Reference与ReferenceQueue之间是如何工作的？

Reference抽象类的初始化时，会start一个线程ReferenceHandler。当一个Reference的referent被回收时，垃圾回收器会把reference添加到pending这个链表里，然后Reference-handler thread不断的读取pending中的reference，把它加入到对应的ReferenceQueue中。

```
transient private Reference<T> discovered;  /* used by VM */

/* List of References waiting to be enqueued.  The collector adds
* References to this list, while the Reference-handler thread removes
* them.  This list is protected by the above lock object. The
* list uses the discovered field to link its elements.
*/
private static Reference<Object> pending = null;

private static class ReferenceHandler extends Thread {

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }

    public void run() {
        for (;;) {
            Reference<Object> r;
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OOME because it may try to allocate
                    // exception objects, so also catch OOME here to avoid silent exit of the
                    // reference handler thread.
                    //
                    // Explicitly define the order of the two exceptions we catch here
                    // when waiting for the lock.
                    //
                    // We do not want to try to potentially load the InterruptedException class
                    // (which would be done if this was its first use, and InterruptedException
                    // were checked first) in this situation.
                    //
                    // This may lead to the VM not ever trying to load the InterruptedException
                    // class again.
                    try {
                        try {
                            lock.wait();
                        } catch (OutOfMemoryError x) { }
                    } catch (InterruptedException x) { }
                    continue;
                }
            }

            // Fast path for cleaners
            if (r instanceof Cleaner) {
                ((Cleaner)r).clean();
                continue;
            }

            ReferenceQueue<Object> q = r.queue;
            if (q != ReferenceQueue.NULL) q.enqueue(r);
        }
    }
}

static {
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
         tgn != null;
         tg = tgn, tgn = tg.getParent());
    Thread handler = new ReferenceHandler(tg, "Reference Handler");
    /* If there were a special system-only priority greater than
     * MAX_PRIORITY, it would be used here
     */
    handler.setPriority(Thread.MAX_PRIORITY);
    handler.setDaemon(true);
    handler.start();
}
```

我们可以通过下面代码来结合SoftReference，WeakReference，PhantomReference与ReferenceQueue使用来验证这个机制。为了确保SoftReference在每次gc后，其引用的referent都被回收，我们需要加入-XX:SoftRefLRUPolicyMSPerMB=0参数，这个原理下文中会在讲。

```
import java.lang.ref.ReferenceQueue;
import java.lang.ref.SoftReference;
import java.lang.ref.WeakReference;
import java.lang.ref.PhantomReference;
import java.lang.ref.Reference;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class ReferenceTest {

    private static List<Reference> roots = new ArrayList<>();

    public static void main(String[] args) throws Exception {

        ReferenceQueue rq = new ReferenceQueue();

        new Thread(new Runnable() {
            @Override
            public void run() {
                int i=0;
                while (true) {
                    try {
                        Reference r = rq.remove();
                        System.out.println("reference:"+r);
                        //为null说明referent被回收
                        System.out.println( "get:"+r.get());
                        i++;
                        System.out.println( "queue remove num:"+i);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();


        for(int i = 0; i < 100000; i++) {
            byte[] a = new byte[1024*1024];
            // 分别验证SoftReference,WeakReference,PhantomReference
            Reference r = new SoftReference(a, rq);
            // Reference r = new WeakReference(a, rq);
            //Reference r = new PhantomReference(a, rq);
            roots.add(r);
            System.gc();

            System.out.println("produce"+i);
            TimeUnit.MILLISECONDS.sleep(100);
        }
    }
}
```

我们打开一个终端来执行上述的代码：

```
java -XX:SoftRefLRUPolicyMSPerMB=0 ReferenceTest
```

得到的输出为：

```
produce1
reference:java.lang.ref.SoftReference@51ba85f9
get:null
queue remove num:1
produce2
reference:java.lang.ref.SoftReference@5f7d71be
get:null
queue remove num:2
...后续的输出省略
```

可以看到当设置SoftRefLRUPolicyMSPerMB参数的值为0时，对应的SoftReference被立即回收了（get:null）

在程序运行的过程当中，我们打开另外一个终端运行jstack抓取堆栈的信息：

```
jps
jstack -p pid

"Reference Handler" #2 daemon prio=10 os_prio=31 tid=0x00007fe0fb003800 nid=0x3703 in Object.wait() [0x00007000037c5000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:157)
	- locked <0x00000006c000ccf0> (a java.lang.ref.Reference$Lock)

   Locked ownable synchronizers:
	- None
```

因此可以看出，当Reference与ReferenQueue联合使用的主要作用就是当Reference指向的referent在回收时(或者要被回收，如下文要讲的Finalizer)，提供一种通知机制，通过queue取到这些reference，来做额外的处理工作。当然，如果我们不需要这种通知机制，我们就不用传入额外的queue，默认使用NULL queue就会入队失败。

# Soft Reference
---

如果一个对象存在GC root对其有直接或间接的Soft Reference，并没有其他强引用路径，该对象就成为了softly-reachable对象。JVM保证在抛出OOM前会回收这些softly-reachable对象。JVM会根据当前内存的情况来决定是否回收softly-reachable对象，但只要referent有强引用存在，该referent就一定不会被清理，因此SoftReference适合用来实现内存敏感的缓存应用。

JVM不仅仅只会考虑当前内存情况，还会考虑软引用所指向的referent最近使用情况和创建时间来综合决定是否回收该referent。

Hotspot在gc时会根据两个标准来回收：
* 根据SoftReference引用实例的timestamp(每次调用softReference.get()会自动更新该字段，把最近一次垃圾回收时间赋值给timestamp)
* 当前JVM heap的内存剩余(free_heap)情况

计算的规则包括了3个变量：
* free_heap: is the amount of free heap space in MB.
* interval: is the time between the last GC's clock and the timestamp of the ref we are currently examining.
* ms_per_mb: is a constant number of milliseconds to keep around a SoftReference for each free megabyte in the heap.(可以通过-XX:SoftRefLRUPolicyMSPerMB来设定)

计算的规则为：
如果interval <= free_heap * ms_per_mb，则SoftReference不会被回收。否则就被回收。

例如：我们有一个SoftReference的timestamp为2000ms，最后一次GC‘s clock是5000ms，jvm设置的-XX:SoftRefLRUPolicyMSPerMB为1000，剩余的堆内存为：1MB。

```
5000 - 2000 > 1 * 1000
```

因此该SoftReference对象会被回收。参考JDK代码当中的[referencePolicy.cpp](http://hg.openjdk.java.net/jdk6/jdk6/hotspot/file/ad8c8ca4ab0f/src/share/vm/memory/referencePolicy.cpp)。

```
LRUCurrentHeapPolicy::LRUCurrentHeapPolicy() {
  setup();
}

// Capture state (of-the-VM) information needed to evaluate the policy
void LRUCurrentHeapPolicy::setup() {
  _max_interval = (Universe::get_heap_free_at_last_gc() / M) * SoftRefLRUPolicyMSPerMB;
  assert(_max_interval >= 0,"Sanity check");
}

// The oop passed in is the SoftReference object, and not
// the object the SoftReference points to.
bool LRUCurrentHeapPolicy::should_clear_reference(oop p) {
  jlong interval = java_lang_ref_SoftReference::clock() - java_lang_ref_SoftReference::timestamp(p);
  assert(interval >= 0, "Sanity check");

  // The interval will be zero if the ref was accessed since the last scavenge/gc.
  if(interval <= _max_interval) {
    return false;
  }

  return true;
}
```

我们来验证一下上述的算法，还是运行ReferenceTest的代码片段，只是这次我们将SoftRefLRUPolicyMSPerMB参数的值调整的为大一些。

```
java -XX:SoftRefLRUPolicyMSPerMB=4 ReferenceTest
```

得到的输出为：
```
produce0
produce1
...
produce147
reference:java.lang.ref.SoftReference@4f279c5
get:null
queue remove num:1
reference:java.lang.ref.SoftReference@a35972b
get:null
queue remove num:2
produce148
reference:java.lang.ref.SoftReference@e0a12b5
get:null
queue remove num:3
```

当生产者产生了147个对象时，此时达到了**interval > free_heap * ms_per_mb**的条件，对应的SoftReference开始被JVM回收掉了。

可见，SoftReference在一定程度上会影响JVM GC的，例如softly-reachable对应的referent多次垃圾回收仍然不满足释放条件，那么它会停留在heap old区，占据很大部分空间，在JVM没有抛出OutOfMemoryError前，它有可能会导致频繁的Full GC。

更多内容，可以参考[这里](http://jeremymanson.blogspot.com/2009/07/how-hotspot-decides-to-clear_07.html)

# Weak Reference
---

与Soft Reference的区别是，WeakReference对JVM GC几乎是没有影响的；当GC发生的时候，当一个对象被WeakReference引用时，处于weakly-reachable状态时，Weak Reference的对象就会立即被回收，并且会把Weak Reference注册到引用队列（ReferenceQueue）中（如果存在的话）。

## WeakHashMap

WeakHashMap在内部定义了一个核心的数据结构Entry<K,V>，它实际上就是继承了WeakReference。WeakHashMap里声明了一个queue，Entry继承WeakReference，构造函数中用key和queue关联构造一个WeakReference，当key不再被使用gc后会自动把把key注册到queue中。完整的代码请参考[这里](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/WeakHashMap.java#WeakHashMap)。

```
/**
 * Reference queue for cleared WeakEntries
 */
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

/**
 * The entries in this hash table extend WeakReference, using its main ref
 * field as the key.
 */
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
	V value;
	final int hash;
	Entry<K,V> next;

	/**
	 * Creates new entry.
	 */
	Entry(Object key, V value,
	      ReferenceQueue<Object> queue,
	      int hash, Entry<K,V> next) {
	    super(key, queue);
	    this.value = value;
	    this.hash  = hash;
	    this.next  = next;
	}
}

/**
 * Expunges stale entries from the table.
 */
private void expungeStaleEntries() {
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

下面我们来编写一个测试代码来分析一下：
我们写一个较大的循环来不断向一个WeakHashMap当中去添加byte数组，执行的过程中我们会发现，caches.size()增加到一定程度之后，又会从1开始增长。

```
import java.util.Map;
import java.util.WeakHashMap;

public class WeakHashMapTest {

    private static Map<String, byte[]> caches = new WeakHashMap<>();

    public static void main(String[] args) throws InterruptedException {
        for (int i = 0; i < 100000; i++) {
            caches.put(i + "", new byte[1024 * 1024 * 10]);
            System.out.println("put num: " + i + " but caches size:" + caches.size());
        }
    }
}
```

```
java -verbose:gc WeakHashMapTest

# 执行结果
put num: 0 but caches size:1
put num: 1 but caches size:2
put num: 2 but caches size:3
put num: 3 but caches size:4
put num: 4 but caches size:5
put num: 5 but caches size:6
[GC (Allocation Failure)  65372K->61912K(251392K), 0.0250445 secs]
put num: 6 but caches size:1
put num: 7 but caches size:2
put num: 8 but caches size:3
put num: 9 but caches size:4
put num: 10 but caches size:5
put num: 11 but caches size:6
[GC (Allocation Failure)  124623K->113104K(316928K), 0.0346340 secs]
```

# Phantom Reference
---

虚引用主要用来跟踪对象被垃圾回收器回收的活动。虚引用与软引用和弱引用的一个区别在于：虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器回收一个对象时，如果发现它还有虚引用，就会把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。由于Object.finalize()方法的不安全性、低效性，常常使用虚引用完成对象回收前的资源释放工作。

下面我们写段代码来进行一下测试：

```
public static void phantom() throws Exception {  
    Object obj = new Object();  
    ReferenceQueue<Object> refQueue = new ReferenceQueue<Object>();  
    PhantomReference<Object> phantom = new PhantomReference<Object>(obj,  
            refQueue);  
    System.out.println(phantom.get()); // null 
    System.out.println(refQueue.poll());// null  
  
    obj = null;  
    System.gc();  
  
    // 调用phanRef.get()不管在什么情况下会一直返回null  
    System.out.println(phantom.get());  
  
    // 当GC发现了虚引用，GC会将phanRef插入进我们之前创建时传入的refQueue队列  
    // 注意，此时phanRef所引用的obj对象，并没有被GC回收，在我们显式地调用refQueue.poll返回phanRef之后  
    // 当GC第二次发现虚引用，而此时JVM将phanRef插入到refQueue会插入失败，此时GC才会对obj进行回收  
    Thread.sleep(200);  
    System.out.println(refQueue.poll());  
} 
```

输出的结果为：

```
null
null
null
java.lang.ref.PhantomReference@1540e19d
```

# 更多的参考资料
---

* http://benjaminwhx.com/2016/09/10/%E8%AF%A6%E8%A7%A3java-lang-ref%E5%8C%85%E4%B8%AD%E7%9A%844%E7%A7%8D%E5%BC%95%E7%94%A8/
* https://my.oschina.net/robinyao/blog/829983
* https://segmentfault.com/a/1190000008853562
* http://blog.csdn.net/aitangyong/article/details/39453365









