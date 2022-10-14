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

网络模式的具体情况可以参考下面的博客（不想自己写一遍了。。。太长了。。。）：

- https://blog.csdn.net/heian_99/article/details/104914945
- https://blog.csdn.net/heian_99/article/details/104944575
- https://blog.csdn.net/heian_99/article/details/104993333

个人总结如下，如果有不正确的地方，或者有补充，欢迎在下方评论。

> 解决的问题

docker网络模式主要解决的是容器与容器间、外界与容器的通信问题。根据实际情况，运行容器时(`docker run`)选择适合的网络模式，可以通过`--network`来指定。

> bridge模式

Docker 服务默认会在主机创建一个 docker0 网桥，docker run 的时候，没有指定network的话**默认使用的网桥模式就是bridge，使用的网桥就是docker0**。

- 所有**使用bridge模式的容器都能通过容器的ip地址通信**，因为都使用了docker0网桥。容器的ip地址范围是由docker0决定的。
- **容器想要被外界访问，run的时候需要`-p`进行端口映射**，通过`主机IP+映射端口`进行访问。

> host模式

容器将不会获得一个独立的Network Namespace， 而是和宿主机共用一个Network Namespace。**容器将不会虚拟出自己的网卡而是使用宿主机的IP和端口。**

- 外部主机与容器可以**直接通过主机的IP地址通信**，容器运行时不用使用`-p`进行端口映射，使用了也不会生效。
- 使用Docker host的网络**最大的好处就是性能**，如果容器对网络传输效率有较高要求，则可以选择host网络。
- 不便之处就是牺牲一些灵活性，比如要**考虑容器间以及容器与主机的端口冲突问题**，Docker host 上**已经使用的端口就不能再用**了。

> none网络

**none网络就是什么都没有的网络。** 一些对安全性要求高并且不需要联网的应用可以使用none网络。

> container网络

**新建的容器和已经存在的一个容器共享一个网络ip配置而不是和宿主机共享**，可以通过`--network=container:容器名`指定。新创建的容器不会创建自己的网卡，配置自己的IP，而是和一个指定的容器共享IP、端口范围等。同样，两个容器除了网络方面，其他的如文件系统、进程列表等还是隔离的。

- 适合不同容器间**高效快速地通信**，比如web Server与App Server。
- 希望**监控其他容器的网络流量**，比如运行在独立容器中的网络监控程序。

> 自定义网络

用户也可以根据业务需要创建user-defined网络，使用`docker network create 网络名`创建，使用`--network 网络名`指定网络。自定义网络会有自己的一个网桥，**使用同一个自定义网络的容器可以通过容器IP地址相互通信**，并且可以**直接通过容器名进行通信**。自定义网络默认使用的就是bridge驱动，除了网桥用的不是docker0之外，其他都是和bridge模式一样的。自定义网络还有overlay 和macvlan这两种网络驱动，用于创建跨主机的网络。



## 常用命令

```bash
docker network  --help
docker network ls   # 列出所有网络
docker network inspect 网络名   #查看网络具体情况
docker network rm 网络名 # 删除网络
docker network create 网络名 # 创建网络（自定义网络）
docker network connect 网络名 容器id #将容器连入一个网络当中
```
