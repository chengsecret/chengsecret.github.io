---
title: "Docker常用容器安装"
subtitle: ""
date: 2022-09-22T15:28:11+08:00
lastmod: 2022-09-22T15:28:11+08:00
draft: false
toc:
  enable: true
weight: false
categories: ["docker"]
tags: ["docker","入门","云原生"]
---



运行一个容器不仅仅是`docker run `，还需要考虑端口、数据的持久化、容器自身的特殊命令等等，这样的容器在生产环境中才有实际意义。



## 安装tomcat

> 安装命令

```bash
docker pull billygoo/tomcat8-jdk8
docker run -d -p 8080:8080 --name mytomcat8 billygoo/tomcat8-jdk8
```

Linux系统安装了浏览器的话，就可以通过http://localhost:8080来访问。

> 问题：访问不了？

- 可能没有映射端口，即： `-p 8080:8080`

- 可能下载的tomcat版本过新，需要把webapps.dist目录名修改为webapps即可。

  ```bash
  docker exec -it 容器id /bin/bash
  ls -l
  rm -r webapps  #是一个空文件夹
  mv webapps.dist webapps #给webapps.dist 文件夹换名字
  ```

> 外网访问不了？

- 如果服务器是云服务器，需要开启**安全组8080端口**和**防火墙8080端口**
- 如果是本地虚拟机，开启**防火墙8080端口**，如果还是访问不了就重启一下。虚拟机ip地址可以在虚拟机中使用`ifconfig`命令获取，ip地址在`ens33`。

## Centos一些常用防火墙命令

```bash
# 开放端口
firewall-cmd --zone=public --add-port=5672/tcp --permanent
firewall-cmd --reload   # 配置立即生效

#关闭端口
firewall-cmd --zone=public --remove-port=5672/tcp --permanent 
firewall-cmd --reload   # 配置立即生效

# 查看防火墙状态
firewall-cmd --state

#查看想开的端口是否已开
firewall-cmd --query-port=8888/tcp

#查看防火墙所有开放的端口
firewall-cmd --zone=public --list-ports

# 开启防火墙
systemctl start firewalld.service  		#启动firewall
systemctl enable firewalld.service		#开机时启动firewall

# 关闭防火墙
systemctl stop firewalld.service        #停止firewall
systemctl disable firewalld.service		#禁止firewall开机启动

#重启防火墙
systemctl restart firewalld.service
```

## 安装mysql

### 常规容器安装

1. 拉取镜像

   ```bash
   docker pull mysql:5.7
   ```

2. 运行容器

   ```bash
   docker run -d -p 3306:3306 --privileged=true -v /mysql/log:/var/log/mysql -v /mysql/data:/var/lib/mysql -v /mysql/conf:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=1234567  --name mysql mysql:5.7
   ```

   - 数据库中的数据不能丢失，所以数据卷一定要配置上
   - `-e MYSQL_ROOT_PASSWORD=1234567`是MySQL容器的特殊命令，表示配置root账户的登陆密码

3. 配置编码（不配置的话插入中文数据会报错）

   ```bash
   cd /mysql/conf/   # 通过数据卷同步到容器中，这个目录是 docker run 时配置的目录
   vim my.cnf
   ```

   将下面这段写入my.cnf:

   ```toml
   [client]
   default_character_set=utf8
   [mysqld]
   collation_server = utf8_general_ci
   character_set_server = utf8
   ```

4. 重启容器，查看字符编码

   ```bash
   docker restart mysql
   docker exec -it mysql bash
   mysql -uroot -p
   # 输入密码
   show variables like 'character%';
   ```

之后，还可以删除容器，再重新跑一个mysql容器，只要数据卷映射的文件夹不变，数据就会自动恢复了。

### 安装主从复制

1. 新建主服务器容器，端口为3307

   ```bash
   docker run -p 3307:3306 --name mysql-master -v /mydata/mysql-master/log:/var/log/mysql -v /mydata/mysql-master/data:/var/lib/mysql -v /mydata/mysql-master/conf:/etc/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
   ```

2. 新建配置文件my.cnf

   ```bash
   cd /mydatamysql-master/conf/
   vim my.cnf
   ```

   my.cnf内容如下：

   ```toml
   [mysqld]
   collation_server = utf8_general_ci
   character_set_server = utf8
   
   ## 设置server_id，同一局域网中需要唯一
   server_id=101 
   ## 指定不需要同步的数据库名称
   binlog-ignore-db=mysql  
   ## 开启二进制日志功能
   log-bin=mall-mysql-bin  
   ## 设置二进制日志使用内存大小（事务）
   binlog_cache_size=1M  
   ## 设置使用的二进制日志格式（mixed,statement,row）
   binlog_format=mixed  
   ## 二进制日志过期清理时间。默认值为0，表示不自动清理。
   expire_logs_days=7  
   ## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
   ## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
   slave_skip_errors=1062
   
   [client]
   default_character_set=utf8
   ```

3. 重启容器后进入容器

   ```bash
   docker restart mysql-master
   docker exec -it mysql-master bash
   root@5d6bc6db6d53:/# mysql -uroot -p
   Enter password: 
   ```

4. master容器内创建用户身份，用于slave容器同步数据

   ```bash
   create user 'slave'@'%' identified by '123456';
   grant replication slave, replication client on *.* to 'slave'@'%';
   ```

5. 新建从服务器容器，端口为3308

   ```bash
   docker run -p 3308:3306 --name mysql-slave -v /mydata/mysql-slave/log:/var/log/mysql -v /mydata/mysql-slave/data:/var/lib/mysql  -v /mydata/mysql-slave/conf:/etc/mysql -e MYSQL_ROOT_PASSWORD=root -d mysql:5.7
   ```

6. 新建配置文件my.cnf

   ```bash
   cd /mydata/mysql-slave/conf/
   vim my.cnf
   ```

   my.cnf内容如下：

   ```toml
   [mysqld]
   collation_server = utf8_general_ci
   character_set_server = utf8
   
   ## 设置server_id，同一局域网中需要唯一
   server_id=102
   ## 指定不需要同步的数据库名称
   binlog-ignore-db=mysql  
   ## 开启二进制日志功能，以备Slave作为其它数据库实例的Master时使用
   log-bin=mall-mysql-slave1-bin  
   ## 设置二进制日志使用内存大小（事务）
   binlog_cache_size=1M  
   ## 设置使用的二进制日志格式（mixed,statement,row）
   binlog_format=mixed  
   ## 二进制日志过期清理时间。默认值为0，表示不自动清理。
   expire_logs_days=7  
   ## 跳过主从复制中遇到的所有错误或指定类型的错误，避免slave端复制中断。
   ## 如：1062错误是指一些主键重复，1032错误是因为主从数据库数据不一致
   slave_skip_errors=1062  
   ## relay_log配置中继日志
   relay_log=mall-mysql-relay-bin  
   ## log_slave_updates表示slave将复制事件写进自己的二进制日志
   log_slave_updates=1  
   ## slave设置为只读（具有super权限的用户除外）
   read_only=1
   
   [client]
   default_character_set=utf8
   ```

7. 重启容器后进入容器

   ```bash
   docker restart mysql-slave
   docker exec -it mysql-slave bash
   root@5d6bc6db6d53:/# mysql -uroot -p
   Enter password: 
   ```

8. 主数据库中查看主从同步状态

   ```bash
   show master status;
   ```

   我的内容如下：

   ```bash
   +-----------------------+----------+--------------+------------------+-------------------+
   | File                  | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
   +-----------------------+----------+--------------+------------------+-------------------+
   | mall-mysql-bin.000001 |      617 |              | mysql            |                   |
   +-----------------------+----------+--------------+------------------+-------------------+
   ```

9. 从服务器中配置主从复制

   ```bash
   change master to master_host='宿主机ip', master_user='slave', master_password='123456', master_port=3307, master_log_file='mall-mysql-bin.000001', master_log_pos=617, master_connect_retry=30;
   ```

   参数说明：

   - master_host：主数据库的IP地址；
   - master_port：主数据库的运行端口；
   - master_user：在主数据库创建的用于同步数据的用户账号；
   - master_password：在主数据库创建的用于同步数据的用户密码；
   - master_log_file：指定从数据库要复制数据的日志文件，**通过查看主数据的状态，获取File参数；**（第八步）
   - master_log_pos：指定从数据库从哪个位置开始复制数据，**通过查看主数据的状态，获取Position参数**；（第八步）
   - master_connect_retry：连接失败重试的时间间隔，单位为秒

10. 从数据库中开启主从复制

    ```bash
    start slave;
    ```

11. 查看主从状态

    ```bash
    show slave status \G;
    ```

    `Slave_IO_Running`与`Slave_SQL_Running`都为**yes**，则成功！

12. 测试主机新增库表，从机查看数据。



## 安装redis

1. 拉取镜像

   ```bash
   docker pull redis:6.0.8
   ```

2. 编辑redis配置文件，利用数据卷挂载到容器中

   ```bash
   mkdir -p /redis  #创建宿主机目录，存放redis容器中的数据
   cd /redis
   vim redis.conf  # 去网上找一个redis初始配置文件，将内容复制进去
   ```

3. 修改redis.conf中的内容

   ```bash
   requirepass 1234567 #redis密码验证，开启会安全很多
   bind 0.0.0.0  #将配置文件中的bind 127.0.0.1注释掉或改成bind 0.0.0.0 ；不改则只能本机访问redis
   daemonize no # 将daemonize yes注释起来或者 daemonize no设置，因为该配置和docker run中-d参数冲突，会导致容器一直启动失败
   # appendonly yes  开启redis数据持久化  可选
   ```

4. 运行容器

   ```bash
   docker run  -p 6379:6379 --name redis --privileged=true -v /redis/redis.conf:/etc/redis/redis.conf -v /redis/data:/data -d redis:6.0.8 redis-server /etc/redis/redis.conf
   ```

5. redis-cli连接

   ```bash
   docker exec -it redis bash
   root@7c5608e916c8:/data# redis-cli
   root@7c5608e916c8:/data# auth 1234567
   root@7c5608e916c8:/data# exit
   ```

   













