---
layout: post
title:  "Reverse Only Letters"
date:   2019-06-07 15：51:22 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/917
description: 反转String
---

## 题目

> Given a string S, return the "reversed" string where all characters that are not a letter stay in the same place, and all letters reverse their positions.

> Example 1:

> Input: "ab-cd"
Output: "dc-ba"
Example 2:

> Input: "a-bC-dEf-ghIj"
Output: "j-Ih-gfE-dCba"
Example 3:

> Input: "Test1ng-Leet=code-Q!"
Output: "Qedo1ct-eeLg=ntse-T!"
 

> Note:

> S.length <= 100
33 <= S[i].ASCIIcode <= 122 
S doesn't contain \ or "

## 分析

利用栈来进行反转，算法很简单。唯一需要注意的是：利用字符的ASCII在33与122之间的特点，将所有字符位置记录下来。

## 代码

``` java
class Solution {
    public String reverseOnlyLetters(String S) {
        
        StringBuilder sb = new StringBuilder();
        
        Stack<Character> letterStack = new Stack<>();
        
        char[] array = S.toCharArray();
        for (int i = 0; i < array.length; i++) {
            if (isLetter(array[i])) {
                letterStack.push(array[i]);
                array[i] = 0;
            }
        }
        
        for(int i = 0; i < array.length; i++) {
            if (array[i] != 0) {
                continue;
            } else {
                array[i] = letterStack.pop();
            }
        }
        return new String(array);
    }
    
    private boolean isLetter(char ch) {
        return (ch >= 'a' && ch <= 'z') || (ch >= 'A' && ch <= 'Z');
    }
    
}
```

## 结果

Runtime: 1 ms, faster than 71.12% of Java online submissions for Reverse Only Letters.
Memory Usage: 34.4 MB, less than 100.00% of Java online submissions for Reverse Only Letters.