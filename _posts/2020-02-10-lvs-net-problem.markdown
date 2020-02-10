---
layout: post
title:  "记一次蛋疼的LVS搭建经历"
date:   2020-02-10 22:28:00 +0800
categories: network gateway lvs
permalink: /network/lvs/problem1
description: 搭建LVS DR集群时遇到的坑爹问题
---

## 环境
最近在研究LoadBalance以及网关相关内容，根据[教程](https://blog.csdn.net/liwei0526vip/article/details/103104496)一步一步搭建DR模式LVS环境，将LVS、RS1、RS2都搞定后请求虚拟IP，无论是curl还是浏览器总是报连接无法建立。具体环境如下：

我采用VMware虚拟机搭建LVS以及RS测试集群，具体组网情况如下图所示

![LVS-network](https://sdww2348115.github.io/resources/img/LVS-network.png)

我创建了两个虚拟网络，网络1由Client、lvs-server、rs-server1、rs-server2组成，代表公网环境，网关IP为192.168.65.1
网络2由lvs-server、rs-server1、rs-server2组成，代表服务器内网环境。按照预想，Client端发起的网络请求完整回路为：

1. Client通过route1发送dst地址为虚拟IP192.168.65.77的报文至LVS服务器192.168.65.10（上图1、2）
2. LVS服务器在观察到报文为虚拟IP后，将报文按照RR的方式修改MAC值，通过内网转发给rs-server（上图3、4）
3. rs-server接收到虚拟IP后，根据src地址，通过route1将response直接回应给Client端

下面为各台服务器网络具体配置：

备注:

lvs-server:
![lvs-server-ifconfig](https://sdww2348115.github.io/resources/img/lvs-server-ifconfig.JPG)

rs-server1:
![rs-server1-ifconfig](https://sdww2348115.github.io/resources/img/rs-server1-ifconfig.JPG)

rs-server2:
![rs-server2-ifconfig](https://sdww2348115.github.io/resources/img/rs-server2-ifconfig.JPG)

1. rs-server上arp_ignore以及arp_announce已配置
2. 由于lvs-server上没有网卡响应虚拟ip192.168.65.77，因此在Client上使用route命令让目的地址为vip的包强制走lvs-server,如下图所示

## 问题现象 & 原因
使用curl http://192.168.65.77命令，现象如下：
![curl-result](https://sdww2348115.github.io/resources/img/curl-result.JPG)

经过检查，rs-server端口正常，lvs-server至rs-server网络通路同样正常。

首先怀疑是IP报文是否到达lvs-server，在lvs-server机器上执行抓包命令`tcpdump -i ens33 'host 192.168.65.77'`，抓包结果如下：
![lvs-server-tcpdump-77](https://sdww2348115.github.io/resources/img/lvs-server-tcpdump-77.JPG)

可以看出，确实接收到了从client端(192.168.65.131)发来的dst为VIP(192.168.65.77)的数据包。但是奇怪的是为啥后面又收到了询问VIP(192.168.65.77)的数据包呢？这个待会后面介绍。既然数据包到了lvs-server，为啥client还是报错不能建立连接？再在client端使用命令`tcpdump -i ens33 'host 192.168.65.10'`抓包看看
![client-tcpdump-10](https://sdww2348115.github.io/resources/img/client-tcpdump-10.JPG)

请注意抓到的第一个报文：`ICMP redirect 192.168.65.77 to host 192.168.65.77`。curl报no route to host的原因找到了，经过还原，整个过程如下：

1. Client端构造了一个dst为VIP(192.168.65.77)的IP报文，查询自身route表，找到了我们的手动配置项，将这个报文直接丢给了lvs-server
2. lvs-server发现自己不是VIP的目标地址，于是查询自身路由表，发现client端与自己以及VIP同网段，把VIP的报文交给我转发还不如你自己发过去呢，于是给Client回了一个ICMP REDIRECT报文告诉客户端发往VIP地址的报文下一跳请走192.168.65.77
3. Client收到Redirect报文后，通过ARP命令查询192.168.65.77的MAC地址，结果发现无人应答，于是经过超时时间后报错:no route to host。

## 解决方案
既然问题是由于ICMP Redirect报文引起的，那么直接禁用lvs-server的ICMP Redirect就能把问题解决。但是回过头来想，lvs的配置需要如此复杂吗？特别是对于client端来说，需要每一个client强制VIP报文走lvs-server本身就是一件不态科学的事，lvs对外是完全透明的，client端不应该有任何设置，那么在我们的配置过程中差了什么？

`route1 -> lvs-server`

只要route1将VIP报文直接交给lvs-server就好了。将lvs，rs看做一个虚拟的服务器，客户端的报文应该经过层层路由直接交付至这个服务器上，而且这个服务器的IP地址就是我们的VIP。但是经过我们的配置，不会有任何一台机器对VIP的ARP请求作出响应，那么网络中发送给VIP的报文都将被丢弃，出现no route to host的错误！因此，我们缺失的那一步操作就是：将VIP的MAC绑定到lvs-server上！执行命令将vip绑定在本机上：
```bash
ifconfig lo:0 192.168.65.77 broadcast 192.168.65.77 netmask 255.255.255.255 up
```

再通过curl命令进行请求：curl http://192.168.65.77，成功得到回应：
![client-tcpdump-10](https://sdww2348115.github.io/resources/img/curl-correct.JPG)