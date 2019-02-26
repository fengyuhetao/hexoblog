---
title: dubbo-启动
tags: dubbo
abbrlink: 11234
date: 2018-09-20 11:25:08
notshow: true
---

## 前言

**长期陷于CRUD，能力只退不进，期待dubbo源码能给点转变**

## 配置环境:

* Intellij idea
* maven
* jdk1.8
* git

## 开发工具: Intellij idea

现将dubbo直接`fork`到自己的仓库中，然后使用`Intellij idea`直接从`https://github.com/fengyuhetao/dubbo.git`拉取项目。

## 启动demo项目

1. 注册中心采用**Multicast 多播**（组播)

   发现报错，如下:

```
[20/09/18 04:37:14:014 CST] main  WARN multicast.MulticastRegistry:  [DUBBO] Ignore empty notify urls for subscribe url provider://169.254.38.13:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=demo-provider&category=configurators&check=false&dubbo=2.0.2&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=4844&side=provider&timestamp=1537432631871, dubbo version: , current host: 169.254.38.13
```

原因在于: **win10 默认没有安装Multicast 协议。**

解决方法,安装可靠多播协议:

![](/assets/dubbo/TIM截图20180920164543.png)

![](/assets/dubbo/TIM截图20180920164747.png)

![](/assets/dubbo/TIM截图20180920164825.png)

![](/assets/dubbo/TIM截图20180920164936.png)

然后，重新启动provider和consumer，这时候发现依然无法成功。

问题在于本地网卡和虚拟网卡过多，导致网络混乱造成的，禁用无关网卡即可。

![](/assets/dubbo/TIM截图20180920173149.png)

仅仅需要关闭vmware虚拟网卡即可，再次重新启动provider和consumer，即可成功。

![](/assets/dubbo/TIM截图20180920173234.png)

2. 注册中心采用zookeeper

按照文档即可,一般不会出什么问题。

**接下来，开始真正的战斗。**

