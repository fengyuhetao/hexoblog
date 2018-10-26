---
title: dubbo项目结构
tags: dubbo
abbrlink: 62824
date: 2018-09-20 19:15:02
password: dubbo
---

参考: 芋道源码      http://svip.iocoder.cn/

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

   ![](/assets/dubbo/TIM截图20181005112345.png)

   * 注册中心下发，有`dubbo-repository`提供该特性

   * 容错
     * `com.alibaba.dubbo.rpc.cluster.Cluster`接口+ `com.alibaba.dubbo.rpc.cluster.support`包
     * Cluster 将 Directory 中的多个 Invoker 伪装成一个 Invoker，对上层透明，伪装过程包含了容错逻辑，调用失败后，重试另一个。
   * 目录
     * `com.alibaba.dubbo.rpc.cluster.Directory` 接口 + `com.alibaba.dubbo.rpc.cluster.directory` 包。
     * Directory 代表了多个 Invoker ，可以把它看成 List ，但与 List 不同的是，它的值可能是动态变化的，比如注册中心推送变更。
   * 路由
     * `com.alibaba.dubbo.rpc.cluster.Router` 接口 + `com.alibaba.dubbo.rpc.cluster.router` 包。
     * 负责从多个 `Invoker` 中按路由规则选出子集，比如读写分离，应用隔离等。
   * 配置
     * `com.alibaba.dubbo.rpc.cluster.Configurator`接口 + `com.alibaba.dubbo.rpc.cluster.configurator` 包。
   * 负载均衡
     * `com.alibaba.dubbo.rpc.cluster.LoadBalance`接口 + `com.alibaba.dubbo.rpc.cluster.loadbalance` 包。
     * LoadBalance 负责从多个 Invoker 中选出具体的一个用于本次调用，选的过程包含了负载均衡算法，调用失败后，需要重选。
   * 合并结果
     * `com.alibaba.dubbo.rpc.cluster.Merger` 接口 + `com.alibaba.dubbo.rpc.cluster.merger` 包。
     * 合并返回结果，用于分组聚合。

8. dubbo-registry
   **注册中心模块**：基于注册中心下发地址的集群方式，以及对各种注册中心的抽象。

   ​						![](/assets/dubbo/TIM截图20181005113216.png)		

   * `dubbo-registry-api`**抽象**注册中心的注册与发现接口。

   * 其他模块，实现`dubbo-registry-api`，提供对应的注册中心发现

   * `dubbo-registry-default`对应Simple注册中心

9. dubbo-monitor

   **监控模块**: 统计服务调用次数，调用时间的，调用链跟踪的服务。

   ![](/assets/dubbo/TIM截图20181010204311.png)

10. dubbo-config

  **配置模块**:是 Dubbo 对外的 API，用户通过 Config 使用Dubbo，隐藏 Dubbo 所有细节。

  ![](/assets/dubbo/TIM截图20181010210410.png)

  * `dubbo-config-api`: 实现了 [API 配置](https://dubbo.gitbooks.io/dubbo-user-book/configuration/api.html) 和 [属性配置](https://dubbo.gitbooks.io/dubbo-user-book/configuration/properties.html) 功能。
  * `dubbo-config-spring`: 实现了 [XML 配置](https://dubbo.gitbooks.io/dubbo-user-book/configuration/xml.html) 和 [注解配置](https://dubbo.gitbooks.io/dubbo-user-book/configuration/annotation.html) 功能。

11. dubbo-container

    **容器模块**:是一个 Standlone 的容器，以简单的 Main 加载 Spring 启动，因为服务通常不需要 Tomcat/JBoss 等 Web 容器的特性，没必要用 Web 容器去加载服务。

    ![](/assets/dubbo/TIM截图20181010210655.png)

    * `dubbo-container-api`:   定义了`com.alibaba.dubbo.container.Container` 接口，并提供 加载所有容器启动的 Main 类
    * 其他模块，实现`dubbo-container-api`
      * `dubbo-container-spring`: 提供`com.alibaba.dubbo.container.spring.SpringContainer`
      * `dubbo-container-log4j`: 提供`com.alibaba.dubbo.container.log4j.Log4jContainer`
      * `dubbo-container-logback`: 提供`com.alibaba.dubbo.container.logback.LogbackContainer`
12. dubbo-filter

    **过滤器模块**：提供了**内置**的过滤器。

    ![](/assets/dubbo/TIM截图20181010214312.png)

    * `dubbo-filter-cache`: 缓存过滤器
    * `dubbo-filter-validation`:参数验证过滤器

13. dubbo-plugin

    **插件模块**: 提供内置的插件

    ![](/assets/dubbo/TIM截图20181010214433.png)

    * `dubbo-qos`: 提供在线运维命令

14. hessian-lite

    Dubbo 对 [Hessian 2](http://hessian.caucho.com/) 的 **序列化** 部分的精简、改进、BugFix 。

15. dubbo-test

    **测试模块**

    ![](/assets/dubbo/TIM截图20181010214611.png)

    * `dubbo-test-benchmark`: 性能测试
    * `dubbo-test-compatibility`: 兼容性测试
    * `dubbo-test-example`: 使用示例

16. dependencies-bom

    统一定义dubbo依赖的第三方库的版本号

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
    
        <parent>
            <groupId>org.sonatype.oss</groupId>
            <artifactId>oss-parent</artifactId>
            <version>7</version>
            <relativePath />
        </parent>
    
        <groupId>com.alibaba</groupId>
        <artifactId>dubbo-dependencies-bom</artifactId>
        <version>2.6.1</version>
        <packaging>pom</packaging>
    
        <name>dubbo-dependencies-bom</name>
        <description>Dubbo dependencies BOM</description>
        <url>http://dubbo.io</url>
    
        <inceptionYear>2011</inceptionYear>
        <licenses>
            <license>
                <name>Apache 2</name>
                <url>http://www.apache.org/licenses/LICENSE-2.0.txt</url>
            </license>
        </licenses>
    
        <organization>
            <name>The Dubbo Project</name>
            <url>http://dubbo.io</url>
        </organization>
    
        <issueManagement>
            <system>Github Issues</system>
            <url>https://github.com/alibaba/dubbo/issues</url>
        </issueManagement>
        <scm>
            <url>https://github.com/alibaba/dubbo/tree/master</url>
            <connection>scm:git:git://github.com/alibaba/dubbo.git</connection>
            <developerConnection>scm:git:git@github.com:alibaba/dubbo.git</developerConnection>
        </scm>
        <mailingLists>
            <mailingList>
                <name>Dubbo User Mailling List</name>
                <subscribe>dubbo+subscribe@googlegroups.com</subscribe>
                <unsubscribe>dubbo+unsubscribe@googlegroups.com</unsubscribe>
                <post>dubbo@googlegroups.com</post>
                <archive>http://groups.google.com/group/dubbo</archive>
            </mailingList>
            <mailingList>
                <name>Dubbo Developer Mailling List</name>
                <subscribe>dubbo-developers+subscribe@googlegroups.com</subscribe>
                <unsubscribe>dubbo-developers+unsubscribe@googlegroups.com</unsubscribe>
                <post>dubbo-developers@googlegroups.com</post>
                <archive>http://groups.google.com/group/dubbo-developers</archive>
            </mailingList>
        </mailingLists>
        <developers>
            <developer>
                <id>dubbo.io</id>
                <name>The Dubbo Project Contributors</name>
                <email>dubbo@googlegroups.com</email>
                <url>http://dubbo.io</url>
                <organization>The Dubbo Project</organization>
                <organizationUrl>http://dubbo.io</organizationUrl>
            </developer>
        </developers>
    
        <properties>
            <!-- Common libs -->
            <spring_version>4.3.10.RELEASE</spring_version>
            <javassist_version>3.20.0-GA</javassist_version>
            <netty_version>3.2.5.Final</netty_version>
            <netty4_version>4.0.35.Final</netty4_version>
            <mina_version>1.1.7</mina_version>
            <grizzly_version>2.1.4</grizzly_version>
            <httpclient_version>4.5.3</httpclient_version>
            <fastjson_version>1.2.46</fastjson_version>
            <zookeeper_version>3.4.9</zookeeper_version>
            <zkclient_version>0.2</zkclient_version>
            <curator_version>2.12.0</curator_version>
            <jedis_version>2.9.0</jedis_version>
            <xmemcached_version>1.3.6</xmemcached_version>
            <cxf_version>3.0.14</cxf_version>
            <thrift_version>0.8.0</thrift_version>
            <hessian_version>4.0.38</hessian_version>
            <servlet_version>3.1.0</servlet_version>
            <jetty_version>6.1.26</jetty_version>
            <validation_version>1.1.0.Final</validation_version>
            <hibernate_validator_version>5.4.1.Final</hibernate_validator_version>
            <jel_version>3.0.1-b08</jel_version>
            <jcache_version>1.0.0</jcache_version>
            <kryo_version>4.0.1</kryo_version>
            <kryo_serializers_version>0.42</kryo_serializers_version>
            <fst_version>2.48-jdk-6</fst_version>
    
            <rs_api_version>2.0</rs_api_version>
            <resteasy_version>3.0.19.Final</resteasy_version>
            <tomcat_embed_version>8.0.11</tomcat_embed_version>
            <!-- Log libs -->
            <slf4j_version>1.7.25</slf4j_version>
            <jcl_version>1.2</jcl_version>
            <log4j_version>1.2.16</log4j_version>
            <logback_version>1.2.2</logback_version>
            <commons_lang3_version>3.4</commons_lang3_version>
        </properties>
    
        。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。
    
    </project>
    ```

    然后dubbo-parent会引入该pom。

    **为了防止用 Maven 管理 Spring 项目时，不同的项目依赖了不同版本的 Spring ，可以使用 Maven BOM 来解决者一问题。**

    ```
    <dependencyManagement>
            <dependencies>
                <dependency>
                    <groupId>com.alibaba</groupId>
                    <artifactId>dubbo-dependencies-bom</artifactId>
                    <version>2.6.1</version>
                    <type>pom</type>
                    <scope>import</scope>
                </dependency>
            </dependencies>
        </dependencyManagement>
    ```

17. dubbo-bom

    Maven BOM(Bill Of Materials) ，**统一**定义了 Dubbo 的版本号

18. dubbo-parent

19. dubbo-all

pom依赖图:

![](/assets/dubbo/pom-dependency.png)