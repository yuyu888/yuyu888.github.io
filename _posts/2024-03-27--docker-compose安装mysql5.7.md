---
layout: mypost
title: docker-compose安装mysql5.7 
categories: [DOCKER, Mysql]
---

## 前言
本来是个很简单的东西， 结果为了打开binglog搞了半天， 所以写个笔记记录下

## docker-compose.yml

````yml
version: '3'
services:

  mysql:
    image: mysql:5.7 #镜像名称以及版本
    restart: always #重启docker后该容器也重启
    container_name: mysql5.7 #容器名称
    privileged: true
    environment:
      MYSQL_ROOT_PASSWORD: 123456 #指定用户密码
      TZ: Asia/Shanghai
    ports:
      - 3306:3306 #本地端口号与容器内部端口号
    volumes: #指定挂载目录
      - /usr/local/docker-mysql5.7/datadir:/var/lib/mysql
      - /usr/local/docker-mysql5.7/log:/var/log/mysql
#      - /usr/local/docker-mysql5.7/config/my.cnf:/etc/mysql/conf.d
````

不做过多的表述， 请注意最后一行我注释掉了，本意是希望他能读取我本地的配置文件， 结果没起作用



执行：
> docker-compose up -d

等他执行完毕之后， 用 docker ps 发现容器已经起起来了

 ![图例](image1.jpg)

然后用msyql 客户端工具 Sequel Ace 连结，非常顺利， 到此为止，一切正常

## 开启binlog日志


在mysql 中执行
> SHOW VARIABLES LIKE 'binlog_format';

也没啥问题，是我想要的

 ![图例](image2.png)

继续执行 
> show variables like '%log_bin%'

 ![图例](image3.jpg)

 log_bin 的值是OFF， 可是我本地my.cnf 文件确实配置了

 >log-bin=mysql-bin  
 >server_id=1

说明本地配置文件没有成功映射到容器中对应的位置，经过一番实验，与各种翻找， 依然没能成功解决，容器内的mysql安装路径，文件存放路径不好找，而且缺失很多必要的命令， 一旦需要都需要安装,想要做个实验都特别麻烦

## 问题解决

经过不懈努力， 终于在容器内部找到了生效的my.cnf配置：/etc/my.conf

默认配置如下：

````conf
# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/5.7/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
# innodb_buffer_pool_size = 128M
#
# Remove leading # to turn on a very important data integrity option: logging
# changes to the binary log between backups.
# log_bin
#
# Remove leading # to set options mainly useful for reporting servers.
# The server defaults are faster for transactions and fast SELECTs.
# Adjust sizes as needed, experiment to find the optimal values.
# join_buffer_size = 128M
# sort_buffer_size = 2M
# read_rnd_buffer_size = 2M
skip-host-cache
skip-name-resolve
datadir=/var/lib/mysql
socket=/var/run/mysqld/mysqld.sock
secure-file-priv=/var/lib/mysql-files
user=mysql

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

#log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
[client]
socket=/var/run/mysqld/mysqld.sock

!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mysql.conf.d/
````

在 [mysqld] 下添加

 >log-bin=mysql-bin  
 >server_id=1

即可

然后重启下容器 

>docker restart 0668eec594d8

再次运行： SHOW VARIABLES LIKE '%log_bin%';

 ![图例](image4.jpg)

 完美


 ## 临时解决

 把容器打包成新的镜像： docker commit 0668eec594d8 local-mysql5.7

 然后 改下 docker-compose.yml 把image 换成新的镜像

 以后需要该配置， 就先进容器该，改完再打包新的镜像

 有空了再重新研究， 怎么能把配置映射正确， 应该是一个很傻逼的问题，但是今天不想弄了
