---
layout: post
title:  "ExecuteModel"
date:   2018-05-15 20:15:01 +0800
categories: jvm
permalink: /jvm/executeModel
---

执行引擎是Java虚拟机最核心的组成部分之一。相对于物理机来说，它不依赖于处理器、指令集、硬件和操作系统。它的功能为输入字节码文件，处理过程就是字节码解析的等效过程，输出的即为执行结果。

## 运行时栈帧结构
栈帧是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时虚拟机栈中的栈元素。栈帧存储了方法的局部变量表、操作数栈、动态链接以及方法返回地址等信息。每一个方法从调用开始至执行完成的过程，对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

![stack-frame](../resources/img/stack-frame.png)

### 局部变量表
局部变量表(Local Variable Table)是一组变量值存储空间，用于存放方法参数和方法内部定义的局部变量。其最大容量在编译时就已经放入class文件中。

局部变量表的容量以变量槽(Variable Slot)为最小单位，并通过索引的方式进行定位，索引的范围从0开始到局部变量表的最大Slot数量为止。其中，局部变量表中第0位索引默认用于传递方法所属的对象实例引用，方法中可以直接通过`this`关键字访问到这个隐含参数。

在方法体的内部，`变量`就代表着一个Slot，当字节码PC计数器的值超过某个变量的作用域后，该变量所使用的Slot可以被其他变量使用。对于某些情况下的gc，这一特性非常重要。
```java
//局部变量表Slot复用与垃圾收集
public static void main(String[] args) {
    byte[] placeholder = new byte[64 * 1024 * 1024];
    System.gc();
}
```

运行上面的代码，当程序运行到System.gc()时，placeholder的生命周期已经完成，此时应该被GC掉，但System.gc()之前没有任何对局部变量表的读写操作，placeholder所占用的Slot未被其他变量所抢去，其值仍然指向byte[]数组，所以该byte[]无法被及时gc。添加一个placeholder = null，将该Slot的值强行设为null，栈帧与byte[]的可达性路径被切断，此时的byte[]就可以被正确的回收掉了。

```java
//局部变量表Slot复用与垃圾收集（优化版）
public static void main(String[] args) {
    byte[] placeholder = new byte[64 * 1024 * 1024];
    placeholder = null; //help gc
    System.gc();
}
```

当使用完一个大对象之后，后面的代码含有一些耗时很长的操作时，我们可以尝试使用XXX = null的方式帮助gc。但是如果没有遇到性能问题，请不要随意使用，它会让代码看起来很丑陋，而且不友好。

局部变量表与类变量不同，它不存在"准备阶段"，也就是说，它不会被赋初始值。所以，在我们在方法中使用一个未初始化的局部变量时，无论编译器或JVM都将会抛出错误，无法执行下去。
```java
public static void main(String[] args) {
    int a;
    System.out.println(a);
}
```
