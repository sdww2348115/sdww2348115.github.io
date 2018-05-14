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