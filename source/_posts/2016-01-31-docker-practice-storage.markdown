---
layout: post
title: "Docker Practice: Storage"
date: 2016-01-31 14:03:30 +0800
comments: true
categories:
---

## Docker基础

### 简介
[Docker](https://www.docker.com/what-docker)作为著名的开源容器引擎，在业界有着广泛的使用。
由于使用了现代Linux内核高级特性(如cgroup，namespace)，它可以更高校更安全地使用宿主机的计算、存储、网络等资源。
容器技术作为轻量级的虚拟化技术，相比传统的基于Hypervisor和VM的虚拟化更适合大规模的横向扩展。
容器技术独立与上层应用语言和底层操作系统，将代码和其所依赖的环境整体打包，实现了“构建一次，处处运行”的目标。
在构建应用方面，容器技术可以大大提升运维工作的效率，缩短应用开发上线发布周期，甚至重新定义了软件开发、测试、交付和部署的流程。

本文作为Docker的初级实践总结，描述了Docker的**安装及基本命令**，并以**Redis**和**PostgreSQL**的服务部署为例，记录Docker官方镜像服务部署的过程。
### 安装
在ubuntu 14.04下安装参考[安装文档](https://docs.docker.com/engine/installation/ubuntulinux/)，过程如下:
```
sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D
sudo su -c "echo 'deb https://apt.dockerproject.org/repo ubuntu-trusty main' > /etc/apt/sources.list.d/docker.list"
sudo apt-get update
sudo apt-get install docker-engine
```


#### 免sudo执行docker
将指定用户加入docker组，并重新登录
```
sudo usermod -aG docker $USER
```

#### 更改docker默认数据目录
更改*/etc/default/docker*
```
DOCKER_OPTS="-g /data/var/lib/docker"
```
重启服务
```
sudo service docker restart
```

### 基本操作

常用启动容器实例模式

后台执行容器 **-d**
```
docker run -d ubuntu /bin/bash -c "vmstat 5"  # docker logs -f 查看输出
```
执行完命令即时删除 **--rm**
```
docker run --rm ubuntu /bin/bash -c "uname -a"
```
交互式 **-it**
```
docker run -it ubuntu /bin/bash   # 操作完成后容器停止，可以start，再attach
docker exec -it CONTAINER_NAME /bin/bash # shell到后台运行的容器
```

查看子命令使用: docker help inspect

列出所有运行的容器列表: docker ps

删除所有容器: docker rm $(docker ps -aq)

搜索镜像: [https://hub.docker.com/](https://hub.docker.com/)

## Redis服务部署

[Redis](http://redis.io/)是开源的内存数据结构存储引擎，常用来实现缓存服务、键值存储引擎以及Pub/Sub引擎等。用Docker部署Redis服务非常便捷。

```
docker run --name myapp-redis -p 127.0.0.1:49514:6379 -d redis
```

Docker会做以下工作:

在本地需找官方[redis官方镜像](https://hub.docker.com/_/redis/)，如果没有则下载；用官方镜像启动名为*myapp-redis*的容器实例；将实例开放的默认redis服务6379端口映射到宿主系统的本地49514端口。

查看redis服务的日志

```
docker logs -f myapp-redis
```

验证:通过宿主系统的redis客户端访问
```
redis-cli -p 49514
```

## PostgreSQL服务部署
[PostgreSQL](http://www.postgresql.org/)是开源的关系型数据库。用Docker部署PostgreSQL服务同样直观便捷。

```
docker run --name myapp-postgres -p 127.0.0.1:49513:5432    \
-v $(pwd)/pg-data:/var/lib/postgresql/data                  \
-e POSTGRES_PASSWORD=postgres                               \
-d postgres
```

同样的，docker会下载[postgresql官方镜像](https://hub.docker.com/_/postgres/)，启动myapp-postgres实例，并将服务端口映射到宿主机的49513端口。

注意这里用到**-v**参数，可以为容器的volume(数据卷)指定映射目录。
官方的postgresql镜像将*/var/lib/postgresql/data/*目录作为volume处理，使得单独存储数据变得简单。
可以简单的理解为将*$(pwd)/pg-data*作为设备挂载到容器的*/var/lib/postgresql/data/*目录。这样，备份和迁移都很方便，不依赖容器。


**-e**参数为创建容器时指定的环境变量，这里将postrgres超级用户的密码设置为postgres。

然后便可以在宿主机通过posgresql的客户端(如psql、createuser、createdb)访问服务了。

```
# 以postgres超级用户测试数据连接
PGPASSWORD=postgres psql -h localhost -p 49513 -U postgres -e
# 创建用户myapp，并设置访问权限(可以创建角色和数据库)及密码
PGPASSWORD=postgres createuser -h localhost -p 49513 -U postgres -rdeP myapp
# 以新用户创建同名数据库
PGPASSWORD=myapp-pass createdb -h localhost -p 49513 -U myapp -e myapp
```

官方镜像还支持若干个环境变量，如初始superuser的用户名及密码等等。以下一个命令即可完成以上流程。
```
docker run --name myapp-postgres -p 127.0.0.1:49513:5432    \
-v $(pwd)/pg-data:/var/lib/postgresql/data                  \
-e POSTGRES_PASSWORD=myapp-pass -e POSTGRES_USER=myapp      \
-d postgres
```

## 总结
通过以上两个例子，创建了两个容器实例，分别是redis服务和有分离数据卷的postgresql服务。
通过应用Docker容器技术，最核心的是简化测试交付部署流程，并使这些流程更容易地实现自动化。
从而使开发者更多地专注于业务逻辑，加速其价值提升。
