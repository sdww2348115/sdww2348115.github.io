---
layout: post
title:  "Multiply Strings"
date:   2019-04-15 22:30:00 +0800
categories: leetcode ARTS algorithm
permalink: /leetcode/43
---

## 题目
> Given two non-negative integers num1 and num2 represented as strings, return the product of num1 and num2, also represented as a string.

>Example 1:
Input: num1 = "2", num2 = "3"
Output: "6"

>Example 2:
Input: num1 = "123", num2 = "456"
Output: "56088"

> Note:
The length of both num1 and num2 is < 110.
Both num1 and num2 contain only digits 0-9.
Both num1 and num2 do not contain any leading zero, except the number 0 itself.
You must not use any built-in BigInteger library or convert the inputs to integer directly.

## 分析
该题的目的是实现一个字符串乘法，计算两个字符串所代表的数字之积。
考虑我们平时数学里面的乘法计算方式，可将题目分解为3个简单的方法，组合起来即可得到结果

* 多位数 * 一位数
* 两个多位数之和
* 多位数 * 10

我们可以将其中一个多位数分解为个位、十位...等不同的一位数，分别计算它们与另一个数的乘积，再按照进位规则相加求和即是结果。
假设两个数的位数分别为m、n，那么将第二个数进行分解，总共要进行n次多位数乘一位数运算，最后相加次数也为n，每次乘法计算都涉及到m个数相乘，因此时间复杂度为O(mn).

## 答案
``` java
class Solution {
    public String multiply(String num1, String num2) {
        int[] num1Ia = strToInt(num1);
        int[] num2Ia = strToInt(num2);

        int[] result = new int[1];
        for (int i = num2Ia.length - 1; i >= 0; i--) {
            int[] val = multiply(num1Ia, num2Ia[i]);
            int offset = num2Ia.length - 1 - i;
            int[] newVal = new int[val.length + offset];
            System.arraycopy(val, 0, newVal, 0, val.length);
            result = add(newVal, result);
        }

        return intArrayToString(result);
    }



    private int[] multiply(int[] num1, int num2) {
        int[] result = new int[num1.length + 1];
        for (int i = num1.length - 1; i >= 0; i--) {
            int multi = num1[i] * num2 + result[i + 1];
            result[i + 1] = multi % 10;
            result[i] = multi / 10;
        }
        return cultZero(result);
    }

    private int[] add(int[] num1, int[] num2) {
        int length = Math.max(num1.length, num2.length);
        int[] result = new int[length + 1];
        for (int i = 0; i < length; i++) {
            int var1 = 0;
            if (num1.length - 1 - i >= 0) {
                var1 = num1[num1.length - 1 - i];
            }
            int var2 = 0;
            if (num2.length - 1 - i >= 0) {
                var2 = num2[num2.length - 1 - i];
            }
            int sum = var1 + var2 + result[length - i];
            result[length - i] = sum % 10;
            result[length - i - 1] = sum / 10;
        }
        return cultZero(result);
    }

    private int[] cultZero(int[] num) {
        int index = 0;
        while (index < num.length && num[index] == 0) {
            index++;
        }
        int newLength = num.length - index;
        if (newLength == 0) {
            return new int[] {0};
        }
        int[] result = new int[newLength];
        System.arraycopy(num, index, result, 0, newLength);
        return result;
    }

    private int[] strToInt(String num) {
        char[] chars = num.toCharArray();
        int[] array = new int[chars.length];
        for (int i = 0; i < chars.length; i++) {
            array[i] = Integer.valueOf(new String(new char[] {chars[i]}));
        }
        return array;
    }

    private String intArrayToString(int[] array) {
        char[] chars = new char[array.length];
        for (int i = 0; i < array.length; i++) {
            chars[i] = String.valueOf(array[i]).charAt(0);
        }
        return new String(chars);
    }
}
```
结果为81.93%与95%