title: 深入浅出算法-基本数据结构-链表linked
date: 2017-06-21 11:34:36
categories: programming
tags:
- algorithms
- data structure
---

# 定义：
---
1. 用于存放数据的**线性表**。
2. 相关的操作：入栈/入队列，出栈/出队列，判断栈/队列是否存储满/空。
3. 空间复杂度：O(n)
4. 单次操作的时间复杂度：O(1)
5. 区别：
	* 栈：先进后出(FILO)。
	* 堆：先进先出(FIFO)。

# 实现：
---
1. 使用数组或者链表来存储（线性表）。
2. 指针（辅助变量）：
	* 栈顶/底指针。
	* 队头/尾指针。
3. 关键：出入元素时同时移动指针。

# 实例分析：
---
我们将使用[leetcode](https://leetcode.com/)上列出的算法题目：[Decode String](https://leetcode.com/)来进行分析。

