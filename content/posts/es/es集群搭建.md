---
title: "Es集群搭建"
subtitle: ""
date: 2023-03-24T21:09:13+08:00
lastmod: 2023-03-24T21:09:13+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["elasticsearch"]
tags: ["elasticsearch"]
featuredImage: ""
---

# es8集群搭建

三台主机进行部署，由于es集群的master是自己选举产生的，所以搭建时无需关心master和普通节点的区别。

### 一、搭建步骤

> 下载解压

下载地址： https://www.elastic.co/cn/downloads/past-releases#elasticsearch

#### 1. 预处理

```bash
sudo tar -zxvf elasticsearch-8.1.1-linux-x86_64.tar.gz -C /usr/local/

# 创建数据文件目录 
mkdir /usr/local/elasticsearch-8.1.1/data 
# 创建证书目录 
mkdir /usr/local/elasticsearch-8.1.1/config/certs

#不可使用root用户直接启动elasticsearch,所以要用其他用户启动
#创建用户组
groupadd ct
#创建用户
useradd ct -g ct -p 123456
#授予用户权限
chown -R ct:ct /usr/local/elasticsearch-8.1.1/
#切换用户
su - ct
```

#### 2. 设置通信秘钥

```bash
#然后进入安装bin目录
cd /usr/local/elasticsearch-8.1.1/
#会有两次回车，第一个是描述，不填跳过；第二是密码，输入123456（你可以设置你自己的密码）
./bin/elasticsearch-certutil ca
#生成密匙，输入刚刚的密码123456，要输入路径直接回车，生成在当前目录下
./bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12
# 将生成的证书文件移动到 config/certs 目录中
mv elastic-stack-ca.p12 elastic-certificates.p12 config/certs
```

凭证移动至每一台集群下面，此处省略各种scp，就是把elastic-certificates.p12这个文件移动到每一个es安装目录的相同路径下。

各个节点添加keystore密码：

```bash
#如果节点证书配置密码的话，这里要加入密码库：
./bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password
./bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
```

生产环境中，还要生成https证书，这里就不做了。

#### 3. 修改配置文件

文件地址：config/elasticsearch.yml

```bash
#集群名称
cluster.name: ctES
#结点名称 多个结点名称不同
node.name: es_node_1
#服务器地址
network.host: 10.4.xx.xx
#端口号
http.port: 9200
transport.profiles.default.port: 9300
#其他结点的路径
discovery.seed_hosts: ["10.4.xx.xx:9300", "10.4.xx.xx:9300", "10.4.xx.xx:9300"]
#相当于指出有权利当master的节点，而不是说三个都是master
cluster.initial_master_nodes: ["es_node_1", "es_node_2", "es_node_3"]

# 允许通配符删除索引
action.destructive_requires_name: true

bootstrap.memory_lock: true

#设置证书密码访问 下面会说怎么生成证书
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: /usr/local/elasticsearch-8.1.1/config/certs/elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: /usr/local/elasticsearch-8.1.1/config/certs/elastic-certificates.p12

#日志和索引存储地址
path.data: /usr/local/elasticsearch-8.1.1/data
path.logs: /usr/local/elasticsearch-8.1.1/logs

#是否支持跨域
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization,X-Requested-With,Content-Type,Content-Length

#不配会报错
ingest.geoip.downloader.enabled: false
```

注意：bootstrap.memory_lock项要设置为true，锁定物理内存地址，防止es内存被交换，从而提高ES性能；但是设置以后因为服务器配置不同可能会启动报错。报错内容：

```bash
bootstrap checks failed. You must address the points described in the following [1] lines before starting Elasticsearch
```

#### 4.修改参数

```bash
vim /etc/security/limits.conf

#添加以下内容：
* soft nofile 65536
* hard nofile 131072
* soft nproc 4096
* hard nproc 4096

# 可以ulimit -Hn 和  ulimit -Sn查看是否生效
# 不生效的话，ssh重新连接即可
```

在以下配置文件中添加参数：

```bash
vim /etc/sysctl.conf
#添加参数
vm.max_map_count=655360
#重启服务
sysctl -p
systemctl daemon-reload
```

 调整JVM运行内存：

```bash
vim config/vm.options

#增加参数(默认是4g，可以自行修改)
-Xms2g
-Xmx2g
```

#### 5.启动

```bash
#后台启动
./bin/elasticsearch -d
```

#### 6 其他节点

只需更改config/elasticsearch.yml中的：

```bash
node.name: es_node_n
network.host: 10.4.xx.xxx
```

剩下操作都一样，证书只需要用第一台节点生成的就行，放到对应的目录`config/certs`下。

#### 7.设置集群访问密码

会设置很多密码 elastic，apm_system，kibana，kibana_system，logstash_system，beats_system，我本地环境全部设置成了123456

```bash
./bin/elasticsearch-setup-passwords  interactive
```

### 二、安装ik分词器

github地址：https://github.com/medcl/elasticsearch-analysis-ik

ik包下载地址： https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.1.1/elasticsearch-analysis-ik-8.1.1.zip

中文想要分词，必须要安装一个分词器插件，这里选择ik。

在集群的每台主机上都要安装ik。

```bash
cd your-es-root/plugins/ && mkdir ik
mv elasticsearch-analysis-ik-8.1.1.zip your-es-root/plugins/ik
unzip your-es-root/plugins/ik/elasticsearch-analysis-ik-8.1.1.zip
```

然后重启es。

```bash
ps -aux | grep elastic
kill -9 两个进程号
./bin/elasticsearch -d
```

### 三、安装kibana可视化平台

```bash
#下载 版本需要对应
cd /usr/local/elasticsearch-8.1.1/kibana #kibana文件夹是自己创的，放哪儿都行
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.1.1-linux-x86_64.tar.gz
#解压到对应目录
tar -zxvf kibana-8.1.1-linux-x86_64.tar.gz
cd kibana-8.1.1/config/
vim kibana.yml
```

```yml
#端口号
server.port: 5601
 
#任意地址都可以访问
server.host: "0.0.0.0"
 
#查询的Elasticsearch实例的URL，我安装在本地，按实际的情况填写
elasticsearch.hosts: ["http://10.4.xx.xx:9200","http://10.4.xx.xx:9200","http://10.4.xx.xx:9200"]
 
#访问elasticsearch开启了认证
elasticsearch.username: "kibana"
elasticsearch.password: "123456"
 
#支持的语言设置
i18n.locale: "zh-CN"
```

```bash
 #保持后台启动
nohup bin/kibana &
#访问地址：10.4.xx.xx:5601
```

## es集群、k8s集群、redis集群

- redis集群需要多个master节点，它的目的是提高存储容量以及提高并发量（槽机制），而高可用是通过slave节点和哨兵模式实现的。
- k8s集群需要多个master节点，因为一个master挂了，其他node并不能自动变为master（master上的apisevice等组件其他node上根本没有），所以需要多个master来保证高可用。master用来管理集群，node用来承载服务。
- es集群中，只能有一个master节点，它挂了其他节点会自动选举出一个master，因此扩展起来非常方便。当集群只有两个节点时，一旦两个节点失去联系，就有可能产生两个master节点，称为“脑裂”，会导致集群数据不一致。





