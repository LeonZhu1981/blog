title: 再谈java内存管理机制-GC基础算法
date: 2017-07-17 14:03:59
categories: programming
tags:
- java
- memory management
- GC
---

# 引用计数（Reference Counting）
---

虽然JVM当中并没有使用引用计数去作为垃圾回收的算法，但本着“存在即合理”的真理，我们有必要再去深究一下该算法的主要优势和劣势。

<!--more-->

## 有哪些语言采用了引用计数的方式？
就我们熟知的语言来说，包括：python，ruby，php。

## 优势：
1. 算法实现简单。当在内存当中分配一个对象之后，如果该对象被另外一个对象引用，该对象的引用计数会加一，如果该对象被删除一个引用，那么引用计数会减一。当一个对象的引用计数为0时，那么垃圾回收器就可以删除掉该对象。
2. 高效，运行期没有停顿。

我们引入一段pyhon代码来直观的感受一下：

```
from sys import getrefcount
a = [1, 2, 3]
getrefcount(a) # output: 2

b = a # add 1 reference to a
getrefcount(a) # output: 3

del b
getrefcount(a) # output: 2
```

## 劣势：
循环引用的问题，简而言之就是a引用b，b又引用了a，造成了“相互引用”的问题。

## 各个语言中是如何解决循环引用问题的？

* python：采用mark-sweep来解决循环引用问题。具体参考：[1](http://hbprotoss.github.io/posts/pythonla-ji-hui-shou-ji-zhi.html) 和 [2](http://www.wklken.me/posts/2015/09/29/python-source-gc.html)。

* php：采用“引用计数系统中的同步周期回收”机制。具体参考：[1](http://docs.php.net/manual/zh/features.gc.collecting-cycles.php) 和 [2](http://www.php-internals.com/book/?p=chapt06/06-04-01-new-garbage-collection)

