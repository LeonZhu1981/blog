title: 深入浅出算法-基础知识-chapter1
date: 2017-06-19 11:34:36
categories: programming
tags:
- algorithms
- data structure
---

# 算法的基本特性：
---
* **有穷性：** 必须**在有限的时间**之内求解。
* **确定性：** 必须**在确定的条件**下求解。
* **可行性：** 必须是**可实现的**。
* **输入&输出：** 必须包含**输入和输出**。

# 算法的基本分类：
---
* **穷举法：**
	* 暴力求解。例如一个较大范围的数字，使用循环排序。
	* 八皇后问题。

* **分而治之（减而治之）：**
	* 二分查找：减而治之。
	* 归并排序：分而治之。

* **贪心算法：**
	* 背包问题。
	* 士兵路径。

* **动态规划：**
	* 最小生成树：Prim，Kruskal。
	* 单源最短路：Dijkstra。

# 什么是优秀的算法？
---
时间效率高，存储空间小。

# 算法效率的度量方法：
---
* 事后统计法: 这种方法有较大的缺陷, 需要依赖于测试数据, 实际完成的程序, 以及有在不同的环境(OS, 硬件环境)造成不同的结果.

* 事先估计法: 在计算机程序编制前，依据统计方法对算法进行估算.

# 函数的渐进增长：
---
给定两个函数f(n)和g(n)，如果存在一个整数N，使得对于所有的n>N，f(n)总是比g(n)大，那么，我们说f(n)的增长渐近快于g(n)。

<!--more-->

# 算法时间复杂度定义：
---
在进行算法分析时，语句总的执行次数T(n)是关于问题规模n的函数，进而分析T(n)随n的变化情况并确定T(n)的数量级。算法的时间复杂度，也就是算法的时间量度，记作：T(n)=O(f(n))。它表示随问题规模n的增大，算法执行时间的增长率和f(n)的增长率相同，称作算法的渐近时间复杂度，简称为时间复杂度。其中f(n)是问题规模n的某个函数。

用O()来体现算法时间复杂度的记法, 称之为: 大O记号法。

# 推导大O记号法：
---
* 用常数1取代运行时间中的所有加法常数。
* 在修改后的运行次数函数中，只保留最高阶项。
* 如果最高阶项存在且不是1，则去除与这个项相乘的常数。得到的结果就是大O阶。

> **总结：** 
> 1. 大O记号法本质是使用[高阶无穷小项](http://baike.baidu.com/item/%E9%AB%98%E9%98%B6%E6%97%A0%E7%A9%B7%E5%B0%8F)的方式表示。
> 2. 讨论算法的大O记号法时，一定要分**最好情况**和**最坏情况**来讨论。例如：快速排序的时间复杂度最好的情况是：O(n\lgn)，最坏情况是：O(n²)。

**常数阶: O(1)**

```
sum = (1 + n) * 2
```

**线性阶: O(n)**

```
for (i = 0; i < 100; i++) {
	...
}
```

**对数阶: O(log2n)**

```
int number = 1;
while(number < n) {
	number = number * 2;
	...
}
```

**平方阶: O(n²)**

```
for (i = 0; i < 100; i++) {
	for (j = 0; j < 100; j++) {
		...	  
	}
} 
```

# 常见的时间渐进复杂度分析：
---
**优秀的时间复杂度：（一般来说能够优化到下面的时间复杂度就算是较为合格的算法）**
O(1)<O(logn)<O(n½)<O(n)<O(nlogn)


**可能需要优化的时间复杂度：**
O(n²)<O(n³)<O(2^n)<O(k^n)<O(n!)

* O(1)：基本运算，+，-，*，/等等。
* O(logn)：二分查找。
* O(n½)：枚举约数。
* O(nlogn)：归并排序；快速排序的期望复杂度；基于比较排序的算法下届。
* O(2^n)：枚举全部的子集。
* O(n!)：枚举全排列。

> 1. 容易忽略的地方：需要明确n到底是多大？如果在n比较小的时候，优化的算法意义并不大。
> 2. 如何估算？我们假设现代CPU每秒钟可以跑1亿条指令：
> * O(n²)：当n >= 10000时，需要跑1s。
> * O(n³)：当n >= 470时，需要跑1s。


# 算法空间复杂度定义：
---
算法的空间复杂度通过计算算法所需的存储空间实现，算法空间复杂度的计算公式记作：S(n)=O(f(n))，其中，n为问题的规模，f(n)为语句关于n所占存储空间的函数。

空间复杂度(Space Complexity)是对一个算法在运行过程中临时占用存储空间大小的量度。一个算法在计算机存储器上所占用的存储空间，包括存储算法本身所占用的存储空间，算法的输入输出数据所占用的存储空间和算法在运行过程中临时占用的存储空间这三个方面。

若输入数据所占空间只取决于问题本身，和算法无关，这样只需要分析该算法在实现时所需的辅助单元即可。若算法执行时所需的辅助空间相对于输入数据量而言是个常数，则称此算法为原地工作，空间复杂度为O(1)。

# 实例分析：
---
我们将使用[leetcode](https://leetcode.com/)上列出的算法题目来进行分析。关于为什么使用leetcode，请参考耗子哥的文章：[LEETCODE 编程训练](http://coolshell.cn/articles/12052.html)。

进入到leetcode网站之后，需要先完成注册和邮箱验证，然后就可以开始你的刷题之旅了。

首先搜索我们此次需要分析的题目:
![leetcode-subarray](http://static.zhuxiaodong.net/blog/static/images/leetcode-subarray.png)

### 题目描述: [here](https://leetcode.com/problems/maximum-subarray/#/description)
> Find the contiguous subarray within an array (containing at least one number) which has the largest sum.
> For example, given the array [-2,1,-3,4,-1,2,1,-5,4],
the contiguous subarray [4,-1,2,1] has the largest sum = 6.

### 算法一：暴力枚举（三重循环）

```
public class Solution {
    public int maxSubArray(int[] nums) {
        int size = nums.length;
        int result = 0;
        for (int start = 0; start < size; start++) {
            for (int end = start + 1; end <= size; end++) {
                int sum = 0;
                for (int i = start; i < end; i++) {
                    sum += nums[i];
                }
                if (sum > result) {
                    result = sum;
                }
            }
        }
        return result;
    }
}
```

上述代码在执行Run Code之后, 显示结果正确:
![leetcode-runcode1](http://static.zhuxiaodong.net/blog/static/images/leetcode-runcode1.png)

但是再执行Submit Solution时，会发现以下的错误：
![sumbit-code1](http://static.zhuxiaodong.net/blog/static/images/submit-code1.png)

![submit-code2](http://static.zhuxiaodong.net/blog/static/images/submit-code2.png)

原因是我们没有考虑到该题目并没有说sum最大值必须是正整数，因此，我们需要修改上面的代码，调整为int的最小值。

```
int result = Integer.MIN_VALUE;
```

此时，我们再次提交代码之后，会发现如下的错误提示：
![submit-code3](http://static.zhuxiaodong.net/blog/static/images/submit-code3.png)

leetcode提示该算法再计算一个较大范围的数组时，执行时间过长。其实我们分析代码不难看出，暴力解法采用了3次for循环，算法的时间复杂度为：**O(n³)**，这显然不是一个很好的解法。

### 算法二：优化枚举（两重循环）：

分析上述算法一，我们不难看出，去掉最内层循环，将sum结果提取到第二层循环，可以将时间复杂度降低为：**O(n²)**

```
public class Solution {
    public int maxSubArray(int[] nums) {
        int size = nums.length;
        int result = Integer.MIN_VALUE;
        for (int start = 0; start < size; start++) {
            int sum = 0;
            for (int end = start + 1; end <= size; end++) {
                sum += nums[end - 1];
                if (sum > result) {
                    result = sum;
                }
            }
        }
        return result;
    }
}
```

虽然算法复杂度降低为了O(n²)，但是leetcode仍然会给出超时的错误。

### 算法三：贪心算法（只循环1次）：

```
public class Solution {
    public int maxSubArray(int[] nums) {
        int size = nums.length;
        int result = Integer.MIN_VALUE;
        int sum = 0;
        for (int start = 0; start < size; start++) {
            sum += nums[start];
            if (sum > result) {
                result = sum;
            }
            // 如果sum的结果小于0，则抛弃掉该值。
            if (sum < 0) {
                sum = 0;
            }
        }
        return result;
    }
}
```

最终算法复杂度被优化为O(n)。

![submit-code4](http://static.zhuxiaodong.net/blog/static/images/submit-code4.png)

