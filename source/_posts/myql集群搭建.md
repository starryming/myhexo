---
title: myql集群搭建
date: 2019-03-05 17:24:07
tags:
- linus
- mysql
categories: technology
keywords: mysql
description: 搭建mysql集群 / 主从备
---


搭建mysql集群 / 主从备

#### 准备

|tar包 |路径                                                      |
| ----------- | ------------------------------------------------------------ |
| mysql5.7.11 | [http://dev.MySQL.com/get/Downloads/MySQL-5.7/mysql-5.7.11-Linux-glibc2.5-x86_64.tar.gz](http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.11-Linux-glibc2.5-x86_64.tar.gz) |



#### 预期结果

![](https://ws1.sinaimg.cn/large/006f2SyGly1fzbuvujq1zj30d208a3yi.jpg)



#### 注意点

```java
1、主从备mysql版本最好一致
2、server_id设置不能相同
3、空库配置主从备，或者数据应该一致的状态下
```





-----

####  mysql 安装配置



###### 1、利用xftp上传tar到 /usr/local/



###### 2、解压并重命名

```java
cd /usr/local
tar -zxvf mysql-5.7.11-linux-glibc2.5-x86_64.tar.gz  //解压到当前文件
mv mysql-5.7.11-linux-glibc2.5-x86_64 mysql	// 重命名 
```



###### 3、创建data目录

```
cd mysql
mkdir data
```



###### 4、用户组/用户 添加

```java
groupadd mysql  // 添加用户组
useradd -r -g mysql mysql  // 添加用户
chown -R mysql:mysql ./	  //修改当前目录拥有者为新建的mysql用户
```



###### 5、安装

```java
./bin/mysqld --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data --initialize	 //安装mysql
```



###### 6、任何目录都可启动服务

```java
cp -a ./support-files/my-default.cnf /etc/my.cnf	// 复制配置文件到 /etc/my.cnf
cp -a ./support-files/mysql.server /etc/init.d/mysqld  // mysql进程放入系统进程
ln -s /usr/local/mysql/ /usr/bin/	// 创建ln
service mysqld start	// 启动服务
```



###### 7、环境配置(设置环境变量后，任意目录可执行mysql -uroot -p)

```java
vi /etc/profile

/*
	...
	//编辑的系统环境变量放在java环境变量后
	export PATH=$PATH:/usr/local/mysql/bin
	...
*/

source /etc/profile   // 配置生效
```



###### 8、修改密码

```java
mysql -u root -p	// 后输入上面记录的随机密码
alter user 'root'@'localhost' identified by 'password';	// root 密码为 password
flush privileges;	// 刷新权限
```



###### 9、远程连接允许

```java
use mysql;
update user set user.Host='%' where user.User='root';	// 允许远程连接数据库
flush privileges;	// 刷新权限
```

```java
systemctl stop firewalld.service	// 关闭防火墙
```





-----

#### mysql 主从备配置 



##### 主配置:

###### 1、主服务my.cnf配置



```java
vim /etc/my.cnf
```



```java
// 对my.cnf配置  上述docker安装配置时，已经提前配置
[mysqld]
log-bin=mysql-bin   // [必须]启用二进制日志
server-id=1      // [必须]服务器唯一ID，默认是1，一般取IP最后一段
```



```java
service mysqld restart // 重启服务
```



###### 2、授权slave账号

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

flush tables with read lock; // 锁定表，防止主服务器状态值变化

show master status;	 // 查看二进制日志 log-bin=mysql-bin 没配置将查询为空 配置从服务器要用
/*
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
*/

unlock tables;  // 配置都完了后进行解锁表 
```



#####  从配置:

###### **1、my.cnf基本配置**

```java
// 对my.cnf配置  上述docker安装配置时，已经提前配置
[mysqld]
server-id=2      //[必须]服务器唯一ID
```



```java
service mysqld restart // 重启服务
```

 

######  2、从服务slave配置

```java
show variables like 'server_id';  // 查看server_id,预防配置未生效  server_id重复

// 从服务器配置 ： 注意不要断开，308数字前后无单引号。
change master to master_host='ip地址', master_user='slave', master_password='password',master_log_file='mysql-bin.000001',master_log_pos=430;

/*
master_host='ip地址'		主服务器ip	
master_user='slave'			主服务器slave账号
master_password='password'			主服务器slave密码
master_log_file='mysql-bin.000001'			binlog名
master_log_pos=430			binlog记录位置
*/

- start slave;    //启动从服务器复制功能
```



###### 3、检查从服务器功能状态

```java
show slave status\G
```



```java
/*
   *************************** 1. row ***************************

              Slave_IO_State: Waiting for master to send event
              Master_Host: ip地址  //主服务器地址
              Master_User: slave   //授权帐户名，尽量避免使用root
              Master_Port: 3306    //数据库端口
              Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
              Read_Master_Log_Pos: 430     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
              Relay_Log_File: techtrain29-test1-rgtj1-tj1-relay-bin.000002
              Relay_Log_Pos: 320
              Relay_Master_Log_File: mysql-bin.000001
              Slave_IO_Running: Yes    //此状态必须YES
              Slave_SQL_Running: Yes     //此状态必须YES
                    ......
注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。
*/
```



###### 4、从库只读

```java
show global variables like "%read_only%";	// 查看状态
set global read_only=1;		// 修改只读
show global variables like "%read_only%";

flush privileges;	// 更新
```



#####  mysql备配置

​	同从配置



#### 数据异常解决方案

```java
// 0、查看各库日志，选择日志大的作为同步点。主库为同步点样例：
show master logs;

// 1、主库锁库，防止操作，日志变动。
flush tables with read lock; 	// 结束记得解锁 unlock tables;

// 2、从库查看slave状态
show slave status\G;
stop slave；	// 停止slave
reset master; // 清空日志
reset slave;  // 重置slave

// 3、利用navicate传输数据：主——>从 (或使用命令导出sql)
    
// 4、从库配置 配置同步点
change master to master_host='ip地址', master_user='slave', master_password='password',master_log_file='mysql-bin.000001',master_log_pos=526962;

// 5、查看状态
show slave status\G;
```



