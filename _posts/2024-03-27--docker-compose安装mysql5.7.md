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

 以后需要该配置， 就先进该容器，改完再打包新的镜像


## 问题解决

/usr/local/docker-mysql5.7/config/my.cnf:/etc/mysql/conf.d

改成：
/usr/local/docker-mysql5.7/config:/etc/mysql/conf.d

太傻B了， 灯下黑， 当时硬是没看出来

## mysql 配置详解

 ````
 [client]
port = 3306
socket = /var/lib/mysql/mysql.sock
default-character-set = utf8

[mysql]
no-auto-rehash          #仅允许使用键值的updates和deletes

[mysqldump]
quick
max_allowed_packet = 64M

[mysqld]
basedir = /usr/local/mysql
datadir = /var/lib/mysql
port = 3306
socket = /var/lib/mysql/mysql.sock

server-id = 1
log-bin = /data/mysql/binlog/mysql-bin
relay-log = /data/mysql/binlog/mysql-relay-bin
binlog-cache-size = 4M          #设置二进制日志缓存大小
max_binlog_cache_size = 8M      #最大的二进制Cache日志缓冲尺寸
max_binlog_size = 1G            #单个二进制日志文件的最大值，默认1G，最大1G
sync-binlog = 1                 #每隔N秒将缓存中的二进制日志记录写回硬盘
binlog_format = mixed          #二进制日志格式（mixed、row、statement）
#log_slave_updates = 1          #从服务器从主服务器收到的更新记入到从服务器自己的二进制日志文件中
expire_logs_days = 30           #二进制日志文件过期时间
replicate-wild-do-table=testdb1.%
replicate-wild-do-table=testdb2.%
replicate-wild-do-table=testdb3.%

log-error = /data/mysql/log/mysql_error.log     #错误日志文件路径
back_log = 384                  #指出在MySQL暂时停止响应新请求之前，短时间内的多少个请求
character-set-server = utf8     #默认字符集
external-locking = FALSE        #避免外部锁定(减少出错几率，增加稳定性)
skip-name-resolv                #禁止外部连接进行DNS解析
thread_concurrency = 32         #CPU核数的两倍
default-storage-engine=InnoDB   #默认表的类型为InnoDB
transaction_isolation = READ-COMMITTED  #事务隔离级别

open_files_limit = 65535        #Mysql能打开文件的最大个数
max_connections = 5000          #指定MySQL允许的最大连接进程数
max_connect_errors = 6000       #设置每个主机的连接请求异常中断的最大次数
max_allowed_packet = 16M        #服务器一次能处理的最大的查询包的值
wait_timeout = 120              #指定一个请求的最大连接时间
interactive_timeout = 8         #连接保持活动的时间
thread_stack = 192K             #设置Mysql每个线程的堆栈大小，默认值足够大，可满足普通操作

long_query_time = 2             #指定多少秒未返回结果的查询属于慢查询
#slow_query_log = on            #开启慢查询
#slow_query_log_file = /data/mysql/slow.log     #指定慢查询日志文件路径
#log-queries-not-using-indexes  #记录所有没有使用到索引的查询语句
#min_examined_row_limit = 1000  #记录那些由于查找了多余1000次而引发的慢查询
#log-slow-admin-statements      #记录那些慢的OPTIMIZE TABLE,ANALYZE TABLE和ALTER TABLE语句
#log-slow-slave-statements      #记录由slave所产生的慢查询

table_open_cache = 512          #设置高速缓存表的数目
tmp_table_size = 64M            #设置内存临时表最大值
max_heap_table_size = 64M       #独立的内存表所允许的最大容量
thread_cache_size = 64          #服务器线程缓存数，与内存大小有关(建议大于3G设置为64)
#query_cache_size = 32M          #指定MySQL查询缓冲区的大小
query_cache_limit = 2M          #只有小于此设置值的结果才会被缓存
query_cache_min_res_unit = 2k   #设置查询缓存分配内存的最小单位

key_buffer_size = 32M           #指定用于索引的缓冲区大小，增加它可得到更好的索引处理性能
sort_buffer_size = 2M           #设置查询排序时所能使用的缓冲区大小，系统默认大小为2MB
join_buffer_size = 2M           #联合查询操作所能使用的缓冲区大小
read_buffer_size = 4M           #读查询操作所能使用的缓冲区大小
read_rnd_buffer_size = 16M      #设置进行随机读的时候所使用的缓冲区
bulk_insert_buffer_size = 8M    #若经常使用批量插入的特殊语句来插入数据，可以适当调整参数至16MB~32MB，建议8MB

myisam_sort_buffer_size = 64M
myisam_max_sort_file_size = 10G
myisam_repair_threads = 1
myisam_recover                  #自动检查和修复没有适当关闭的MyISAM表

innodb_additional_mem_pool_size = 16M   #设置InnoDB存储的数据目录信息和其他内部数据结构的内存池大小
innodb_buffer_pool_size = 128M          #InnoDB使用一个缓冲池来保存索引和原始数据
innodb_file_io_threads = 4              #InnoDB中的文件I/O线程，通常设置为4
innodb_thread_concurrency = 8           #服务器有几个CPU就设置为几，建议用默认设置，一般设为8
innodb_flush_log_at_trx_commit = 2      #设置为0就等于innodb_log_buffer_size队列满后再统一存储，默认为1
innodb_log_buffer_size = 16M    #默认为1MB，通常设置为6-8MB就足够
innodb_log_file_size = 128M     #确定日志文件的大小，更大的设置可以提高性能，但也会增加恢复数据库的时间
innodb_log_files_in_group = 3   #为提高性能，MySQL可以以循环方式将日志文件写到多个文件。推荐设置为3
innodb_max_dirty_pages_pct = 90 #InnoDB主线程刷新缓存池中的数据
innodb_lock_wait_timeout = 120  #InnoDB事务被回滚之前可以等待一个锁定的超时秒数
innodb_file_per_table = 0       #InnoDB为独立表空间模式，每个数据库的每个表都会生成一个数据空间,0关闭,1开启
 ````

参考文档：  
https://www.cnblogs.com/rmticocean/articles/17761195.html  
https://blog.csdn.net/weixin_33753003/article/details/91599420 