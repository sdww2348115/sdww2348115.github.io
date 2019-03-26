---
layout: post
title:  "Range Sum Query 2D - Immutable"
date:   2019-03-26 22:07:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/304
---

## 题目
>Given a 2D matrix matrix, find the sum of the elements inside the rectangle defined by its upper left corner (row1, col1) and lower right corner (row2, col2).

>Example:
Given matrix = [
  [3, 0, 1, 4, 2],
  [5, 6, 3, 2, 1],
  [1, 2, 0, 1, 5],
  [4, 1, 0, 1, 7],
  [1, 0, 3, 0, 5]
]
sumRegion(2, 1, 4, 3) -> 8
sumRegion(1, 1, 2, 2) -> 11
sumRegion(1, 2, 2, 4) -> 12

>Note:
You may assume that the matrix does not change.
There are many calls to sumRegion function.
You may assume that row1 ≤ row2 and col1 ≤ col2.

## 分析
该题的目的是多次调用某一函数，求一个数字矩阵中某一子块的数字之和。如果不考虑时间复杂度，该算法实现非常简单，直接按照题目要求每次执行加法操作即可。而且题目中有准确提到：矩阵中的数据不会改变，而且该方法会多次调用，说明该问题主要考察的是在这种情况下如何通过缓存的方式加快后续计算的速度。

该题的难点也在于此：通常缓存只会记录方法的输入/输出值，在输入/输出值相同的情况下直接从缓存中取出结果进行返回，省略掉中间的计算开销。该题的目的肯定不止于此，因此我们还要考虑如何充分利用过程数据加速计算。

首先，我们把子区域块的求和分解为多个同一行内多个数字之和。以sumRegion(2, 1, 4, 3)为例，我们将其分解为：
sum(2, 1 -> 3) + sum (3, 1 -> 3) + sum(4, 1 -> 3)。在这种情况下，我们可以记录下每一行所作的每一次加法计算操作的值。

缓存的查询同样需要耗费CPU时间，因此其数据结构的设计也非常重要，最快的查询方式肯定是查表。我们设计一个3维数组：每一个值val(x,y,z)表示第x行从第y个值一直连加到第z个值之和。因此，我们查询的时间复杂度为O(1)。空间复杂度为O(n^3)，其中n为二维数组中值的个数。

## 答案
```java
class NumMatrix {
    
    public int[][][] cache;
    
    public int[][] matrix;

    public NumMatrix(int[][] matrix) {
        
        this.matrix = matrix;
        
        if (matrix.length > 0) {
            int rows = matrix.length;
            int cols = matrix[0].length;
            cache = new int[rows][cols][cols];
            for (int i = 0; i < rows; i++) {
                for (int j = 0; j < cols; j++) {
                    for (int k = j; k < cols; k++) {
                        cache[i][j][k] = -1;
                    }
                }
            }
        }
    }
    
    public int sumRegion(int row1, int col1, int row2, int col2) {
        if (matrix.length == 0) return 0;
        int sum = 0;
        for (int i = row1; i <= row2; i++) {
            sum += caculate(i, col1, col2);
        }
        return sum;
    }
    
    public final int caculate(int row, int colStart, int colEnd) {
        int cacheNum = cache[row][colStart][colEnd];
        if (cacheNum != -1) {
            return cacheNum;
        }
        
        int sum = 0;
        for (int i = colStart; i <= colEnd; i++) {
            cacheNum = cache[row][i][colEnd];
            if (cacheNum != -1) {
                sum += cacheNum;
                break;
            } else {
                sum += matrix[row][i];
                cache[row][colStart][i] = sum;
            }
        }
        return sum;
    }
}
```