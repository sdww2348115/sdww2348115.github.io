---
layout: post
title:  "红黑树简单分析"
date:   2019-08-22 23:57:00 +0800
categories: java datastructure map tree
permalink: /datastructure/RedBlackTree
description: 红黑树简单分析
---

## 2-3-4树

2-3-4树是一种平衡、查找树，它由以下3种节点组成：

* 2-node:node中含有一个值，两个children。左子树中所有节点的值都比当前node中的值小，右子树中所有节点的值都比当前node的值大。
* 3-node:node中含有两个值，三个children。val1 < val2，且chiledren1子树中的所有值都小于val1,children2子树中所有值大于val1且小于val2,children3子树中的值大于val2。
* 4-node:node中含有三个值，四个children。有序关系同上。
如下图所示：

![red-black-tree](../img/2-3-4-node.png)

除节点外，2-3-4树可以保持完美平衡：从根节点到叶节点的路径高度完全一致。
