---
title: "Compose容器编排"
subtitle: ""
date: 2022-10-15T13:14:53+08:00
lastmod: 2022-10-15T13:14:53+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","云原生"]
featuredImage: ""
---

##  一、docker-compose是什么

之前使用 Docker 的时候，定义 Dockerfile 文件，然后使用 docker build、docker run 等命令操作容器。然而微服务架构的**应用系统一般包含若干个微服务，每个微服务一般都会部署多个实例，如果每个微服务都要手动启停，那么效率之低，维护量之大可想而知。**

Compose 是 Docker 公司推出的一个工具软件，**可以管理多个 Docker 容器组成一个应用，实现对docker容器的快速编排**。你需要写一个 YAML 格式的配置文件`docker-compose.yml`，来**定义一组相关联的应用容器为一个项目（project）**，写好多个容器之间的调用关系。然后，只要一个命令，就能同时启动/关闭这些容器。

compose安装步骤：

```bash
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

Compose 使用的三个步骤：

- 使用 Dockerfile 定义各个微服务并构建出对应的镜像文件。
- 使用 docker-compose.yml 定义构成应用程序的服务，这样它们可以在隔离环境中一起运行。
- 最后，执行 docker-compose up 命令来启动并运行整个应用程序。



## 二、compose常用命令

```bash
docker-compose -h                           # 查看帮助

docker-compose up                           # 启动所有docker-compose服务

docker-compose up -d                        # 启动所有docker-compose服务并后台运行

docker-compose down                         # 停止并删除容器、网络、卷、镜像。

docker-compose exec  yml里面的服务id                 # 进入容器实例内部  docker-compose exec docker-compose.yml文件中写的服务id /bin/bash

docker-compose ps                      # 展示当前docker-compose编排过的运行的所有容器

docker-compose top                     # 展示当前docker-compose编排过的容器进程

docker-compose logs  yml里面的服务id     # 查看容器输出日志

docker-compose config     # 检查配置

docker-compose config -q  # 检查配置，有问题才有输出

docker-compose restart   # 重启服务

docker-compose start     # 启动服务

docker-compose stop      # 停止服务
```

## 三、compose实战

有一个springboot项目，用到了MySQL和redis，要用docker部署。需要三个容器：一个放redis、一个放MySQL、一个放服务（jar包）。【实际情况的话，redis和mysql一般都不会放容器中。compose编排的基本是各个微服务。这里只是为了测试一下compose才这么去做的。】

### 不使用compose如何实现？

1. 制作Java服务的镜像：打jar包，编写Dockerfile文件，构建镜像。Dockerfile文件可以如下：

   ```toml
   # 基础镜像使用java
   FROM java:8
   # 作者
   MAINTAINER liaoyuan
   # VOLUME 指定临时文件目录为/tmp，在主机/var/lib/docker目录下创建了一个临时文件并链接到容器的/tmp
   VOLUME /tmp
   # 将jar包添加到容器中并更名为zzyy_docker.jar
   ADD docker_boot-0.0.1-SNAPSHOT.jar zzyy_docker.jar
   # 运行jar包
   RUN bash -c 'touch /zzyy_docker.jar'
   ENTRYPOINT ["java","-jar","/zzyy_docker.jar"]
   #暴露6001端口作为微服务
   EXPOSE 6001
   ```

2. 新建MySQL容器，建库建表。

3. 建立redis容器。

4. 运行Java服务容器。

**这么做产生了哪些问题？**

- 先后顺序必须固定，先redis+MySQL，再Java服务。
- 需要多个run命令，一个个去开启容器。
- 服务器间的停启，可能导致ip地址变动。（这可以通过自定义网络解决）

### 使用compose如何实现？

1. 编写docker-compose,yml文件

   ```yml
   version: "3"     # 指定compose的版本
   
   services:   # 定义一个项目的所有服务
   
     microService:  # 定义服务名
       image: zzyy_docker:1.6  # 由哪个镜像构建
       container_name: ms01  # 定义容器名称
       ports:
         - "6001:6001"
       volumes:
         - /app/microService:/data
       networks: 
         - liaoyuan_net   # 指定使用的网络
       depends_on:  # redis服务和myql服务构建完成后才启动microService服务
         - redis
         - mysql 
   
     redis: # 定义服务名
       image: redis:6.0.8
       ports:
         - "6379:6379"
       volumes:
         - /app/redis/redis.conf:/etc/redis/redis.conf
         - /app/redis/data:/data
       networks: 
         - liaoyuan_net
       command: redis-server /etc/redis/redis.conf
   
     mysql:
       image: mysql:5.7
       environment:
         MYSQL_ROOT_PASSWORD: '1234567'
         MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
         MYSQL_DATABASE: 'db2021'
         MYSQL_USER: 'liaoyuan'
         MYSQL_PASSWORD: '123'
       ports:
          - "3306:3306"
       volumes:
          - /app/mysql/db:/var/lib/mysql
          - /app/mysql/conf/my.cnf:/etc/my.cnf
          - /app/mysql/init:/docker-entrypoint-initdb.d
       networks:
         - liaoyuan_net
       command: --default-authentication-plugin=mysql_native_password #解决外部无法访问
    
   networks: 
      liaoyuan_net:  # 自定义网络
   ```

   `docker-compose,yml`文件的其他配置可见：https://cloud.tencent.com/developer/article/1982631

2. 制作Java服务的镜像：打jar包，编写Dockerfile文件，构建镜像。此时，Java服务的配置文件`application.yml`中，调用redis和MySQL的**ip地址可以用`docker-compose,yml`中定义的服务名来表示**。

3. 执行`docker-compose up -d`启动。

4. 进入MySQL容器，建库建表。

## 四、compose、swarm、k8s的比较

它们三者主要都是用于容器编排，区别是什么？

- compose定义的是单机里，一个项目中多个容器的编排。
- swarm也是docker公司出品的，解决的是多个主机间，docker容器集群的编排。但是不适合大型项目的管理。
- k8s比swarm还要高级，对于大型项目，是肯定需要靠k8s去管理。所以说就容器编排而言，k8s目前来说是终点。

参考连接：

> https://blog.csdn.net/qq_40585800/article/details/109120484
>
> https://blog.csdn.net/lgxzzz/article/details/124454585
>
> https://www.zhihu.com/question/312917608















