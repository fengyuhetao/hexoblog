---
title: dubbo项目结构
tags: dubbo
abbrlink: 62824
date: 2018-09-20 19:15:02
---

1. 概览

   可以看到，代码相当多，坚持就是胜利。

![](/assets/dubbo/TIM截图20180920200955.png)

2. 代码统计

   - 可以使用`IDEA Statistic`插件。

   ![](/assets/dubbo/TIM截图20180921223152.png)

   - 使用shell命令

   这个命令只过滤了部分注释，所以相比 [IDEA Statistic](https://plugins.jetbrains.com/plugin/4509-statistic) 会偏多。

   > find . -name "*.java"|xargs cat|grep -v -e ^$ -e ^\s*\/\/.*$|wc -

3. dubbo模块示意图

   ![](/assets/dubbo/dubbo-modules.jpg)

4. dubbo-common

   **公共逻辑模块**：提供工具类和通用模型。

   ![](/assets/dubbo/TIM截图20180921224344.png)

   通用模型: 比如`com.alibaba.dubbo.common.URL`：

   - 所有扩展点参数都包含 URL 参数，URL 作为上下文信息贯穿整个扩展点设计体系。
   - URL 采用标准格式：`protocol://username:password@host:port/path?key=value&key=value`

5. dubbo-remoting

   **远程通信模块**：提供**通用**的客户端和服务端的通讯功能。、

   ![](/assets/dubbo/TIM截图20180921224942.png)

   * dubbo-remoting-zookeeper

     相当于`Zookeeper Client`，负责和`Zookeeper Server`通信。

   * dubbo-remoting-api

     定义`Dubbo client`和`Dubbo Server`的接口。

   * 其他，均实现了`dubbo-remoting-api`

      * dubbo-remoting-grizzly
      * dubbo-remoting-http
      * dubbo-remoting-mina
      * dubbo-remoting-netty(netty3)
      * dubbo-remoting-netty4
      * dubbo-remoting-p2p            P2P 服务器，注册中心 `dubbo-registry-multicast` 使用该项目。

   目前看来，仅仅需要看如下:

   * `dubbo-remoting-api` + `dubbo-remoting-netty4`
   * `dubbo-remoting-zookeeper`

6. dubbo-rpc

   **远程调用模块**：抽象各种协议，以及动态代理，只包含一对一的调用，**不关心集群的管理**。集群相关的管理，由 `dubbo-cluster` 提供特性。同时，该模块也是整个项目的核心部分。

   ![](/assets/dubbo/TIM截图20180923212827.png)

   * dubbo-rpc-api

     **抽象**各种协议以及动态代理，**实现**了一对一的调用。

   * 其他模块则分别实现了`dubbo-rpc-api`，特别注意:`dubbo-rpc-default`对应`dubbo://`

7. dubbo-cluster

   **集群模块**：将多个服务提供方伪装为一个提供方，包括：负载均衡, 集群容错，路由，分组聚合等。集群的地址列表可以是静态配置的，也可以是由注册中心下发。