---
layout: post
title:  "String"
date:   2019-05-28 23:00:00 +0800
categories: JDK String format 杂谈
permalink: /JDK/StringFormat
description: String.format()方法浅谈
---

## 问题

今天在开发的过程中遇到一个很2的问题：使用String.format()对一个字符串进行处理时，程序报错如下：
``` 
Exception in thread "main" java.util.UnknownFormatConversionException: Conversion = '''
	at java.util.Formatter.checkText(Formatter.java:2579)
	at java.util.Formatter.parse(Formatter.java:2555)
	at java.util.Formatter.format(Formatter.java:2501)
	at java.util.Formatter.format(Formatter.java:2455)
	at java.lang.String.format(String.java:2940)
	at com.sdww8591.tools.tmp.FormatTest.main(FormatTest.java:13)
```
其中字符串为:"SET PASSWORD FOR 'admin'@'%' = PASSWORD('%s');",一个设置Mysql用户密码的标准String模板。

设置断点进入Formatter.java:2579，发现此时cursor位于字符数组的26位置处，指向@字符后面的那个%字符。异常抛出的根本原因是程序遇到%字符后，发现后面并未接数字、-+、tsd等规范化字符，所以程序直接抛出了UnknownFormatConversionException。

代码如下：
``` java
private static void checkText(String s, int start, int end) {
    for (int i = start; i < end; i++) {
        // Any '%' found in the region starts an invalid format specifier.
        // 对于任何单独出现的%或者位于字符串末尾的%都会抛出UnknownFormatConversionException
        if (s.charAt(i) == '%') {
            char c = (i == end - 1) ? '%' : s.charAt(i + 1);
            throw new UnknownFormatConversionException(String.valueOf(c));
        }
    }
}
```

外层代码：
``` java
    Matcher m = fsPattern.matcher(s);
    for (int i = 0, len = s.length(); i < len; ) {
        if (m.find(i)) {
            // Anything between the start of the string and the beginning
            // of the format specifier is either fixed text or contains
            // an invalid format string.
            if (m.start() != i) {
                // Make sure we didn't miss any invalid format specifiers
                checkText(s, i, m.start());
                // Assume previous characters were fixed text
                al.add(new FixedString(s.substring(i, m.start())));
            }

            al.add(new FormatSpecifier(m));
            i = m.end();
        } else {
        、、、
        }
    }
```
这里仅截取了源代码的一部分，其中fsPattern的正则表达式为：
> "%(\\d+\\$)?([-#+ 0,(\\<]*)?(\\d+)?(\\.\\d+)?([tT])?([a-zA-Z%])"

包含了所有的格式化表达式，因此上面代码的逻辑为：
1. 找到所有匹配格式化表达式的关键字符串组合，将字符串切分为数段
2. 对于每一个不包含格式化表达式的字符段，采用checkText方法验证其是否含有单独的%,如果含有，则抛出UnknownFormatConversionException异常。

当需要format的字符串中含有%字符时，可通过字符%%代替

## String.format方法浅析
既然走到这里了，不妨看看String.format()方法的具体实现。从核心上来说，String.format()方法主要分为两个步骤：

1. parse步骤：将需要格式化的字符串进行解析，生成待Format的字符数组。
2. print步骤：将第一步生成的字符数组与参数列表组合起来，生成准确结果。

### Parse
核心代码位于Formatter.parse()中，JDK核心代码采用了正则表达式的方式，将所有支持的格式化表达式写入到Pattern中，使用该Pattern对字符串进行处理，将字符串m模板切分为数段。例如：

|-------------------------||--------------||--|
SET PASSWORD FOR 'admin'@'%%' = PASSWORD('%s');

上面的字符串将被切分为5段：
SET PASSWORD FOR 'admin'@'
%%
' = PASSWORD('
%s
');

从中我们可以看出，每个字符串段可分为两种类型：准确字符串与待格式化字符串,代码中对应的类型为FixedString与FormatSpecifier。

### Print
核心代码位于Formatter.format()中。每个Formatter实例都含有一个StringBuilder用于拼接字符串，输出结果。在该方法中，代码逻辑将把上面生成的字符串段数组与参数数组组合起来，使用字符串替换的方式将最终结果一点一点组装起来。

具体的代码这里就不细讲了，大家可以去Formatter.format()方法中自行查看