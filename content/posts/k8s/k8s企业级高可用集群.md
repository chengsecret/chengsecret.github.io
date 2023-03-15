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

**细节架构图如下：**

![image-20200406232838722](/k8s高可用集群/架构.png)

**为了看的更清晰，服务器情况如下：**

![image-20200406232838722](/k8s高可用集群/node.png)

为什么master和etcd最少要三个？这和redis集群master节点最少三个的原因是一样的，都是因为选举。负载均衡是为了node节点与master节点的通信，若不配置，则所有node只能和一台master通信，那台master挂了，集群也就挂了，因此必须配置。VIP是node访问master的地址，用于负载均衡。负载均衡与VIP后面会细讲。

### 二、etcd的两种部署方式

#### 1.堆叠（Stacked）etcd 拓扑

使用堆叠（stacked）控制平面节点，其中 etcd 节点与控制平面节点共存。每个控制平面节点创建一个本地 etcd 成员（member），这个 etcd 成员只与该节点的 `kube-apiserver` 通信。 

[官网](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology/)描述如下：

>这种拓扑将控制平面和 etcd 成员耦合在同一节点上。相对使用外部 etcd 集群， 设置起来更简单，而且更易于副本管理。
>
>然而，堆叠集群存在耦合失败的风险。如果一个节点发生故障，则 etcd 成员和控制平面实例都将丢失， 并且冗余会受到影响。你可以通过添加更多控制平面节点来降低此风险。
>
>因此，你应该为 HA 集群运行至少三个堆叠的控制平面节点。
>
>**这是 kubeadm 中的默认拓扑**。当使用 `kubeadm init` 和 `kubeadm join --control-plane` 时， 在控制平面节点上会自动创建本地 etcd 成员。

#### 2.外部 etcd 拓扑

使用外部 etcd 节点，其中 etcd 在与控制平面不同的节点上运行。

[官网](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology/)描述如下：

> 就像堆叠的 etcd 拓扑一样，外部 etcd 拓扑中的每个控制平面节点都会运行 `kube-apiserver`、`kube-scheduler` 和 `kube-controller-manager` 实例。 同样，`kube-apiserver` 使用负载均衡器暴露给工作节点。但是 etcd 成员在不同的主机上运行， 每个 etcd 主机与每个控制平面节点的 `kube-apiserver` 通信。
>
> 这种拓扑结构解耦了控制平面和 etcd 成员。因此它提供了一种 HA 设置， 其中失去控制平面实例或者 etcd 成员的影响较小，并且不会像堆叠的 HA 拓扑那样影响集群冗余。
>
> 但此拓扑需要两倍于堆叠 HA 拓扑的主机数量。 具有此拓扑的 HA 集群至少需要三个用于控制平面节点的主机和三个用于 etcd 节点的主机。



chatGPT给出的两者间的比较：

> **单独部署：**
>
> 单独部署是指将etcd作为一个独立的进程运行在集群之外。这种方式的优点是：
>
> - 独立部署可以为etcd提供更好的资源隔离，以保障其稳定性。
> - 可以单独升级etcd而无需影响其他服务。
> - 可以将etcd部署到其他机器或云服务上，以降低硬件成本。
>
> 不过，单独部署的缺点也是显而易见的：
>
> - 部署和维护etcd需要一些额外的工作。
> - etcd的性能和可用性可能会受到网络延迟和带宽限制的影响。
>
> **内嵌部署：**
>
> 内嵌部署是将etcd作为一个容器或进程内的模块运行在Kubernetes集群中。这种方式的优点是：
>
> - 简化了部署和维护etcd的工作，因为etcd与Kubernetes一起打包并升级。
> - 可以直接访问Kubernetes集群中的网络，以获得更好的性能和可用性。
>
> 不过，内嵌部署的缺点也是显而易见的：
>
> - etcd与其他Kubernetes组件共享资源，可能会受到其他组件的影响。
> - 如果Kubernetes本身出现故障，则etcd也可能受到影响。
>
> 综合来说，单独部署适用于那些需要对etcd进行更精细化控制和管理的场景，而内嵌部署则更适用于那些需要快速部署和简化管理的场景。

### 三、高可用负载均衡器

[官网](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/ha-topology/)文档主要是解决了高可用场景下apiserver与etcd集群的关系, 三master节点防止单点故障。但是集群master访问接口不可能将三个apiserver都暴露出去，一个挂掉时还是不能自动切换到其他节点。官方文档只提到了一句“使用负载均衡器将apiserver暴露给工作程序节点”，而这恰恰是生产环境中需要解决的重点问题。

**Notes:** **此处的负载均衡器并不是kube-proxy，也不是提供service外部访问的Load Balance，此处的Load Balancer是针对apiserver的。**

![image-20200406232838722](/k8s高可用集群/vip.png)

1. 图中的keepalived+haproxy就是充当负载均衡器的，也可以用nginx+keepalived实现。当然也可以购买云服务商的负载均衡器。公有云部署的话要用公有云自带的负载均衡，比如阿里云的 SLB，腾讯云的 ELB，用来替代 haproxy 和 keepalived ，因为公有云大部分都是不支持 keepalived 的。结合上面的图就可以发现，VIP是给工作节点来访问的。
2. 由负载均衡器提供一个vip（虚拟ip），流量负载到keepalived master节点上。vip只要和网络中的其他ip不冲突即可。
3. 当keepalived节点出现故障, vip自动漂到其他可用节点。所以说负载均衡器也要建一个集群，不然负载均衡器挂了集群也就挂了。
4. haproxy负责将流量负载到apiserver节点。
5. 负载均衡器如何部署，本文不做探究。

[官网]()中使用kubeadm部署高可用k8s集群时，在`kubeadm init`命令中使用了负载均衡器：

```bash
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs 
#实际部署过程中，还要其他参数，如--pod-network-cidr
```

[官网](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)是这么描述的：

> (推荐）如果计划将单个控制平面 kubeadm 集群升级成高可用， 你应该指定 `--control-plane-endpoint` 为所有控制平面节点设置共享端点。 端点可以是负载均衡器的 DNS 名称或 IP 地址。
>
> kubeadm 不支持将没有 `--control-plane-endpoint` 参数的单个控制平面集群转换为高可用性集群。

`--control-plane-endpoint`也可以指定为一个master节点的ip，单master应该默认就是如此。当有多个master时，若指定的是中一个master的ip地址，那个master挂了，集群就狗带了。这就是要配置负载均衡器的原因了。

使用``kubeadm``命令添加master节点：

```bash
# 原master执行 获取certificate-key
kubeadm init phase upload-certs --upload-certs
# 原master执行 获取token
kubeadm token create --print-join-command

# 想加入集群的master执行
kubeadm join <原masterIP>:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07
```

- 这个 `--control-plane` 标志通知 `kubeadm join` 创建一个新的控制平面。
- `--certificate-key ...` 将导致从集群中的 `kubeadm-certs` Secret 下载控制平面证书并使用给定的密钥进行解密。

### 四、service LoadBalancer类型

为什么还将这个呢，因为这个load balance容易跟上文的负载均衡器搞混，我本人一开始就以为是同一个。

**这里的LoadBalancer是为了暴露内部服务给外部访问的，而上文的负载均衡器是为访问master节点中的apiserver。**

k8s中创建service时，需要指定`type`类型，可以分别指定`ClustrerIP,NodePort,LoadBalancer`三种，其中前面两种无论在内网还是公网环境下使用都很常见，只有`LoadBalancer`大部分情况下只适用于支持外部负载均衡器的云提供商（AWS,阿里云,华为云等）使用。

如果想要在内网环境中使用`type=LoadBalancer`就需要部署另外的插件，比如MetalLB组件。

> 参考文章：

- [ MetalLB](https://zhuanlan.zhihu.com/p/266422557)
- [K8S集群中的服务发现和流量暴露机制](https://www.zhihu.com/question/526869937)
- [利用 kubeadm 创建高可用集群](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [kubeadm安装k8s高可用集群](https://www.yuque.com/fairy-era/yg511q/gf89ub)
- [Kubernetes核心架构与高可用集群详解](https://zhuanlan.zhihu.com/p/444114515)
- [k8s生产环境](https://kubernetes.io/zh-cn/docs/setup/production-environment/)
- [k8s高可用部署：keepalived + haproxy](https://www.kubernetes.org.cn/6964.html)

























