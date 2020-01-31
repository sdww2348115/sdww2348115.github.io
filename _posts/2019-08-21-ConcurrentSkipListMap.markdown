---
layout: post
title:  "ConcurrentSkipListMap简单剖析"
date:   2019-08-21 23:37:00 +0800
categories: java concurrent datastructure map
permalink: /concurrent/ConcurrentSkipListMap
description: ConcurrentSkipListMap简单剖析
---

## 背景
类比于ConcurrentHashMap与HashMap，为了解决TreeMap在高并发情况下的线程安全问题，JDK1.8引入了新的一个新的类：`ConcurrentSkipListMap`，该类实现了`ConcurrentMap`、`SortedMap`以及`NavigableMap`所有方法，所有节点有序排列，支持多个线程进行并发读写，其查找的时间复杂度为O(lgN),插入\删除的时间复杂度同样为O(lgN)。

## 基本原理
ConcurrentSkipList的核心数据结构为SkipList（跳表），本质上所有的数据以链表的形式进行有序存放，并利用多层索引链表实现对数据的快速查找。整体如下图所示：

![skip-list](../resources/img/skiplist.png)

SkipList由3种节点组成：

* 数据节点Node，存放Key，Value数据。所有的Node节点位于最底层，以链表的方式有序排列。
* 普通Index节点：拥有指向数据节点的指针，下一层Index节点的指针以及后续Index节点的指针。从图中可清晰的看出来：1. 除level为1的Index节点外，所有Index节点的down指针一定不为null，即Index节点不会存在悬空的情况。2. 每一层index节点同样是有序递增排列的。
* Index头节点：header节点同样拥有right与down两种指针，但header节点不指向任何数据节点，并且记录了当前的level。

## 查询

ConcurrentSkipList的查询过程主要分为两个部分：

### 根据Index定位到最大的小于目标Key值的最底层Index Node

整个定位过程与查表类似，主要逻辑位于方法findPredecessor中，
```java
private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
        if (key == null)
            throw new NullPointerException(); // don't postpone errors
        for (;;) {
            for (Index<K,V> q = head, r = q.right, d;;) {
                if (r != null) {
                    Node<K,V> n = r.node;
                    K k = n.key;
                    if (n.value == null) {
                        if (!q.unlink(r))
                            break;           // restart
                        r = q.right;         // reread r
                        continue;
                    }
                    if (cpr(cmp, key, k) > 0) {
                        q = r;
                        r = r.right;
                        continue;
                    }
                }
                if ((d = q.down) == null)
                    return q.node;
                q = d;
                r = d.right;
            }
        }
    }
```

1. 从最顶层level的head结点开始，current保存的当前Index结点引用，right保存的是当前结点的右结点引用
2. if (target > current.node.key && right != null),则current = right, rught = right.right
3. if (current.down != null && right == null),则指针下移，分别指向下一层的当前结点以及右侧结点
4. 反复2，3步骤，直到找到current.down == null的结点，返回其Node值

### 在底层链表中按照条件进行进一步查找

进一步查找的过程主要是处理各种并发性问题，具体情况请参考下列代码块。lower、higher key的查找与等于相类似，这里就不一一列举了。

```java
private V doGet(Object key) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                // 整个跳表中没有任何数据，或者target比跳表中所有数据都小
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                // 此时有另一个线程刚于此处执行了插入操作，需要重新进行定位
                if (n != b.next)                // inconsistent read
                    break;
                // n结点刚刚被移除，需要重新进行定位
                if ((v = n.value) == null) {    // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                // b结点刚被移除，需要重新进行定位
                if (b.value == null || v == n)  // b is deleted
                    break;
                // 判断数据node的值与目标值是否相等，即是否存在目标target
                if ((c = cpr(cmp, key, n.key)) == 0) {
                    @SuppressWarnings("unchecked") V vv = (V)v;
                    return vv;
                }
                // 目标值不存在
                if (c < 0)
                    break outer;
                b = n;
                n = f;
            }
        }
        return null;
    }
```

## 插入

ConcurrentSkipListMap的插入过程主要分为3个步骤：1.通过查询定位目标位置（新建数据Node）；2.计算Index level；3.插入Index结点

在插入数据结点的过程中，仅涉及到CAS操作，未使用任何lock结构，只要目标Node以及之前的两个Node未做修改即可完成整个过程。

```java
private V doPut(K key, V value, boolean onlyIfAbsent) {
        Node<K,V> z;             // added node
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        //插入数据结点
        outer: for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                //如果n == null，则说明target比ConcurrentSkipListMap中所有结点都要大
                if (n != null) {
                    Object v; int c;
                    Node<K,V> f = n.next;
                    //同上，此时有另外的线程正在执行插入操作
                    if (n != b.next)               // inconsistent read
                        break;
                    //同上，此时有另外的线程正在执行删除
                    if ((v = n.value) == null) {   // n is deleted
                        n.helpDelete(b, f);
                        break;
                    }
                    //同上，此时另外的线程正在执行b的删除操作
                    if (b.value == null || v == n) // b is deleted
                        break;
                    //由于并发原因，target比n.key更大，此时需要向后继续查找
                    if ((c = cpr(cmp, key, n.key)) > 0) {
                        b = n;
                        n = f;
                        continue;
                    }
                    //找到目标node，执行替换操作
                    if (c == 0) {
                        if (onlyIfAbsent || n.casValue(v, value)) {
                            @SuppressWarnings("unchecked") V vv = (V)v;
                            return vv;
                        }
                        break; // restart if lost race to replace value
                    }
                    // else c < 0; fall through
                }
                
                //在对应位置执行插入操作
                z = new Node<K,V>(key, value, n);
                if (!b.casNext(n, z))
                    break;         // restart if lost race to append to b
                break outer;
            }
        }

        /* 通过一个随机int值来决定当前插入node的Index层数，只有第2位与末位为0的数时才会生成Index结点，因此每个结点生成Index的概率为1/4，每一层index上浮的概率为1/2。*/
        int rnd = ThreadLocalRandom.nextSecondarySeed();
        if ((rnd & 0x80000001) == 0) { // test highest and lowest bits
            int level = 1, max;
            while (((rnd >>>= 1) & 1) != 0)
                ++level;
            Index<K,V> idx = null;
            HeadIndex<K,V> h = head;
            // ConcurrentHashMap中原有Index的层数大于当前数据结点index层数，从下至上依次构建Index结点
            if (level <= (max = h.level)) {
                for (int i = 1; i <= level; ++i)
                    idx = new Index<K,V>(z, idx, null);
            }
            // 新结点的level大于原有ConcurrentSkipListMap的level，原有的level需要 + 1
            else { // try to grow by one level
                level = max + 1; // hold in array and later pick the one to use
                @SuppressWarnings("unchecked")Index<K,V>[] idxs =
                    (Index<K,V>[])new Index<?,?>[level+1];
                //从下至上构建基于插入node的Index数组
                for (int i = 1; i <= level; ++i)
                    idxs[i] = idx = new Index<K,V>(z, idx, null);
                for (;;) {
                    h = head;
                    int oldLevel = h.level;
                    if (level <= oldLevel) // lost race to add level
                        break;
                    HeadIndex<K,V> newh = h;
                    Node<K,V> oldbase = h.node;
                    //插入node的Index数组可能比原来head的level高了不止1层，需要依次进行添加
                    for (int j = oldLevel+1; j <= level; ++j)
                        newh = new HeadIndex<K,V>(oldbase, newh, idxs[j], j);
                    if (casHead(h, newh)) {
                        h = newh;
                        idx = idxs[level = oldLevel];
                        break;
                    }
                }
            }
            // find insertion points and splice in
            // 按层次依次插入新构建的Index Node
            splice: for (int insertionLevel = level;;) {
                int j = h.level;
                for (Index<K,V> q = h, r = q.right, t = idx;;) {
                    if (q == null || t == null)
                        break splice;
                    //从左至右找到Index可插入的目标地址
                    if (r != null) {
                        Node<K,V> n = r.node;
                        // compare before deletion check avoids needing recheck
                        int c = cpr(cmp, key, n.key);
                        //这里再次进行check，防止后续节点n已经被删除
                        if (n.value == null) {
                            if (!q.unlink(r))
                                break;
                            r = q.right;
                            continue;
                        }
                        //向后继续查找
                        if (c > 0) {
                            q = r;
                            r = r.right;
                            continue;
                        }
                    }

                    //每次仅对一个level的Index链表进行插入，这里检查的就是当前level是否与目标level相等
                    if (j == insertionLevel) {
                        if (!q.link(r, t))
                            break; // restart
                        if (t.node.value == null) {
                            findNode(key);
                            break splice;
                        }
                        if (--insertionLevel == 0)
                            break splice;
                    }

                    //insertionLevel小于j，则level下移，处理下一层的插入操作
                    if (--j >= insertionLevel && j < level)
                        t = t.down;
                    q = q.down;
                    r = q.right;
                }
            }
        }
        return null;
    }
```

## 删除

ConcurrentSkipList的删除操作最核心的仍然是查找目标node，随后检查并发问题并进行删除。

```java
final V doRemove(Object key, Object value) {
        if (key == null)
            throw new NullPointerException();
        Comparator<? super K> cmp = comparator;
        outer: for (;;) {
            for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
                Object v; int c;
                // 有其他线程正在执行删除操作，重新进行处理
                if (n == null)
                    break outer;
                Node<K,V> f = n.next;
                // 有其他线程正在执行插入操作，重新进行处理
                if (n != b.next)                    // inconsistent read
                    break;
                if ((v = n.value) == null) {        // n is deleted
                    n.helpDelete(b, f);
                    break;
                }
                if (b.value == null || v == n)      // b is deleted
                    break;
                if ((c = cpr(cmp, key, n.key)) < 0)
                    break outer;
                // 此时有另外的插入节点操作已经成功，需要b，n节点均往后移动一位
                if (c > 0) {
                    b = n;
                    n = f;
                    continue;
                }
                //判断value是否满足删除的需求
                if (value != null && !value.equals(v))
                    break outer;
                if (!n.casValue(v, null))
                    break;
                if (!n.appendMarker(f) || !b.casNext(n, f))
                    findNode(key);                  // retry via findNode
                else {
                    findPredecessor(key, cmp);      // clean index
                    if (head.right == null)
                        //请注意这里，见代码块后续部分
                        tryReduceLevel();
                }
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
        }
        return null;
    }
    
    /**
      删除数据节点需要同时至删除所有指向该数据节点的Index节点，如果恰好该Index节点又属于某些高层level唯一的节点，则需要将这些level的Index删掉。即删除最顶上n层的Index节点。
      如果有其他线程正在同时进行节点的插入操作，且插入操作同时触发了level增加时，该方法可能会导致错误产生。（即丢失较为顶层的部分Index节点）
      为了预防错误，重置level的只会对3层以上的ConcurrentSkipListMap进行，且必须连续三层header的next均为null时才会进行。在使用cas重置header节点后还会再次进行检查，如果被替换的header.next不为null时，将会滚原header指向的对象。
      仍然会出错的场景：假设此时ConcurrentSkipList的level为5，有另一个线程正在执行插入操作，并且会将该SkipList的level拉满直到6，当前线程需要将level降低到4，即使回滚依然会将level回滚为5，新插入节点的顶层index就丢失了。
    **/
    private void tryReduceLevel() {
        HeadIndex<K,V> h = head;
        HeadIndex<K,V> d;
        HeadIndex<K,V> e;
        if (h.level > 3 &&
            (d = (HeadIndex<K,V>)h.down) != null &&
            (e = (HeadIndex<K,V>)d.down) != null &&
            e.right == null &&
            d.right == null &&
            h.right == null &&
            casHead(h, d) && // try to set
            h.right != null) // recheck
            casHead(d, h);   // try to backout
    }
```

## 总结

1. ConcurrentSkipListMap可认为是TreeMap的高并发版本
2. ConcurrentSkipListMap在高并发情况下的插入/删除仅能保证数据节点的插入一定正确，但不保证Index节点一定正确，但是能够保证数据一定能够被检索到
