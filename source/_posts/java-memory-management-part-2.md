title: 初探java内存管理机制-垃圾回收算法基础
date: 2017-07-10 10:02:51
categories: programming
tags:
- java
- memory management
- GC
---

# 对象是否存活的算法
---

## 引用计数法（Reference Counting）：

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是可以被回收的。

<!--more-->

### 引用计数法的缺陷：

Java虚拟机并没有采用引用计数法作为垃圾回收的主要算法，原因是由于引用计数法很难够去解决循环引用（reference cycle）的问题。

来看看下面的代码：

```
public class ReferenceCountingGC {
    private Object instance = null;
    private static final int _1MB = 1024 * 1024;
    private byte[] bigSize = new byte[2 * _1MB];

    public static void main(String[] args) {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;
        objA=null;
        objB=null;
        //假设在这行发生GC,objA和objB是否能被回收?
        System.gc();
    }
}
```

objA引用了objB，而objB又引用了objA，是否内存就不会被回收了？我们需要实际的验证一下：

```
# 使用-XX:+PrintGCDetails参数打印出详细的GC信息
java -XX:+PrintGCDetails ReferenceCountingGC
```

得到的结果是，内存从5458K降低了至331K。这说明JVM并没有使用引用计数算法。

```
# JDK1.6
[Full GC (System) [CMS: 0K->331K(63872K), 0.0097213 secs] 5458K->331K(83008K), [CMS Perm : 4664K->4663K(21248K)], 0.0104391 secs] [Times: user=0.01 sys=0.00, real=0.01 secs]
Heap
 par new generation   total 19136K, used 1021K [7f3000000, 7f44c0000, 7f44c0000)
  eden space 17024K,   6% used [7f3000000, 7f30ff6a0, 7f40a0000)
  from space 2112K,   0% used [7f40a0000, 7f40a0000, 7f42b0000)
  to   space 2112K,   0% used [7f42b0000, 7f42b0000, 7f44c0000)
 concurrent mark-sweep generation total 63872K, used 331K [7f44c0000, 7f8320000, 7fae00000)
 concurrent-mark-sweep perm gen total 21248K, used 4726K [7fae00000, 7fc2c0000, 800000000)
```

## 可达性分析算法（Reachability Analysis）：

通过一系列的称为"GC Roots"的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链（Reference Chain），当一个对象到GC Roots没有任何引用链相连（用图论的话来说，就是从GC Roots到这个对象不可达）时，则证明此对象是不可用的。

参考下图，对象object5，object6，object7没有GC Root的引用，因此被称为不可达对象，它们将被GC回收：

![reachability-analysis](https://www.zhuxiaodong.net/static/images/reachability-analysis.png)

再来看一张图（关于StrongReference，SoftReference，WeakReference，PhantomReference的内容，后续会进行讲解）。从GC Root到达一个对象往往会存在多个引用路径，这个时候垃圾回收器会根据两个原则来判断对象是否可达：
* 引用的强弱关系为：Strong > Soft > Weak > Phantom
* 单一路径中，以最弱的引用为准
* 多路径中，以最强的引用为准

![mutil-reference](https://www.zhuxiaodong.net/static/images/mutil-reference.png)

上述图中，obj4存在3个引用路径，分别是1 -> 6，2 -> 5，3 -> 4，根据多路径中以最强的引用为准的原则，2 -> 5都是强引用。如果仅仅存在一个路径对Obj4有引用时，比如现在只剩1->6，那么根对象到Obj4的引用就是以最弱的为准，就是SoftReference引用，Obj4就是softly-reachable对象。

### 什么是GC Root？

参考知乎上R大的[解释](https://www.zhihu.com/question/53613423/answer/135743258)

> 所谓“GC roots”，或者说tracing GC的“根集合”，就是一组必须活跃的引用。这些引用可能包括：

> * 所有Java线程当前活跃的栈帧里指向GC堆里的对象的引用；换句话说，当前所有正在被调用的方法的引用类型的参数/局部变量/临时值。
> * VM的一些静态数据结构里指向GC堆里的对象的引用，例如说HotSpot VM里的Universe里有很多这样的引用。
> * JNI handles，包括global handles和local handles（看情况）
> * 所有当前被加载的Java类（看情况）
> * Java类的引用类型静态变量（看情况）
> * Java类的运行时常量池里的引用类型常量（String或Class类型）（看情况）
> * String常量池（StringTable）里的引用

注意，是一组必须活跃的引用，不是对象。

# 关于引用（Reference）：
---

无论是通过引用计数算法判断对象的引用数量，还是通过可达性分析算法判断对象的引用链是否可达，判定对象是否存活都与“引用”有关。在JDK 1.2以前，Java中的引用的定义很传统：如果reference类型的数据中存储的数值代表的是另外一块内存的起始地址，就称这块内存代表着一个引用。这种定义很纯粹，但是太过狭隘，一个对象在这种定义下只有被引用或者没有被引用两种状态，对于如何描述一些“食之无味，弃之可惜”的对象就显得无能为力。我们希望能描述这样一类对象：当内存空间还足够时，则能保留在内存之中；如果内存空间在进行垃圾收集后还是非常紧张，则可以抛弃这些对象。很多系统的缓存功能都符合这样的应用场景。

因此我们需要将引用进行分类，具体的内容请参考我的另外一篇[文章](http://www.zhuxiaodong.net/2017/strong-soft-weak-phantom-reference-in-java)。

# 关于Finalize：
---

一个对象到达不可达状态了之后，并不会立即在内存上清除掉，需要至少两次标记的过程：如果对象在进行可达性分析后发现没有与GC Roots相连接的引用链，那它将会被第一次标记并且进行一次筛选，筛选的条件是此对象是否有必要执行finalize()方法。当对象没有override finalize()方法，或者finalize()方法已经被虚拟机调用过，虚拟机将这两种情况都视为“没有必要执行”。

如果一个对象被判定为“有必须执行” finalize方法，就会将这个对象放置到一个叫F-Queue的队列中，稍后会由一个现成优先级较低的Finalizer线程去执行。虚拟机会触发这个方法，但是并不保证等待其执行的结果。这样处理的原因是，如果开发人员在重写的finalize方法中编写了一些耗时的代码，会导致F-Queue当中的其它对象永远处于等待状态，甚至导致整个内存回收系统崩溃。

重写了finalize方法的对象，可以通过重新与引用链上的任何一个对象建立关联（例如将该对象赋值给某个类对象或者成员变量），来让GC重新将该对象“复活”。否则，该对象将会被立即回收。

下面我们使用代码来验证一下：

```
public class FinalizeEscapeGC {
    public static FinalizeEscapeGC SAVE_HOOK = null;

    public void isAlive() {
        System.out.println("yes,i am still alive. :)");
    }

    @Override
    protected void finalize() throws Throwable {
        super.finalize();
        System.out.println("finalize mehtod executed!");
        FinalizeEscapeGC.SAVE_HOOK = this;    
    }

    public static void main(String[] args) throws Throwable {
        SAVE_HOOK = new FinalizeEscapeGC();
        /*对象第一次成功拯救自己*/
        SAVE_HOOK = null;
        System.gc();
        // 因为finalize方法优先级很低, 所以暂停0.5秒以等待它
        Thread.sleep(500);
        if(SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no,i am dead. :(");
        }
        //下面这段代码与上面的完全相同, 但是这次自救却失败了
        SAVE_HOOK=null;
        System.gc();
        //因为finalize方法优先级很低, 所以暂停0.5秒以等待它
        Thread.sleep(500);
        if(SAVE_HOOK != null) {
            SAVE_HOOK.isAlive();
        } else {
            System.out.println("no,i am dead. :(");
        }
    }
}
```

执行的结果为：
```
finalize mehtod executed!
yes,i am still alive. :)
no,i am dead. :(
```

从上面的代码我们可以分析出，finalize方法被执行一次了之后，由于我们执行了**FinalizeEscapeGC.SAVE_HOOK = this; **，对象被成功“复活”了。但是由于finalize方法只会被执行一次，因此第二次gc的时候，就被完全回收掉了。

由此我们可以分析出，finalize方法的执行时间点并可控，同时可能造成对象被推迟回收的问题，我们在日常的开发过程当中，应该避免使用finalize方法来做资源的回收。可以使用try {} finally {}或者PhantomReference来达到类似的功能。

# 方法区的垃圾回收
---

永久代的垃圾收集主要回收两部分内容：废弃常量和无用的类。

## 回收常量池：

以常量池中字面量的回收为例，假如一个字符串"abc"已经进入了常量池中，但是当前系统没有任何一个String对象是叫做"abc"的，换句话说，就是没有任何String对象引用常量池中的"abc"常量，也没有其他地方引用了这个字面量，如果这时发生内存回收，而且必要的话，这个"abc"常量就会被系统清理出常量池。常量池中的其他类（接口）、方法、字段的符号引用也与此类似。

## 回收无用的类：

下面3种情况会导致无用的类“可以进行”回收，仅仅是“可以进行”，但并不是真正的被回收：

* 该类所有的实例都已经被回收，也就是Java堆中不存在该类的任何实例。
* 加载该类的ClassLoader已经被回收。
* 该类对应的java.lang.Class对象没有在任何地方被引用，无法在任何地方通过反射访问该类的方法。

是否对真正对类进行回收，HotSpot虚拟机提供了-Xnoclassgc参数进行控制。同时，还可以使用-verbose:class，以及-XX:+TraceClassLoading、-XX:+TraceClassUnLoading查看类加载和卸载信息，其中-verbose:class和-XX:+TraceClassLoading可以在Product版的虚拟机中使用，-XX:+TraceClassUnLoading参数需要FastDebug版的虚拟机支持。

在大量使用反射、动态代理、CGLib等ByteCode框架、动态生成JSP以及OSGi这类频繁自定义ClassLoader的场景都需要虚拟机具备类卸载的功能，以保证永久代不会溢出。

# 垃圾回收算法
---

## 标记-清除算法（Mark-Sweep）
---
最基础的算法是标记-清除算法，该算法分为了“标记”和“清除”两个阶段，首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。

下面参考[《垃圾回收的算法与实现》](http://www.ituring.com.cn/book/1460)，来实现标记-清除算法的伪代码：

### 标记阶段

```
mark_phase(){
    for(r : $roots){
        mark(*r)
    }
}

mark(obj){
    if(obj.mark == FALSE){
        obj.mark = TRUE
        for(child : children(obj)){
            mark(*child)
        }
    }
}
```
标记还是比较简单的。也就是遍历每一个根节点，由于每个根节点是一个树，这个时候可以采用广度优先算法或者深度优先算法来遍历这棵树，标记出所有的节点。

![mark-sweep](https://www.zhuxiaodong.net/static/images/mark-sweep.png)

### 清除阶段

```
sweep_phase(){
    sweeping = $heap_start
    while(sweeping < $heap_end){
        if(sweeping.mark == TRUE){
            sweeping.mark = FALSE
        } else {
            sweeping.next = $free_list
            $free_list = sweeping
        }
        sweeping += sweeping.size
    }
}
```
这里主要是为了把清除的内存标记到空闲链表中，用于下一次分配内存。

### 缺点

* 内存碎片问题：
该算法会造成大量不连续的内存碎片，空间碎片太多可能会导致以后在程序运行过程中需要分配较大对象时，无法找到足够的连续内存而不得不提前触发另一次垃圾收集动作。

* 消耗的时间：
标记的时候需要遍历 GC roots，标记出活动对象。清除的时候需要遍历整个堆。

## 复制收集算法（Copy and Collection）
---
复制-收集算法将可用内存按容量划分为大小相等的两块，每次只使用其中的一块。当这一块的内存用完了，就将还存活着的对象复制到另外一块上面，然后再把已使用过的内存空间一次清理掉。

### 优点
每次都是对整个半区进行内存回收，内存分配时也就不用考虑内存碎片等复杂情况，只要移动堆顶指针，按顺序分配内存即可，实现简单，运行高效。

### 缺点
内存利用的效率不高。由于将内存区域分为了分为了2块，实际可用的内存区域缩小为了原来的一半。

### 实现（新生代采用复制收集算法）
目前几乎所有的虚拟机实现，在新生代的回收都是采用的复制收集算法。原因是IBM公司的专门研究表明，新生代中的对象98%是“朝生夕死”的，所以并不需要按照1:1的比例来划分内存空间，而是将内存分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中一块Survivor。

当回收的时候，将Eden和其中一块Survivor区域中还存活的对象一次性的复制到另一块Survivor空间上，最后在清理掉Eden和之前用过的Survivor。

HotSpot虚拟机默认Eden和Survivor的大小比例是8:1，也就是每次新生代中可用内存空间为整个新生代容量的90%（80%+10%），只有10%的内存会被“浪费”。

### 内存分配担保（Handle Promotion）
98%的对象可回收只是一般场景下的数据，我们没有办法保证每次回收都只有不多于10%的对象存活，当Survivor空间不够用时，需要依赖其他内存（这里指老年代）进行分配担保（Handle Promotion）。

内存的分配担保也一样，如果另外一块Survivor空间没有足够空间存放上一次新生代收集下来的存活对象时，这些对象将直接通过分配担保机制进入老年代。

## 标记-整理算法（Mark-Compact）
---
### 老年代不采用复制收集（Copy and Collection）算法的原因
复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。更关键的是，如果不想浪费50%的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。

### 定义
标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。

## 分代收集算法（Generational Collection）
---
根据对象存活周期的不同将内存划分为几块。一般是把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批对象死去，只有少量存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率高、没有额外空间对它进行分配担保，就必须使用“标记—清理”或者“标记—整理”算法来进行回收。

# Hotspot的算法实现
---
HotSpot虚拟机上实现对象存活判定和垃圾收集算法时，必须对算法的执行效率有严格的考量，才能保证虚拟机高效运行。

## 枚举根路径
---
从可达性分析中的GC Roots节点找引用链这个操作为例，可作为GC Roots的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（例如栈帧中的本地变量表）中，现在很多应用仅仅方法区就有数百兆，如果要逐个检查这里面的引用，那么必然会消耗很多时间。

另外，可达性分析对执行时间的敏感还体现在GC停顿上，因为这项分析工作必须在一个能确保一致性的快照中进行——这里“一致性”的意思是指在整个分析期间整个执行系统看起来就像被冻结在某个时间点上，不可以出现分析过程中对象引用关系还在不断变化的情况，该点不满足的话分析结果准确性就无法得到保证。这点是导致GC进行时必须停顿所有Java执行线程（Stop The World）的其中一个重要原因，即使是在号称（几乎）不会发生停顿的CMS收集器中，枚举根节点时也是必须要停顿的。

由于目前的主流Java虚拟机使用的都是准确式GC，所以当执行系统停顿下来后，并不需要一个不漏地检查完所有执行上下文和全局的引用位置，虚拟机应当是有办法直接得知哪些地方存放着对象引用。在HotSpot的实现中，是使用一组称为OopMap的数据结构来达到这个目的的，在类加载完成的时候，HotSpot就把对象内什么偏移量上是什么类型的数据计算出来，在JIT编译过程中，也会特定的位置记录下栈和寄存器中哪些位置是引用。这样，GC在扫描时就可以直接得知这些信息了。

下面的代码是HotSpot Client VM生成的一段String.hashCode()方法的本地代码，可以看到在0x026eb7a9处的call指令有OopMap记录，它指明了EBX寄存器和栈中偏移量为16的内存区域中各有一个普通对象指针（Ordinary Object Pointer）的引用，有效范围为从call指令开始直到0x026eb730（指令流的起始位置）+142（OopMap记录的偏移量）=0x026eb7be，即hlt指令为止。

```
[Verified Entry Point]
0x026eb730:mov%eax, -0x8000(%esp)
……
；ImplicitNullCheckStub slow case
0x026eb7a9:call 0x026e83e0
；OopMap{ebx=Oop[16]=Oop off=142}
；*caload
；-java.lang.String:hashCode@48(line 1489)
；{runtime_call}
0x026eb7ae:push$0x83c5c18
；{external_word}
0x026eb7b3:call 0x026eb7b8
0x026eb7b8:pusha
0x026eb7b9:call 0x0822bec0；{runtime_call}
0x026eb7be:hlt 
```

## 安全点
---
在OopMap的协助下，HotSpot可以快速且准确地完成GC Roots枚举，但一个很现实的问题随之而来：可能导致引用关系变化，或者说OopMap内容变化的指令非常多，如果为每一条指令都生成对应的OopMap，那将会需要大量的额外空间，这样GC的空间成本将会变得很高。

实际上，HotSpot也的确没有为每条指令都生成OopMap，前面已经提到，只是在“特定的位置”记录了这些信息，这些位置称为安全点（Safepoint），即程序执行时并非在所有地方都能停顿下来开始GC，只有在到达安全点时才能暂停。Safepoint的选定既不能太少以致于让GC等待时间太长，也不能过于频繁以致于过分增大运行时的负荷。所以，安全点的选定基本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的——因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这个原因而过长时间运行，“长时间执行”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生Safepoint。

**如何确保在GC发生时让所有线程都“跑”到最近的安全点上再停顿下来？**

* 抢先式中断（Preemptive Suspension）：
抢先式中断不需要线程的执行代码主动去配合，在GC发生时，首先把所有线程全部中断，如果发现有线程中断的地方不在安全点上，就恢复线程，让它“跑”到安全点上。由于此种方式的成本过高，Hotspot并没有采用。

* 主动式中断（Voluntary Suspension）：
当GC需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起。轮询标志的地方和安全点是重合的，另外再加上创建对象需要分配内存的地方。

## 安全区域
---

使用Safepoint似乎已经完美地解决了如何进入GC的问题，但实际情况却并不一定。Safepoint机制保证了程序执行时，在不太长的时间内就会遇到可进入GC的Safepoint。但是，程序“不执行”的时候呢？所谓的程序不执行就是没有分配CPU时间，典型的例子就是线程处于Sleep状态或者Blocked状态，这时候线程无法响应JVM的中断请求，“走”到安全的地方去中断挂起，JVM也显然不太可能等待线程重新被分配CPU时间。

对于这种情况，就需要安全区域（Safe Region）来解决。

安全区域是指在一段代码片段之中，引用关系不会发生变化。在这个区域中的任意地方开始GC都是安全的。我们也可以把Safe Region看做是被扩展了的Safepoint。

在线程执行到Safe Region中的代码时，首先标识自己已经进入了Safe Region，那样，当在这段时间里JVM要发起GC时，就不用管标识自己为Safe Region状态的线程了。在线程要离开Safe Region时，它要检查系统是否已经完成了根节点枚举（或者是整个GC过程），如果完成了，那线程就继续执行，否则它就必须等待直到收到可以安全离开Safe Region的信号为止。




