title: CSAPP读书笔记-第一章
date: 2017-01-09 21:18:55
categories: programming
tags: 
- csapp
toc: true
---
# 计算机的信息的基本表示单位：bit（位）
系统当中所有的信息都是--包括磁盘文件，内存中的程序和数据，网络当中传输的数据，都是由一串 bit 表示。 在不同的上下文当中， 一个同样的字节序列可能表示一个整数、浮点数、字符串、或者是机器指令。

# 编译链接过程

![compile-link](http://static.zhuxiaodong.net/blog/static/images/compile-link.png)

* **预处理阶段：**预处理器（cpp）根据以字符#开头的命令，修改原始的C程序。例如#include<stdio.h>命令会使预处理器读取系统的头文件stdio.h，并将其内容直接插入至程序文件中，经过预处理器处理之后得到另外一个C程序，通常以.i作为扩展名的文件。

* **编译阶段：**编译器（ccl）将文本文件hello.i翻译成文本文件hello.s，即一个汇编语言程序。

```
main:
	subq $8, %rsp
	movl $.LCO, %edi
	call puts
	movl $0, %eax
	addq $8, %rsp
	ret
```

* **汇编阶段：**
汇编器（as）将hello.s翻译成机器语言指令，这些机器指令打包成可重定向目标程序（relocatable object program）的格式，以二进制格式的方式保存到目标文件hello.o中。

* **链接阶段：**
链接器（ld）负责将源代码hello.c文件当中所使用到标准库函数或外部函数对应的目标文件.o（这里是printf.o），以特定的形式合并到hello.o文件中，最终得到的是hello可执行文件。


