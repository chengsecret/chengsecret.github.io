---
title: "Docker常用容器安装"
subtitle: ""
date: 2022-09-22T15:28:11+08:00
lastmod: 2022-09-22T15:28:11+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","入门","云原生"]
---



运行一个容器不仅仅是`docker run `，还需要考虑端口、数据的持久化、容器自身的特殊命令等等，这样的容器在生产环境中才有实际意义。



## 安装tomcat

> 安装命令

```bash
docker pull billygoo/tomcat8-jdk8
docker run -d -p 8080:8080 --name mytomcat8 billygoo/tomcat8-jdk8
```

Linux系统安装了浏览器的话，就可以通过http://localhost:8080来访问。

> 问题：访问不了？

- 可能没有映射端口，即： `-p 8080:8080`

- 可能下载的tomcat版本过新，需要把webapps.dist目录名修改为webapps即可。

  ```bash
  docker exec -it 容器id /bin/bash
  ls -l
  rm -r webapps  #是一个空文件夹
  mv webapps.dist webapps #给webapps.dist 文件夹换名字
  ```

> 外网如何访问？

- 如果服务器是云服务器，需要开启**安全组8080端口**和**防火墙8080端口**
- 如果是本地虚拟机，开启**防火墙8080端口**，如果还是访问不了就重启一下（我重启后才能访问的）。

## Centos一些常用防火墙命令

```bash
# 开放端口
firewall-cmd --zone=public --add-port=5672/tcp --permanent
firewall-cmd --reload   # 配置立即生效

#关闭端口
firewall-cmd --zone=public --remove-port=5672/tcp --permanent 
firewall-cmd --reload   # 配置立即生效

# 查看防火墙状态
firewall-cmd --state

#查看想开的端口是否已开
firewall-cmd --query-port=8888/tcp

#查看防火墙所有开放的端口
firewall-cmd --zone=public --list-ports

# 开启防火墙
systemctl start firewalld.service  		#启动firewall
systemctl enable firewalld.service		#开机时启动firewall

# 关闭防火墙
systemctl stop firewalld.service        #停止firewall
systemctl disable firewalld.service		#禁止firewall开机启动

#重启防火墙
systemctl restart firewalld.service
```

















