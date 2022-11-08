---
title: "K8s介绍与环境安装"
subtitle: ""
date: 2022-11-07T13:31:29+08:00
lastmod: 2022-11-07T13:31:29+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["k8s"]
tags: ["k8s","云原生"]
featuredImage: ""
---

# 一. Kubernetes介绍

## 1 应用部署方式演变

在部署应用程序的方式上，主要经历了三个时代：

- **传统部署**：互联网早期，会直接将应用程序部署在物理机上

  > 优点：简单，不需要其它技术的参与
  >
  > 缺点：不能为应用程序定义资源使用边界，很难合理地分配计算资源，而且程序之间容易产生影响

- **虚拟化部署**：可以在一台物理机上运行多个虚拟机，每个虚拟机都是独立的一个环境

  > 优点：程序环境不会相互产生影响，提供了一定程度的安全性
  >
  > 缺点：增加了操作系统，浪费了部分资源

- **容器化部署**：与虚拟化类似，但是共享了操作系统

  > 优点：
  >
  > 可以保证每个容器拥有自己的文件系统、CPU、内存、进程空间等
  >
  > 运行应用程序所需要的资源都被容器包装，并和底层基础架构解耦
  >
  > 容器化的应用程序可以跨云服务商、跨Linux操作系统发行版进行部署



**容器化部署方式给带来很多的便利，但是也会出现一些问题**，比如说：

- 一个容器故障停机了，怎么样让另外一个容器立刻启动去替补停机的容器
- 当并发访问量变大的时候，怎么样做到横向扩展容器数量

这些容器管理的问题统称为**容器编排**问题，为了解决这些容器编排问题，就产生了一些容器编排的软件：

- **Swarm**：Docker自己的容器编排工具
- **Mesos**：Apache的一个资源统一管控的工具，需要和Marathon结合使用
- **Kubernetes**：Google开源的的容器编排工具(最强大，应用最广)

## 2 kubernetes简介

![image-20200406232838722](/k8s介绍与环境安装/image-20200406232838722.png)

kubernetes，是一个全新的基于容器技术的分布式架构领先方案，是谷歌严格保密十几年的秘密武器----Borg系统的一个开源版本，于2014年9月发布第一个版本，2015年7月发布第一个正式版本。

kubernetes的本质是**一组服务器集群**，它可以在集群的每个节点上运行特定的程序，来对节点中的容器进行管理。目的是**实现资源管理的自动化**，主要提供了如下的主要功能：

- **自我修复**：一旦某一个容器崩溃，能够在1秒中左右迅速启动新的容器
- **弹性伸缩**：可以根据需要，自动对集群中正在运行的容器数量进行调整
- **服务发现**：服务可以通过自动发现的形式找到它所依赖的服务
- **负载均衡**：如果一个服务起动了多个容器，能够自动实现请求的负载均衡
- **版本回退**：如果发现新发布的程序版本有问题，可以立即回退到原来的版本
- **存储编排**：可以根据容器自身的需求自动创建存储卷

## 3 kubernetes组件

一个kubernetes集群主要是由**控制节点(master)**、 **工作节点(node)** 构成，每个节点上都会安装不同的组件。

**master：集群的控制平面，负责集群的决策 ( 管理 )**

> **ApiServer** : 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API注册和发现等机制
>
> **Scheduler** : 负责集群资源调度，按照预定的调度策略将Pod调度到相应的node节点上
>
> **ControllerManager** : 负责维护集群的状态，比如程序部署安排、故障检测、自动扩展、滚动更新等
>
> **Etcd** ：负责存储集群中各种资源对象的信息

**node：集群的数据平面，负责为容器提供运行环境 ( 干活 )**

> **Kubelet** : 负责维护容器的生命周期，即通过控制docker，来创建、更新、销毁容器
>
> **KubeProxy** : 负责提供集群内部的服务发现和负载均衡
>
> **Docker** : 负责节点上容器的各种操作

![image-20200406184656917](/k8s介绍与环境安装/image-20200406184656917.png)

下面，以部署一个nginx服务来说明kubernetes系统各个组件调用关系：

1. 首先要明确，一旦kubernetes环境启动之后，master和node都会将自身的信息存储到etcd数据库中

2. 一个nginx服务的安装请求会首先被发送到master节点的apiServer组件

3. apiServer组件会调用scheduler组件来决定到底应该把这个服务安装到哪个node节点上

   在此时，它会从etcd中读取各个node节点的信息，然后按照一定的算法进行选择，并将结果告知apiServer

4. apiServer调用controller-manager去调度Node节点安装nginx服务

5. kubelet接收到指令后，会通知docker，然后由docker来启动一个nginx的pod

   pod是kubernetes的最小操作单元，容器必须跑在pod中

6. 至此，一个nginx服务就运行了，如果需要访问nginx，就需要通过kube-proxy来对pod产生访问的代理

这样，外界用户就可以访问集群中的nginx服务了。

## 4 kubernetes概念

**Master**：集群控制节点，每个集群需要至少一个master节点负责集群的管控

**Node**：工作负载节点，由master分配容器到这些node工作节点上，然后node节点上的docker负责容器的运行

**Pod**：kubernetes的最小控制单元，容器都是运行在pod中的，一个pod中可以有1个或者多个容器

**Controller**：控制器，通过它来实现对pod的管理，比如启动pod、停止pod、伸缩pod的数量等等

**Service**：pod对外服务的统一入口，其下面可以维护同一类的多个 Pod 

**Label**：标签，用于对pod进行分类，同一类pod会拥有相同的标签

**NameSpace**：命名空间，用来隔离pod的运行环境



# 二. kubernetes集群环境搭建

## 1.  环境规划

### 1.1集群类型

- Kubernetes集群大致分为两类：一主多从和多主多从。

- 一主多从：一个Master节点和多台Node节点，搭建简单，但是有**单机故障风险**，适合用于测试环境。

- 多主多从：多台Master和多台Node节点，搭建麻烦，安全性高，适合用于生产环境。

> 因为刚学，我还是准备在VMware装三个虚拟机进行测试，采用是**一主多从**类型的集群。等过一遍k8s后，再用两台腾讯云+一台实验室自己配的服务器，搭建生产环境的k8s。



### 1.2安装方式

kubernetes有多种部署方式，目前主流的方式有kubeadm、minikube、二进制包。

- ① minikube：一个用于快速搭建单节点的kubernetes工具。

- ② kubeadm：一个用于快速搭建kubernetes集群的工具。

- ③ 二进制包：从官网上下载每个组件的二进制包，依次去安装，此方式对于理解kubernetes组件更加有效。

我们需要安装kubernetes的集群环境，但是又不想过于麻烦，所以选择kubeadm方式。可以参考[官网](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)安装。



### 1.3 主机规划

| 角色   | IP地址         | 操作系统  | 配置                    |
| ------ | -------------- | --------- | ----------------------- |
| Master | 192.168.18.130 | CentOS7.9 | 2核CPU，2G内存，20G硬盘 |
| Node1  | 192.168.18.103 | CentOS7.9 | 2核CPU，2G内存，20G硬盘 |
| Node2  | 192.168.18.104 | CentOS7.9 | 2核CPU，2G内存，20G硬盘 |

每台机器要求：

- cpu 两核以上
- 内存 至少2GB
- 三台机器网络要能互通（公网和内网都可以）
- 系统内核版本在7.5以上

其他要求（在安装过程中进行即可，见下节`环境搭建`）：

- 开启机器上的某些端口，见[k8s端口](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)。也可以直接关掉防火墙
- 节点之中不可以有重复的主机名、MAC 地址或 product_uuid
- 禁用SELinux
- 关闭swap分区
- 时间需要同步

## 2. 环境搭建

最终目标：

- 在所有节点上安装Docker 和kubeadm
- 部署Kubernetes Master
- 部署容器网络插件
- 部署Kubernetes Node，将节点加入Kubernetes 集群中
- 部署Dashboard Web 页面，可视化查看Kubernetes 资源

### 2.1 环境初始化

**如果没有特殊说明，需要在三台主机上都操作**。

1. 检查操作系统的版本（要求操作系统的版本至少在7.5以上）

   ```bash
   cat /etc/redhat-release
   ```

2. **关闭防火墙**（如果不关闭防火墙，就需要开启对应端口，见[k8s端口](https://kubernetes.io/zh-cn/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#check-required-ports)，目前没试过开端口的方式，以后在服务器部署k8s的时候试试）

   ```bash
   systemctl stop firewalld
   systemctl disable firewalld
   systemctl status firewalld
   ```

3. **设置主机名**。防止三台主机的主机名相同

   ```bash
   # 192.168.18.130执行
   hostnamectl set-hostname master
   #192.168.18.103执行
   hostnamectl set-hostname node1
   # 192.168.18.104执行
   hostnamectl set-hostname node2
   ```

4. **配置主机名解析**。为了方便后面集群节点间的直接调用，需要配置一下主机名解析，企业中推荐使用内部的DNS服务器

   ```bash
   cat >> /etc/hosts<<EOF
   192.168.18.130 master
   192.168.18.103 node1
   192.168.18.104 node2
   EOF
   ```

5. **配置时间同步**。kubernetes要求集群中的节点时间必须精确一致，所以在每个节点上添加时间同步。企业中建议配置内部的会见同步服务器

   ```bash
   yum install ntpdate -y
   # 使用阿里服务器进行时间更新
   ntpdate ntp1.aliyun.com
   #查看当前时间
   date
   ```

   如果是杭电佬，主机使用的是hdu内网，上述同步时间命令极有可能失败（hdu内网是真的辣鸡），暂时没找到解决办法。（我是在虚拟机安装k8s的，装虚拟机的时候同步好了时间）

6. **关闭selinux**。让容器可以读取主机文件系统

   ```bash
   sed -i 's/enforcing/disabled/' /etc/selinux/config #永久关闭
   ```

7. **关闭swap分区**。swap分区指的是虚拟内存分区，它的作用是物理内存使用完，之后将磁盘空间虚拟成内存来使用，启用swap设备会对系统的性能产生非常负面的影响，因此kubernetes要求每个节点都要禁用swap设备，但是如果因为某些原因确实不能关闭swap分区，就需要在集群安装过程中通过明确的参数进行配置说明

   ```bash
   free -m  # 查看swap分区
   swapoff -a
   sed -ri 's/.*swap.*/#&/' /etc/fstab #永久关闭
   free -m 
   ```

8. 修改linux的内核参数，将桥接的IPv4流量传递到iptables的链

   ```bash
   #添加网桥过滤和地址转发功能
   cat > /etc/sysctl.d/k8s.conf << EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   net.ipv4.ip_forward = 1
   EOF
   # 加载br_netfilter模块
   modprobe br_netfilter
   # 查看是否加载
   lsmod | grep br_netfilter
   # 生效
   sysctl --system
   ```

9. **开启ipvs**。在Kubernetes中Service有两种带来模型，一种是基于iptables的，一种是基于ipvs的两者比较的话，ipvs的性能明显要高一些，但是如果要使用它，需要手动载入ipvs模块

   ```bash
   #安装ipset和ipvsadm
   yum -y install ipset ipvsadm
   #添加需要加载的模块写入脚本文件
   cat > /etc/sysconfig/modules/ipvs.modules <<EOF
   #!/bin/bash
   modprobe -- ip_vs
   modprobe -- ip_vs_rr
   modprobe -- ip_vs_wrr
   modprobe -- ip_vs_sh
   modprobe -- nf_conntrack_ipv4
   EOF
   # 授权、运行、检查是否加载
   chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
   #检查是否加载
   lsmod | grep -e ipvs -e nf_conntrack_ipv4
   ```

10. 重启

    ```bash
    reboot
    ```

### 2.2 每个节点安装Docker、kubeadm、kubelet和kubectl

#### 2.2.1安装Docker

安装可参考： https://blog.koisecret.site/docker%E5%AE%89%E8%A3%85/#centos%E5%AE%89%E8%A3%85docker，需要注意的点如下：

- 最好指定docker的安装版本，否则安装最新版（万一不兼容呢

  ```bash
  # 查看存储库中 Docker 的版本
  yum list docker-ce --showduplicates | sort -r
  yum list docker-ce-cli --showduplicates | sort -r
  # 安装指定版本的 Docker,如20.10.8-3
  yum -y install docker-ce-3:20.10.8-3.el7.x86_64 docker-ce-cli-1:20.10.8-3.el7.x86_64 containerd.io
  ```

- Docker 在默认情况下使用Vgroup Driver为cgroupfs，而Kubernetes推荐使用systemd来替代cgroupfs

  ```bash
  # 可以与 配置阿里云镜像加速 的操作合并
  sudo tee /etc/docker/daemon.json <<-'EOF'
  {
      "exec-opts": ["native.cgroupdriver=systemd"],
  	"registry-mirrors": ["https://kn0t2bca.mirror.aliyuncs.com"]
  }
  EOF
  ```

#### 2.2.2添加阿里云的YUM软件源

- 由于kubernetes的镜像在国外，速度比较慢，这里切换成国内的镜像源。

```bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

#### 2.2.3安装kubeadm、kubelet和kubectl

- yum安装，注意指定版本

  ```bash
  yum install -y kubelet-1.23.9 kubectl-1.23.9 kubeadm-1.23.9
  ```

  - `kubeadm`：用来初始化集群的指令。
  - `kubelet`：在集群中的每个节点上用来启动 Pod 和容器等。
  - `kubectl`：用来与集群通信的命令行工具。

- 为了实现 Docker 使用的 cgroup drvier 和 kubelet 使用的 cgroup drver 一致，建议修改 `/etc/sysconfig/kubelet` 文件的内容

  ```bash
  vim /etc/sysconfig/kubelet
  
  # 修改
  KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
  KUBE_PROXY_MODE="ipvs"
  ```

- 设置为开机自启动即可，由于没有生成配置文件，集群初始化后自动启动

  ```bash
  systemctl enable kubelet
  ```

### 2.3 准备集群镜像





















