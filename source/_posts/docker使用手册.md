---
title: docker使用手册
date: 2018-05-29 14:36:55
tags:
---

## Docker命令

1. 列出所有镜像

   > docker images -a

2. 构建镜像

   >  docker build -t \[镜像名称\]        \[Dockerfile所在目录\]

3. 运行镜像

   > docker run --name [容器名称(可选)] -d -p 80:8080 [镜像名称]

4. 查看镜像状态

   > docker stats
   >

5. 显示所有的容器，包括未运行的

   > docker ps -a

6. 显示所有容器ID

   > docker ps -a -q

7. 停止运行

   > docker stop [容器名称]

8. 删除容器

   > docker rm [容器名称]
   >
   > docker rm [容器ID]

9. 创建一个守护态的Docker容器，然后使用docker attach命令进入该容器。

   >  sudo docker run -itd ubuntu:14.04 /bin/bash 

10. 进入容器,这方法不怎么好

  >  sudo docker attach [容器id]

11. 进入容器

    > docker exec -it [容器id|容器名称] /bin/bash

12. 删除image

    > docker rmi [ImageID]

13. 查看端口开启情况

    > docker port [容器名称]

14. 从容器中复制文件

    > docker cp [容器名称]:[path\]             \[主机path\]

15. 从主机复制文件到容器

    > docker cp [主机path]                            \[容器名称\]:[path]
