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

   > docker run -d -p 80:8080 [镜像名称]

4. 查看镜像状态

   > docker stats

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

