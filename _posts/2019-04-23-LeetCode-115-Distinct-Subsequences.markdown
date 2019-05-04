---
layout: post
title:  "Distinct Subsequences"
date:   2019-04-23 22:14:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/115
description: 通用的动态规划场景
---

## 题目

> Given a string S and a string T, count the number of distinct subsequences of S which equals T.
> A subsequence of a string is a new string which is formed from the original string by deleting some (can be none) of the characters without disturbing the relative positions of the remaining characters. (ie, "ACE" is a subsequence of "ABCDE" while "AEC" is not).

> Example 1:
Input: S = "rabbbit", T = "rabbit"
Output: 3
Explanation:

> As shown below, there are 3 ways you can generate "rabbit" from S.
(The caret symbol ^ means the chosen letters)

> rabbbit
^^^^ ^^
rabbbit
^^ ^^^^
rabbbit
^^^ ^^^


> Example 2:
Input: S = "babgbag", T = "bag"
Output: 5
Explanation:

> As shown below, there are 5 ways you can generate "bag" from S.
(The caret symbol ^ means the chosen letters)

> babgbag
^^ ^
babgbag
^^    ^
babgbag
^    ^^
babgbag
  ^  ^^
babgbag
    ^^^
    
## 分析
分析该题，意思是要求一个String s有多少种方式可变为另一个String t。仔细分析不难看出这是一个动态规划的问题，假设f(s, t)得到的结果就是s转换到t的方法数，因此当s[0] == t[0]时，f(s, t) = f(s.sub(1), t) + f(s.sub(1), t.sub(1))。其中s.sub(1)指s去掉字符s[0]的子串，这里的两个数分别代表了选择s[0]与不选择两种情况。

当s[0] != t[0]时很简单，f(s, t) = f(s.sub(1), t)，直到s.length < t为止。

递归过程中会产生许多相同的f(s, t)，因此使用一个Map作为cache将这些中间结果暂存起来。

## 代码

``` java
class Solution {
    
    private Map<Pair, Integer> cache = new HashMap<>();
    
    public int numDistinct(String s, String t) {
        Set<Character> set = new HashSet<>();
        for (char ch : t.toCharArray()) {
            set.add(ch);
        }
         
        StringBuffer sb = new StringBuffer();
        for (char ch : s.toCharArray()) {
            if (set.contains(ch)) {
                sb.append(ch);
            }
        }
        
        s = sb.toString();
        
        return caculate(s, t);
    }
    
    private Integer caculate(String s, String t) {
        Pair pair = new Pair(s, t);
        Integer count = cache.get(pair);
        if (count != null) {
            return count;
        }
        
        if (s.length() < t.length()) {
            return 0;
        }
        
        if (t.length() == 0) {
            return 1;
        }  
        
        Integer result = null;
        if (s.charAt(0) != t.charAt(0)) {
            result = caculate(s.substring(1), t);
        } else {
            result = caculate(s.substring(1), t) + caculate(s.substring(1), t.substring(1));
        }
        cache.put(pair, result);
        return result;
    }
    
    class Pair {
        String s;
        String t;
        public int hashCode() {
            return s.hashCode() + t.hashCode();
        }
        public boolean equals(Object obj) {
            if (!Pair.class.isInstance(obj)) {
                return false;
            }
            Pair other = (Pair) obj;
            return (s.equals(other.s) && t.equals(other.t));
        }
        
        public Pair(String s, String t) {
            this.s = s;
            this.t = t;
        }
    }
}
```