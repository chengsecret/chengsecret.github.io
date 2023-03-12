---
title: "在云服务器安装k8s集群"
subtitle: ""
date: 2023-03-12T16:59:39+08:00
lastmod: 2023-03-12T16:59:39+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["k8s"]
tags: [""]
featuredImage: ""
---

> 参考文章：
>
> - https://blog.csdn.net/weixin_43988498/article/details/122639595 (主要参考)
> - https://blog.koisecret.site/k8s%E4%BB%8B%E7%BB%8D%E4%B8%8E%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85/

此次部署本人使用了三台阿里云服务器，系统都是Ubuntu，内网不互通。

## 主要问题

我主要是根据上述的第一篇博客来部署k8s集群，因此多余的废话也不说了，只针对部署过程中该博客中存在的一些问题，进行修改。

> 1.重启网卡命令失败

```bash
apt install ifupdown 
/etc/init.d/networking restart
ifconfig # 查看是否写入成功
```

> 2.关闭selinux

ubuntu执行`sed -i 's/enforcing/disabled/' /etc/selinux/config`时会报错，原因是没有config这个文件。在一些操作系统中，如 CentOS 7， SELinux 默认是启用的，并且可以通过编辑 `/etc/selinux/config` 文件来关闭它。但在某些操作系统中，如 Ubuntu， SELinux 是被禁用的，因此无需关闭。

> 3.没有开启ipvs

可以部署完k8s后再添加，不添加也无所谓（性能下降点）。

> 4.安装docker

Ubuntu安装docker可以参考： https://blog.csdn.net/qq_44732146/article/details/121207737

若想下载指定版本的docker，可以采用：

```bash
apt list -a docker-ce #列出 Docker 软件源中所有可用的版本

apt install docker-ce=5:20.10.22~3-0~ubuntu-jammy docker-ce-cli=5:20.10.22~3-0~ubuntu-jammy containerd.io
```

下载完成后，仍按自己博客中进行配置： https://blog.koisecret.site/docker%E5%AE%89%E8%A3%85/#centos%E5%AE%89%E8%A3%85docker

别忘了修改`/etc/docker/daemon.json`文件：

```json
{
    "exec-opts": ["native.cgroupdriver=systemd"],
	"registry-mirrors": ["https://kn0t2bca.mirror.aliyuncs.com"]
}
```

> 5.下载kubelet、kubeadm、kubectl 

```bash
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -  # 添加证书

cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF   # 添加apt源

apt-get update

apt-cache madison kubelet #查看可安装版本

apt-get install -y kubectl=1.23.0-00 kubeadm=1.23.0-00 kubelet=1.23.0-00 #安装

#为了实现 Docker 使用的 cgroup drvier 和 kubelet 使用的 cgroup drver 一致
vim /etc/sysconfig/kubelet
# 修改
KUBELET_EXTRA_ARGS="--cgroup-driver=systemd"
KUBE_PROXY_MODE="ipvs"

systemctl enable kubelet && sudo systemctl start kubelet #设置开机启动
```

> 6.kubeadm init

- 内存不足（阿里云是真🐕，说好的2核2G，结果内存只有1680M，虽然服务器是我白嫖的）

  ```bash
  --ignore-preflight-errors=NumCPU,Mem  #kubeadm init 后加上
  ```

- 命令执行失败

  ```bash
  kubeadm reset # 相当于撤回kubeadm init 
  ```

最终也没解决，估计是公网服务器直接的通信还有问题吧。明天换centos操作系统试试。









