---
layout: post
title:  "ClassLoader"
date:   2018-05-14 23:13:01 +0800
categories: jvm
permalink: /jvm/classLoader
---

Jvm中的类加载器是指实现`通过一个类的全限定名来获取描述此类的二进制字节流`功能的代码模块。

## 类与类加载器
对于任意一个类，都需要由加载它的类加载器和这个类本身一同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类命名空间。通俗来说，只有由同一个类加载器所加载的类才会"相等"，不同类加载器所加载的类，即使全限定名一样，其本质仍然是不相等的。注：这里的相等包含equals()方法、isAssignableFrom()方法、isInstance()方法、以及instanceof关键字等。
```java
public class ClassLoaderEqualsTest {
    public static void main(String[] args) throws Exception{
        ClassLoader myClassLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name, b, 0, b.length);
                } catch (Exception e) {
                    throw new RuntimeException(e);
                }
            }
        };
        Object object = myClassLoader.loadClass("jvm.BigClazz").newInstance();
        System.out.println(object.getClass());
        System.out.println(object instanceof BigClazz);
        System.out.println(object.getClass().getClassLoader());
        System.out.println(BigClazz.class.getClassLoader());
    }
}
```
程序运行结果为：
```
class jvm.BigClazz
false
jvm.ClassLoderEqualsTest$1@1540e19d
sun.misc.Launcher$AppClassLoader@18b4aac2
```
可以看出，两个jvm.BigClazz类并不相等，因为它们的类加载器并不是同一个。

## 双亲委派模型
从Java虚拟机的角度来说，类加载器分为两类：
* Bootstrap ClassLoader:由C++实现，属于虚拟机的一部分，用于加载Java基本的类。
* 其他ClassLoader:由Java实现，独立于虚拟机外部，全都继承于抽象类java.lang.ClassLoader,用于加载其他的Java类。

