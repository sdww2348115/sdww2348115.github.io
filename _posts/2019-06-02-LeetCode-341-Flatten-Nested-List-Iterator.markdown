---
layout: post
title:  "Flatten Nested List Iterator"
date:   2019-06-02 21:22:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/341
description: 迭代器处理
---

## 题目

> Given a nested list of integers, implement an iterator to flatten it.

> Each element is either an integer, or a list -- whose elements may also be integers or other lists.

> Example 1:

> Input: [[1,1],2,[1,1]]
> Output: [1,1,2,1,1]
> Explanation: By calling next repeatedly until hasNext returns false, 
             the order of elements returned by next should be: [1,1,2,1,1].
> Example 2:

> Input: [1,[4,[6]]]
> Output: [1,4,6]
> Explanation: By calling next repeatedly until hasNext returns false, 
             the order of elements returned by next should be: [1,4,6].