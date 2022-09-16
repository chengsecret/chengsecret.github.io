---
title: "Second_post"
date: 2022-09-16T00:21:35+08:00
draft: true
---

# 第二篇博客 Linux服务器部署

## 前提条件

- 操作系统：CentOS 7.3
- Tomcat 版本：Tomcat9.0.38  去官网找一下
- JDK 版本：JDK1.8.0_181 JDK链接: https://pan.baidu.com/s/1FUlpRB-e5fTXYmxHTLHi_w 提取码: ei4a 
- Mysql5.7    Mysql链接: https://pan.baidu.com/s/168FfNkuP_zb-j56wSrRLtQ 提取码: y2e3 
- 工具FileZilla



## 安装前准备

- 将下载好的Apache Tomcat和JDK安装压缩包上传到Linux实例的根目录下
- 查看安全组规则，8080 22 3306等这些端口是否有用
- 关闭防火墙
  - **systemctl status firewalld**查看防火墙，若为inactive，则是关闭状态；若是开启的，运行**systemctl disable firewalld**
- 关闭SELinux
  - **getenforce**：若为Disabled，则是关闭状态



## 1. Mysql数据库



1. 卸载自带的mariadb-lib

   ```bash
   rpm -qa|grep mariadb  //查看有没有
   ```

   如果有：

   ```bash
   rpm -e mariadb-libs-5.5.44-2.el7.centos.x86_64 //看具体名字是什么
   ```

   再次查看：

   ```bash
   rpm -qa|grep mariadb
   ```

   

2. 检查linux服务器上是否已安全mysql

   ```bash 
   rpm -qa|grep -i mysql
   ```

   若有则删除：

   ```bash
   [root@ecs-sn3-medium-2-linux-20191118115914 ~]# rpm -qa|grep -i mysql
   mysql-libs-5.1.73-8.el6_8.x86_64
   [root@ecs-sn3-medium-2-linux-20191118115914 ~]# rpm -e mysql-libs-5.1.73-8.el6_8.x86_64 --nodeps
   [root@ecs-sn3-medium-2-linux-20191118115914 ~]# rpm -qa|grep -i mysql
   [root@ecs-sn3-medium-2-linux-20191118115914 ~]# 
   ```

3. 上传mysql-5.7.24-1.el6.x86_64.rpm-bundle.tar文件，解压

   ```bash
   tar xvf mysql-5.7.24-1.el6.x86_64.rpm-bundle.tar
   ```

4. 安装对应的包（必须按照顺序）

   ```bash
   rpm -ivh mysql-community-common-5.7.24-1.el6.x86_64.rpm
   rpm -ivh mysql-community-libs-5.7.24-1.el6.x86_64.rpm
   rpm -ivh mysql-community-devel-5.7.24-1.el6.x86_64.rpm //可有可无
   rpm -ivh mysql-community-client-5.7.24-1.el6.x86_64.rpm
   rpm -ivh mysql-community-server-5.7.24-1.el6.x86_64.rpm 
   ```

   若失败，可加上--force --nodeps，如rpm -ivh mysql-community-server-5.7.24-1.el6.x86_64.rpm --force --nodeps

5. 启动数据库

   ```bash
   sudo service mysqld start
   ```

6. 查看MySQL默认密码

   ```bash
   grep 'temporary password' /var/log/mysqld.log
   ```

7. 登陆MySql，输入用户名和密码（密码为上面输出的）

   ```bash 
   mysql -uroot -p
   ```

8. 修改当前用户密码 注意看下面的报错

   ```bash
   //先让密码等级变低
   set global validate_password_policy=LOW;
   //设置密码，密码还是不能太简单
   set password = password('hwy123456');
   ```

9. 开启远程登录，授权root远程登录(否则无法远程登录)

   ```bash
   GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'hwy123456' WITH GRANT OPTION;   //这里的alliance要换成你自己mysql数据库的密码
   
   ```

10. 让命令生效

    ```bash
    flush privileges;
    ```

11. 服务器开启防火墙

    ```bash
    ####设置防火墙开放端口
    [root@localhost ~]# firewall-cmd --zone=public --add-port=3306/tcp --permanent
    
    ####重启防火墙
    [root@localhost ~]# service firewalld  restart
    
    ```

12. 打开3306的安全组



数据库部署完毕。



## 2. JDK

见：https://help.aliyun.com/document_detail/51376.html?spm=5176.doc52806.6.757.bJq7gM

## 3. Tomcat

见：https://help.aliyun.com/document_detail/51376.html?spm=5176.doc52806.6.757.bJq7gM





### Tomcat无法自动解压war包

```bash
cd /usr/local/tomcat/conf
vi server.xml
```

找到这行：

<Host name="localhost" appBase="webapps" unpackWARs="true" autoDeploy="true">

把appBase的值改为webapps