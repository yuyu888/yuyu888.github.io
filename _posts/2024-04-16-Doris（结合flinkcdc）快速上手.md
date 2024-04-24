---
layout: mypost
title: Doris（结合flinkcdc）快速上手
categories: [FlinkCDC, Doris]
---

## Doris 简介

[看这里](https://doris.apache.org/zh-CN/docs/1.2/summary/basic-summary){:target="_blank"}

## Flink 环境搭建

[看这里](https://yuyu888.github.io/posts/2024/03/28/FlinkCDC%E5%90%8C%E6%AD%A5mysql%E6%95%B0%E6%8D%AE%E8%87%B3mysql.html){:target="_blank"} 里的 “Flink 搭建”

下载[Apache Doris pipeline connector 3.0.0](https://github.com/apache/flink-cdc/releases/tag/release-3.0.0){:target="_blank"}  的jra包，并放入 放入 flink-1.19.0 的安装目录下的lib 下

## Doris 安装

[看这里](https://nightlies.apache.org/flink/flink-cdc-docs-master/zh/docs/get-started/quickstart/mysql-to-doris/){:target="_blank"}

docker-compose.yml

````yml
version: '2.1'
services:
  doris:
    image: yagagagaga/doris-standalone
    ports:
      - "8030:8030"
      - "8040:8040"
      - "9030:9030"
````

执行
> docker-compose up -d 


访问 http://localhost:8030/  

![DorisLogin](image0.png)

默认用户名：root 密码为空， 输入用户root 点击登录成功进入


## 在 Doris 里创建库表：demo1.example_tbl

1、创建一个数据库
> create database demo1;  

2、创建数据表
````SQL
use demo;

CREATE TABLE IF NOT EXISTS demo1.example_tbl
(
    `user_id` LARGEINT NOT NULL COMMENT "用户id",
    `create_date` CHAR(10) NOT NULL COMMENT "数据灌入日期时间",
    `city` VARCHAR(20) COMMENT "用户所在城市",
    `age` SMALLINT COMMENT "用户年龄",
    `sex` TINYINT COMMENT "用户性别",
    `last_visit_date` VARCHAR(20) REPLACE DEFAULT "1970-01-01 00:00:00" COMMENT "用户最后一次访问时间",
    `cost` BIGINT SUM DEFAULT "0" COMMENT "用户总消费",
    `max_dwell_time` INT MAX DEFAULT "0" COMMENT "用户最大停留时间",
    `min_dwell_time` INT MIN DEFAULT "99999" COMMENT "用户最小停留时间"
)
AGGREGATE KEY(`user_id`, `create_date`, `city`, `age`, `sex`)
DISTRIBUTED BY HASH(`user_id`) BUCKETS 1
PROPERTIES (
    "replication_allocation" = "tag.location.default: 1"
);
````

表中的列按照是否设置了 AggregationType，分为 Key (维度列) 和 Value（指标列）。没有设置 AggregationType 的，如 user_id、create_date、age ... 等称为 Key，而设置了 AggregationType 的称为 Value。

当我们导入数据时，对于 Key 列相同的行会聚合成一行，而 Value 列会按照设置的 AggregationType 进行聚合。 AggregationType 本例中实现了四种聚合方式：

>SUM：求和，多行的 Value 进行累加。  
>REPLACE：替代，下一批数据中的 Value 会替换之前导入过的行中的 Value。  
>MAX：保留最大值。  
>MIN：保留最小值。  

[字段说明](https://doris.apache.org/zh-CN/docs/1.2/sql-manual/sql-reference/Data-Definition-Statements/Create/CREATE-TABLE){:target="_blank"}

![DorisTable](image1.png)


## 使用flinkCDC 同步mysql数据到Doris

### 创建mysql 数据表

````SQL
CREATE TABLE `doris_example_tbl` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `user_id` bigint(20) NOT NULL COMMENT '用户id',
  `create_date` char(10) NOT NULL COMMENT '数据灌入日期时间',
  `city` varchar(20) DEFAULT NULL COMMENT '用户所在城市',
  `age` smallint(6) DEFAULT NULL COMMENT '用户年龄',
  `sex` tinyint(4) DEFAULT NULL COMMENT '用户性别',
  `last_visit_date` varchar(20) DEFAULT '1970-01-01 00:00:00' COMMENT '用户最后一次访问时间',
  `cost` bigint(20) DEFAULT '0' COMMENT '用户总消费',
  `max_dwell_time` int(11) DEFAULT '0' COMMENT '用户最大停留时间',
  `min_dwell_time` int(11) DEFAULT '99999' COMMENT '用户最小停留时间',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8mb4;
````

### 执行flinkCDC同步命令

> ./bin/start-cluster.sh  
> ./bin/sql-client.sh

### flinkSQL

````SQL
CREATE TABLE `doris_example_tbl`(
            `id` INT NOT NULL,
            `age` SMALLINT NOT NULL,
            `sex` TINYINT NOT NULL,
            `user_id` INT NOT NULL,
            `city` STRING NOT NULL,
            `create_date` STRING NOT NULL,
            `last_visit_date` STRING NOT NULL,
            `cost` BIGINT NOT NULL,
            `max_dwell_time` INT NOT NULL,
            `min_dwell_time` INT NOT NULL,
            PRIMARY KEY (`id`) NOT ENFORCED 
          ) WITH (
          'connector' = 'mysql-cdc', 
          'hostname' = '127.0.0.1',
          'port' = '3306', 
          'username' = 'root', 
            'password' = '123456', 
          'database-name' = 'mytest', 
          'table-name' = 'doris_example_tbl' 
          );

CREATE TABLE `example_tbl`(
            `age` SMALLINT NOT NULL,
            `sex` TINYINT NOT NULL,
            `user_id` INT NOT NULL,
            `city` STRING NOT NULL,
            `create_date` STRING NOT NULL,
            `last_visit_date` STRING NOT NULL,
            `cost` BIGINT NOT NULL,
            `max_dwell_time` INT NOT NULL,
            `min_dwell_time` INT NOT NULL
          ) WITH (
            'connector' = 'doris', 
            'fenodes' = '127.0.0.1:8030', 
            'table.identifier' = 'demo1.example_tbl', 
            'username' = 'root',
            'password' = ''
          );

insert into example_tbl select age, sex,  user_id, city, create_date, last_visit_date, cost, max_dwell_time,  min_dwell_time from doris_example_tbl;

````

![DorisTable](image2.png)

### 在mysql中插入数据

````SQL
INSERT INTO `doris_example_tbl` (`user_id`, `create_date`, `city`, `age`, `sex`, `last_visit_date`, `cost`, `max_dwell_time`, `min_dwell_time`)
VALUES
	(10000, '2017-10-01', '北京', 20, 0, '2017-10-01 06:00:00', 20, 10, 10),
	(10000, '2017-10-01', '北京', 20, 0, '2017-10-01 07:00:00', 15, 2, 2),
	(10001, '2017-10-01', '北京', 30, 1, '2017-10-01 17:05:45', 2, 22, 22),
	(10002, '2017-10-02', '上海', 20, 1, '2017-10-02 12:59:12', 200, 5, 5),
	(10003, '2017-10-02', '广州', 32, 0, '2017-10-02 11:20:00', 30, 11, 11),
	(10004, '2017-10-01', '深圳', 35, 0, '2017-10-01 10:00:15', 100, 3, 3),
	(10004, '2017-10-03', '深圳', 35, 0, '2017-10-03 10:20:22', 11, 12, 6);

````

### 结果对比

mysql的数据  
![mysqlData](image3.png)

Doris的数据  
![DorisData](image4.png)

通过对比发现，user_id的值为10000数据在mysql里是两条，在Doris中合并成了一条  

last_visit_date 被最新入库的值替换  
cost 做了累加
max_dwell_time 取了两条记录中最大的值
min_dwell_time 取了两条记录中最小的值

注意，如果变更mysql里的值，cost会被累加，所以在创建source（CREATE TABLE `doris_example_tbl`....）的时候， 需要加一个参数 debezium.skipped.operations=u,d 

如果需要监听更新和删除的操作并更新结果，需要使用flink DataStreamApi 监听前后变化，计算出新的数据插入到doris
 

