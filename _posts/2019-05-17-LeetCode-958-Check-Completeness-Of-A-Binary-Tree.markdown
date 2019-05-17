---
layout: post
title:  "Check Completeness of a Binary Tree"
date:   2019-05-17 23:21:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/958
description: leetcode第958题，利用层次遍历判断是否是完全二叉树
---

## 题目
> Given a binary tree, determine if it is a complete binary tree.

> Definition of a complete binary tree from Wikipedia:
> In a complete binary tree every level, except possibly the last, is completely filled, and all nodes in the last level are as far left as possible. It can have between 1 and 2h nodes inclusive at the last level h.

> Example 1:

> Input: [1,2,3,4,5,6]
Output: true
Explanation: Every level before the last is full (ie. levels with node-values {1} and {2, 3}), and all nodes in the last level ({4, 5, 6}) are as far left as possible.

> Example 2:



> Input: [1,2,3,4,5,null,7]
Output: false
Explanation: The node with value 7 isn't as far left as possible.
 
> Note:
The tree will have between 1 and 100 nodes.

## 分析
按照定义，对完全二叉树进行层次遍历，所有的null节点将被放在最后，因此利用层次遍历处理一下目标树，通过判断null节点是否在末尾即可得知是否是完全二叉树。
时间复杂度为O(N)，空间复杂度为O(lgN).

## 答案
``` java
class Solution {
    public boolean isCompleteTree(TreeNode root) {
        
        List<TreeNode> list1 = new LinkedList<>();
        List<TreeNode> list2 = new LinkedList<>();
        
        list1.add(root);
        
        boolean isPreNull = false;
        while (list1.size() > 0) {
            for (TreeNode node : list1) {
                if (node != null) {
                    if (isPreNull) {
                        return false;
                    }
                    list2.add(node.left);
                    list2.add(node.right);
                } else {
                    isPreNull = true;
                }
            }
            list1 = list2;
            list2 = new LinkedList<>();
        }
        return true;
    }
}
```
Runtime: 1 ms, faster than 95.93% of Java online submissions for Check Completeness of a Binary Tree.
Memory Usage: 35.4 MB, less than 95.39% of Java online submissions for Check Completeness of a Binary Tree.