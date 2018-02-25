title: OutOfMemoryError异常实践
date: 2017-07-06 21:17:35
categories: programming
tags:
- java
- memory management
- OutOfMemoryError
---

在[上一篇](http://www.zhuxiaodong.net/2017/java-memory-management-part-1/)的文章当中，我们知道在JVM当中除了程序计数器之外，其它的区域都可能发生OOM异常，本章我们将通过实际的Demo来进行验证。

<!--more-->

# Java堆内存溢出
---

我们通过如下的代码来模拟无限循环的创建对象，最终导致OOM异常的结果：

```
import java.util.List;
import java.util.ArrayList;

public class HeapOOM {
    static class OOMObject {

    }

    public static void main(String[]args) {
    	List<OOMObject>list = new ArrayList<OOMObject>();
        while(true) {
            list.add(new OOMObject());
        }
    }
}
```

编译上述代码，并在执行时设置最大的堆内存为20MB，Crash之后自动记录堆快照的Dump文件。

```
javac HeapOOM.java
java -Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError HeapOOM
```

最终我们发现程序的输出为：

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid51553.hprof ...
Heap dump file created [27567077 bytes in 0.146 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
at java.util.Arrays.copyOf(Arrays.java:3210)
at java.util.Arrays.copyOf(Arrays.java:3181)
at java.util.ArrayList.grow(ArrayList.java:261)
at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)
at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)
at java.util.ArrayList.add(ArrayList.java:458)
at HeapOOM.main(HeapOOM.java:10)
```

可以看到上述的信息中已经输出OOM异常，并打印出了完整的stack trace。根据stack trace我们已经能够很轻易地找到出现问题的代码位置。

那么如果是在生产环境当中，由于无法获得显式的stack trace，我们又应该如何去分析OOM异常出现的原因了？答案是根据-XX:+HeapDumpOnOutOfMemoryError的JVM参数去记录下发生OOM的堆快照文件，然后通过[Eclipse Memory Analyzer](http://www.eclipse.org/mat/)工具来进行离线分析。

**Eclipse Memory Analyzer使用方式的简单教程**：

我们将演示1.7.0版本在mac os x下的使用方式。其中关于一部分Memory Analyzer的配置（MemoryAnalyzer.ini配置文件的路径在: **mat.app/Contents/Eclipse/MemoryAnalyzer.ini**），可以参考[官方文档](https://wiki.eclipse.org/MemoryAnalyzer/FAQ)。

step 1：下载EMA的mac os x版本，由于我的本机没有安装eclipse IDE，因此是使用stand-alone的方式。download url：[here](http://www.eclipse.org/mat/downloads.php)

PS: 以下是官方对于stand-alone的简单说明
> The stand-alone Memory Analyzer is based on Eclipse RCP. It is useful if you do not want to install a full-fledged IDE on the system you are running the heap analysis.

> To install the Memory Analyzer into an Eclipse IDE use the update site URL provided below. The Memory Analyzer (Chart) feature is optional. The chart feature requires the BIRT Chart Engine (Version 2.3.0 or greater).

> The minimum Java version required to run Memory Analyzer is 1.7

step 2：下载解压了之后，会得到mat.app的一个文件。

step 3：双击直接执行，不出意外的话你会得到如下的错误信息，log文件中具体的错误信息为：

```
java.lang.IllegalStateException: The platform metadata area could not be written: /private/var/folders/lf/6h5w3k697hn3t51wp405fm3c0000gp/T/AppTranslocation/05ED3CF2-90A8-4A74-82F6-7A07D2889A3E/d/mat.app/Contents/MacOS/workspace/.metadata.  By default the platform writes its content
under the current working directory when the platform is launched.  Use the -data parameter to
specify a different content area for the platform.
```

![mat-error](https://www.zhuxiaodong.net/static/images/mat-error.png)

step 4：上述错误信息指出，某个本地文件无法写入，按照提示使用-data参数指定working directory的方式并不成功；猜测是由于执行权限的问题，因此改换用sudo的方式执行，就能够成功地运行EMA。

```
sudo ./mat.app/Contents/MacOS/MemoryAnalyzer
```

step 5：使用EAM打开对应的dump文件（java_pid<pid>.hprof），由于我们是分析OOM异常，因此选择Leak Suspects Report。

![ema-wizard](https://www.zhuxiaodong.net/static/images/ema-wizard.png)

step 6：此时我们能够看到Leak Suspects视图的一些汇总信息，以及stack trace信息，此外还有大对象的引用关系（从列表当中，我们可以很快速地定位到HeapOOM类的OOMObject对象）。

![ema-report-summary](https://www.zhuxiaodong.net/static/images/ema-report-summary.png)

![ema-report-stack-trace](https://www.zhuxiaodong.net/static/images/ema-report-stack-trace.png)

![ema-report-list](https://www.zhuxiaodong.net/static/images/ema-report-list.png)

# 虚拟机栈和本地方法栈溢出
---

由于在HotSpot虚拟机中并不区分虚拟机栈和本地方法栈，因此，对于HotSpot来说，虽然-Xoss参数（设置本地方法栈大小）存在，但实际上是无效的，栈容量只由-Xss参数设定。

回顾我们在[之前的文章中](http://www.zhuxiaodong.net/2017/java-memory-management-part-1/)介绍的，此区域分为了两种类型的异常：
* 如果请求的栈深度大于虚拟机允许的深度，将会抛出StackOverflowError异常。
* 如果虚拟机栈动态可扩展，并且在扩展时无法申请到足够的内存，会抛出OutOfMemoryError异常。

虽然定义了两种不同类型的异常，但是仔细思考一下，两者其实有一定地重合，当栈空间无法继续分配时，到底是内存太小，还是已使用的栈空间太大？

### **单线程**场景下的实验

下面我们通过实际的代码来验证一下：

```
public class JavaVMStackSOF{
    private int stackLength = 1;

    public void stackLeak(){
        stackLength++;
        stackLeak();
    }

    public static void main(String[]args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch(Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}
```

编译，并设置-Xss160k JVM参数来执行：

```
javac JavaVMStackSOF
java -Xss160k JavaVMStackSOF
```

最终得到的输出结果为：

```
stack length:771
Exception in thread "main" java.lang.StackOverflowError
at JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:4)
```

因此说明，在单线程场景下，无论是由于栈帧太大还是虚拟机栈容量太小，当内存无法分配的时候，虚拟机抛出的都是StackOverflowError异常。

### **多线程**场景下的实验

我们先需要明确一些概念，OS分配给每一个进程的内存空间是有限的，例如在32bit的windows下，能够分配给进程的内存大概是2GB，2GB 减去 Xmx（最大堆内存），再减去MaxPermSize（最大方法区容量），剩余的内存就被虚拟机栈和本地方法栈所使用了。（程序计数器使用的内存几乎可以忽略，另外我们假设虚拟机本身消耗的内存可以忽略不计）

因此每个线程分配到的栈容量越大，可以建立的线程数量自然就越少，建立线程时就越容易把剩下的内存耗尽，这个时候出现OOM异常的几率会增大。

需要注意的是，我们在开发多线程应用的时候，如果是建立过多线程导致的内存溢出，在不能减少线程数或者更换64位虚拟机的情况下，可以通过减少Xmx（java堆）的设置，或者减少Xss（java虚拟机栈）的方式来换取更多的线程。

下面我们通过的实际的代码来验证一下:

```
public class JavaVMStackOOM {
    private void dontStop(){
        while(true) {
        }
    }

    public void stackLeakByThread() { 
        while(true){
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    dontStop();
                }
            });
            thread.start();
        }
    }

    public static void main(String[]args) throws Throwable {
        JavaVMStackOOM oom = new JavaVMStackOOM();
        oom.stackLeakByThread();
    }
} 
```

> 注意，执行的时候会导致本机的CPU使用率100%，整个系统处于卡死的状态，并且无法使用Ctrl+C停止进程，只能用kill -9才能强制结束进程。

```
javac JavaVMStackOOM
java -Xss2M JavaVMStackOOM
```

最终得到的输出为:

```
Exception in thread "main" java.lang.OutOfMemoryError: unable to create new native thread
        at java.lang.Thread.start0(Native Method)
        at java.lang.Thread.start(Thread.java:714)
        at JavaVMStackOOM.stackLeakByThread(JavaVMStackOOM.java:15)
        at JavaVMStackOOM.main(JavaVMStackOOM.java:20)
^CJava HotSpot(TM) 64-Bit Server VM warning: Exception java.lang.OutOfMemoryError occurred dispatching signal SIGINT to handler- the VM may need to be forcibly terminated
```

# 方法区和运行时常量池溢出
---

### 运行常量池溢出

我们使用String.intern()方法来进行测试，String.intern()是一个Native方法，它的作用是：如果字符串常量池中已经包含一个等于此String对象的字符串，则返回代表池中这个字符串的String对象；否则，将此String对象包含的字符串添加到常量池中，并且返回此String对象的引用。在JDK 1.6及之前的版本中，由于常量池分配在永久代内，我们可以通过-XX:PermSize和-XX:MaxPermSize限制方法区大小，从而间接限制其中常量池的容量。

> 注意：由于我们的jvm测试版本为jdk1.8，在jdk1.8当中已经移除了永久代，转而使用Metaspace来替代，详情请参考[这里](http://ifeve.com/java-permgen-removed/)和[这里](https://my.oschina.net/benhaile/blog/214159?p=2&temp=1499398217668)

```
public class RuntimeConstantPoolOOM {
  public static void main(String[]args) {
    //使用List保持着常量池引用, 避免Full GC回收常量池行为
    List<String> list = new ArrayList<String>();
    int i = 0;
    while(true) {
      list.add(String.valueOf(i++).intern())；
    }
  }
} 
```

```
javac RuntimeConstantPoolOOM
java -verbose -verbose:gc -XX:MaxMetaspaceSize=2m RuntimeConstantPoolOOM 
```

得到结果输出为:

```
[GC (Last ditch collection)  175K->175K(3134464K), 0.0003352 secs]
[Full GC (Last ditch collection)  175K->175K(3134464K), 0.0024552 secs]
[GC (Metadata GC Threshold)  175K->175K(3134464K), 0.0003683 secs]
[Full GC (Metadata GC Threshold)  175K->174K(3134464K), 0.0024257 secs]
[GC (Last ditch collection)  174K->174K(3143168K), 0.0003871 secs]
[Full GC (Last ditch collection)  174K->174K(3143168K), 0.0024017 secs]
Error occurred during initialization of VM
java.lang.OutOfMemoryError: Metaspace
```

结论是，如果将-XX:MaxMetaspaceSize设置得很小，jvm在启动的时候就已经报错了。如果将该值设置大一些，或者默认不添加该参数，该程序会无限执行下去，这与JDK1.7的效果是一致的。


接下来我们需要测试一下在JDK1.7和JDK1.6下的实验结果，由于多个jdk的切换，因此我们使用[jenv](http://www.jenv.be/)来切换不同的java版本。

* jenv的安装和使用可以参考官方文档，也可以参考这篇[blog](http://boxingp.github.io/blog/2015/01/25/manage-multiple-versions-of-java-on-os-x/)。NOTE：记得设置完你的bashrc or zshrc之后，执行一下**source ~/.bashrc** or **source ~/.zshrc**，其次记得重启shell：**exec $SHELL -l**。
* 由于Oracle已不再维护老版本的JDK，官方的download url笔者也是花了一些时间才找到。[Java 6在这里下载](http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html)；[Java 7在这里下载](http://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html)。
* 完成安装了之后，我们能够在对应的目录下找到安装路径：

```
ll /Library/Java/JavaVirtualMachines/

/Library/Java/JavaVirtualMachines/
├── 1.6.0.jdk
├── jdk1.7.0_80.jdk
└── jdk1.8.0_65.jdk

# 添加jdk的路径至jenv中.
jenv add /Library/Java/JavaVirtualMachines/1.6.0.jdk/Contents/Home/

jenv add /Library/Java/JavaVirtualMachines/jdk1.7.0_80.jdk/Contents/Home/

jenv add /Library/Java/JavaVirtualMachines/jdk1.8.0_65.jdk/Contents/Home/
```

首先来测试一下JDK1.6下的实验结果:

```
jenv local 1.6

java -XX:PermSize=10M -XX:MaxPermSize=10M RuntimeConstantPoolOOM

Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
	at java.lang.String.intern(Native Method)
	at RuntimeConstantPoolOOM.main(RuntimeConstantPoolOOM.java:10)
```
运行时常量池溢出，在OutOfMemoryError后面跟随的提示信息是"PermGen space"，说明运行时常量池属于方法区（HotSpot虚拟机中的永久代）的一部分。

接下来测试JDK1.7时，程序不会出现任何异常，与JDK1.8的效果一致。

### 方法区溢出

方法区用于存放Class的相关信息，如类名、访问修饰符、常量池、字段描述、方法描述等。对于这些区域的测试，基本的思路是运行时产生大量的类去填满方法区，直到溢出。我们在如下的实验中使用cglib来在运行时产生大量的动态类。

值得注意的是，运行时产生大量的类，常常会出现在我们使用的一些框架中，例如：Spring，Hibernate等；另外一些jvm上的动态语言（Groovy）也会持续的在运行时创建类来实现语言的动态性。大量JSP或动态产生JSP文件的应用（JSP第一次运行时需要编译为Java类）、基于OSGi的应用（即使是同一个类文件，被不同的加载器加载也会视为不同的类，也会导致方法区溢出。

由于使用到了cglib第三方库，我们使用maven来进行管理:

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>net.zhuxiaodong</groupId>
    <artifactId>methodareademo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <dependencies>
        <dependency>
            <groupId>cglib</groupId>
            <artifactId>cglib</artifactId>
            <version>3.2.5</version>
        </dependency>
    </dependencies>

</project>
```

```
package net.zhuxiaodong.methodareademo;


import net.sf.cglib.proxy.Enhancer;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

/**
 * Created by leonzhu on 07/07/2017.
 */
public class JavaMethodAreaOOM {
	public static void main(String[] args) {
		while(true) {
			Enhancer enhancer=new Enhancer();
			enhancer.setSuperclass(OOMObject.class);
			enhancer.setUseCache(false);
			enhancer.setCallback(new MethodInterceptor(){
				public Object intercept(Object obj, Method method, Object[]args, MethodProxy proxy)throws Throwable{
					return proxy.invokeSuper(obj, args);
				}
			});
			enhancer.create();
		}
	}

	static class OOMObject{
	}
}
```

我们设置在IDEA当中设置一下JVM的VM options：-XX:PermSize=10M -XX:MaxPermSize=10M，并且使用JDK1.6进行测试。
![IDEA-setting](https://www.zhuxiaodong.net/static/images/IDEA-setting.png)

得到的异常信息java.lang.OutOfMemoryError: PermGen space，说明永久代的内存已经溢出。

```
Exception in thread "main" net.sf.cglib.core.CodeGenerationException: java.lang.reflect.InvocationTargetException-->null
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:345)
	at net.sf.cglib.proxy.Enhancer.generate(Enhancer.java:492)
	at net.sf.cglib.core.AbstractClassGenerator$ClassLoaderData.get(AbstractClassGenerator.java:114)
	at net.sf.cglib.core.AbstractClassGenerator.create(AbstractClassGenerator.java:291)
	at net.sf.cglib.proxy.Enhancer.createHelper(Enhancer.java:480)
	at net.sf.cglib.proxy.Enhancer.create(Enhancer.java:305)
	at net.zhuxiaodong.methodareademo.JavaMethodAreaOOM.main(JavaMethodAreaOOM.java:24)
Caused by: java.lang.reflect.InvocationTargetException
	at sun.reflect.GeneratedMethodAccessor1.invoke(Unknown Source)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25)
	at java.lang.reflect.Method.invoke(Method.java:597)
	at net.sf.cglib.core.ReflectUtils.defineClass(ReflectUtils.java:459)
	at net.sf.cglib.core.AbstractClassGenerator.generate(AbstractClassGenerator.java:336)
	... 6 more
Caused by: java.lang.OutOfMemoryError: PermGen space
	at java.lang.ClassLoader.defineClass1(Native Method)
	at java.lang.ClassLoader.defineClassCond(ClassLoader.java:637)
	at java.lang.ClassLoader.defineClass(ClassLoader.java:621)
```

如果使用JDK1.7（u80版本），得到异常信息为：

```
Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"
```

JDK1.8版本由于已经移除了“永久代”，因此不做该版本的实验。

# 直接内存溢出
---

直接内存的容量可以通过-XX:MaxDirectMemorySize来指定，如果不指定，则大小与-Xmx的值一致。

如下的代码模拟了直接内存溢出的情况，我们并没有直接使用NIO的DirectByteBuffer类，而是通过反射获取Unsafe实例进行内存分配。原因是虽然使用DirectByteBuffer分配内存也会抛出内存溢出异常，但它抛出异常时并没有真正向操作系统申请分配内存，而是通过计算得知内存无法分配，于是手动抛出异常，真正申请分配内存的方法是unsafe.allocateMemory()。

```
import sun.misc.Unsafe;
import java.lang.reflect.Field;

public class DriectMemoryOOM {
    private static final int _1MB = 1024*1024;

    public static void main(String[]args) throws Exception {
        Field unsafeField = Unsafe.class.getDeclaredFields()[0];
        unsafeField.setAccessible(true);
        Unsafe unsafe=(Unsafe)unsafeField.get(null);

        while(true){
            unsafe.allocateMemory(_1MB);
        }
    }
} 
```

```
java -Xmx20M -XX:MaxDirectMemorySize=10M DriectMemoryOOM
```

由DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果我们发现OOM之后，Dump下的堆快照文件很小，而程序中又直接或间接使用了NIO，那就可以考虑检查一下是不是这方面的原因。
