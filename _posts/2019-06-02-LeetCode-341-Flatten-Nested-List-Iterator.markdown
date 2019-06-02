---
layout: post
title:  "Flatten Nested List Iterator"
date:   2019-06-02 21:22:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/341
description: 迭代器处理
---

## 分析
迭代器的作用是遍历整个数据结构，并含有next()与hasNext()两个典型方法。回到最基本的ArrayList上，迭代器最关键的作用是记录当前遍历的位置。调用next()方法时，返回游标所指向的值，hasNext()方法将返回游标后是否存在元素。

题目的目的是要我们实现一个迭代器，但是list中有两种数据类型：单元素与列表。当元素为列表时，我们需要保存当前cursor的位置，然后去遍历子列表的数据，这里引入栈来保存递归的上下文信息。如果迭代器指向单元素时，按照一般的迭代器方式进行处理；如果迭代器指向列表时，我们将当前迭代器压入栈中，采用子列表的迭代器设置为当前迭代器。

简单来说，我们构造了一个虚拟的游标来不断迭代next()与hasNext().通过栈来保存游标的递归调用关系。

处理hasNext()方法时遇到一个问题：如果顶层的iter.hasNext()方法返回为true，代表后面还有NestedInteger元素，但这个元素中是否含有Integer节点却无从得知。我们必须调用next()方法获取到真实的下一个节点后才能判断后续是否存在Integer节点，从而得知hasNext()应该如何返回。因此我所设计的迭代器在初始化完成/next()调用后会将下一次next()方法要返回的值放在缓存中，这样就能通过cache == null判断后续节点是否存在整数节点了。

## 代码
``` java
public class NestedIterator implements Iterator<Integer> {
    
    public Stack<Iterator<NestedInteger>> stack = new Stack<>();
    
    public Iterator<NestedInteger> iter;
    
    public Integer nextVal;

    public NestedIterator(List<NestedInteger> nestedList) {
        iter = nestedList.iterator();
        moveToNext();
    }

    @Override
    public Integer next() {
        if (nextVal == null) {
            throw new NoSuchElementException();
        }
        Integer result = nextVal;
        moveToNext();
        return result;
    }

    @Override
    public boolean hasNext() {
        return nextVal != null;
    }
    
    //move cursor to the next Integer and cache the next val
    public void moveToNext() {
        while (iter.hasNext() || !stack.isEmpty()) {
            if (iter.hasNext()) {
                NestedInteger node = iter.next();
                if (node.isInteger()) {
                    nextVal = node.getInteger();
                    return;
                } else {
                    stack.push(iter);
                    iter = node.getList().iterator();
                }
            } else {
                iter = stack.pop();
            }
        }
        nextVal = null;
    }
}
```

## 结果
Runtime: 2 ms, faster than 100.00% of Java online submissions for Flatten Nested List Iterator.
Memory Usage: 36.6 MB, less than 99.86% of Java online submissions for Flatten Nested List Iterator.