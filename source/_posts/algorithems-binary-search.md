title: 深入浅出算法-二分查找
date: 2017-10-20 15:50:00
categories: programming
tags:
- algorithms
- data structure
---

# 二分查找的定义：
---

下面是维基百科中的定义:
> 在计算机科学中，二分搜索（英语：binary search），也称折半搜索（英语：half-interval search）[1]、对数搜索（英语：logarithmic search）[2]，是一种在有序数组中查找某一特定元素的搜索算法。搜索过程从数组的中间元素开始，如果中间元素正好是要查找的元素，则搜索过程结束；如果某一特定元素大于或者小于中间元素，则在数组大于或小于中间元素的那一半中查找，而且跟开始一样从中间元素开始比较。如果在某一步骤数组为空，则代表找不到。这种搜索算法每一次比较都使搜索范围缩小一半。

**关键点：**
* 有序：这个很好理解，因为算法当中必须获取中间的元素。
* 数组：因为只有数组才能够通过索引去获取元素，例如：item[mid]。


<!--more-->

# 算法的复杂度：
---
**对数阶: O(log2n)**

# python ：
---

```
# coding=utf-8 

def binary_search(list, item):
    low = 0
    high = len(list) - 1

    while low <= high: # 只要范围没有缩小到包含一个元素
        mid = (low + high) / 2 # 取中间值
        guess = list[mid]
        if guess == item: # 如果找到了元素，直接返回
            return mid
        if guess > item: # 如果猜得数字大了，则缩小范围
            high = mid - 1
        else: # 如果猜得数字小了，同样缩小范围
            low = mid + 1
    return None

# 注意：下面的数组将数据直接按照顺序进行初始化
LIST = [1, 3, 5, 7, 9, 11]

print binary_search(LIST, 5)
print binary_search(LIST, 9)
print binary_search(LIST, 10)
```

output:

```
2
4
None
```

# javascript ：
---

```
function binarySearch(list, item) {
    let low = 0;
    let high = list.length - 1;

    while (low <= high) {
        let middle = Math.ceil((low + high) / 2, 2);
        let guess = list[middle];
        if (guess > item) {
            high = middle - 1;
        } else if (guess < item) {
            low = middle + 1;
        } else if (guess === item) {
            return middle;
        }
    }
    return null;
}

const sortList = [1, 2, 3, 4, 5, 8, 9, 10, 11, 13, 15, 18];

console.log(binarySearch(sortList, 4));
console.log(binarySearch(sortList, 18));
console.log(binarySearch(sortList, 23));
```

output:

```
3
11
null
```

# Java :
---

```
import java.util.Arrays;

public class BinarySearch {
    private static int rank(int[] list, int item) {
        int low = 0;
        int high = list.length - 1;

        while (low <= high) {
            int middle = (low + high) / 2;
            int guess = list[middle];
            if (guess > item) {
                high = middle - 1;
            } else if (guess < item) {
                low = middle + 1;
            } else if (guess == item) {
                return middle;
            }
        }
        return null;
    }

    public static void main(String[] args) {
        int[] myList = {1, 3, 7, 2, 5, 9, 11, 10, 98};
        Arrays.sort(myList);
        System.out.println(rank(myList, 2));
        System.out.println(rank(myList, 98));
        System.out.println(rank(myList, -1));
    }
}
```

output:

```
1
8
null
```

# JDK 中 BinarySearch 的实现：
---

java.utils 包中的 Arrays 类包含了对二分查找算法的实现，让我们来看看其中的一些实现细节：

```
public static int binarySearch(int[] a, int key) {
    return binarySearch0(a, 0, a.length, key);
}

// Like public version, but without range checks.
private static int binarySearch0(int[] a, int fromIndex, int toIndex,
                                 int key) {
    int low = fromIndex;
    int high = toIndex - 1;

    while (low <= high) {
        int mid = (low + high) >>> 1;
        int midVal = a[mid];

        if (midVal < key)
            low = mid + 1;
        else if (midVal > key)
            high = mid - 1;
        else
            return mid; // key found
    }
    return -(low + 1);  // key not found.
}
```

可见，区别是一些实现细节的不同，例如：

* 提供了 fromIndex ， toIndex ，用来指定查找的范围。
* 使用位运算 `int mid = (low + high) >>> 1` 来提升了性能。
* 使用返回值为负数来表示位查找到值。

除此之外， Arrays 类还提供了许多重载版本的方法：
![java-arrays-binary-search](http://static.zhuxiaodong.net/blog/static/images/java-arrays-binary-search.png)
