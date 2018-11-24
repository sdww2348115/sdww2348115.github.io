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

