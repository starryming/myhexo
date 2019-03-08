---
title: Nginx服务器搭建
date: 2019-03-05 17:11:31
tags: 
- linus
categories: technology
keywords: Nginx
description: 搭建Nginx服务器 / 图片服务器。
---

搭建Nginx服务器 / 图片服务器。

参考 https://blog.csdn.net/CSDN_LQR/article/details/53334583

```java
Nginx:
ftpuser
qweQWE123456
```



## 一、搭建Ngnix



#### 1、nginx安装环境

##### gcc

```java
yum install gcc-c++ 	// gcc 编译依赖gcc环境
```



##### PCRE

```java
yum install -y pcre pcre-devel		// Perl库 nginx的http模块使用pcre来解析正则表达式
```



##### zlib

```java
yum install -y zlib zlib-devel		// nginx使用zlib对http包的内容进行gzip
```



##### openssl

```java
yum install -y openssl openssl-devel
/*OpenSSL 是一个强大的安全套接字层密码库，囊括主要的密码算法、常用的密钥和证书封装管理功能及SSL协议，并提供丰富的应用程序供测试或其它目的使用。nginx不仅支持http协议，还支持https（即在ssl协议上传输http），所以需要在linux安装openssl库。*/
```



-------

#### 2、编译安装

##### tar.gz上传 /usr/



##### 解压

```
tar -zxvf nginx-1.8.0.tar.gz
cd nginx-1.8.0
```



##### 配置

```java
// 新开一个连接窗口
cd /var
mkdir temp
cd /temp
mkdir nginx
```

```java
/*回原窗口执行下面命令*/
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi！
```



##### 编译安装

```java
make	// 编译
make  install	// 安装

cd /usr/local/nginx
ll
```



##### 运行

```java
cd /usr/local/nginx/sbin
./nginx		// 运行
```



#### 3、相关命令

```java
cd /usr/local/nginx/sbin

/*停止*/
./nginx -s quit

/*重启*/
./nginx -s quit
./nginx

/*重新加载配置文件*/
./nginx -s reload
```



#### 4、外部访问 

```java
firewall-cmd --zone=public --add-port=80/tcp --permanent 	// 开发80端口
systemctl stop firewalld.service 	// 重启防火墙
systemctl start firewalld.service 

/* 
	http://129.204.123.212/ 	
*/

```



## 二、搭建vsftp

[搭建参考](https://blog.csdn.net/csdn_lqr/article/details/53333946)

##### 安装：

```java
yum install vsftpd
service vsftpd start
```



```java
使用xftp进行匿名访问测试vsftpd安装成功否
```



##### 关闭匿名访问：

```java
vim /etc/vsftpd/vsftpd.conf

/*
annoymous_enable=YES ——> annoymous_enable=NO

最后面添加端口范围开启：
pasv_min_port=40000
pasv_max_port=40999
*/

service vsftpd restart

chkconfig vsftpd on
```



##### 防火墙开放40000-40999范围端口：

```java
firewall-cmd --list-ports

firewall-cmd --zone=public --add-port=40000-40999/tcp --permanent  

firewall-cmd --reload  
```



```java
getsebool -a | grep ftp

/*
ftpd_anon_write --> off
ftpd_connect_all_unreserved --> off
ftpd_connect_db --> off
ftpd_full_access --> on # 需要开启
ftpd_use_cifs --> off
ftpd_use_fusefs --> off
ftpd_use_nfs --> off
ftpd_use_passive_mode --> off
httpd_can_connect_ftp --> off
httpd_enable_ftp_server --> off
tftp_anon_write --> off
tftp_home_dir --> on # 需要开启
*/

setsebool -P allow_ftpd_full_access on
setsebool -P tftp_home_dir on
```



##### 添加ftp用户:

```java
useradd -d /home/ftpuser1 ftpuser1
passwd ftpuser1
usermod -s /sbin/nologin ftpuser1	// 只允许ftp

service vsftpd restart
```

## 三、搭建Nginx图片服务器



#### 1、nginx/html下创建images文件夹

```java
mkdir /usr/local/nginx/html/image
```



#### 2、修改nginx/conf/nginx.conf在默认的server里再添加一个location并指定实际路径:

```java
location / {
    root html;
    index index.html index.htm;
}
// 添加下面
location /images/ {
    root  /home/ftpuser/www/;
    autoindex on;
}
/*
1)root则是将images映射到/home/ftpuser/www/images/ 
2)autoindex on便是打开浏览功能。
*/
```



#### 3、重启nginx

```
./nginx 0s reload
```



#### 4、修改用户访问权限

```
chown ftpuser /home/ftpuser
chmod 777 -R /home/ftpuser
```



#### 5、测试

```
放一张图片于 /home/ftpuser/www/images

http://129.204.123.212/images/Simple-Order.png
```

