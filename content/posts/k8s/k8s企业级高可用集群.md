---
title: "K8s企业级高可用集群"
subtitle: ""
date: 2023-03-15T13:33:14+08:00
lastmod: 2023-03-15T13:33:14+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["k8s"]
tags: ["k8s"]
featuredImage: ""
---

之前部署的k8s集群是单master结构，显然不可能用于生产中。之前搭建的时候就在想，生产环境中的k8s集群仅仅是多个master节点那么简单吗？显然不是。这两天翻了很多博客，也看了k8s官网，对如何构建一个高可用集群已经心中有数了（虽然我还没搭过）。

### 一、搭建方式

生产环境搭建k8s集群的方式一般是两种：kubeadm方式、二进制方式，对比一下就会发现，二进制部署实在太太太麻烦了。因此，我会选择用kubeadm进行部署。

Kubernetes 采用的是中心辐射型（Hub-and-Spoke）API 模式。 所有从集群（或所运行的 Pods）发出的 API 调用都终止于 API server，而API Server直接与ETCD数据库通讯。若仅部署单一的API server ，当API server所在的 VM 关机或者 API 服务器崩溃将导致不能停止、更新或者启动新的 Pod、服务或副本控制器；而ETCD存储若发生丢失，API 服务器将不能启动。若要实现Kubernetes高可用，则Master组件必须是冗余的，尤其是ETCD、ApiServer必须部署在多个节点上。

**在正式环境中确保Master高可用，并启用访问安全机制，至少应包括以下几个方面**：

1. Master的kube-apiserver、kube-controller-mansger和kube-scheduler服务至少三个结点的多实例方式部署。

2. ETCD至少以3个结点的集群模式部署
3. Load Balance使用双机热备，向Node暴露虚拟IP作为入口地址，供客户端访问。负载均衡节点承担着Worker Node集群和Master集群通讯的职责。

细节架构图如下：

![image-20200406232838722](/k8s高可用集群/架构.png)

为了看的更清晰，服务器情况如下：

![image-20200406232838722](/k8s高可用集群/node.png)

为什么master和etcd最少要三个？这和redis集群master节点最少三个的原因是一样的，都是因为选举。负载均衡是为了node节点与master节点的通信，若不配置，则所有node只能和一台master通信，那台master挂了，集群也就挂了，因此必须配置。VIP是node访问master的地址，用于负载均衡。负载均衡与VIP后面会细讲。







