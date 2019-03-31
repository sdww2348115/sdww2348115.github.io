---
layout: post
title:  "Create Maximum Number"
date:   2019-03-31 22:07:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/321
---

## 题目
> Given two arrays of length m and n with digits 0-9 representing two numbers. Create the maximum number of length k <= m + n from digits of the two. The relative order of the digits from the same array must be preserved. Return an array of the k digits.

> Example 1:
Input:
nums1 = [3, 4, 6, 5]
nums2 = [9, 1, 2, 5, 8, 3]
k = 5
Output:
[9, 8, 6, 5, 3]

> Example 2:
Input:
nums1 = [6, 7]
nums2 = [6, 0, 4]
k = 5
Output:
[6, 7, 6, 0, 4]

## 分析
分析题意：有两个分别长为m,n的数组，数组中的数字位1-9中的一个。我们要创建一个长度为k的数组，其中k <= m + n,k
中的数组必须来自这两个数组，要求使得数组中数字最大。

从简单的情况开始分析：当k = 1时，新数组等于数组1中最大的值或者数组2中最大的值
当k = 2时，新数组等于数组1中最大的两个数的子序列或者数组2中最大两个数的子序列或者数组1中最大的值加上数组2中最大的值合并。
由此可递推下去：设maxSub(array, k)为数组array中，长度为k的最大数子序列，则题目的解就是
for (int i = 0; i <= k; i++) maxSub(array1, i) + maxSub(array2, k - i)中的最大值。题目最核心的难点就是求单个数组中长度为p且最大数字的子序列。

求单个数组中长度为p最大数字子序列：一个典型的动态规划问题，经观察可知，Max（x, y）表示序列中sub（x, nums.length）中长度为y的最大和子序列，则Max(x, y) = Math.max(xMax(x-1, y-1), Max(x-1, y)),如果将中间结果全部缓存下来，时间复杂度为O(n^2),空间复杂度为O(n^3).

merge的时间复杂度为O(k)，因此整个算法的时间复杂度为O(n^3 + m^3 + k)，即O(math.max(n^3, m^3))。空间复杂度为O(n^3 + m^3)

## 答案