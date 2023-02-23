---
title: 关于Java PriorityQueue类的使用场景
tags:
  - java
  - 数据结构
categories:
  - 刷题
toc: true
abbrlink: 76a5661e
date: 2023-02-23 16:53:35
---
最近在leetcode刷题的时候，发现很多题推荐解法是用**优先队列**的特性，比如：[滑动窗口的最大值](https://leetcode.cn/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/) 、[丑数](https://leetcode.cn/problems/chou-shu-lcof/)

以前完全没有用个这个类，所以在此整理一下对优先队列的认识和刷题场景


# 优先队列的特性
很明显，优先队列也是一种队列，只不过其出队顺序和一般队列不同，优先队列的出队顺序是按照一定的优先级来的，也就是说出队规则可以随意定制

优先队列ADT是一种数据结构，它支持插入、删除最小值操作（返回并删除最小元素）、删除最大值操作（返回并删除最大元素）

优先队列的主要操作：优先队列是元素的容器，每个元素有一个相关的键值
* insert(key, data)：插入键值为key的数据到优先队列中，元素以其key进行排序
* deleteMin/deleteMax：删除并返回最小/最大键值的元素
* getMinimum/getMaximum：返回最小/最大剑指的元素，但不删除它

优先队列的辅助操作：
* 第k最小/第k最大：返回优先队列中键值为第k个最小/最大的元素
* 大小（size）：返回优先队列中的元素个数
* 堆排序（Heap Sort）：基于键值的优先级将优先队列中的元素进行排序

在某些场景下，比如要求队列中的最小元素先出即可用优先队列，在java中的实现类为`java.util.PriorityQueue`。


# 认识下PriorityQueue类的方法
## 创建对象
```java
// 默认情况下，优先级队列的头是队列中最小的元素，元素将按升序从队列中移除
PriorityQueue<Integer> nums = new PriorityQueue<>();

// 借助 Comparator 接口自定义元素的顺序，头是队列中最大的元素，按降序从队列中移除
PriorityQueue<int[]> win = new PriorityQueue<int[]>(new Comparator<int[]>() {
    public int compare(int[] a, int[] b) {
        return a[0] != b[0] ? b[0] - a[0] : b[1] - a[1];
        }});
```

## 插入元素
```java
class Main {
	public static void main(String[] args) {

		//创建优先队列
		PriorityQueue<Integer> numbers = new PriorityQueue<>();

		//使用add()方法，如果队列已满，则会引发异常
		numbers.add(4);
		numbers.add(2);
		System.out.println("PriorityQueue: " + numbers);

		//使用offer()方法，如果队列已满，则返回false
		numbers.offer(1);
		System.out.println("更新后的PriorityQueue: " + numbers);
	}
}

/*
 * 输出结果：
 * PriorityQueue: [2, 4]
 * 更新后的PriorityQueue: [1, 4, 2]
 * 
 * 以上结果中，队列的头是最小元素
 */

```

## 访问元素
```java
//使用 peek() 方法
int number = nums.peek();
System.out.println("访问元素: " + number);

/*
 * 输出结果：
 * 访问元素: 1
 */
```

## 删除元素
```java
//使用remove()方法，从队列中删除指定的元素
boolean result = numbers.remove(2);
System.out.println("元素2是否已删除? " + result);

//使用poll()方法，返回并删除队列的头
int number = numbers.poll();
System.out.println("使用poll()删除的元素: " + number);

/*
 * 输出结果：
 * 元素2是否已删除? true
 * 使用poll()删除的元素: 1
 */
```

## 是否包含元素
```java
//使用contains()方法，从队列中搜索指定的元素，找到则返回true，否则false。
boolean result = numbers.contains(4);
System.out.println("队列中是否有元素 4 ？" + result);

/*
 * 输出结果：
 * 队列中是否有元素 4 ？ true
 */
```

# 刷题场景
## [滑动窗口的最大值](https://leetcode.cn/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/)
```java
//给定一个数组 nums 和滑动窗口的大小 k，请找出所有滑动窗口里的最大值。 
//示例: 
//输入: nums = [1,3,-1,-3,5,3,6,7], 和 k = 3
//输出: [3,3,5,5,6,7] 
//解释:
//  滑动窗口的位置                最大值
//---------------               -----
//[1  3  -1] -3  5  3  6  7       3
// 1 [3  -1  -3] 5  3  6  7       3
// 1  3 [-1  -3  5] 3  6  7       5
// 1  3  -1 [-3  5  3] 6  7       5
// 1  3  -1  -3 [5  3  6] 7       6
// 1  3  -1  -3  5 [3  6  7]      7

import java.util.Comparator;
import java.util.PriorityQueue;
/*
 * 解题思路：利用优先队列的特性，规定堆顶元素就是窗口最大值
 * */
class Solution {
	public int[] maxSlidingWindow(int[] nums, int k) {
		int len = nums.length;
		PriorityQueue<int[]> win = new PriorityQueue<int[]>(new Comparator<int[]>() {
			// 重新定义出队规则
			@Override
			public int compare(int[] a, int[] b) {
				return a[0] != b[0] ? b[0] - a[0] : b[1] - a[1];
			}
		});
		// 初始化窗口
		for (int i = 0; i < k; ++i) {
			win.offer(new int[]{nums[i], i});
		}
		// 创建指定长度的结果数组
		int[] res = new int[len-k+1];
		// 第一个窗口的最大值
		res[0] = win.peek()[0];
		// 遍历滑动窗口
		for (int i = k; i < len; ++i) {
			// 添加新元素
			win.offer(new int[]{nums[i], i});
			// 删除窗口长度外的元素
			while (win.peek()[1] <= i - k) {
				win.poll();
			}
			// 返回当前窗口的最大值
			res[i - k + 1] = win.peek()[0];
		}
		return res;
	}
}
```

## [丑数](https://leetcode.cn/problems/chou-shu-lcof/)
```java
// 我们把只包含质因子 2、3 和 5 的数称作丑数（Ugly Number）。求按从小到大的顺序的第 n 个丑数。
// 示例: 
// 输入: n = 10
// 输出: 12
// 解释: 1, 2, 3, 4, 5, 6, 8, 9, 10, 12 是前 10 个丑数。 
// 说明:
// 1 是丑数。 
// n 不超过1690。


import java.util.HashSet;
import java.util.PriorityQueue;
import java.util.Set;

/*
 * 解题思路：最小堆，需借助java的PriorityQueue类的特性实现：https://www.cainiaojc.com/java/java-priorityqueue.html
 * 初始化堆，将最小丑数1放入堆
 * 每次取出堆顶元素x，x元素也是堆中最小的丑数，需排除重复元素，依次将 2x,3x,5x 加入堆
 * 第n次取出的堆顶元素就是第n个丑数
 * */
class Solution {
	public int nthUglyNumber(int n) {
		int[] factors = {2,3,5};
		PriorityQueue<Long> heap = new PriorityQueue<Long>();   //优先级队列的头是队列中最小的元素
		heap.offer(1L);     // 初始化最小堆，放入最小丑数1
		int ugly = 0;
		for(int i=0; i<n; i++){
			long cur = heap.poll();     //返回并删除队列的头，即最小元素
			ugly = (int) cur;
			for (int factor : factors){
				long next = cur*factor;
				//检查是否有重复元素
				if(!heap.contains(next)){
					heap.offer(next);
				}
			}
		}
		return ugly;
	}
}
```
# 参考资料
[数据结构与算法(4)——优先队列和堆](https://zhuanlan.zhihu.com/p/39615266) 

[Java PriorityQueue](https://www.cainiaojc.com/java/java-priorityqueue.html)