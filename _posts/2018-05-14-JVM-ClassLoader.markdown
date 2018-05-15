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

从Java开发者的角度来看，类的加载器分为以下三类

* `启动类加载器`：就是上面的Bootstrap ClassLoader，负责将存放在<JAVA_HOME>/lib目录中，或者被-Xbootclasspath参数所指定路径中的，且能够被虚拟机所识别的（依照文件名）类库加载到JVM中。
* `扩展类加载器`(Extension ClassLoader)：该加载器负责加载<JAVA_HOME>/lib/ext目录中的，或者被`java.ext.dirs`系统变量所指定的路径中的所有类库。请注意：如果使用java.ext.dirs系统变量设定java程序加载类库，该java程序将不会加载<JAVA_HOME>/lib/ext目录中的jar包！譬如各种加密算法都无法使用。
* `应用程序类加载器`(Application ClassLoader)：该加载器负责加载ClassPath上所指定的类库。

上述三类加载器之间的层次模型被称为双亲委派模型：除了顶层的启动类加载器外，其余的类加载器都应该有自己的父类加载器，如下所示：

![ParentsDelegationModel.png](../resources/img/ParentsDelegationModel.png)

双亲委派模型的工作流程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委派给自己的父类加载器，如此递归，所有的加载请求都会被送到顶层的启动类加载器中，只有当父加载器无法完成加载任务时，子加载器才会自己加载。简单来说：可以保证每一个类都是从上至下依次进行加载的。

Java类随着类加载器一起具备了一种带有优先级的层级关系，这保证了Java类型的唯一性，对于保证Java程序运行的稳定性很重要。双亲委派模型的代码都在ClassLoader.loadClass()方法中，一般来说，如果我们要实现自己的类加载器，不要重载这个方法，而是重载类加载器中的findClass()方法。

```java
//双亲委派模型实现
protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                //首先尝试通过父类加载
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
                
                //父类无法加载的情况下才会在子类进行加载
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

但是在实际的使用过程中，位于上面的类加载器所加载的类可能会使用子加载器所加载的类，所以双亲委派模型并不是强制性约束模型。比如JNDI，其代码由启动类加载器所加载，但是实际实现类却是由应用类加载器所加载。OSGI中的类可能会平级调用其他类加载器等。

