title: 初探java内存管理机制-运行时数据区域
date: 2017-06-28 11:40:38
categories: programming
tags:
- java
- memory management
- GC
---

# 运行时数据区域：
---

JVM Runtime会将所管理的内存分成若干的数据区域，参考下图：

![jvm-runtime-data-area](https://www.zhuxiaodong.net/static/images/jvm-runtime-data-area.png)

<!--more-->

下面我们对以上的数据区域一一进行解释：

### 程序计数器（Program Counter Register）：

* 一块比较小的内存区域，可以看作是**当前线程**所执行字节码的行号指示器。
* 字节码解释器通过改变程序计数器的值，来选取下一条需要执行的字节码指令。此外，分支、循环、跳转、异常处理、线程恢复等功能需要依赖于程序计数器来完成。
* 程序计数器是**线程私有**的，这意味着每一个线程的程序计数器彼此独立，互不影响。
* 程序计数器只会记录jvm字节码的指令地址，对于native代码（C，C++编写的本地代码），计数器记录的是空（Undefined）。
* 程序计数器是所有运行时数据区域中，唯一不会出现OutOfMemoryError异常的区域。

### Java虚拟机栈（Java Virtual Machine Stacks）：

* **线程私有**，且与线程的生命周期一致。
* 描述了Java方法执行的内存模型：每个方法执行时会创建一个Stack Frame，用于存储局部变量表、操作数栈、动态链接、方法入口等。每个方法在从执行开始到结束的过程中，会对应Stack Frame在虚拟机栈中从**入栈**到**出栈**的过程。Stack -> Stack Frame (Local Variable Table/Operand Stack/Dynamic Linking/Return Address)的结构请参考下图:
![java-stack](https://www.zhuxiaodong.net/static/images/java-stack.png)
> 更多关于局部变量表、操作数栈、动态链接、方法入口的知识，会在后续的章节当中介绍。
* 定义了2种内存异常：
	* 如果请求的栈深度大于虚拟机允许的深度，将会抛出StackOverflowError异常。
	* 如果虚拟机栈动态可扩展，并且在扩展时无法申请到足够的内存，会抛出OutOfMemoryError异常。

### 本地方法栈（Native Method Stack）：

* 本地方法栈与Java虚拟机栈有着类似的功能，只是本地方法栈为Native方法服务。
* 与Java虚拟机栈一样，也会抛出StackOverflowError异常和OutOfMemoryError异常。

### Java堆（Java Heap）：

* 对于大多数应用来说，Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块。
* 被所有线程共享，在虚拟机启动时创建。
* 所有的对象实例和数组都必须在Java堆上分配。（随着JIT编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换优化技术将会导致一些微妙的变化发生，所有的对象都分配在堆上也渐渐变得不是那么“绝对”了。）
* Java Heap由: 新生代(New Generation)、老年代(Old Generation)组成，其中New Generation又分为了一个Eden和一个From Survivor和To Survivor区域。
* TLAB（Thread Local Allocation Buffer）线程私有的缓冲区。
* 如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

### 方法区（Method Area）：

* 线程共享的内存区域。
* 存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码。
* 方法区在HotSpot虚拟机实现当中又被称为**永久代（Permanment Generation）**，原因是由于HotSpot的设计团队选择把GC分代收集扩展至方法区，这样可以重用Java堆内存管理的代码。（事实证明之前永久代的设计并不是一个好的方案，原因是使用Java堆的内存回收算法更容易受到内存溢出问题[-XX:MaxPermSize上限]；目前Hotspot已经规划将在今后的版本当中移除永久代，使用Native内存实现方法区，参考[这里](http://openjdk.java.net/jeps/122)；JDK 1.7的HotSpot中，已经把原本放在永久代的字符串常量池移出。）
* 方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

### 运行常量池（Runtime Constant Pool）：

* 运行时常量池是方法区的一部分。Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池（Constant Pool Table），用于存放编译期生成的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。更多信息，参考[这里](https://www.zhihu.com/question/30300585)。
* 除了保存Class文件中描述的符号引用外，还会把翻译出来的直接引用也存储在运行时常量池中。
* 运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有编译期才能产生，也就是并非预置入Class文件中常量池的内容才能进入方法区运行时常量池，运行期间也可能将新的常量放入池中，这种特性被开发人员利用得比较多的便是String类的intern()方法。
* 常量池无法再申请到内存时会抛出OutOfMemoryError异常。

### 直接内存（Direct Memory）：

* 直接内存并不是虚拟机运行时数据区的一部分，也不是Java虚拟机规范中定义的内存区域。
* JDK 1.4中新加入了NIO（New Input/Output）类，引入了一种基于通道（Channel）与缓冲区（Buffer）的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在Java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java堆和Native堆中来回复制数据。
* 直接内存可能导致OutOfMemoryError异常出现。