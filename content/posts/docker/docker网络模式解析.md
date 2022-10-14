---
title: "Docker网络模式解析"
subtitle: ""
date: 2022-10-13T15:52:37+08:00
lastmod: 2022-10-13T15:52:37+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","网络","云原生"]
---

## docker网络模式介绍

docker网络模式主要解决的是容器与容器间、外界与容器的通信问题。docker网络模式分为host、bridge、none、container四种模式，并且可以自定义网络。容器运行时使用哪种网络，可以通过`--network`来指定。

- none网络：**none网络就是什么都没有的网络。**一些对安全性要求高并且不需要联网的应用可以使用none网络。
- host网络：容器将不会获得一个独立的Network Namespace， 而是和宿主机共用一个Network Namespace。容器将不会虚拟出自己的网卡而是使用宿主机的IP和端口。
  - 因此容器运行时不用使用`-p`进行端口映射，使用了也不会生效。外部主机与容器可以直接通过主机的IP地址通信。
  - 使用Docker host的网络最大的好处就是性能，如果容器对网络传输效率有较高要求，则可以选择host网络。
  - 便之处就是牺牲一些灵活性，比如要**考虑端口冲突问题**，Docker host 上**已经使用的端口就不能再用**了。
- bridge网络：默认的网络模式。Docker 服务默认会在主机创建一个 docker0 网桥，docker run 的时候，没有指定network的话默认使用的网桥模式就是bridge，使用的就是docker0。Docker启动一个容器时会根据Docker网桥的网段分配给容器一个IP地址，称为Container-IP，同时Docker网桥是每个容器的默认网关。因为在同一宿主机内的容器都接入同一个网桥，这样容器之间就能够通过容器的Container-IP直接通信。
  - 所有使用bridge模式的容器都可以互相通过Container-IP进行通信，因为它们都连在docker0网桥上。其他的容器或者是外界，是不能通过Container-IP与对应容器通信的。
  - 使用bridge网络，外界如果想访问容器内的服务，必须要`-p`端口映射。
- container网络：新建的容器和已经存在的一个容器共享一个网络ip配置而不是和宿主机共享。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。可以通过`--network=container:容器名`指定。
  - 适合不同容器中的程序希望通过loopback高效快速地通信，比如web Server与App Server.
  - 希望监控其他容器的网络流量，比如运行在独立容器中的网络监控程序。
- 自定义网络：自定义网络会有自己的一个网桥，使用同一个自定义网络的容器可以直接相互通信，并且可以直接通过容器名进行通信。自定义网络默认使用的就是bridge，除了网桥用的不是docker0之外，其他都是一样的。

网络模式的具体情况可以参考：

- https://blog.csdn.net/heian_99/article/details/104914945
- https://blog.csdn.net/heian_99/article/details/104944575
- https://blog.csdn.net/heian_99/article/details/104993333



## 常用命令

```bash
docker network  --help
docker network ls   # 列出所有网络
docker network inspect 网络名   #查看网络具体情况
docker network rm 网络名 # 删除网络
docker network create 网络名 # 创建网络（自定义网络）
docker network connect 网络名 容器id #将容器连入一个网络当中
```
