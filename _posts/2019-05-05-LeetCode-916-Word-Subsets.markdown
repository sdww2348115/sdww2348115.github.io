---
layout: post
title:  "Word Subsets"
date:   2019-05-05 22:53:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/916
description: leetcode第916题，采用一种合并转化的思路进行解答
---

## 题目
> We are given two arrays A and B of words.  Each word is a string of lowercase letters.

> Now, say that word b is a subset of word a if every letter in b occurs in a, including multiplicity.  For example, "wrr" is a subset of "warrior", but is not a subset of "world".

> Now say a word a from A is universal if for every b in B, b is a subset of a. 

> Return a list of all universal words in A.  You can return the words in any order.

> Example 1:

> Input: A = ["amazon","apple","facebook","google","leetcode"], B = ["e","o"]
Output: ["facebook","google","leetcode"]

> Example 2:

> Input: A = ["amazon","apple","facebook","google","leetcode"], B = ["l","e"]
Output: ["apple","google","leetcode"]
Example 3:

> Input: A = ["amazon","apple","facebook","google","leetcode"], B = ["e","oo"]
Output: ["facebook","google"]
Example 4:

> Input: A = ["amazon","apple","facebook","google","leetcode"], B = ["lo","eo"]
Output: ["google","leetcode"]
Example 5:

> Input: A = ["amazon","apple","facebook","google","leetcode"], B = ["ec","oc","ceo"]
Output: ["facebook","leetcode"]
 

> Note:
1 <= A.length, B.length <= 10000
1 <= A[i].length, B[i].length <= 10
A[i] and B[i] consist only of lowercase letters.
All words in A[i] are unique: there isn't i != j with A[i] == A[j].

## 分析
题目很简单，定义sub(A, B)为B中的字符能够完全在A中找到。也就是说：对于任意字符b属于B，num(A, b) > num(B, b)。
num(X, n)代表X字符串中含有n字符的个数。
给定两个字符串集合，要找到所有满足条件的a：1. a属于A；2. 对于任意b属于B，都有sub(a, b)为true。
算法的时间复杂度为O((NB*b + a) * NA)即O(NB*NA*b),其中NB表示N集合中字符串个数，b代表平均每个字符串中字符的个数。空间复杂度为O(a/b)。
仔细观察可以发现：不一定需要每次都计算字符串A与字符串集合中每一个字符串的结果，我们可以将字符串集合B中所有的结果组合起来，取每个字符的最大值，只需要一个中间结果就能满足条件。时间复杂度为O(NB*b + NA*a),即O(NA*a)。空间复杂度仍然为O(1)

## 答案
```java
class Solution {
    public List<String> wordSubsets(String[] A, String[] B) {
        int[] totalB = new int[26];
        for (String b : B) {
            int[] wordCount = convertWord(b);
            for (int i = 0; i < 26; i++) {
                if (wordCount[i] > totalB[i]) {
                    totalB[i]  = wordCount[i];
                }
            }
        }
        
        List<String> result = new LinkedList<>();
        for (String a : A) {
            int[] wordCount = convertWord(a);
            if (sub(wordCount, totalB)) {
                result.add(a);
            }
        }
        return result;
    }
    
    private boolean sub(int[] wordA, int[] wordB) {
        for (int i = 0; i < wordB.length; i++) {
            if (wordA[i] < wordB[i]) {
                return false;
            }
        }
        return true;
    }
    
    private int[] convertWord(String string) {
        int[] count = new int[26];
        for (char ch : string.toCharArray()) {
            int index = ch - 'a';
            count[index] += 1;
        }
        return count;
    }
}
```

## 结果
Runtime: 17 ms, faster than 92.96% of Java online submissions for Word Subsets.
Memory Usage: 49.9 MB, less than 73.77% of Java online submissions for Word Subsets.