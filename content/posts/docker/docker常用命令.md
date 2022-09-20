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

```



















