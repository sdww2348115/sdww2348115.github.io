---
layout: post
title:  "IOC"
date:   2018-06-18 11:31:01 +0800
categories: spring
permalink: /spring/ioc
---

## 引言
由于其优越的特性，Spring Framework已经成为企业级Java应用事实上的标准。Spring提供了一系列开箱即用的模块，让构建企业级应用变得非常的简单、方便。

往小了说，在构建Web应用的时候，表现层有SpringMVC解决方案；持久化有JPA作为依托，同时还可以使用MyBatis/Hibernate等持久化框架；权限认证可采用Spring Security；缓存也有一系列解决方案......

对于构建一个企业级的大应用，Spring也针对性地推出了一系列产品：针对快速开发和容器化部署的SpringBoot；针对快速构建批处理的SpringBatch;大系统微服务框架的SpringCloud

上述的一切，都离不开Spring框架的核心：IOC与AOP。

## IOC概念
在没有IOC概念之前，当程序员需要一个对象的时候，往往通过直接创建实现。创建对象通常有以下两步：

 1. 使用构造函数得到实例化的类
 2. 执行对象的初始化工作

在实际的程序中，对象的创建过程往往是非常复杂的，每一个对象初始化都涉及到许多步骤，包括创建与之相关的类、将自己注册到某个组件中，执行某些初始化方法，设置类的初始化状态等等。这些代码片段往往分散于程序的每个角落中，难于管理与复用，因此，人们总结了一系列工厂设计模式作为对象创建的范式，它们共同的特点是：`将散乱的对象初始化逻辑集中起来放到工厂类中，由工厂类来执行对象的初始化工作`。

工厂类的出现一定程度上缓解了对象创建的复杂性，但是仍有一些问题无法得到解决：

 * 使用对象时，除了创建之外，程序往往还会有其他方式。比如获取单例对象。
 * 企业级应用逻辑繁复，对象既多又杂，无限制使用工厂模式将导致工厂类泛滥。
 * 企业级应用中，对象之间的逻辑关系紧密，往往一个对象将依赖于不同层次的多个其他特定对象。
 * 对象泛滥，往往创建一个业务逻辑对象需要创建数十甚至数百个底层对象。

总的来说，现有的设计模式无法满足于企业级应用的对象管理。因此，软件科学家们提出了一种新的设计模式IOC：Inversion of Control，即`将所有对象的创建与管理逻辑一并交予框架实现`。相比各种工厂模式IOC更进一步，将对象的管理功能也一并实现，在编写代码时， 用户只管从框架中拿出自己所需的对象即可。于此，`对象的创建管理与使用完全解耦`。

IOC设计模式所带来的优点：

 * 组件的使用与实现完全解耦，这是spring各种开箱即用组件方便性的基础。
 * 采用资源池的思想，使得许多对象可以复用
 * 由于IOC容器接管了所有对象的创建与管理，因此一些对于对象的批量修改功能有了实现的基础（AOP）

该章节主要是个人理解，这里有一篇Martin大佬关于IOC的文章：[Inversion of Control Containers and the Dependency Injection pattern](https://www.martinfowler.com/articles/injection.html)

## IOC容器
业务对象通过IOC声明相应的依赖，最终由IOC容器将这些相互依赖的对象绑定到一起。IOC容器的职责主要就是这两方面：业务对象的构建管理与业务对象间的依赖绑定。

 * `业务对象的构建管理`：在IOC场景中，业务对象无需关心所依赖的对象如何构建如何取得，但是这部分工作始终需要有人来做。IOC容器将对象的构建逻辑从逻辑代码中剥离出来，以免这部分逻辑污染业务对象的实现。
 * `业务对象见的依赖绑定`：对于IOC容器来说，这是其核心职责。IOC容器通过结合之前构建和管理的所有业务对象，以及各个业务对象间可识别的依赖关系，将这些对象所依赖的对象注入绑定，从而保证每个业务对象在使用的时候，可以处于就绪状态。

Spring提供了两种容器类型：`BeanFactory`和`ApplicationContext`。

* BeanFactory：基础类型IOC容器，提供完整的IOC服务支持。如果没有特殊指定，默认采用延迟初始化策略。只有当客户端对象需要访问容器中的某个受管对象时，才会对该受管对象进行初始化以及依赖注入操作。
* ApplicationContext。ApplicationContext在BeanFactory的基础上进行构建，是相对比较高级的容器实现，除了拥有BeanFactory的所有支持，ApplicationContext还提供了其他比较高级的特性，比如事件发布、国际化信息支持等。ApplicationContext所管理的对象，在该类型容器启动之后，默认全部初始化并绑定完成。

![ApplicationContext](../resources/img/ApplicationContext.png)

上图为ApplicationContext的继承结构，除了BeanFactory外，ApplicationContext还继承了以下接口：

* MessageSource:用于处理参数化或国际化的消息。
* ResourceLoader：用于定位resource资源地址
* ApplicationEventPublisher：用于实现Spring内部事件触发机制
* EnvironmentCapable：ApplicationContext可以根据该接口判断自己的BeanFactory具体使用哪些方法，用于区分各种环境。

## Spring中的IOC容器

![SpringIOC](../resources/img/SpringIOC.jpg)

上图为Spring中IOC容器的主要组件，它们配合起来共同完成IOC的功能。

 * BeanDefinitionReader 负责从配置文件、代码等处获取Bean的描述信息，并将这些信息组织为标准的可使用的BeanDefinition
 * BeanDefinitionRegistry 从BeanDefinitionReader处获取BeanDefinition，并按照规则将它们组织成Bean，放入自身容器中保管起来
 * BeanFactory 向外部模块提供获取Bean的接口 

### BeanFactory
BeanFactory是整个SpringIOC技术的基石。

BeanFactory中包含许多Bean的定义，这些Bean以String类型的name作为唯一标识符。与ListableBeanFactory中的接口不同，当BeanFactory的实现属于HierarchicalBeanFactory时，如果当前BeanFactory中找不到目标Bean的定义，该Factory将会查找当前BeanFactory的直接Parent。如果出现Bean同名的情况，当前BeanFactory中的Bean将会覆盖Parent Factory中的Bean。

#### ListableBeanFactory
ListableBeanFactory接口继承自BeanFactory，它在BeanFactory所提供的的获取Bean接口的基础上，提供了一大批获取BeanDefination的接口，比如根据Type获取Bean的名字；根据注解获取Bean的名字；判断Bean是否属于该BeanFactory等。使用该接口中的方法需要注意两点：

1. 接口中的方法只会返回当前BeanFactory的结果，即使BeanFactory实现了HierarchicalBeanFactory接口，这些方法依然不会返回Parent Factory中的信息。通过`BeanFactoryUtils`可以判断Bean属于哪个Factory。
2. 这些方法也不会返回那些经过特殊方式交由该Factory管理的Bean。比如通过ConfigurableBeanFactory.registerSingleton方法所注册进该Factory中的Bean。

#### HierarchicalBeanFactory
HierarchicalBeanFactory表明BeanFactory可以含有Parent Factory，在运用时只需要注意其提供了一个判断当前Factory是否含有这个Bean的方法：containsLocalBean。

### BeanDefinitionRegistry
BeanDefinitionRegistry用于注册和保管BeanDefinitions，通常由BeanFactories实现。在Spring框架中，所有注册与保管BeanDefinition的接口都被封装在BeanDefinitionRegistry中。

### BeanDefinitionReader
BeanDefinitionReader负责从外部收集信息，将这些信息处理为Spring标准的描述形式：BeanDefinition。并将这些BeanDefinition提供给BeanDefinitionRegistry。

### BeanDefinition
BeanDefinition用于描述一个Bean的信息，它被保存于BeanDefinitionRegistry中，由BeanFactory将其实例化为具体的Bean。

BeanDefinition继承关系如下：

![BeanDefinition](../resources/img/BeanDefinition.png)