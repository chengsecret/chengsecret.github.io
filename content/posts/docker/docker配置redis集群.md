---
title: "Docker配置redis集群"
subtitle: ""
date: 2022-09-27T10:15:34+08:00
lastmod: 2022-09-27T10:15:34+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker","redis"]
tags: ["docker","redis","分布式存储","云原生"]
---

当有上亿条数据需要缓存，该如何解决这个问题？   ——**分布式存储**

## 一 redis回顾

> 持久化

**RDB快照**：默认情况下，Redis将内存数据库快照保存为dump.rdb二进制文件，可以设置他的初始化规则为N秒内发生了M次改动时就做一次保存动作 例： “60秒内有至少有1000个键被改动”这一条件时，自动保存一次数据集：配置文件：`# save 60 1000`

**AOF** : RDB有个缺点，当服务宕机后，会缺失最近没达到持久化条件一段时间内的数据。所以AOF出现了，他是规定多长时间就持久化一次，推荐每秒同步一次，即使故障也就那一秒的数据没有。这样的话文件就会较大，所以当恢复采用aof文件，进度就会稍慢。Redis4.0混合持久化，先采用RDB快照恢复，再用AOF补充缺失的部分数据。重启效率大大提高。



> 主从复制

- 主从复制架构用来解决数据的**冗余备份**,从节点仅仅用来同步数据。

- **无法解决:master节点出现故障的自动故障转移**。

- **数据的复制是单向的，只能由主节点到从节点**。**Master以写为主，Slave以读为主**。

**一般来说，要将Redis运用于工程项目中，只使用一台Redis是万万不能的（宕机），原因如下：**

- 从结构上，单个Redis服务器会发生单点故障，并且一台服务器需要处理所有的请求负载，压力较大！
- 从容量上，单个Redis服务器内存容量有限，就算一台Redis服务器内容容量为256G，也不能将所有内存用作Redis存储内存，一般来说，**单台Redis最大使用内存不应该超过20G**。



> 哨兵模式

- Sentinel（哨兵）是Redis 的高可用性解决方案：由一个或多个Sentinel 实例 组成的Sentinel 系统可以监视任意多个主服务器，以及这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器。简单的说哨兵就是带有**自动故障转移功能的主从架构**。**只有Redis的主节点对外提供服务，从节点不提供读写**。

- **无法解决: 1.单节点并发压力问题   2.单节点内存和磁盘物理上限 3.很难扩缩容** 



> cluster集群

即使使用哨兵，redis每个实例也是全量存储，每个redis存储的内容都是完整的数据，浪费内存且有木桶效应。为了最大化利用内存，可以采用**cluster群集**，就是**分布式存储**。即每台redis存储不同的内容。Redis在3.0后开始支持Cluster(模式)模式,它具有**高可用、可扩展性、分布式、容错**等特性。

- Redis集群**不需要sentinel哨兵**，也能完成节点移除和故障转移的功能。
- 需要将每个节点设置成集群模式，这种集群模式没有中心节点，可水平扩展，据官方文档称可以线性扩展到上万个节点(官方推荐不超过1000个节点)。redis集群的性能和高可用性均优于之前版本的哨兵模式，且集群配置非常简单。
- 数据分布的实现原理：Redis 集群将所有数据划分为16384个槽位，每个节点（主节点）负责其中的一部分，当我们根据key存放的时候，Redis根据CRC16算法得出一个结果，然后对16384求余数，这样每个key都会落在0-16383之间，然后找到对应的节点区间。

判断一个是集群中的节点是否可用,是集群中的所用主节点选举过程,如果半数以上的节点认为当前节点挂掉,那么当前节点就是挂掉了,所以搭建redis集群时建议节点数最好为奇数，**搭建集群至少需要三个主节点,三个从节点,至少需要6个节点**。



## 二 分布式缓存集群系统常用的三种算法

### 1.哈希取余算法

2亿条记录就是2亿个k,v，我们单机不行必须要分布式多机，假设有3台机器构成一个集群，用户每次读写操作都是根据公式：`hash(key) % N`个机器台数，计算出哈希值，用来决定数据映射到哪一个节点上。

**优点**：简单粗暴，直接有效，只需要预估好数据规划好节点，例如3台、8台、10台，就能保证一段时间的数据支撑。使用Hash算法让固定的一部分请求落到同一台服务器上，这样每台服务器固定处理一部分请求（并维护这些请求的信息），起到负载均衡+分而治之的作用。

**缺点**：  原来规划好的节点，进行扩容或者缩容就比较麻烦。某个redis机器宕机了，由于台数数量变化，会导致hash取余全部数据重新洗牌。

### 2.一致性哈希算法

​        一致性Hash算法是对2^32取模，简单来说，一致性Hash算法将整个哈希值空间组织成一个虚拟的圆环，整个哈希环为：整个空间按顺时针方向组织，圆环的正上方的点代表0，0点右侧的第一个点代表1，以此类推，2、3、4、……直到2^32-1，也就是说0点左侧的第一个点代表2^32-1， 0和2^32-1在零点中方向重合，我们把这个由2^32个点组成的圆环称为Hash环。

​        将各个服务器使用Hash进行一个哈希，具体可以选择服务器的IP或主机名作为关键字进行哈希，这样每台机器就能确定其在哈希环上的位置。

​		当我们需要存储一个kv键值对时，首先计算key的hash值，hash(key)，将这个key使用相同的函数Hash计算出哈希值并确定此数据在环上的位置，**从此位置沿环顺时针“行走”**，第一台遇到的服务器就是其应该定位到的服务器，并将该键值对存储在该节点上。

**优点**：**容错性**（如果一台服务器不可用，则受影响的数据仅仅是此服务器到其环空间中前一台服务器（即沿着逆时针方向行走遇到的第一台服务器）之间数据，其它不会受到影响。）、**扩展性**

**缺点**：**数据倾斜问题**（被缓存的对象大部分集中缓存在某一台服务器上）

### 3.哈希槽算法

​		哈希槽实质就是一个数组，数组[0,2^14 -1]形成hash slot空间。redis集群采用的就是哈希槽算法。

​		解决均匀分配的问题，在数据和节点之间又加入了一层，把这层称为哈希槽（slot），用于管理数据和节点之间的关系，现在就相当于节点上放的是槽，槽里放的是数据。

​		一个集群只能有16384个槽，编号0-16383（0-2^14-1）。这些槽会分配给集群中的所有主节点，分配策略没有要求。可以指定哪些编号的槽分配给哪个主节点。集群会记录节点和槽的对应关系。解决了节点和槽的关系后，接下来就需要对key求哈希值，然后对16384取余，余数是几key就落入对应的槽里。slot = CRC16(key) % 16384。以槽为单位移动数据，因为槽的数目是固定的，处理起来比较容易，这样数据移动问题就解决了。



## 三 docker容器配置redis集群

### 搭建集群环境

1. 新建六个redis容器实例

   ```bash
   docker run -d --name redis-node-1 --net host --privileged=true -v /data/redis/share/redis-node-1:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6381
   
   docker run -d --name redis-node-2 --net host --privileged=true -v /data/redis/share/redis-node-2:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6382
   
   docker run -d --name redis-node-3 --net host --privileged=true -v /data/redis/share/redis-node-3:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6383
   
   docker run -d --name redis-node-4 --net host --privileged=true -v /data/redis/share/redis-node-4:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6384
   
   docker run -d --name redis-node-5 --net host --privileged=true -v /data/redis/share/redis-node-5:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6385
   
   docker run -d --name redis-node-6 --net host --privileged=true -v /data/redis/share/redis-node-6:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6386
   ```

   - **--net host** : 使用宿主机的端口和ip地址
   - **--cluster-enabled yes**：开启redis集群
   - **--appendonly yes**：开启持久化

   生产环境下，建议修改配置文件，以下是最小的Redis集群配置文件：

   ```properties
   port 7000
   cluster-enabled yes
   cluster-config-file nodes.conf
   cluster-node-timeout 5000
   appendonly yes
   # 想要外网连接，还要注释掉bind 127.0.0.1
   ```

2. 进入容器redis-node-1，并为六个容器构建集群关系

   ```bash
   docker exec -it redis-node-1 /bin/bash
   ```

   ```bash
   redis-cli --cluster create 192.168.188.128:6381 192.168.188.128:6382 192.168.188.128:6383 192.168.188.128:6384 192.168.188.128:6385 192.168.188.128:6386 --cluster-replicas 1   # ip地址修改为你自己的
   			# --cluster-replicas 1 表示为每个master创建一个slave节点	
   ```

   运行结果如下：

   ![image-20220916221934686](/docker配置redis集群/a.png)

3. redis客户端连接6381容器，查看节点状态

   ```bash
   redis-cli -p 6381
   cluster info 
   cluster nodes
   ```

4. 数据读写

   若以`redis-cli -p 6381`的方式连接redis，会发现一些key存不进去（因为key对应的槽位不在6381容器中），要采用集群的方式连接

   ```bash
   redis-cli -p 6381 -c
   ```

   运行结果如下：

   ![image-20220916221934686](/docker配置redis集群/b.png)

5. 查看集群信息

   ```bash
   redis-cli --cluster check 192.168.188.128:6381
   ```

### 容错切换迁移

1.将redis-node-1容器关掉会发生什么？

```bash
docker stop redis-node-1
docker exec -it redis-node-2 /bin/bash
redis-cli --cluster check 192.168.188.128:6382
```

会发现redis-node-1的从机自动上位成了master节点。

2.如何还原之前的三主三从？

1. 开启node1容器，此时node1为从节点
2. 关闭node1的主节点容器，此时node1升级为主节点
3. 开启刚刚关闭的node1主节点容器，此时该节点变为从节点

3.将主节点node1以及它的从节点都关闭会怎么样？

    - 槽位未完全分配，集群就无法提供任何服务。

### 主从扩容

1. 新建6387、6388两个节点

   ```bash
   docker run -d --name redis-node-7 --net host --privileged=true -v /data/redis/share/redis-node-7:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6387
   docker run -d --name redis-node-8 --net host --privileged=true -v /data/redis/share/redis-node-8:/data redis:6.0.8 --cluster-enabled yes --appendonly yes --port 6388
   ```

2. 进入6387容器实例内部

   ```bash 
   docker exec -it redis-node-7 bash
   ```

3. 将6387节点作为master节点加入集群

   ```bash
   redis-cli --cluster add-node 192.168.188.128:6387 192.168.188.128:6381
   
   redis-cli --cluster check 192.168.188.128:6381 # 查看集群情况
   ```

   - 6387 就是将要作为master新增节点

   - 6381 就是原来集群节点里面的领路人，相当于6387拜拜6381的码头从而找到组织加入集群
   - 此时6387节点是主节点，但是槽为是空的

4. 重新分配曹号

   ```bash
   redis-cli --cluster reshard 192.168.111.147:6381
   ```

   运行结果如下：

   ![image-20220916221934686](/docker配置redis集群/c.png)

   可以再次查看集群情况`redis-cli --cluster check 192.168.188.128:6381`，会发现6387节点的槽是分段的，分成了三个区间，因为重新分配成本太高，所以前3家各自匀出来一部分，从6381/6382/6383三个旧节点分别匀出1364个坑位给新节点6387。

5. 为主节点分配从节点6388

   ```bash
   redis-cli --cluster add-node 192.168.188.128:6388 192.168.188.128:6387 --cluster-slave --cluster-master-id e4781f644d4a4e4d4b4d107157b9ba8144631451
   # 命令：redis-cli --cluster add-node ip地址:新slave端口 ip地址:新master端口 --cluster-slave --cluster-master-id master节点ID
   ```

### 主从缩容

目的：将6387和6388节点下线。

1. 获取节点id，将6388节点删除

   ```bash
   redis-cli --cluster check 192.168.188.128:6382
   redis-cli --cluster del-node 192.168.188.128:6388 5d149074b7e57b802287d1797a874ed7a1a284a8 #6388节点id
   ```

2. 将6387槽号清空，重新分配（本例将槽位都给6382）

   ```bash
   redis-cli --cluster check 192.168.188.128:6382
   ```

   运行结果如下：

   ![image-20220916221934686](/docker配置redis集群/d.png)

   此时6387节点的槽位已经清空了。

3. 删除6387节点

   ```bash
   redis-cli --cluster del-node 192.168.188.128:6387 e4781f644d4a4e4d4b4d107157b9ba8144631451 #6387节点id
   ```



## 四 本人的一些疑问及解答

> 网上一些教程上说的是“主从复制，master与slave读写分离”，而很多博客说的是slave不会提供读服务。所以**slave到底会不会提供读服务？提供的话，是读请求自动分配到slave还是说得靠程序逻辑去指定？**

- Redis的从节点默认情况下是不会接受任何读写数据的请求的，从节点的目标只有一个就是作为主节点数据的副本，以提供主节点宕机后的快速恢复。换句话说Redis现在的主从架构默认情况下是不支持读写分离的。
- 再回到读写分离的本质上，为什么需要进行读写分离？无非就是分担主节点的压力，所以如果能够将主节点的压力降低到足够承受的地步，那么久自然不需要所谓的读写分离了。而主节点的压力绝大部分都是来自于主线程的阻塞，所以归根究底就是要把Redis的阻塞扼杀掉。Redis的性能瓶颈常见项。所以总的来说，Redis的主从模式是可以实现读写分离的，但是需要自己修改相关配置，因为Redis的默认机制是所有命令必须由主节点接受并执行，从节点只负责从主节点中拷贝数据，作为一个纯粹的副本。另外，实现读写分离的性价比不高，实际上是可以通过避免使用可能会导致主线程阻塞的相关命令来达到更高的吞吐量。
- 对于读占比较高的场景，确实可以通过把一部分读流量分摊到从节点来减轻主主节点的压力，同时所有写操作必须在主节点中执行。假设Redis现在的从节点可以执行读命令，那么使用了读写分离会有以下几个问题：主从数据延迟问题、主服务器数据过期问题、从节点宕机后路由问题。
- 在redis的官方文档中，对redis-cluster架构上，有这样的说明：在cluster架构下，默认的，一般redis-master用于接收读写，而redis-slave则用于备份，当有请求是在向slave发起时，会直接重定向到对应key所在的master来处理。但如果不介意读取的是redis-cluster中有可能过期的数据并且对写请求不感兴趣时，则亦可通过readonly命令，将slave设置成可读，然后通过slave获取相关的key，达到读写分离。对于redis -cluster主从架构，若要进行读写分离，官方其实是不建议的。



参考链接：

- https://www.cnblogs.com/zhenjungan/p/16504092.html 
- https://www.cnblogs.com/zhenjungan/category/2190380.html 
- https://www.cnblogs.com/kaleidoscope/p/9636264.html

---

> 哨兵模式和集群模式下，springboot该如何去连接？(由于不涉及读写分离，因此问题也简化了)

- 哨兵模式下的yml文件

  ```yml
  spring:
    redis:
      # 数据库（默认为0号库）
      # database: 2
      # 密码（默认空），操作redis需要使用的密码
      password: jwssw
      # 端口号
      #    port: 6379
      #连接超时时间（毫秒）
      timeout: 10000ms
      sentinel:
        master: mymaster  # master书写是使用哨兵监听的那个名称
        nodes:    # 连接的不再是一个具体redis主机,书写的是多个哨兵节点
          - 192.168.229.200:26379
          - 192.168.229.201:26379
          - 192.168.229.202:26379
        # 操作sentinel时需要提供的密码
        password: jwssw
      # 使用lettuce配置
      lettuce:
        pool:
          # 连接池最大连接数（使用负值表示没有限制）
          max-active: 200
          # 连接池中的最大空闲连接
          max-idle: 20
          # 连接池中的最小空闲连接
          min-idle: 5
          # 连接池最大阻塞等待时间（使用负值表示没有限制）
          max-wait: -1ms
  ```

- 集群下的yml文件

  ```yml
  spring:
    redis:
      cluster:
        # 集群节点
        nodes: 192.168.211.158:6382,192.168.211.158:6383,192.168.211.158:6384,192.168.211.158:6385,192.168.211.158:6386,192.168.211.158:6387
        # 最大重定向次数
        max-redirects: 5
      # 密码
      password: myredis
      lettuce:
        pool:
          min-idle: 0
          max-active: 8
          max-wait: -1
          max-idle: 8
          enabled: true
  ```











