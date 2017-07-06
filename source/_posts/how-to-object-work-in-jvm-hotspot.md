title: 对象在HotSpot虚拟机中是如何工作的
date: 2017-07-06 14:13:01
categories: programming
tags:
- java
- memory management
---

# 对象的创建：
---

### 内存分配：

当在创建一个java对象时(例如：new关键字、反序列化、clone等)，首先将去检查这个指令的参数是否能在常量池中定位到一个类的符号引用，并且检查这个符号引用代表的类是否已被加载、解析和初始化过。如果没有，那必须先执行相应的类加载过程。

<!--more-->

在类加载过程完成了之后，**该对象所占用的内存大小就可以完全确定下来**，jvm就会在内存上分配这个对象。HotSpot采用两种方式来为对象分配内存：

* **指针碰撞（Bump the Pointer）**：如果Java堆内存是绝对规整的，会使用一个指针将内存区域分为两大块，所有使用过的内存放在一边，未分配的内存放在另一边，在新创建一个对象时，只需要将该指针移动与该对象大小相等的位置。

* **空闲列表（Free List）**：如果Java堆内存不是规整的，已使用的内存和未分配的内存是交错的，虚拟机会维护一个列表，记录哪些内存块是可用的，在分配的时候会从列表当中找到一块足够大的空间划分给待分配的对象，并更新列表上的记录。

如何决定使用上述的哪种分配机制，取决于采用的垃圾回收器是否带有压缩整理功能。例如：Serial、ParNew等带有Compact过程的收集器会使用**指针碰撞**的方式，而CMS基于Mark-Sweep算法的会使用**空闲列表**的方式。

### 内存分配的线程安全：

由于对象的分配是一个非常频繁的操作，Java堆区域又是线程共享的区域，那么整个分配的过程是如何来确保线程安全的呢？

* 通过CAS（Compare And Swap）+ 失败重试的方式来保证更新操作的原子性。
* 使用TLAB（Thread Local Allocation Buffer）：即哪个线程要分配内存，就在对应的TLAB上分配。只有TLAB用完并分配新的TLAB时，才需要同步锁定。虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB参数来设定。

### 初始化为零值：

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头），如果使用TLAB，这一工作过程也可以提前至TLAB分配时进行。这一步操作保证了对象的实例字段在Java代码中可以不赋初始值就直接使用，程序能访问到这些字段的数据类型所对应的零值。

### 对象头（Object Header）的设置：

对象头的设置的信息包括：这个对象是哪一个类的实例，元数据的信息存储在哪里，对象的Hashcode，对象的GC分代信息，是否启用偏向锁等。

### HotSpot虚拟机内存分配的实现代码：

以下的C++代码截取了HotSpot的jdk7u，完整的代码可以参考[这里](https://github.com/openjdk-mirror/jdk7u-hotspot/blob/master/src/share/vm/interpreter/bytecodeInterpreter.cpp):


```
 CASE(_new): {
    u2 index = Bytes::get_Java_u2(pc+1);
    constantPoolOop constants = istate->method()->constants();
    if (!constants->tag_at(index).is_unresolved_klass()) {
      // Make sure klass is initialized and doesn't have a finalizer
      oop entry = constants->slot_at(index).get_oop();
      assert(entry->is_klass(), "Should be resolved klass");
      klassOop k_entry = (klassOop) entry;
      assert(k_entry->klass_part()->oop_is_instance(), "Should be instanceKlass");
      instanceKlass* ik = (instanceKlass*) k_entry->klass_part();
      if ( ik->is_initialized() && ik->can_be_fastpath_allocated() ) {
        size_t obj_size = ik->size_helper();
        oop result = NULL;
        // If the TLAB isn't pre-zeroed then we'll have to do it
        bool need_zero = !ZeroTLAB;
        if (UseTLAB) {
          result = (oop) THREAD->tlab().allocate(obj_size);
        }
        if (result == NULL) {
          need_zero = true;
          // Try allocate in shared eden
    retry:
          HeapWord* compare_to = *Universe::heap()->top_addr();
          HeapWord* new_top = compare_to + obj_size;
          if (new_top <= *Universe::heap()->end_addr()) {
            //以下的代码使用CPU提供的原子CAS操作指令: cmpxchg.
            if (Atomic::cmpxchg_ptr(new_top, Universe::heap()->top_addr(), compare_to) != compare_to) {
              goto retry;
            }
            result = (oop) compare_to;
          }
        }
        if (result != NULL) {
          // Initialize object (if nonzero size and need) and then the header
          if (need_zero ) {
            HeapWord* to_zero = (HeapWord*) result + sizeof(oopDesc) / oopSize;
            obj_size -= sizeof(oopDesc) / oopSize;
            if (obj_size > 0 ) {
              memset(to_zero, 0, obj_size * HeapWordSize);
            }
          }
          //是否使用偏向锁
          if (UseBiasedLocking) {
            result->set_mark(ik->prototype_header());
          } else {
            result->set_mark(markOopDesc::prototype());
          }
          result->set_klass_gap(0);
          result->set_klass(k_entry);
          SET_STACK_OBJECT(result, 0);
          UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
        }
      }
    }
	// Slow case allocation
	CALL_VM(InterpreterRuntime::_new(THREAD, METHOD->constants(), index),
	        handle_exception);
	SET_STACK_OBJECT(THREAD->vm_result(), 0);
	THREAD->set_vm_result(NULL);
	UPDATE_PC_AND_TOS_AND_CONTINUE(3, 1);
}
```

# 对象的内存布局：
---

HotSpot虚拟机中，对象在内存中的布局分为3部分：
* 对象头（Object Header）
* 实例数据（Instance Data）
* 对齐填充（Padding）

### 对象头：

对象头由两部分构成：

**Mark Word**：
* 用于存储对象的运行时数据，例如：HashCode、GC分代年龄、锁状态标识、线程持有锁、偏向线程ID、偏向时间戳等。
* 长度为32bit（32位操作系统中）/ 64bit(32位操作系统中)。
* Mark Word被设计为动态的存储结构以便提升存储效率。

**类型指针**：
对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。

### 实例数据：

存储了对象真正的有效信息，即程序代码中所定义的各种类型的字段内容。无论是从父类继承下来的，还是在子类中定义的，都需要记录起来。这部分的存储顺序会受到虚拟机分配策略参数（FieldsAllocationStyle）和字段在Java源码中定义顺序的影响。HotSpot虚拟机默认的分配策略为longs/doubles、ints、shorts/chars、bytes/booleans、oops（Ordinary Object Pointers），从分配策略中可以看出，相同宽度的字段总是被分配到一起。在满足这个前提条件的情况下，在父类中定义的变量会出现在子类之前。如果CompactFields参数值为true（默认为true），那么子类之中较窄的变量也可能会插入到父类变量的空隙之中。

### 对齐填充：

对齐填充仅仅起到占位符的作用。HotSpot要求对象的起始地址必须是8字节的整数倍，也就是说，对象的大小必须为8字节的整数倍。又由于对象头部分正好为8字节的整数倍，而实例数据部分的大小是动态的，因此对齐填充会将实例数据中，剩余未满足整数倍的部分补齐。


# 对象的访问定位：
---

一个Java对象完成内存分配了之后，在Java栈上会有一个引用(reference)数据来访问Java堆上的具体对象。

Java虚拟机规范只规定了引用类型是一个指向对象的引用，并没有定义这个引用应该通过何种方式去定位、访问堆中的对象的具体位置，所以对象访问方式也是取决于虚拟机实现而定的。

有两种方式来实现对象的访问：

**句柄访问**：
Java堆中将会划分出一块内存来作为句柄池，reference中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与类型数据各自的具体地址信息，参考下图：

![handler-access](http://static.zhuxiaodong.net/blog/static/images/handler-access.png)

* 优点：使用句柄来访问的最大好处就是reference中存储的是稳定的句柄地址，在对象被移动（垃圾收集时移动对象是非常普遍的行为）时只会改变句柄中的实例数据指针，而reference本身不需要修改。
* 缺点：由于访问对象时，会多一次句柄池的指针定位访问，相对于直接访问来说效率会比较低。

**直接指针**：
Java堆对象的布局中就必须考虑如何放置访问类型数据的相关信息，而reference中存储的直接就是对象地址，参考下图：

![directly-access](http://static.zhuxiaodong.net/blog/static/images/directly-access.png)

* 优点：相对于句柄访问来说，性能会更高。
* 缺点：对象被移动时，可能需要对相关联的内存部分进行移动。

HotSpot使用的是直接内存的方式来访问对象。


