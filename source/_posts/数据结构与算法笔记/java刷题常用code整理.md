---
title: java刷题常用code整理
tags:
  - 文章内容的关键词
  - private
categories:
  - 对照文件存放的目录名称
toc: true
date: 2023-10-10 18:31:02
---

# 排序问题
## 数组排序
```java
int[] arr = new int[]{1,2,3};
Arrays.sort(arr);
System.out.println(Arrays.toString(arr));
```

## List排序
```java
List<Integer> arr = new ArrayList<>();
arr.add(1);
arr.add(2);
arr.add(3);
Collections.sort(arr);
arr.sort((o1, o2) -> o2-o1);
System.out.println(arr.toString());
```

## 二维数组排序
```java
int[][] arr2 = new int[][]{{0,1},{1,2},{3,0}};
Arrays.sort(arr2, Comparator.comparingInt(o -> o[1])); // 按第二列升序排序
```
```java
// 自定义比较器Comparator
int[][] arr2 = new int[][]{{0,1},{1,2},{3,0}};
Arrays.sort(price_rating, new MyComparator());

static class MyComparator implements Comparator<int[]> {
	@Override
	public int compare(int[] arr1, int[] arr2) {
		return arr1[1] - arr2[1];
	}
}
```
        
# 最值问题
## 数组求最值
```java
// int[]数组的最大值
int[] counts = new int[]{1, 2, 3};
int max = Arrays.stream(counts).max().getAsInt();
```

# 类型转换问题
## char[]和String互相转换
```java
String str = 'letter';
// String转换为char[]
char[] strToChars = str.toCharArray();
// char[]转换为String
String charsToStr = String.valueOf(chars);
```

## int[]和String[]互相转换
```java
// String[] 转换为 int[]
String[] strs;
int[] nums = Arrays.stream(strs).mapToInt(Integer::parseInt).toArray();
```

# ArrayList方法
## remove 移除元素
```java
// 删除指定元素。remove() 方法仅将对象作为其参数。如果 obj 元素出现多次，则删除在动态数组中最第一次出现的元素。
arraylist.remove(Object obj);
// 如果list元素是Integer类型，则需要用Integer.valueOf()将 删除元素 从 int 类型转变成一个 Integer 对象。
randomNumbers.remove(Integer.valueOf(13));

        
// 删除指定索引位置的元素
arraylist.remove(int index);
```