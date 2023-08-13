---
title: Docker常用命令
abbrlink: 29dc6fe8
tags:
  - Docker
categories:
  - Docker
  - 常用命令
index_img: /img/pages/docker.jpeg
date: 2023-08-13 17:40:00
---

#### Docker常用命令
> 查找镜像→ docker search <镜像名称>
> 
> 下载镜像→ docker pull <镜像名称>:<镜像版本>
> 
>  查看所有镜像→ docker images <镜像名称 || 镜像ID>
> 
>  查看正在运行的容器→ docker ps
> 
> 查看所有的容器→ docker ps -a
> 
> 根据名称或id停止容器→ docker stop <容器名称 || 容器ID>
> 
> 根据名称或id启动容器→ docker start <容器名称 || 容器ID>
> 
> 根据名称或id重启容器→ docker restart <容器名称 || 容器ID>
> 
> 根据名称或id删除容器→ docker rm <容器名称 || 容器ID>
> 
> 查看容器详细信息→ docker inspect <容器名称 || 容器ID>
> 
> 可以查看容器内部的标准输出→ docker logs -f <容器名称 || 容器ID>
> 
> 进入容器内部→ docker exec -it <容器名称 || 容器ID>  bash
> 
> 从主机复制文件到容器内部→ docker cp <文件> <容器名称 || 容器ID>:<路径>
> 
> 从容器内部复制文件到主机→ docker cp <容器名称 || 容器ID>:<文件> <路径>
> 
> 查看所有容器的id→ docker ps -aq
> 
> 停止所有的容器→ docker stop $(docker ps -aq)
> 
> 删除所有的容器→ docker rm $(docker ps -aq)
> 
> 删除所有的镜像→ docker rmi $(docker images -q)
> 
> 删除所有none镜像→ docker rmi $(docker images -f "dangling=true" -q)
> 
> 删除所有不使用的镜像→ docker image prune -f -a
> 
> 删除所有停止的容器→ docker container prune -f
> 
> 创建自定义网络→ docker network create --subnet 172.18.0.0/24 static
> 
> 查看网络类型→ docker network ls
> 
> 删除所有没用的网络→ docker network prune -f
> 
> 清除build缓存→ docker builder prune