---
layout: post
title:  "K Closest Points to Origin"
date:   2019-04-19 22:55:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/973
---

## 题目
> We have a list of points on the plane.  Find the K closest points to the origin (0, 0).

> (Here, the distance between two points on a plane is the Euclidean distance.)

> You may return the answer in any order.  The answer is guaranteed to be unique (except for the order that it is in.)

> Example 1:
Input: points = [[1,3],[-2,2]], K = 1
Output: [[-2,2]]
Explanation: 
The distance between (1, 3) and the origin is sqrt(10).
The distance between (-2, 2) and the origin is sqrt(8).
Since sqrt(8) < sqrt(10), (-2, 2) is closer to the origin.
We only want the closest K = 1 points from the origin, so the answer is just [[-2,2]].

> Example 2:
Input: points = [[3,3],[5,-1],[-2,4]], K = 2
Output: [[3,3],[-2,4]]
(The answer [[-2,4],[3,3]] would also be accepted.)

> Note:
1. 1 <= K <= points.length <= 10000
2. -10000 < points[i][0] < 10000
3. -10000 < points[i][1] < 10000

## 分析
每个point的绝对值小于10000，因此平方后得到的值肯定小于100000000，两数相加小于200000000，使用int进行平方/开方/加减完全足够。

这个题是一个标注的维护前N个最小值的数据结构，如果我们把所有的points插入后，进行一次排序即可得到最终答案，按照快排计算，时间开销为O(NlgN),额外的空间开销为O(1).

由于只用比较最后的值的大小，因此实际上比较耗时的开方运算没有执行的必要；我们可以只保存前K小值的集合，比K中最大值还大的组合没有计算和排序的必要。因此，构建一个空间为K的大根堆，每次需要加入新的数时首先与大根堆中的最大值进行比较，如果大于大根堆中最大值，则去掉大根堆最大值，并将该值填入。时间复杂度可优化为O(NlgK).

## 代码
``` java
class Solution {
    public int[][] kClosest(int[][] points, int K) {
        if (points.length == 0 || K == 0) {
            return new int[0][2];
        }
        Node[] nodes = new Node[K];
        for (int i = 0; i < K; i++) {
            nodes[i] = new Node(points[i]);
        }
        MaxHeap maxHeap = new MaxHeap(nodes);

        for (int i = K; i < points.length; i++) {
            Node node = new Node(points[i]);
            if (maxHeap.getMax().val > node.val) {
                maxHeap.replaceHead(node);
            }
        }

        int[][] result = new int[K][2];
        Node[] resultNodes = maxHeap.getEntities();
        for (int i = 0; i < K; i++) {
            result[i] = resultNodes[i].point;
        }
        return result;
    }

    class Node {
        public int[] point;
        public int val;
        public Node(int[] point) {
            this.point = point;
            val = point[0] * point[0] + point[1] * point[1];
        }
    }

    class MaxHeap {

        public Node[] entities;

        public MaxHeap(Node[] entities) {
            this.entities = entities;
            //initialize
            for (int i = entities.length - 1; i > 0; i--) {
                int indexParent = (i - 1) >> 1;
                if (entities[i].val > entities[indexParent].val) {
                    swap(i, indexParent);
                    adjust(i);
                }
            }
        }

        public Node[] getEntities() {
            return entities;
        }

        public Node getMax() {
            return entities[0];
        }

        public void replaceHead(Node node) {
            entities[0] = node;
            adjust(0);
        }

        public void adjust(int i) {
            int rightChildIndex = (i + 1) * 2;
            int leftChildIndex = rightChildIndex - 1;
            if (leftChildIndex >= entities.length) {
                return;
            } else if (rightChildIndex >=  entities.length) {
                if (entities[leftChildIndex].val > entities[i].val) {
                    swap(leftChildIndex, i);
                }
                return;
            } else {
                if (entities[leftChildIndex].val > entities[i].val
                        && entities[leftChildIndex].val > entities[rightChildIndex].val) {
                    swap(leftChildIndex, i);
                    adjust(leftChildIndex);
                    return;
                } else if (entities[rightChildIndex].val > entities[i].val
                        && entities[rightChildIndex].val > entities[leftChildIndex].val){
                    swap(rightChildIndex, i);
                    adjust(rightChildIndex);
                    return;
                }
            }
        }

        private void swap(int index1, int index2) {
            Node tmp = entities[index1];
            entities[index1] = entities[index2];
            entities[index2] = tmp;
        }
    }
}
```