---
layout: post
title:  "Proxy Of Java"
date:   2019-04-03 22:23:00 +0800
categories: Java Proxy 
permalink: /tools/proxy
---

# 使用JAVA实现的Proxy

标签（空格分隔）： java proxy net

---

## 背景
公司内部环境网络复杂，每个部门都有自己以192开头的局域网，又名小网，与之相对的是整个公司层面上以10开头的大网。大网与小网环境不能互通，因此其他部门同事如果需要访问小网内的资源将非常不方便。

以前的做法是通过Nginx搭建一个反向代理，使用一台既能访问大网又能访问小网的机器作为桥梁沟通两个网络。使用的方式是端口映射（portProxy）:类似路由器的做法，将指向本机某个端口的TCP数据完全推送至目标地址，是基于TCP层之上的代理转发机制。长久以来工作非常顺利，但是有几点不好：

1. 每当有新需求来临时，需要我手动修改Nginx配置并重启，虽然消耗时间不多，但被打扰的次数不少。
2. 其他同事之间信息不通，热门端口的代理被反复提起，沟通的成本较高。
3. Nginx状态对于其他同事是黑盒，他们无从判断是我本机的Nginx失效还是服务器网络出现问题。

鉴于以上问题，我准备以Java实现一个简单的TCP层代理服务，希望能够解决工作中所遇到的痛点。

## 思路
所有的痛点都是由于缺乏一个有效的信息通道造成。

* 对于第一个痛点，由于公司内部都是相熟的同事，在安全性得到保障的情况下我可以将反向代理的配置权释放出来，可以供任何人在外部进行修改，这样就减少修改Nginx所带来的时间成本，同时也使得我被打断的时间大大减少。
* 将本机的所有端口代理信息整理起来以网页形式进行展示，并加以备注说明，可以解决不同同事之间的信息墙。同时也更方便该项目的推广和应用。
* 对于代理程序以及其中每一个代理端口的健康状态加以监控，并展示在端口信息页面上，大家即可方便的观察代理程序与各端口的使用信息。甚至流量信息！

综合起来，整个项目可分为三个模块：
1. 管理控制模块：用于保存所有代理端口的信息，以网页方式进行展示，支持与用户进行互动，用户可通过网页直接对代理信息进行增删改查操作。
2. 核心代理模块：实现端口流量转发的核心模块。
3. 监控模块：对代理的端口执行监控，目前仅考虑监控网络连接以及流量。

## 实现

### 核心代理模块
JAVA中可通过Socket接口直接操作TCP层以及之上的数据，根据网络协议，TCP中的数据是以流的方式进行传输，只要将接收到的TCP字节流原封不动的传递给另一个Socket即可完成代理转发功能。在转发的过程中，需要注意几点：

1. 转发的数据流应是互相隔离的，即使目的地址一致的情况下也不能将多个输入流的数据混在同一个输出流进行输出。
2. 原有的TCP连接逻辑上变为了两个连接，程序应即使对TCP连接的关闭和异常做出响应，任何连接断开都将被视为整个链路的断开
3. TCP连接应是全双工通信。

代码如下：
``` java
/**
     * 开启代理监听端口，接收代理请求
     * @throws Exception
     */
    public static void proxyStart() throws Exception {
        ServerSocket serverSocket = new ServerSocket(LISTEN);
        Socket socket = null;
        while ((socket = serverSocket.accept()) != null) {
            handleSocket(socket);
        }
    }

    /**
     * 代理数据传输具体实现，每一个端口代理将开启3个线程：
     * 1. 监控两个TCP连接状态的总线程
     * 2. 将源数据传递给目的地址的数据传输线程
     * 3. 将目的地址返回的数据传递给源地址的传输线程
     * @param socket
     */
    public static void handleSocket(Socket socket) {
        Thread in = null;
        Thread out = null;
        try (Socket target = new Socket()) {
            target.connect(new InetSocketAddress(HOST, PORT));
            //使用CountDownLatch作为线程间同步工具，任一传输线程异常退出都将导致链路整体断开
            CountDownLatch shouldTerminate = new CountDownLatch(1);
            in = new Thread(() -> {
                transfer(socket, target, shouldTerminate);
            });
            in.start();

            out = new Thread(() -> {
                transfer(target, socket, shouldTerminate);
            });
            out.start();

            shouldTerminate.await();
        } catch (IOException e) {
            throw new RuntimeException();
        } catch (InterruptedException e) {
            throw new RuntimeException();
        } finally{
            if (in != null) {
                in.interrupt();
            }
            if (out != null) {
                out.interrupt();
            }
        }
    }
    
    public static void transfer(Socket src, Socket target, CountDownLatch shouldTerminate) {
        try {
            byte[] buffer = new byte[8192];
            while (src.getInputStream().read(buffer) > 0) {
                target.getOutputStream().write(buffer);
            }
        } catch (IOException e) {
            throw new RuntimeException();
        } finally {
            shouldTerminate.countDown();
        }
    }
```

#### 系统优化

上面的代理程序能够完美实现TCP数据转发的目的。对于每一个到来的连接，程序将开启三个线程为其服务：管理线程与两个数据传输线程。
上面的代理程序能够完美实现TCP数据转发的目的。对于每一个到来的连接，程序将开启三个线程为其服务：管理线程与两个数据传输线程。



