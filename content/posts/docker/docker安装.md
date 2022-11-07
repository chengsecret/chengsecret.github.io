---
title: "Docker入门"
subtitle: ""
date: 2022-09-18T20:29:23+08:00
lastmod: 2022-09-18T20:29:23+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","入门","云原生"]
---

![docker](/docker安装/docker01.png)

## docker简介                                                                 

- Docker 是基于容器技术的轻量级**虚拟化解决方案**，是一种**容器化技术**的落地。容器作为一类操作系统层面的虚拟化技术，其目标是在单一Linux主机交付多套隔离性环境，容器共享同一套主机操作系统内核。
- Docker的主要目标是“**Build，Ship and Run Any App,Anywhere**”，也就是通过对应用组件的封装、分发、部署、运行等生命周期的管理，使用户的APP（可以是一个WEB应用或数据库应用等等）及其运行环境能够做到“**一次镜像，处处运行**”。
- Docker将应用打成镜像，通过镜像成为运行在Docker容器上面的实例，而 Docker容器在任何操作系统上都是一致的，这就实现了**跨平台、跨服务器**。只需要一次配置好环境，换到别的机子上就可以一键部署好，大大简化了操作。

一句话，**docker是解决了运行环境和配置问题，方便做持续集成并有助于整体发布的容器虚拟化技术**。



## 与传统虚拟机的对比

- 传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；

- 容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

- 每个容器之间互相隔离，每个容器有自己的文件系统 ，容器之间进程不会相互影响，能区分计算资源。



## docker三组件

- 镜像：镜像可以用来创建Docker容器的。一个镜像可以包含一个完整的操作系统环境和用户需要的其它应用程序，docker的镜像是只可读的，一个镜像可以创建多个容器。

- 容器：容器是镜像创建的实例。它可以被启动、开始、停止、删除。每个容器都是相互隔离的、保证安全的平台。容器，**是一个运行时环境**。

- 仓库：仓库是集中存放镜像文件的场所。每个仓库中又包含了多个镜像，每个镜像有不同的标签（tag）。

Docker 本身则是一个容器运行载体或称之为管理引擎。是一个Client-Server结构的系统，Docker守护进程运行在主机上， 然后通过Socket连接从客户端访问，守护进程从客户端接受命令并管理运行在主机上的容器。 



## Centos安装docker

### 前提条件

1. CentOS 仅发行版本中的内核支持 Docker。Docker 运行在CentOS 7 (64-bit)上，要求系统为64位、Linux系统内核版本为 3.8以上，要选用Centos7.x+。查看内核版本：

   ```bash
   cat /etc/redhat-release
   ```

2. 卸载旧版本

   ```bash
   sudo yum remove docker \
                     docker-client \
                     docker-client-latest \
                     docker-common \
                     docker-latest \
                     docker-latest-logrotate \
                     docker-logrotate \
                     docker-engine
   ```

3. yum安装gcc

   ```bash
   yum -y install gcc
   yum -y install gcc-c++
   ```

### 安装docker

1. 安装软件包

   ```bash
   sudo yum install -y yum-utils
   ```

2. 设置镜像仓库(官网的很慢，不推荐)

   ```bash
   sudo yum-config-manager \
       --add-repo \
       http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
   ```

3. 先更新yum软件包索引

   ```bash
   yum makecache fast
   ```

4. 安装docker-ce（社区版-免费的）

   ```bash
   sudo yum install docker-ce docker-ce-cli containerd.io
   ```

5. 启动docker

   ```bash
   sudo systemctl start docker
   sudo systemctl enable  docker
   ```

   通过`docker version `判断是否启动成功。

6. 将用户加入到docker组

   ```bash
   # 添加docker用户组，一般已存在，不需要执行
   sudo groupadd docker
   # 将登陆用户加入到docker用户组中
   sudo gpasswd -a $USER docker
   # 更新用户组
   newgrp docker
   # 测试docker命令是否可以使用sudo正常使用
   docker version
   ```

7. 查看镜像(空的)

   ```bash
   docker images
   ```

8. 卸载（只教别做）

   ```bash
   systemctl stop docker
   sudo yum remove docker-ce docker-ce-cli containerd.io
   sudo rm -rf /var/lib/docker
   sudo rm -rf /var/lib/containerd
   ```

### 阿里云镜像加速

使用阿里云镜像仓库，可以配置阿里云的镜像加速，速度更快。如果不配，一些操作可能会因为网速不行而失败。

1. 进入阿里云官网，登录注册账号，搜索**容器镜像服务**，依此点击**镜像工具–镜像加速器–选择CentOS**

   ![阿里云镜像加速](/docker安装/ali.png)

2. 按阿里云官网说，创建目录

   ```bash
   sudo mkdir -p /etc/docker
   ```

3. 编辑配置文件

   ```bash
   sudo tee /etc/docker/daemon.json <<-'EOF'
   {
     "registry-mirrors": ["xxxx替换自己的加速地址"]
   }
   EOF
   ```

4. 重启服务

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart docker
   ```

## HelloWord

使用docker命令，运行hello-word镜像：

```bash
docker run hello-world
```

查看镜像：

```bash
docker images 
```

会发现多了一个hello-world镜像，这其实就是docker从远程docker仓库中下载下来的。

查看容器：

```bash
#查看运行的容器
docker ps 
#查看所有的容器
docker ps -a
```

会发现多了hello-word容器。通过镜像，即可运行相应的容器！
