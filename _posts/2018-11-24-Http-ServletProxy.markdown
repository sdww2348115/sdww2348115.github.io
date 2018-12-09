---
layout: post
title:  "ServletProxy"
date:   2018-11-24 22:43:00 +0800
categories: http servlet util
permalink: /util/servlet-proxy
---

## 背景
目前公司开始全面开展微服务化整改，以前的项目计划都会被迁移至CloudOS所提供的微服务平台上。对于以前的项目而言，都是大而全的，功能完备、结构紧密；一时半会完成不了项目级别的服务拆分与微服务化，因此大多都采取了先封装再拆分的方式。

1. step1:将整个项目Docker化，进行前后端分离等操作
2. step2:在Docker化的基础上，再进行微服务拆分，最终达到微服务化的目的

项目Docker化往往意味着请求方式的变动：之前是UI界面直接访问后端Server，变为UI界面访问Portal，再由Portal访问后端Service。由此带来了许多问题：代码重复开发、原有项目中的上传下载功能无法使用、UI无法直通Service层等。

为了解决以上问题，考虑1.是否能搭建一条VPN通道直接将UI与Service连接起来；2.portal对于请求的管控功能必须实现，能否在VPN通道处加上一些业务处理逻辑？

## 目标
根据上文需求的分析，我们将要在Portal层中实现一个http"透传"通道，在"透传"的时候还可以执行一些必须的业务逻辑。目前Portal层基本由spring项目组成，目前需要考虑业务逻辑在spring环境中执行。

## 分析
Http协议是基于字节的传输协议，理论上我们可以将收到的HTTP请求字节流完整传递给service层，然后将service层所返回的字节流一字不差的返回给前端UI界面。但是这样会带来一个问题：portal完全成为了一个代理，我们无法获取字节流的信息，也无法对字节流做出处理。完整地解析整个http请求所带来的的时间空间消耗也不是我们想要见到的。那么，如何把握这个度呢？
个人给出的答案是：HTTP头部携带有请求的大量信息，而且通常不会太长。我们仅需要解析并重构HTTP header，就能获取足够多的信息，同时避免接收、缓存、解析http请求中硕大的body。

## 整体架构
http请求流在portal层将依次完成以下工作：

1. portal对于http请求进行逻辑处理，并修改其为core层所需的请求
2. portal将修改后的http请求发送至core层
3. portal针对core层返回进行业务处理，并将处理完成后的resp返回给UI层

参考Netty的InboundHandler与OutbandHandler，整体结构如下图所示
![ServletProxy](../resources/img/servlet-proxy.jpg)

## 详细设计

### 如何在Servlet中实现请求的转发
我们的portal项目采用的是SpringBoot，Servlet容器选用的是嵌入式tomcat，而且之前并未考虑多servlet项目之间的划分。采用Servlet容器来实现该需求的话会遇到以下问题：

 1. 多Servlet之间url的划分。由于之前默认只采用一个Servlet，该Servlet的url-pattern为`/*`，在此基础上直接添加Servlet可能会导致目标请求不能按照我们预想的方式进行跳转。
 2. SpringBoot的web核心实现仍然为SpringMVC，其双层的applicationContext架构可能会让新的Servlet无法注入预期的bean。

基于以上考虑，该方案将基于Servlet容器中的Filter进行实现。为了方便处理，这里采用javax.servlet.http.HttpFilter作为基类。

### 发送/接收Http请求的框架选择
目前java使用的httpclient一般为jdk自带的URLconnection/Apache HttpClient/Netty。各个不同实现都有自己的长处与短处：

|实现|优点|缺点|
| :------:| :------: | :------: |
|JDK URLConnection|JDK自带，不用额外引入lib|据说有bug,使用起来不太习惯|
|Apache HttpClient|通用实现方式，使用面广|需要额外引入lib|
|Netty|NIO实现|使用复杂，操作性较弱|

在无法确定哪条路能够走通或者比较好走的情况下，把这一块抽象出来，使其可以适配各种实现是一种较优的选择。

### 实体类设计
当我们处理与UI之间交互的时候，我们使用的用于表示HTTP请求的类为`HttpServletRequest`与`HttpServletResponse`，而与后端Core之间交互时，使用的类与传输框架密切相关。因此，在该框架中我们使用两个简化后的类来表示、操作HTTP协议中的Request和Response。

 * Request:类的属性包括:URL、HTTP Method、HTTP Header、Http Body
 * Response:类的属性包括:StatusCode、HTTP Header、Http Body

## stream closed问题
代码完成编写后，测试时proxy总是报server error。关键错误堆栈如下：

从错误堆栈可以看出，产生问题的原因是stream closed。当前程序运行时，我们有两个InputStream，一个来自UI页面，另一个来自后端server。错误堆栈中的包名：org.apache.catalina可以看出，该问题是出自于UI页面发来的文件上传请求。
![ServletProxy](../resources/img/exception-stack.PNG)

Tomcat由service与connector两部分组成，connector提供数据交互相关的逻辑，整体并发框架等；所有的逻辑由Service完成，其中主要是由service中的filter完成各项工作(servlet的相关逻辑同样是由filter完成）。当Http请求到达filter的时候，tomcat会对请求做最基本的解析，将其处理为两部分：包含基本信息的Header，以及封装为ServletInputStream类型的Body。该逻辑是我们实现serlvet-proxy的基础。

因此，当我们调用HttpServletRequest.getInputstream.read()方法时，实际上是读取通过socket传来的Http请求中的body部分，正常情况下如果该方法有问题的话，带有body的请求将都不能被正确解析，因此这里的问题一定不是出在Tomcat处。仔细查看错误堆栈，发现来自UI的请求在被我们的filter进行处理之前，已经被spring中的几个filter进行处理了。其中包括：OncePerRequestFilter，CharacterEncodingFilter，HiddenHttpMethodFilter，FormContentFilter，RequestContextFilter。
初步怀疑是这里Filter的处理导致了InputStream异常的表现。

由于我们的目的是将Http请求直接透传至后方服务，对于请求的任何框架性修改都是不必要的。因此，在设置自定义filter处，添加代码registration.setOrder(Ordered.HIGHEST_PRECEDENCE)将ServletProxyFilter添加在所有Filter的最前方。经过测试，Http请求的InputStream在读取时不会再报stream closed异常。

//TODO:通过断点跟踪，可以发现，在HiddenHttpMethodFilter中，请求被发现为multipart/form-data格式，因此tomcat对该请求的body做了最基本的解析，包括文件原始名称等，导致inputstream被破坏，无法被正常读取。该流程貌似与form中file只能放在最后也有关系。

当我将上传通道打通之后，又遇到了新的问题：将http的inputstream与servlet的responsestream进行数据copy时，又出现了inputstream closed问题，详细错误堆栈如下：

> java.io.IOException: Attempted read from closed stream.
	at org.apache.http.impl.io.ContentLengthInputStream.read(ContentLengthInputStream.java:165) ~[httpcore-4.4.10.jar:4.4.10]
	at org.apache.http.conn.EofSensorInputStream.read(EofSensorInputStream.java:137) ~[httpclient-4.5.2.jar:4.5.2]
	at org.apache.http.conn.EofSensorInputStream.read(EofSensorInputStream.java:150) ~[httpclient-4.5.2.jar:4.5.2]
	at org.apache.commons.io.IOUtils.copyLarge(IOUtils.java:1025) ~[commons-io-1.3.2.jar:1.3.2]
	
查看对应class源码，异常抛出处非常清晰明白：ContentLengthInputStream在read时会首先check是否关闭，如果stream关闭则会抛出以上错误。check代码， 可以发现在该类中只有一个close方法会将close的值置为true，将这里打上断点进行跟踪，问题终于被找到了：CloseableHttpClient在执行execute方法时，当responseHandler处理完毕请求后，client会将inputstream切断。代码节选自org.apache.http.impl.client.CloseableHttpClient,如下：

```java
    final T result = responseHandler.handleResponse(response);
    final HttpEntity entity = response.getEntity();
    EntityUtils.consume(entity);
    return result;
```

因此，当我们的Http请求返回后，此时程序hold住了两个请求：servlet请求与http请求。如果以servlet请求为主，则走默认的ServletFilter逻辑，由Servlet控制请求周期；如果以http请求为主，则走HttpClient逻辑，代码的控制权将交给HttpClient。为了使代码可用，这里采用了callback的形式，将所有后续逻辑以callback的形式放入HttpClient中，由responseHandler负责将两个response连接起来，达到“透传”的目的。
