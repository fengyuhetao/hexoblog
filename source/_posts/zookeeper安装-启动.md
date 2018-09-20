---
title: zookeeper安装-启动
tags: zookeeper
abbrlink: 28842
date: 2018-09-20 12:23:57
---

## 单机模式

* 下载，并解压zookeeper

> wget http://mirrors.hust.edu.cn/apache/zookeeper/zookeeper-3.4.13/zookeeper-3.4.13.tar.gz

* 创建data和logs两个目录,用于存储数据和日志

> ht@TIANJI:/mnt/d/zookeeper\$ cd zookeeper-3.4.13/
> ht@TIANJI:/mnt/d/zookeeper/zookeeper-3.4.13\$ mkdir data
> ht@TIANJI:/mnt/d/zookeeper/zookeeper-3.4.13\$ mkdir logs

* 在`conf`目录下写入zoo.cfg文件

```
tickTime=2000
dataDir=/mnt/d/zookeeper/zookeeper-3.4.13/data
dataLogDir=/mnt/d/zookeeper/zookeeper-3.4.13/logs
clientPort=2181
```

* 进入bin目录，启动、停止、重启分和查看当前节点状态（包括集群中是何角色）别执行：

```
linux:
./zkServer.sh start
./zkServer.sh stop
./zkServer.sh restart
./zkServer.sh status

windows:
./zkServer.cmd
```

## 伪集群模式

同时在每一个dataDir目录下，创建myid文件，文件内容即服务器ID，具体如下:

server.A=B:C:D     其中 A 是一个数字，就是myid里的那个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址,C和D是两个端口,C、D两个端口相互也不能重复（仅限伪分布式模式下）。

官方解释如下:

**表单server.X的条目列出构成ZooKeeper服务的服务器。当服务器启动时，它通过查找数据目录中的文件myid来知道它是哪个服务器 。该文件包含服务器编号，以ASCII格式显示。**

**最后，请注意每个服务器名称后面的两个端口号：“2888”和“3888”。对等体使用前端口连接到其他对等体。这样的连接是必要的，使得对等体可以进行通信，例如，以商定更新的顺序。更具体地说，一个ZooKeeper服务器使用这个端口来连接追随者到领导者。当新的领导者出现时，追随者使用此端口打开与领导者的TCP连接。因为默认领导选举也使用TCP，所以我们目前需要另外一个端口进行领导选举。这是服务器条目中的第二个端口。**

复制三个zookeeper文件夹。

```
ht@TIANJI:/mnt/d/zookeeper$ tree -L 1
.
├── zookeeper_1
├── zookeeper_2
├── zookeeper_3
└── zookeeper-3.4.13.tar.gz
```

修改配置文件:

zookeeper_1/zoo.cfg:

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/mnt/d/zookeeper/zookeeper_1/data
clientPort=2182
dataLogDir=/mnt/d/zookeeper/zookeeper_1/logs

server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```

zookeeper_2/zoo.cfg:

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/mnt/d/zookeeper/zookeeper_2/data
clientPort=2182
dataLogDir=/mnt/d/zookeeper/zookeeper_2/logs

server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```

zookeeper_3/zoo.cfg:

```
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/mnt/d/zookeeper/zookeeper_3/data
clientPort=2182
dataLogDir=/mnt/d/zookeeper/zookeeper_3/logs

server.1=localhost:2287:3387
server.2=localhost:2288:3388
server.3=localhost:2289:3389
```

然后，挨个启动即可。

