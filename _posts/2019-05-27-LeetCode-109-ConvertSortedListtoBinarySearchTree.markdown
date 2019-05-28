---
layout: post
title:  "Convert Sorted List to Binary Search Tree"
date:   2019-05-27 23:51:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/109
description: 递归算法处理二叉树问题
---

## 题目

> Given a singly linked list where elements are sorted in ascending order, convert it to a height balanced BST.

> For this problem, a height-balanced binary tree is defined as a binary tree in which the depth of the two subtrees of every node never differ by more than 1.

> Example:

> Given the sorted linked list: [-10,-3,0,5,9],

> One possible answer is: [0,-3,9,-10,null,5], which represents the following height balanced BST:

>      0
>     / \
>   -3   9
>   /   /
>  -10  5


## 分析
遇到二叉树的构建问题，我们可以直接使用递归的方式对其进行处理。由于是平衡二叉树，左右子树最大高度最多相差1，因此采用二分的方式递归：将排好序的node从中剖开，分为中间点，左子队列与右子队列，中间点作为root，左子队列构建的树作为root左节点，右子队列同理。
优化：原始数据结构为链表形式，而二分法的过程中需要大量使用随机访问，因此首先将原始数据结构转换为方便随机访问的数组形式。总时间复杂度为O(n)，空间复杂度主要是递归的栈开销O(lgn)

## 代码
``` java
class Solution {
    public TreeNode sortedListToBST(ListNode head) {
        
        if (head == null) {
            return null;
        }
        
        //convert to array
        int i = 0;
        ListNode node = head;
        while (node != null) {
            i++;
            node = node.next;
        }
        ArrayList<Integer> valsInarray = new ArrayList<>(i);
        i = 0;
        node = head;
        while (node != null) {
            valsInarray.add(i++, node.val);
            node = node.next;
        }
        
        //recursion process
        return sortedArrayToBST(valsInarray, 0, valsInarray.size() - 1);
    }
    
    private TreeNode sortedArrayToBST(ArrayList<Integer> array, int startIndex, int endIndex) {
        
        if (startIndex == endIndex) {
            return new TreeNode(array.get(startIndex));
        }
        
        int middleIndex = (startIndex + endIndex) / 2;
        TreeNode midNode = new TreeNode(array.get(middleIndex));
        if (startIndex < middleIndex) {
            midNode.left = sortedArrayToBST(array, startIndex, middleIndex - 1);
        }
        if (middleIndex < endIndex) {
            midNode.right = sortedArrayToBST(array, middleIndex + 1, endIndex);
        }
        return midNode;
    }
}
```

## 结果
Runtime: 1 ms, faster than 97.28% of Java online submissions for Convert Sorted List to Binary Search Tree.
Memory Usage: 37 MB, less than 99.98% of Java online submissions for Convert Sorted List to Binary Search Tree.