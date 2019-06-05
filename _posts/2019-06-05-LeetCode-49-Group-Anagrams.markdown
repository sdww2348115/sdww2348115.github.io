---
layout: post
title:  "Group Anagrams"
date:   2019-06-05 23:39:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/49
description: String分类
---

## 题目

> Given an array of strings, group anagrams together.

> Example:

> Input: ["eat", "tea", "tan", "ate", "nat", "bat"],
> Output:
> [
  ["ate","eat","tea"],
  ["nat","tan"],
  ["bat"]
]
> Note:

> All inputs will be in lowercase.
> The order of your output does not matter.

## 分析

典型的分类转换题目：我们需要用另一种数据结构来表示String,并定义其equals()方法：字母相同即代表相对，最后根据equals方法将其进行分类输出。

由于要使用equals方法进行分类，需要建立String->同类集合的快速查找关系，因此HashMap必然是首选数据结构。

算法如下：
	将每一个String转为另一种数据结构，并重新定义其equals()方法与hashcode()方法。使用该数据结构为Key，找到其Value所对应的Collection，将String放入进去。最后遍历Map得到结果。
	
HashMap查找/插入的时间复杂度为O(1)，String转换时间复杂度为O(1),Colletion可采用List，插入复杂度也为O(1),n个String则为O(n),最后转为结果的复杂度同样为O(n),因此算法的时间复杂度为O(n).
借用了HashMap存储中间元素，空间复杂度为O(n)

算法的优化可从hashcode()方法入手，如果大量元素积聚在一起，将导致HashMap查找的时间变长，严重情况下将退化为O(n)(退化为List)或者O(lgn)(退化为treeMap)。因此要考虑如何将元素的hashCode尽可能分散。

由于String对象大多长度较短，因此采用简单的取反、移位再&的方式让String对象尽量均匀分布，同时位运算也不会消耗太多CPU资源。

## 代码

``` java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        HashMap<Node, List<String>> map = new HashMap<>();
        for (String str: strs) {
            Node node = new Node(str);
            map.putIfAbsent(node, new LinkedList<String>());
            map.get(node).add(str);
        }
        
        return new LinkedList<List<String>>(map.values());
    }
    
    class Node {
        Integer hashCode = 0;
        int[] charCount = new int[26];
        public Node(String str) {
            for (char ch : str.toCharArray()) {
                Integer index= ch - 'a';
                charCount[index]++;
                hashCode ^= index;
            }
            hashCode = (hashCode << (str.length() % 6 + 1)) ^ hashCode;
        }
        
        public int hashCode() {
            return hashCode;
        }
        
        public boolean equals(Object obj) {
            if (obj == null || !Node.class.isInstance(obj)) {
                return false;
            } else {
                Node other = (Node) obj;
                for (int i = 0; i < 26; i++) {
                    if (charCount[i] != other.charCount[i]) {
                        return false;
                    }
                }
                return true;
            }
        }
    }
}
```

## 结果

Runtime: 41 ms, faster than 12.04% of Java online submissions for Group Anagrams.
Memory Usage: 42.2 MB, less than 89.14% of Java online submissions for Group Anagrams.