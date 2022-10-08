---
title: "Dockerfile解析与服务部署"
subtitle: ""
date: 2022-10-08T16:43:10+08:00
lastmod: 2022-10-08T16:43:10+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","云原生"]
---

## Dockerfile是什么

DockerFile是用来**构建docker镜像**的文本文件，是一条条构建镜像所需的指令和参数构成的**脚本**。

- 每条保留字指令必须是**大写字母**并且后面要`跟随至少一个参数`
- 指令从上到下顺序执行
- ’# ‘为注释
- 每条指令都会创建一个新的镜像层，最终生成目标镜像（镜像的分层结构）

---

**Dockerfile执行流程分析：**

1. `docker`从基础镜像运行一个容器
2. 执行一条指令并对容器作出修改
3. 执行类似`docker commit`的操作提交一个新的镜像层。
4. `docker`再基于刚提交的镜像运行一个新容器
5. 执行`dockerfile`中的下一条指令直到所有指令都执行完成

---

**构建步骤**：编写DockerFile文件--------》docker build构建镜像--------》docker run根据镜像运行容器


Dockerfile面向开发，Docker镜像成为交付标准，Docker容器则涉及部署与运维，三者缺一不可，合力充当Docker体系的基石。

1. **Dockerfile**，需要定义一个Dockerfile，Dockerfile定义了进程需要的一切东西。Dockerfile涉及的内容包括执行代码或者是文件、环境变量、依赖包、运行时环境、动态链接库、操作系统的发行版、服务进程和内核进程(当应用进程需要和系统服务和内核进程打交道，这时需要考虑如何设计namespace的权限控制)等等;

2. **Docker镜像**，在用Dockerfile定义一个文件之后，docker build时会产生一个Docker镜像，当运行 Docker镜像时会真正开始提供服务;

3. **Docker容器**，容器是直接提供服务的。



## Dockerfile常用保留字指令

构建tomcat9.0镜像的DockerFile文件如下：

```bash
FROM amazoncorretto:8-al2-jdk

ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME"
WORKDIR $CATALINA_HOME

# let "Tomcat Native" live somewhere isolated
ENV TOMCAT_NATIVE_LIBDIR $CATALINA_HOME/native-jni-lib
ENV LD_LIBRARY_PATH ${LD_LIBRARY_PATH:+$LD_LIBRARY_PATH:}$TOMCAT_NATIVE_LIBDIR

# see https://www.apache.org/dist/tomcat/tomcat-9/KEYS
# see also "versions.sh" (https://github.com/docker-library/tomcat/blob/master/versions.sh)
ENV GPG_KEYS 48F8E69F6390C9F25CFEDCD268248959359E722B A9C5DF4D22E99998D9875A5110C01C5A2F6059E7 DCFD35E0BF8CA7344752DE8B6FB21E8933C60243

ENV TOMCAT_MAJOR 9
ENV TOMCAT_VERSION 9.0.68
ENV TOMCAT_SHA512 840b21c5cd2bfea7bbfed99741c166608fa1515bb00475ebd4eccfd4f3ed2a1be6bfffd590b2a6600003205b62f271b6ba0937e557fc65a536df61cb4f7b7c8f

RUN set -eux; \
	\
# http://yum.baseurl.org/wiki/YumDB.html
	if ! command -v yumdb > /dev/null; then \
		yum install -y --setopt=skip_missing_names_on_install=False yum-utils; \
		yumdb set reason dep yum-utils; \
	fi; \
# a helper function to "yum install" things, but only if they aren't installed (and to set their "reason" to "dep" so "yum autoremove" can purge them for us)
	_yum_install_temporary() { ( set -eu +x; \
		local pkg todo=''; \
		for pkg; do \
			if ! rpm --query "$pkg" > /dev/null 2>&1; then \
				todo="$todo $pkg"; \
			fi; \
		done; \
		if [ -n "$todo" ]; then \
			set -x; \
			yum install -y --setopt=skip_missing_names_on_install=False $todo; \
			yumdb set reason dep $todo; \
		fi; \
	) }; \
	_yum_install_temporary gzip tar; \
	\
	ddist() { \
		local f="$1"; shift; \
		local distFile="$1"; shift; \
		local mvnFile="${1:-}"; \
		local success=; \
		local distUrl=; \
		for distUrl in \
# https://issues.apache.org/jira/browse/INFRA-8753?focusedCommentId=14735394#comment-14735394
			"https://www.apache.org/dyn/closer.cgi?action=download&filename=$distFile" \
# if the version is outdated (or we're grabbing the .asc file), we might have to pull from the dist/archive :/
			"https://downloads.apache.org/$distFile" \
			"https://www-us.apache.org/dist/$distFile" \
			"https://www.apache.org/dist/$distFile" \
			"https://archive.apache.org/dist/$distFile" \
# if all else fails, let's try Maven (https://www.mail-archive.com/users@tomcat.apache.org/msg134940.html; https://mvnrepository.com/artifact/org.apache.tomcat/tomcat; https://repo1.maven.org/maven2/org/apache/tomcat/tomcat/)
			${mvnFile:+"https://repo1.maven.org/maven2/org/apache/tomcat/tomcat/$mvnFile"} \
		; do \
			if curl -fL -o "$f" "$distUrl" && [ -s "$f" ]; then \
				success=1; \
				break; \
			fi; \
		done; \
		[ -n "$success" ]; \
	}; \
	\
	ddist 'tomcat.tar.gz' "tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz" "$TOMCAT_VERSION/tomcat-$TOMCAT_VERSION.tar.gz"; \
	echo "$TOMCAT_SHA512 *tomcat.tar.gz" | sha512sum --strict --check -; \
	ddist 'tomcat.tar.gz.asc' "tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz.asc" "$TOMCAT_VERSION/tomcat-$TOMCAT_VERSION.tar.gz.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
		gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
	done; \
	gpg --batch --verify tomcat.tar.gz.asc tomcat.tar.gz; \
	tar -xf tomcat.tar.gz --strip-components=1; \
	rm bin/*.bat; \
	rm tomcat.tar.gz*; \
	command -v gpgconf && gpgconf --kill all || :; \
	rm -rf "$GNUPGHOME"; \
	\
# https://tomcat.apache.org/tomcat-9.0-doc/security-howto.html#Default_web_applications
	mv webapps webapps.dist; \
	mkdir webapps; \
# we don't delete them completely because they're frankly a pain to get back for users who do want them, and they're generally tiny (~7MB)
	\
	nativeBuildDir="$(mktemp -d)"; \
	tar -xf bin/tomcat-native.tar.gz -C "$nativeBuildDir" --strip-components=1; \
	_yum_install_temporary \
		apr-devel \
		gcc \
		make \
		openssl11-devel \
	; \
	( \
		export CATALINA_HOME="$PWD"; \
		cd "$nativeBuildDir/native"; \
		aprConfig="$(command -v apr-1-config)"; \
		./configure \
			--libdir="$TOMCAT_NATIVE_LIBDIR" \
			--prefix="$CATALINA_HOME" \
			--with-apr="$aprConfig" \
			--with-java-home="$JAVA_HOME" \
			--with-ssl \
		; \
		nproc="$(nproc)"; \
		make -j "$nproc"; \
		make install; \
	); \
	rm -rf "$nativeBuildDir"; \
	rm bin/tomcat-native.tar.gz; \
	\
# mark any explicit dependencies as manually installed
	find "$TOMCAT_NATIVE_LIBDIR" -type f -executable -exec ldd '{}' ';' \
		| awk '/=>/ && $(NF-1) != "=>" { print $(NF-1) }' \
		| xargs -rt readlink -e \
		| sort -u \
		| xargs -rt rpm --query --whatprovides \
		| sort -u \
		| tee "$TOMCAT_NATIVE_LIBDIR/.dependencies.txt" \
		| xargs -r yumdb set reason user \
	; \
	\
# clean up anything added temporarily and not later marked as necessary
	yum autoremove -y; \
	yum clean all; \
	rm -rf /var/cache/yum; \
	\
# sh removes env vars it doesn't support (ones with periods)
# https://github.com/docker-library/tomcat/issues/77
	find ./bin/ -name '*.sh' -exec sed -ri 's|^#!/bin/sh$|#!/usr/bin/env bash|' '{}' +; \
	\
# fix permissions (especially for running as non-root)
# https://github.com/docker-library/tomcat/issues/35
	chmod -R +rX .; \
	chmod 777 logs temp work; \
	\
# smoke test
	catalina.sh version

# verify Tomcat Native is working properly
RUN set -eux; \
	nativeLines="$(catalina.sh configtest 2>&1)"; \
	nativeLines="$(echo "$nativeLines" | grep 'Apache Tomcat Native')"; \
	nativeLines="$(echo "$nativeLines" | sort -u)"; \
	if ! echo "$nativeLines" | grep -E 'INFO: Loaded( APR based)? Apache Tomcat Native library' >&2; then \
		echo >&2 "$nativeLines"; \
		exit 1; \
	fi

EXPOSE 8080
CMD ["catalina.sh", "run"]
```

- **FROM**：基础镜像，即当前新镜像是基于哪个镜像创建的

- **MAINTAINER**：镜像维护者的姓名和邮箱地址

- **RUN**：容器**构建时**需要运行的指令，有两种格式

  - shell格式，如：`RUN yum -y install vim`
  - exec格式，`RUN ["可执行文件","参数一","参数二"]`

- **EXPOSE**：当前容器对外暴露的端口

- **WORKDIR**：指定在创建容器后，终端默认登录进来的工作目录

- **ENV**：用来在构建镜像过程中设置环境变量。这个环境变量可以在后续的任何RUN指令中使用，这就如同在命令前面指定了环境变量前缀一样;也可以在其它指令中直接使用这些环境变量。如：`ENV MYCAT_HOME=/usr/local/mycat`  `WORKDIR $MYCAT_HOME/mycat`

- **ADD** 和 **COPY**：

  - ADD：将宿主机目录下的文件拷贝进镜像，并且ADD命令会自动处理URL和解压tar压缩包，如`ADD centos-6-docker.tar.xz / 

  - COPY：将从构建上下文目录中<源路径>的文件/目录 复制到新的一层的镜像内的<目标路径>位置，如`COPY src destCOPY` 或 `COPY ["src" "dest"]`

  - 两者功能基本一致，区别为：

    第一处：ADD指令可以添加URL资源,或者说可以直接从远程添加文件到镜像中，或者说，而COPY不具备这样的能力；
    第二处：ADD指令将这些资源添加到镜像镜像的文件系统；而COPY指令是将这些资源添加到容器的文件系统中；ADD指令是将资源添加到静态文件系统中，COPY指令是将资源添加到动态文件系统中，具体有什么实质性的体现呢？
    下面详细说明ADD指令带来的实际影响。
    如果src是文件类型的资源，dest以/结尾， 则docker会把dest当作一个目录看待，会把资源拷贝到dest目录下。如果dest不存在，则会创建dest目录；

    如果src是个文件类型的资源，dest不以/结尾，则docker会把dest当作一个文件看待；

    如果dest存在，则会把src拷贝到dest的上级目录中，并重命名为dest；如果目标文件dest是已存在，会用源文件中的内容覆盖掉dest中的内容。

    如果src是个目录，且dest不存在，则docker会创建一个名为dest的目录，把src目录下的文件拷贝进来。如果dest是个已经存在的目录，则docker会把src目录下的文件拷贝到该目录下。

    如果源文件是压缩文件，则docker会自动解压。

- **VOLUME**：容器数据卷，用于数据持久化和数据保存。如： `VOLUME /share/data　#声明容器中/share/data为匿名卷`。volume和run -v的区别见[一文说清楚Dockerfile 中VOLUME到底有什么用？](https://blog.csdn.net/qq32933432/article/details/120944205)

- **CMD** 和 **ENTRYPOINT**：指定一个**容器启动时**要运行的命令。

  **CMD**的指令和RUN相似，也是两种格式：

  - `shell`格式：CMD<命令>，如`CMD cat /conf/my.cnfCMD` `cat /bin/bash`
  - `exec`格式：CMD ["可执行文件“，”参数1“，”参数2“…]，如`CMD ["catalina.sh", "run"]`

  `Dockerfile`中可以有多个CMD指令，**但只有最后一个生效**，CMD会**被`docker run`之后的参数替换**。`docker run`运行时不能够额外追加命令，否则会覆盖掉`Dockerfile`中的`CMD`命令。

  而`ENTRYPOINT`则不同，可以将`ENTRYPOINT`简单理解为追加。**ENTRYPOINT不会被docker run后面的命令覆盖，而且这些命令行参数会被当做参数传给ENTRYPOINT指令指定的程序。ENTRYPOINT同样只有最后一个生效。**
  
  ENTRYPOINT可以和CMD一起用，一般是变参才会使用 CMD ，这里的 CMD 等于是在给 ENTRYPOINT 传参。当指定了ENTRYPOINT后，CMD的含义就发生了变化，不再是直接运行其命令而是将CMD的内容作为参数传递给ENTRYPOINT指令。如：`ENTRYPOINT ["nginx","-c"] #定参` `CMD ["/etc/nginx/mginx.conf"] #变参`
  
- **onbuild**：当构建一个被继承的`Dockerfile`时运行命令，父镜像在被子继承后，父镜像的`onbuild`被触发。



## Dockerfile定义镜像实战

目标：镜像具有centos + vim + ifconfig + jdk8

1. 新建一个文件夹，文件夹中有jdk1.8的Linux版压缩包、Dockerfile文件（D大写）

2. 编辑Dockerfile文件

   ```bash
   FROM centos:7
   MAINTAINER ct<ct@126.com>
    
   ENV MYPATH /usr/local
   WORKDIR $MYPATH
    
   #安装vim编辑器
   RUN yum -y install vim
   #安装ifconfig命令查看网络IP
   RUN yum -y install net-tools
   #安装java8及lib库
   RUN yum -y install glibc.i686
   RUN mkdir /usr/local/java
   #ADD 是相对路径jar,把jdk-8u171-linux-x64.tar.gz添加到容器中,安装包和Dockerfile文件在同一位置
   ADD jdk-8u171-linux-x64.tar.gz /usr/local/java/
   #配置java环境变量
   ENV JAVA_HOME /usr/local/java/jdk1.8.0_171
   ENV JRE_HOME $JAVA_HOME/jre
   ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
   ENV PATH $JAVA_HOME/bin:$PATH
    
   EXPOSE 80
    
   CMD echo $MYPATH
   CMD echo "success--------------ok"
   CMD /bin/bash #上两个CMD不生效
   ```

3. 构建镜像

   ```bash
   docker build -t centosjava8:1.5 . # docker build -t 镜像名:版本号 Dockerfile所在目录
   ```

4. 运行

   ```bash
   docker run -it centosjava8:1.5
   ```

> 虚悬镜像

- 即仓库名、标签名都是`<none>`的镜像，也称为**dangling image**

- 查看所以虚悬镜像命令：`docker image ls -f dangling=true`
- 删除所有虚悬镜像：`docker image prune`



## 简单实战

1. idea建一个springboot项目，简单写个controller，打成一个jar包

2. 将jar包和Dockerfile文件放在同一个文件夹下，编写Dockerfile文件

   ```bash
   # 基础镜像使用java
   FROM java:8
   # 作者
   MAINTAINER ct
   
   # VOLUME 在主机/var/lib/docker目录下创建了一个临时文件并链接到容器的/tmp
   VOLUME /tmp
   
   # 将jar包添加到容器中并更名为dockertest.jar
   ADD docker_boot-0.0.1-SNAPSHOT.jar dockertest.jar
   
   # 运行jar包
   RUN bash -c 'touch /dockertest.jar'
   
   ENTRYPOINT ["java","-jar","/dockertest.jar"]
   
   #暴露6001端口作为微服务
   EXPOSE 6001
   ```

3. 构建镜像

   ```bash
   docker build -t dockertest:1.8 .
   ```

4. 运行容器

   ```bash
    docker run -d -p 6001:6001 dockertest:1.8
   ```



