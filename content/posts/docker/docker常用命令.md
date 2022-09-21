---
title: "Docker常用命令"
subtitle: ""
date: 2022-09-19T15:12:39+08:00
lastmod: 2022-09-19T15:12:39+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","入门","云原生"]
---

## 帮助启动类命令

```bash
systemctl start docker  # 启动docker
systemctl stop docker # 停止docker
systemctl restart docker # 重启docker
systemctl status docker # 查看docker状态
systemctl enable docker # 开机时自动启动docker
docker info # 查看docker信息
docker --help #查看docker全部命令的帮助文档
docker 具体命令 --help # 查看具体命令的帮助文档
```

## 镜像命令

```bash
docker images # 列出本机上的镜像 
              ## -a 列出所有镜像（包括已经删除过的历史印象层）
              ## -q 只显示镜像id
docker search [--limit 25] 镜像名称  #从远端镜像仓库中查找镜像  limit参数可选，表示显示的条数（默认25）。
									#基本选第一个即可（点赞最多，官方认证）
docker pull 镜像名[:tag] # 从云端镜像仓库下载镜像 tag表示版本，不写则默认最新（latest）
						# docker pull redis:6.0.8，再docker images，
						#可以发现下载了仓库名为redis，tag为6.0.8的镜像，说明不同版本的redis镜像放置在同一个redis仓库中
docker system df # 查看镜像、容器、数据卷所占的空间
docker rmi [-f] 镜像id   # 删除镜像  
					    #docker rmi -f $(docker iamges -qa) 删除全部

```

**虚悬镜像**：仓库名、tag都为<none>的镜像，没什么用，基本是下载失败的镜像。

## 容器命令

```bash
docker run [options] 镜像名[:tag] [command]  # 新建且启动容器
				# options说明
                # --name="容器新名字"         # 为容器指定一个名称
                # -d       # 后台运行
                # -i：以交互模式运行容器，通常与 -t 同时使用；
				# -t：为容器重新分配一个伪输入终端，通常与 -i 同时使用；
				## 如 docker run -it ubuntu /bin/bash  ### /bin/bash为指定终端命令
				# -P: 随机端口映射，大写P
				# -p: 指定端口映射，小写p  如：-p 1234:6379 表示本机端口1234映射容器中的6379端口

docker start 容器id或容器名  # 启动已停止运行的容器
docker restart  容器id或容器名  # 重启容器
docker stop  容器id或容器名  # 停止运行容器
docker kill 容器id或容器名  # 强制停止运行容器

docker ps   # 列出当前正在运行的容器
			# -a 列出所有容器
			
# 以交互模式运行时退出容器 如：docker run -it ubuntu /bin/bash 
exit   # 容器停止
ctrl+p+q  # 退出后容器在后台运行

#进入后台正在运行的容器
docker exec -it 容器id command ## 如docker exec -it id /bin/bash
docker attach 容器id  ##区别： exec与attach都是进入后台正在运行的容器，
							# exec是在容器中启动新的终端，因此exit后不会导致容器停止运行，推荐使用
							# attach直接进入容器启动命令的终端，exit后容器停止运行

docker rm 容器id或容器名 # 删除停止运行的容器
docker rm -f $(docker ps -aq)

docker logs 容器id  # 查看容器日志
docker top 容器id  # 查看容器内运行的进程
docker inspect 容器id # 查看容器内部细节

docker cp 容器id:容器内文件路径 目的主机路径  # 复制容器内的文件到主机中
docker export 容器id > 文件名.tar  # 导出整个容器内容为一个tar文件，放在当前目录中
cat 文件名.tar | docker import -包名/镜像名:版本号 # 根据tar文件再次生成镜像

docker commit -m="描述信息" -a="作者" 容器id 创建的镜像名:[版本号] #提交容器副本使之成为一个新的镜像
								# 如ubuntu镜像是没有vim命令的，ubuntu容器内下载vim，再commit成一个新镜像，新镜像就有vim命令

```

## 镜像加载原理

- docker的镜像实际上由一层一层的文件系统组成，称为层级的文件系统UnionFS。Union文件系统（UnionFS）是一种分层、轻量级并且高性能的文件系统，**它支持对文件系统的修改作为一次提交来一层层的叠加**，镜**像可以通过分层来进行继承**，基于基础镜像（没有父镜像），可以制作各种具体的应用镜像。

- bootfs(boot file system)主要包含bootloader和kernel, bootloader主要是引导加载kernel, Linux刚启动时会加载bootfs文件系统，在Docker镜像的最底层是引导文件系统bootfs。这一层与我们典型的Linux/Unix系统是一样的，包含boot加载器和内核。当boot加载完成之后整个内核就都在内存中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。

- rootfs (root file system) ，在bootfs之上。包含的就是典型 Linux 系统中的 /dev, /proc, /bin, /etc 等标准目录和文件。rootfs就是各种不同的操作系统发行版，比如Ubuntu，Centos等等。 对于一个精简的OS，rootfs可以很小，只需要包括最基本的命令、工具和程序库就可以了，因为底层直接用Host的kernel，自己只需要提供 rootfs 就行了。由此可见对于不同的linux发行版, bootfs基本是一致的, rootfs会有差别, 因此不同的发行版可以公用bootfs。
- **镜像分层最大的一个好处就是共享资源，方便复制迁移，就是为了复用。**
- 当容器启动时，一个新的可写层被加载到镜像的顶部。这一层通常被称作“容器层”，“容器层”之下的都叫“镜像层”。所有对容器的改动 - 无论添加、删除、还是修改文件都只会发生在容器层中。只有容器层是可写的，容器层下面的所有镜像层都是只读的。



## 本地镜像发布

### 发布到阿里云

1. 阿里云官网搜索`容器镜像服务`,创建个人实例 ，新建**命名空间**和**仓库名称**，进入**管理**界面

2. 按照管理界面的命令，将镜像push到阿里云镜像仓库

   ```bash
   docker login --username=xxxxxx registry.cn-hangzhou.aliyuncs.com
   docker tag [ImageId] registry.cn-hangzhou.aliyuncs.com/ctstudy/ubuntu-test:[镜像版本号]
   docker push registry.cn-hangzhou.aliyuncs.com/ctstudy/ubuntu-test:[镜像版本号]
   ```

3. 从阿里云镜像仓库拉取镜像

   ```bash
   docker pull registry.cn-hangzhou.aliyuncs.com/ctstudy/ubuntu-test:[镜像版本号]
   ```

### 发布到私有库

> - 官方Docker Hub地址：https://hub.docker.com/，中国大陆访问太慢了且准备被阿里云取代的趋势，不太主流。
>
> - Dockerhub、阿里云这样的公共镜像仓库可能不太方便，涉及机密的公司不可能提供镜像给公网，所以需要创建一个本地私人仓库供给团队使用，基于公司内部项目构建镜像。
>
> - Docker Registry镜像是官方提供的工具，可以用于构建私有镜像仓库。

1. 下载镜像 

   ```bash
   docker pull registry
   ```

2. 运行镜像

   ```bash
   docker run -d -p 5000:5000 -v /zzyyuse/myregistry/:/tmp/registry --privileged=true registry
    ## -v 数据卷            个人认为这么挂载是有问题的，后续再回来看看
   ```

3. curl命令查看容器中的镜像

   ```bash
   curl -XGET http://localhost:5000/v2/_catalog  # 显示 {"repositories":[]}
   ```

4. 将要上传的镜像（ubuntu）修改为符合私服规范的tag

   ```bash
   docker tag 镜像:Tag   Host:Port/Repository:Tag # 如：docker tag  ubuntu:1.2  localhost:5000/ubuntu:1.2
   ```

5. 修改配置文件使之支持http

   ```bash
   vim /etc/docker/daemon.json
   ```

   修改为：

   ```json
   {
     "registry-mirrors": ["https://xxxxxx.mirror.aliyuncs.com"],  // 你自己的阿里云镜像，不用改，别忘了加逗号
     "insecure-registries": ["localhost:5000"]
   }
   ```

   修改完后重启docker。

6. push推送到私服库

   ```bash
   docker push localhost:5000/ubuntu:1.2
   ```

   再次验证：

   ```bash
   curl -XGET http://localhost:5000/v2/_catalog  # 显示 {"repositories":[ubuntu]}
   ```

7. pull到本地

   ```bash
   docker pull localhost:5000/zzyyubuntu:1.2
   ```

## 容器数据卷

```bash
docker run --privileged=true -v /宿主机绝对路径目录:/容器内目录  镜像名 #  --privileged=true 记得带上，否则可能权限不够
```

我们对数据的要求希望是持久化的，Docker容器产生的数据，如果不备份，那么当容器实例删除后，容器内的数据自然也就没有了。为了能保存数据在docker中我们使用卷。

1. 数据卷可在容器之间共享或重用数据

2. 卷中的更改可以直接实时生效，爽

3. 数据卷中的更改不会包含在镜像的更新中

- 查看是否挂载成功：

  ```bash
  docker inspect 容器ID
  ```

- 映射的读写规则

  ```bash
  docker run --privileged=true -v /宿主机绝对路径目录:/容器内目录:ro  镜像名  #容器内文件只读
  docker run --privileged=true -v /宿主机绝对路径目录:/容器内目录:rw  镜像名  #默认，读写都可以
  ```

- 数据卷的继承与共享

  ```bash
  docker run -it  --privileged=true -v /mydocker/u:/tmp --name u1 ubuntu  # 容器一
  docker run -it  --privileged=true --volumes-from u1 --name u2 ubuntu # 容器二，他们共享容器卷
  ```

  











