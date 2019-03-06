---
title: Docker 部署
date: 2019-03-05 14:40:17
tags: 
- Docker
- linus
categories: technology
keywords: Docker
description: 在云服务器上、centos7，使用docker容器来搭建各种服务。
---

在云服务器上、centos7，使用docker容器来搭建各种服务。

- docker安装
- redis
- mysql 主从
- rabbitmq
- zookeeper





## Dockers命令

```js
# 镜像
docker images  // 查看
docker rmi [images id]  // 删除

# 容器
docker ps  // 查看
docker stop [container id]	// 停止
docker rm [container id]  // 删除
```



------

## 2019/1/7	**docker安装:**   [view](https://blog.csdn.net/u010046908/article/details/79553227)

(centos需要7.2以上)

  ```javascript
uname -r 	// Docker要求Linux系统的内核版本高与3.10，所以安装前通过命令检查内核版本
sudo yum update		// 更新yum
    
sudo yum install -y yum-utils device-mapper-persistent-data lvm2	// 安装需要的软件包
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo		// 设置docker的yum源
sudo yum install docker-ce	// 安装
  ```

```
sudo systemctl start docker		// 启动
sudo systemctl enable docker	// 开机自启

docker version	// 检查安装成功
```




##  2019/1/10	**docker安装mysql**   [view](https://www.cnblogs.com/sweetchildomine/p/7814692.html)



  **- 数据卷配置**

```java
mkdir /docker
mkdir /docker/mysql
mkdir /docker/mysql/data
mkdir /docker/mysql/mysql-files //安装mysql8.0才需要这个文件夹，5.7不需要
vim /docker/mysql/my.cnf 
```

```java
//下面的内容输入到my.cnf中
[mysqld]
user=mysql
character-set-server=utf8
default_authentication_plugin=mysql_native_password
table_definition_cache=400
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```



  ```java
sudo docker pull mysql:5.7.24	// 拉取镜像
     
docker run -d --privileged=true -p 3306:3306 -v /docker/mysql/my.cnf:/etc/mysql/my.cnf -v /docker/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=password --name mysql mysql:5.7.24

sudo docker ps	// 查看镜像运行
  ```

  ```java
systemctl start firewalld	// 开启防火墙
sudo firewall-cmd --add-port=3306/tcp	// 防火墙开发端口
    
/*sudo systemctl stop firewalld	 关闭防火墙 ——> 可能被入侵	*/
      
docker exec -it mysql bash	// docker进入mysql容器
mysql -u root -p'password' // 然后给mysql设置权限
      
  
// 这样就可以在外面用navicate访问了
  ```





------

## 2019/1/11	mysql主从复制   [view](https://blog.csdn.net/deeplearnings/article/details/78398526)



###  **主配置:**

#### 1、主服务my.cnf配置

```java
// 对my.cnf配置  上述docker安装配置时，已经提前配置
[mysqld]
log-bin=mysql-bin   //[必须]启用二进制日志
server-id=1      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```



```java
service mysql restart // 重启服务
```



#### 2、授权slave账号

```java
// 主服务器上建立帐户并授权slave
GRANT replication slave ON *.* TO 'slave'@'%' IDENTIFIED BY 'password'; 


show variables like 'server_id';  // 查看server_id
/*
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id     | 1     |
+---------------+-------+
*/

show master status;	 // 查看二进制日志 log-bin=mysql-bin 没配置将查询为空 配置从服务器要用
/*
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
*/

/* 注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化 */
```



###  **从配置:**  

#### **1、my.cnf基本配置**

```java
// 对my.cnf配置  上述docker安装配置时，已经提前配置
[mysqld]
server-id=2      //[必须]服务器唯一ID，默认是1，一般取IP最后一段
```



```java
service mysql restart // 重启服务
```

 

#### **2、从服务slave配置** 

```java
show variables like 'server_id';  // 查看server_id,预防配置未生效  server_id重复

// 从服务器配置 ： 注意不要断开，308数字前后无单引号。
change master to master_host='云服务ip', master_user='slave', master_password='password',master_log_file='mysql-bin.000001',master_log_pos=154;

start slave;    //启动从服务器复制功能
```



#### 3、检查从服务器功能状态

```java
show slave status\G
```



```java
/*
   *************************** 1. row ***************************

              Slave_IO_State: Waiting for master to send event
              Master_Host: ***  //主服务器地址
              Master_User: slave   //授权帐户名，尽量避免使用root
              Master_Port: 3306    //数据库端口，部分版本没有此行
              Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
              Read_Master_Log_Pos: 600     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
              Relay_Log_File: DESKTOP-6EUBVFN-relay-bin.000002
              Relay_Log_Pos: 320
              Relay_Master_Log_File: mysql-bin.000001
              Slave_IO_Running: Yes    //此状态必须YES
              Slave_SQL_Running: Yes     //此状态必须YES
                    ......
注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。
*/
```





------

##  2019/1/10  **docker安装redis**    [view](https://blog.csdn.net/woniu211111/article/details/80970560)



###  数据卷配置

  ```java
mkdir /docker
mkdir /docker/redis
mkdir /docker/redis/conf
mkdir /docker/redis/data
   
  
touch /docker/redis/conf/redis.conf  //创建redis.conf配置文件
  
/*	redis.conf文件内容自行添加：
  	切记注释掉：#daemonize yes 否则无法启动容器
  	重要话说三遍：注释掉#daemonize yes，注释掉#daemonize yes，注释掉#daemonize yes
*/
  ```

###  镜像安装

  ```java
docker pull redis // 拉取镜像
docker run -d --privileged=true -p 6379:6379 -v /docker/redis/conf/redis.conf:/etc/redis/redis.conf -v /docker/redis/data:/data --name redistest2 redis:4.0 redis-server /etc/redis/redis.conf --appendonly yes
  
/* 命令说明：
    privileged=true ：容器内的root拥有真正root权限，否则容器内root只是外部普通用户权限
    -p 6379:6379 ： 将容器的6379端口映射到主机的6379端口
    -v /docker/redis/conf/redis.conf:/etc/redis/redis.conf ：映射配置文件
    -v /docker/redis/data:/data ：映射数据目录
    redis-server /etc/redis/redis.conf ：指定配置文件启动redis-server进程
    --appendonly yes ：开启数据持久化
*/
      
docker ps
  
docker exec -ti 'Container Id' redis-cli  // docker进入redis
  ```

 



------

##  2019/1/10	**docker安装rabbitMq:**    [view](https://www.jianshu.com/p/c40166cb4e86)


```java
docker pull rabbitmq:management  // 拉取镜像
docker run -d --name rabbitmq --publish 5671:5671 --publish 5672:5672 --publish 4369:4369 --publish 25672:25672 --publish 15671:15671 --publish 15672:15672 rabbitmq:management		// 启动容器
 
```

```java
// 防火墙开放端口
sudo firewall-cmd --add-port=5672/tcp
sudo firewall-cmd --add-port=15672/tcp

// 浏览器：http://ip:15672/#/
```



### - 设置账户password，因为guest默认password不允许外部ip登陆

```
docker exec -it rabbitmq bash
# 这里的admin可以替换成你的用户名和password
rabbitmqctl add_user admin admin
rabbitmqctl set_user_tags admin administrator
```





-----

##  2019/1/12   **docker安装zookeeper** 



### 1、数据卷 

```java
mkdir /docker/zookeeper
mkdir /docker/zookeeper/data
vim /docker/zookeeper/zoo.cfg
/* 配置内容
clientPort=2181
dataDir=/data
dataLogDir=/datalog
tickTime=2000
initLimit=5
syncLimit=2
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
*/
```



### 2、运行 

```java
docker run -d --privileged=true -p 2181:2181 -v /docker/zookeeper/zoo.cfg:/conf/zoo.cfg -v /docker/zookeeper/data:/data --name zookeeper zookeeper:latest
```



---

##  2019/1/14  **docker 搭建zookeeper集群**

### 安装docker-compose

```java
sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose

docker-compose --version
```



###  yml搭建zookeeper集群

```java
vim docker-compose.yml
/*
version: '3.1'

services:
  zoo1:
    image: zookeeper
    restart: always
    hostname: zoo1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888 server.2=zoo2:2888:3888 server.3=zoo3:2888:3888

  zoo2:
    image: zookeeper
    restart: always
    hostname: zoo2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zoo3:2888:3888

  zoo3:
    image: zookeeper
    restart: always
    hostname: zoo3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888 server.2=zoo2:2888:3888 server.3=0.0.0.0:2888:3888
*/
```

```java
docker-compose up -d
```