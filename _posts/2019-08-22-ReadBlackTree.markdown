---
layout: post
title:  "红黑树简单分析"
date:   2019-08-22 23:57:00 +0800
categories: java datastructure map tree
permalink: /datastructure/RedBlackTree
description: 红黑树简单分析
---

## 性质
红黑树是计算机工程中最常见的二叉平衡查询树。原因是它具有以下优秀性质：

* 相比于一般的二叉查询树，它具有一定程度的平衡性质：即从root结点出发，到最远结点的距离最多不超过到最近结点的两倍。
* 相比AVL树，红黑树并不是在每一次进行插入/删除操作时就会出发左旋右旋操作，插入/删除的平均开销较AVL树更小
* 相比于原型2-3-4树，红黑树不涉及到结点类型的变更，因此不会大量创建/删除树结点对象；实现起来不需要考虑结点类型的差异

红黑树由2-3-4推导得出，并拥有以下性质：

1. 所有节点都有颜色，且为红色或者黑色
2. 根节点为黑色
3. 所有叶子节点（包含null）都是黑色
4. 红色节点的子节点一定为黑色
5. 根节点到任何叶子节点的距离不超过距离最短路径叶子节点距离的2倍（从root到任何叶子节点所经过的黑色节点数目相等）

## 节点定义

类似于一般二叉树，红黑树的节点仅仅是多了color属性，如下
```java
    @Data
    private class Node {
        private K key;
        private V value;
        private Boolean color;
        private Node left;
        private Node right;

        public Node (K key, V value, Boolean color) {
            this.key = key;
            this.value = value;
            this.color = color;
        }
    }
```

## 搜索

由于红黑树是一棵基本平衡的二叉树，查询过程与一般二叉树的查询过程保持一致。
``` java
    /**
     * 搜索目标key值节点，如果不存在则返回null
     * @param target 
     */
    public Node searchNode(K target, Node root) {
        if (root == null) {
            return null;
        }
        int cmp = root.key.compareTo(target);
        if (cmp == 0) {
            return root;
        } else if (cmp > 0) {
            return searchNode(target, root.left);
        } else {
            return searchNode(target, root.right);
        }
    }
```

## 插入

红黑树的插入同样带有强烈的二叉查找树性质以及平衡树的性质。根据2-3-4树的插入过程可知：2-3-4树插入时仅会向最底层叶子节点进行插入，且都是2节点变为3节点、3节点变为4节点类型的变化，对树的高度没有影响；2-3-4树高度的变化是由下至上通过节点的不断分裂与组合实现的，因此能够保证根节点到最远节点的距离与最近的节点保持一致。

简单来说：2-3-4树与红黑树都是向上长的，因此能够保证叶子节点的高度必然一致。

第一部分：二叉查找树的插入

根据比较结果，递归处理树的插入。
```java
    /**
     * 向红黑树中插入新的数据
     * @param key 键值对的key值，索引依赖项
     * @param value 键值对的value值
     * @param root 插入目标树的根节点
     * @return 插入后所生成树的根节点
     */
    public Node insert(K key, V value, Node root) {
        
        if (root == null) {
            return new Node(key, value, RED);
        }
        
        int cmp = root.key.compareTo(key);
        if (cmp == 0) {
            root.value = value;
        } else if (cmp > 0) {
            root.left = insert(key, value, root.left);
        } else {
            root.right = insert(key, value, root.right);
        }
        return root;
    }
```

第二部分：平衡性的保证

这部分的保证完全由2-3-4树的性质推导得来，简单起见，这里仅考虑LeftLeaningRedBlackTree，即对于整个红黑树以及其中的每一个子树，不会存在左结点为Black，而右节点为Red的情况。

```java
    /**
     * 向红黑树中插入新的数据
     * @param key 键值对的key值，索引依赖项
     * @param value 键值对的value值
     * @param root 插入目标树的根节点
     * @return 插入后所生成树的根节点
     */
    public Node insert(K key, V value, Node root) {

        if (root == null) {
            return new Node(key, value, RED);
        }

        if (isRed(root.left) && isRed(root.right)) {
            colorFlip(root);
        }

        int cmp = root.key.compareTo(key);
        if (cmp == 0) {
            root.value = value;
        } else if (cmp > 0) {
            root.left = insert(key, value, root.left);
        } else {
            root.right = insert(key, value, root.right);
        }

        if (isRed(root.right)) {
            root = rotateLeft(root);
        }

        if (isRed(root.left) && isRed(root.left.left)) {
            root = rotateRight(root);
        }

        return root;
    }

    /**
     * 左旋当前节点与right child，旋转完成后仍然是红黑树的一部分
     * @param root 左旋目标节点
     * @return 左旋后新的root节点
     */
    private Node rotateLeft(Node root) {
        Node newRoot = root.right;
        root.right = newRoot.left;
        newRoot.left = root;

        newRoot.color = root.color;
        root.color = RED;

        return newRoot;
    }

    /**
     * 右旋当前节点
     * @param root 右旋的目标节点
     * @return 右旋后的新root节点
     */
    private Node rotateRight(Node root) {
        Node newRoot = root.left;
        root.left = newRoot.right;
        newRoot.right = root;

        newRoot.color = root.color;
        root.color = RED;
        return newRoot;
    }

    /**
     * 颜色切换，用于提升或下沉节点所在位置
     * @param root 提升/下沉目标节点
     */
    private void colorFlip(Node root) {
        root.color = !root.color;
        root.left.color = !root.left.color;
        root.right.color = !root.right.color;
    }

    /**
     * 判断当前节点是否为红色
     * @param node 目标节点
     * @return
     */
    private Boolean isRed(Node node) {
        return node != null && node.color.equals(RED);
    }
```
