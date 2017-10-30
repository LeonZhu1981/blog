title: 深入浅出算法-3种简单排序-选择-冒泡-插入
date: 2017-10-27 10:15:16
categories: programming
tags:
- algorithms
- data structure
---

# 选择排序（ Selection sort ）
---

## 定义：
---

下面是维基百科中的定义:
> 选择排序（Selection sort）是一种简单直观的排序算法。它的工作原理如下。首先在未排序序列中找到最小（大）元素，存放到排序序列的起始位置，然后，再从剩余未排序元素中继续寻找最小（大）元素，然后放到已排序序列的末尾。以此类推，直到所有元素均排序完毕。
> 选择排序的主要优点与数据移动有关。如果某个元素位于正确的最终位置上，则它不会被移动。选择排序每次交换一对元素，它们当中至少有一个将被移到其最终位置上，因此对n个元素的表进行排序总共进行至多n-1次交换。在所有的完全依靠交换去移动元素的排序方法中，选择排序属于非常好的一种。

![selection-sort](https://upload.wikimedia.org/wikipedia/commons/b/b0/Selection_sort_animation.gif)


![selection-sort](https://upload.wikimedia.org/wikipedia/commons/9/94/Selection-Sort-Animation.gif)

上图中红色表示当前最小值，黄色表示已排序序列，蓝色表示当前位置。

<!--more-->

## 算法的复杂度：
---

数据结构：数组
最坏时间复杂度：О(n²)
最优时间复杂度：О(n²)
平均时间复杂度：О(n²)
空间复杂度：О(n) total, O(1) auxiliary

## python ：
---

```
# coding=utf-8 

def find_smallest(arr):
    smallest = arr[0]
    smallest_index = 0

    for i in range(1, len(arr)):
    	if arr[i] < smallest:
    		smallest = arr[i]
    		smallest_index = i
    return smallest_index

def selection_sort(arr):
	newArr = []
	for i in range(len(arr)):
		smallest = find_smallest(arr)
		newArr.append(arr.pop(smallest))
	return newArr

print selection_sort([3,2,5,9,110,1,8,20])
```

output:

```
[1, 2, 3, 5, 8, 9, 20, 110]
```

## javascript ：
---

```
function findSmallest(array) {
    let smallestItem = array[0];
    let smallestIndex = 0;

    for (let i = 0; i < array.length; i++) {
        if (array[i] < smallestItem) {
            smallestItem = array[i];
            smallestIndex = i;
        }
    }
    return smallestIndex;
}

function selectionSort(array) {
    let newArray = [];
    for (let i = array.length; i > 0; i--) {
        let smallestIndex = findSmallest(array);
        newArray.push(array[smallestIndex]);
        array.splice(smallestIndex, 1);
    }
    return newArray;
}

function selectionSortV2(array) {
    let min = 0;
    let temp = [];
    for (let i = 0; i < array.length - 1; i++) {
        min = i;
        for (let j = i + 1; j < array.length; j++) {
            if (array[min] > array[j]) {
                min = j;
            }
        }
        temp = array[min];
        array[min] = array[i];
        array[i] = temp;
    }
    return array;
};

console.log("v1 version:" + selectionSort([3, 2, 5, 9, 110, 1, 8, 20]));
console.log("v2 version:" + selectionSortV2([3, 2, 5, 9, 110, 1, 8, 20]));
```

output:

```
v1 version:1,2,3,5,8,9,20,110
v2 version:1,2,3,5,8,9,20,110
```

## java ：
---

```
import java.util.Arrays;

public class SelectionSort {
    private static void sort(int[] array) {
        int arrayLength = array.length;
        int temp, smallestIndex = 0;
        for (int i = 0; i < arrayLength - 1; i++) {
            smallestIndex = i;
            for (int j = i + 1; j < arrayLength; j++) {
                if (array[smallestIndex] > array[j]) {
                    smallestIndex = j;
                }
            }
            temp = array[smallestIndex];
            array[smallestIndex] = array[i];
            array[i] = temp;
        }
        System.out.println(Arrays.toString(array));
    }

    public static void main(String[] args) {
        int[] testCase = {0, 5, 9, 10, 4, 6, 3, 2};
        sort(testCase);
    }
}
```

output:

```
[0, 2, 3, 4, 5, 6, 9, 10]
```

# 冒泡排序（ Bubble sort ）
---

## 定义：
---

冒泡排序的基本思想是，对相邻的元素进行两两比较，顺序相反则进行交换，这样，每一趟会将最小或最大的元素“浮”到顶端，最终达到完全有序。

![bubble-sort](http://static.zhuxiaodong.net/blog/static/images/bubble-sort.png)

## 算法复杂度：
---

数据结构	数组
最坏时间复杂度：О(n²)
最优时间复杂度：О(n²)
平均时间复杂度：О(n²)
空间复杂度：总共 O(n)，需要辅助空间 O(1)

## python ：
---

```
# coding=utf-8 

def bubble_sort(array):
	for i in range(len(array) - 1, 0, -1):
		flag = True
		for j in range(0, i):
			if array[j] > array[j + 1]:
				flag = False
				array[j], array[j + 1] = array[j + 1], array[j]
		if flag:
			return array
	return array

test_case = [2, 1, 3, 5, 4, 35, 22, 20, 8]
print(bubble_sort(test_case))
```

## javascript ：
---

```
function bubbleSort(array) {
    let length = array.length;
    let temp = 0;
    for (let i = 0; i < length - 1; i++) {
        let flag = true;
        for (let j = 0; j < length - 1 - i; j++) {
            if (array[j] > array[j + 1]) {
                flag = false;
                temp = array[j];
                array[j] = array[j + 1];
                array[j + 1] = temp;
            }
        }
        if (flag) {
            break;
        }
    }
    return array;
}

let test_case = [2, 1, 3, 5, 4, 35, 22, 20, 8]
console.log(bubbleSort(test_case))
```

## java ：
---

```
import java.util.Arrays;

public class BubbleSort {
    private static int[] sort(int[] array) {
        int arrayLength = array.length;
        int temp = 0;

        for (int i = 0; i < arrayLength - 1; i++) {
            boolean flag = true;
            for (int j = 0; j < arrayLength - 1 - i; j++) {
                if (array[j] > array[j + 1]) {
                    temp = array[j];
                    array[j] = array[j + 1];
                    array[j + 1] = temp;
                    flag = false; 
                }
            }
            if (flag) {
                break;
            }
        }

        return array;
    }

    public static void main(String[] args) {
        int[] testCase = {0, 5, 9, 10, 4, 6, 3, 2};
        System.out.println(Arrays.toString(sort(testCase)));
    }
}
```

# 插入排序（ Insertion sort ）
---

## 定义：
---

直接插入排序基本思想是每一步将一个待排序的记录，插入到前面已经排好序的有序序列中去，直到插完所有元素为止。

具体算法描述如下：
1. 从第一个元素开始，该元素可以认为已经被排序
2. 取出下一个元素，在已经排序的元素序列中从后向前扫描
3. 如果该元素（已排序）大于新元素，将该元素移到下一位置
4. 重复步骤 3，直到找到已排序的元素小于或者等于新元素的位置
5. 将新元素插入到该位置后
6. 重复步骤 2~5

如果比较操作的代价比交换操作大的话，可以采用二分查找法来减少比较操作的数目。该算法可以认为是插入排序的一个变种，称为二分查找排序。

**实例：**
现有一组数组 arr = [5, 6, 3, 1, 8, 7, 2, 4]，共有八个记录，排序过程如下：

```
[5]   6   3   1   8   7   2   4
  ↑   │
  └───┘
[5, 6]   3   1   8   7   2   4
↑        │
└────────┘
[3, 5, 6]  1   8   7   2   4
↑          │
└──────────┘
[1, 3, 5, 6]  8   7   2   4
           ↑  │
           └──┘
[1, 3, 5, 6, 8]  7   2   4
            ↑    │
            └────┘
[1, 3, 5, 6, 7, 8]  2   4
   ↑                │
   └────────────────┘
[1, 2, 3, 5, 6, 7, 8]  4
         ↑             │
         └─────────────┘
 
[1, 2, 3, 4, 5, 6, 7, 8]
```

## 算法复杂度：
---

数据结构	数组
最坏时间复杂度：О(n²)
最优时间复杂度：О(n²)
平均时间复杂度：О(n²)
空间复杂度：总共 O(n)，需要辅助空间 O(1)


## python ：
---

```
# coding=utf-8 

def insertion_sort(array):
    if len(array) == 1:
        return array

    for i in range(1, len(array)):
        temp = array[i]
        j = i - 1
        while j >= 0 and temp < array[j]:
            array[j + 1] = array[j]
            j -= 1
        array[j + 1] = temp

    return array

print insertion_sort([3,2,5,9,110,1,8,20])
```

## javascript ：
---

```
function insertionSort(array) {
    function swap(array, i, j) {
        let temp = array[i];
        array[i] = array[j];
        array[j] = temp;
    }

    let length = array.length;

    for (let i = 1; i < length; i++) {
        for (let j = i; j > 0; j--) {
            if (array[j - 1] > array[j]) {
                swap(array, j - 1, j);
            } else {
                break;
            }
        }
    }

    return array;
}

let test_case = [2, 1, 3, 5, 4, 35, 22, 20, 8]
console.log(insertionSort(test_case))
```

## java ：
---

```
import java.util.Arrays;

public class InsertionSort {
    public static int[] sort(int[] array) {
        int length = array.length;
        int temp = 0;

        for (int i = 1; i < length; i++) {
            temp = array[i];
            for (int j = i; j >= 0; j--) {
                if (array[j - 1] > temp) {
                    array[j] = array[j - 1];
                } else {
                    array[j] = temp;
                    break;
                }
            }
        }

        return array;
    }

    public static void main(String[] args) {
        int[] testCase = {0, 2, 5, 4, 3, 55, 88, 35, 22, 11, 9};
        System.out.println(Arrays.toString(sort(testCase)));
    }
}
```

