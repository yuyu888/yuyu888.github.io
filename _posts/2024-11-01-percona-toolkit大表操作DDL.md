---
layout: mypost
title: percona-toolkit大表操作DDL
categories: [Mysql]
---

## 前言
很多互联网业务都面临着无法停机，需要在线变更数据库结构的情况。但是在线修改数据量较大的表，可能对线上业务产生较大影响，比如：
1. 在线修改大表的表结构执行时间往往不可预估，一般时间较长。
2. 由于修改表结构是表级锁，因此在修改表结构时，影响表写入操作。
3. 如果长时间的修改表结构，中途修改失败，由于修改表结构是一个事务，因此失败后会还原表结构，在这个过程中表都是锁着不可写入。
4. 修改大表结构容易导致数据库 CPU、IO 等性能消耗，使 MySQL 服务器性能降低。
5. 在线修改大表结构容易导致主从延时，从而影响业务读取。

## Percona-Toolkit
Percona-Toolkit 源自 Maatkit 和 Aspersa 工具，这两个工具是管理 MySQL 的最有名的工具，但 Maatkit 已经不维护了，全部归并到 Percona-Toolkit。Percona Toolkit 是一组高级的命令行工具，用来管理 MySQL 和系统任务，主要包括以下功能：
1. 验证主节点和复制数据的一致性
2. 有效的对记录行进行归档
3. 找出重复的索引
4. 总结 MySQL 服务器
5. 从日志和 tcpdump 中分析查询
6. 问题发生时收集重要的系统信息
7. 在线修改表结构

### 安装

官网：https://github.com/percona/percona-toolkit

````sh
git clone https://github.com/percona/percona-toolkit.git

cd your_path/percona-toolkit

perl Makefile.PL
make
make test
make install
````

实际上还需要安装mysql， DBD::MySQL； 安装教程：https://segmentfault.com/a/1190000008664272

安装过程中会遇到各种问题，软件下载不下来，或者文件缺失等， 解决起来特别麻烦
索性就使用docker安装
````sh
docker pull docker.io/perconalab/percona-toolkit

docker run -it perconalab/percona-toolkit 

/usr/bin/pt-online-schema-change  {yourParams} 
````

### percona-toolkit工具介绍

| 工具类别 | 工具命令               | 工具作用                               | 备注     |
|----------|------------------------|----------------------------------------|----------|
| 开发类   | pt-duplicate-key-checker | 列出并删除重复的索引和外键             |          |
| 开发类   | pt-online-schema-change | 在线修改表结构                         |          |
| 开发类   | pt-query-advisor       | 分析查询语句，并给出建议，有bug       | 已废弃   |
| 开发类   | pt-show-grants         | 规范化和打印权限                       |          |
| 开发类   | pt-upgrade             | 在多个服务器上执行查询，并比较不同     |          |
| 性能类   | pt-index-usage         | 分析日志中索引使用情况，并出报告       |          |
| 性能类   | pt-pmp                 | 为查询结果跟踪，并汇总跟踪结果         |          |
| 性能类   | pt-visual-explain      | 格式化执行计划                         |          |
| 性能类   | pt-table-usage         | 分析日志中查询并分析表使用情况         |          |
| 配置类   | pt-config-diff         | 比较配置文件和参数                     |          |
| 配置类   | pt-mysql-summary       | 对mysql配置和status进行汇总            |          |
| 配置类   | pt-variable-advisor    | 分析参数，并提出建议                   |          |
| 监控类   | pt-deadlock-logger     | 提取和记录mysql死锁信息               |          |
| 监控类   | pt-fk-error-logger     | 提取和记录外键信息                     |          |
| 监控类   | pt-mext                | 并行查看status样本信息                 |          |
| 监控类   | pt-query-digest        | 分析查询日志，并产生报告               | 常用命令 |
| 监控类   | pt-trend               | 按照时间段读取slow日志信息             | 已废弃   |
| 复制类   | pt-heartbeat           | 监控mysql复制延迟                     |          |
| 复制类   | pt-slave-delay         | 设定从落后主的时间                     |          |
| 复制类   | pt-slave-find          | 查找和打印所有mysql复制层级关系       |          |
| 复制类   | pt-slave-restart       | 监控salve错误，并尝试重启salve         |          |
| 复制类   | pt-table-checksum      | 校验主从复制一致性                     |          |
| 复制类   | pt-table-sync          | 高效同步表数据                         |          |
| 系统类   | pt-diskstats           | 查看系统磁盘状态                       |          |
| 系统类   | pt-fifo-split          | 模拟切割文件并输出                     |          |
| 系统类   | pt-summary             | 收集和显示系统概况                     |          |
| 系统类   | pt-stalk               | 出现问题时，收集诊断数据               |          |
| 系统类   | pt-sift                | 浏览由pt-stalk创建的文件               |          |
| 系统类   | pt-ioprofile           | 查询进程IO并打印一个IO活动表           |          |
| 实用类   | pt-archiver            | 将表数据归档到另一个表或文件中         |          |
| 实用类   | pt-find                | 查找表并执行命令                       |          |
| 实用类   | pt-kill                | Kill掉符合条件的sql                    | 常用命令 |
| 实用类   | pt-align               | 对齐其他工具的输出                     |          |
| 实用类   | pt-fingerprint         | 将查询转成密文     


## pt-online-schema-change 介绍

pt-online-schema-change 是 Percona-Toolkit 工具集中的一个组件，很多 DBA 在使用 Percona-Toolkit 时第一个使用的工具就是它，同时也是使用最频繁的一个工具。它可以做到在修改表结构的同时（即进行 DDL 操作）不阻塞数据库表 DML 的进行，这样降低了对生产环境数据库的影响。
在 MySQL 5.6.7 之前是不支持 Online DDL 特性的，即使在添加二级索引的时候有 FIC 特性，但是在修改表字段的时候还是会有锁表并阻止表的 DML 操作。这样对于 DBA 来说是非常痛苦的，好在有 pt-online-schema-change 工具在没有 Online DDL 时解决了这一问题，pt-online-schema-change 其主要特点就是在数据库结构修改过程中不会造成读写阻塞。

## 原理对比

### 原生DDL操作
在MySQL5.6之前，在alter这个时间段里面，表是被加了锁的(写锁)，加写锁时其他用户只能select表不能update、insert表。表数据量越大，耗时越长。
mysql在线ddl(加字段、加索引等修改表结构之类的操作）过程如下：
    1、对表加锁(表此时只读)
    2、复制原表物理结构
    3、修改表的物理结构
    4、把原表数据导入中间表中，数据同步完后，锁定中间表，并删除原表
    5、rename中间表为原表
    6、刷新数据字典，并释放锁

### 使用pt-osc工具修改表结构
  pt-osc工具是PT工具包里面的一种，它的全称是pt-online-schema-change，看这个名字，不难猜出来，它是为了在线修改表结构来才创建出来的，所谓的在线修改表，也就是不影响线上业务从而实现修改表结构的效果。

 pt-osc工具的工作原理及步骤 ：
1. 创建需要执行alter操作的原表的一个临时表，然后在临时表中更改表结构。
2. 在原表中创建触发器（3个）三个触发器分别对应insert,update,delete操作
3. 从原表拷贝数据到临时表，拷贝过程中在原表进行的写操作都会更新到新建的临时表。
4. Rename 原表到old表中，在把临时表Rename为原表，最后将原表删除，将原表上所创建的触发器删除。

## pt-online-schema-change 常用参数

````shell
--alter： 
# 结构变更语句，不需要alter table关键字。可以指定多个更改，用逗号分隔
--alter-foreign-keys-method：
#    这个参数是用来处理需要修改的表上具有外键的情况的，如果表上有外键，则需要使用该参数来处理，该参数有4个值，分别是auto、rebuild_constraints、drop_swap、none,一般情况下，选用auto即可，默认值也是auto
--execute 
#  确定修改表，则指定该参数。真正执行。
--charset=utf8
#使用utf8编码，避免中文乱码
--chunk-size
# 对每次导入行数进行控制，已减少对原表的锁定时间。
--print 
# 打印SQL语句到标准输出。指定此选项可以让你看到该工具所执行的语句
--user=    
# 连接mysql的用户名
--password=   
# 连接mysql的密码
--host=     
# 连接mysql的地址
P=            
# 连接mysql的端口号
D=       
#  连接mysql的库名
t=     
# 连接mysql的表名
--recursion-method  
# 发现从的方法，  默认是show processlist，可以指定none来不检查Slave

````
坑点：  
实际实验中由于没有 设置--charset=utf8， 导致COMMENT乱码；  
没有设置--recursion-method=none；导致从库连接失败

查看从库信息 
> SHOW SLAVE HOSTS; 


## 实践

### 原生执行
> ALTER TABLE attendance_20240115 MODIFY COLUMN right_name CHAR(60) NOT NULL;

然后再执行 INSERT 一直锁着 无法执行成功

### 使用 pt-online-schema-change

> /usr/bin/pt-online-schema-change --host=10.130.130.87 --user=root --password=xxxxxx --alter "MODIFY COLUMN right_name CHAR(60) NOT NULL" D=hec_attendance,t=attendance_20240115 --print --execute --recursion-method=none

执行完毕，过一会等触发器建立完毕，插入一条数据能执行成功， 等全部执行完毕，数据依然在

期间可以执行 SHOW TRIGGERS 查看 触发器

执行 show tables 可以看到新增了一个表 _attendance_20240115_new
执行明细：
````log
bash-5.1$ /usr/bin/pt-online-schema-change --host=10.130.130.87 --user=root --password=123456 --alter "MODIFY COLUMN right_name CHAR(60) NOT NULL" D=hec_attendance,t=attendance_20240115 --print --execute --recursion-method=none
No slaves found.  See --recursion-method if host test.contract11.happyelements.net has slaves.
Not checking slave lag because no slaves were found and --check-slave-lag was not specified.
Operation, tries, wait:
  analyze_table, 10, 1
  copy_rows, 10, 0.25
  create_triggers, 10, 1
  drop_triggers, 10, 1
  swap_tables, 10, 1
  update_foreign_keys, 10, 1
Altering `hec_attendance`.`attendance_20240115`...
Creating new table...
CREATE TABLE `hec_attendance`.`_attendance_20240115_new` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `unique_id` char(64) NOT NULL,
  `day` date NOT NULL,
  `begin` datetime NOT NULL,
  `end` datetime NOT NULL,
  `work_hour` decimal(4,2) NOT NULL,
  `week` char(32) NOT NULL,
  `is_holiday` tinyint(4) DEFAULT NULL,
  `official_holiday` tinyint(4) NOT NULL DEFAULT '0' COMMENT '????????0:???1??',
  `late` tinyint(4) NOT NULL,
  `leave_early` tinyint(4) NOT NULL,
  `work_time_enough` tinyint(4) NOT NULL,
  `right_name` char(128) NOT NULL,
  `right_flag` tinyint(4) DEFAULT NULL,
  `overtime_totaltime` decimal(4,2) NOT NULL DEFAULT '0.00' COMMENT '?????',
  `overtime_list` text COMMENT '????',
  `application_totaltime` decimal(11,2) NOT NULL DEFAULT '0.00',
  `application_list` text COMMENT '????',
  `is_valid` tinyint(1) NOT NULL DEFAULT '1' COMMENT '???????1????0???',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '????',
  `period_name` varchar(10) NOT NULL DEFAULT '' COMMENT '??????',
  PRIMARY KEY (`id`),
  UNIQUE KEY `unique_id_day` (`unique_id`,`day`) USING BTREE,
  KEY `day` (`day`),
  KEY `is_holiday_index` (`is_holiday`),
  KEY `official_holiday_index` (`official_holiday`),
  KEY `unique_id_late` (`unique_id`,`late`),
  KEY `unique_id_right_flag` (`unique_id`,`right_flag`),
  KEY `unique_id_is_holiday` (`unique_id`,`is_holiday`),
  KEY `unique_id` (`unique_id`)
) ENGINE=InnoDB AUTO_INCREMENT=3240226 DEFAULT CHARSET=utf8
Created new table hec_attendance._attendance_20240115_new OK.
Altering new table...
ALTER TABLE `hec_attendance`.`_attendance_20240115_new` MODIFY COLUMN right_name CHAR(60) NOT NULL
Altered `hec_attendance`.`_attendance_20240115_new` OK.
2024-11-01T02:07:42 Creating triggers...
-----------------------------------------------------------
Event : DELETE
Name  : pt_osc_hec_attendance_attendance_20240115_del
SQL   : CREATE TRIGGER `pt_osc_hec_attendance_attendance_20240115_del` AFTER DELETE ON `hec_attendance`.`attendance_20240115` FOR EACH ROW BEGIN DECLARE CONTINUE HANDLER FOR 1146 begin end; DELETE IGNORE FROM `hec_attendance`.`_attendance_20240115_new` WHERE `hec_attendance`.`_attendance_20240115_new`.`id` <=> OLD.`id`; END
Suffix: del
Time  : AFTER
-----------------------------------------------------------
-----------------------------------------------------------
Event : UPDATE
Name  : pt_osc_hec_attendance_attendance_20240115_upd
SQL   : CREATE TRIGGER `pt_osc_hec_attendance_attendance_20240115_upd` AFTER UPDATE ON `hec_attendance`.`attendance_20240115` FOR EACH ROW BEGIN DECLARE CONTINUE HANDLER FOR 1146 begin end; DELETE IGNORE FROM `hec_attendance`.`_attendance_20240115_new` WHERE !(OLD.`id` <=> NEW.`id`) AND `hec_attendance`.`_attendance_20240115_new`.`id` <=> OLD.`id`; REPLACE INTO `hec_attendance`.`_attendance_20240115_new` (`id`, `unique_id`, `day`, `begin`, `end`, `work_hour`, `week`, `is_holiday`, `official_holiday`, `late`, `leave_early`, `work_time_enough`, `right_name`, `right_flag`, `overtime_totaltime`, `overtime_list`, `application_totaltime`, `application_list`, `is_valid`, `update_time`, `period_name`) VALUES (NEW.`id`, NEW.`unique_id`, NEW.`day`, NEW.`begin`, NEW.`end`, NEW.`work_hour`, NEW.`week`, NEW.`is_holiday`, NEW.`official_holiday`, NEW.`late`, NEW.`leave_early`, NEW.`work_time_enough`, NEW.`right_name`, NEW.`right_flag`, NEW.`overtime_totaltime`, NEW.`overtime_list`, NEW.`application_totaltime`, NEW.`application_list`, NEW.`is_valid`, NEW.`update_time`, NEW.`period_name`); END
Suffix: upd
Time  : AFTER
-----------------------------------------------------------
-----------------------------------------------------------
Event : INSERT
Name  : pt_osc_hec_attendance_attendance_20240115_ins
SQL   : CREATE TRIGGER `pt_osc_hec_attendance_attendance_20240115_ins` AFTER INSERT ON `hec_attendance`.`attendance_20240115` FOR EACH ROW BEGIN DECLARE CONTINUE HANDLER FOR 1146 begin end; REPLACE INTO `hec_attendance`.`_attendance_20240115_new` (`id`, `unique_id`, `day`, `begin`, `end`, `work_hour`, `week`, `is_holiday`, `official_holiday`, `late`, `leave_early`, `work_time_enough`, `right_name`, `right_flag`, `overtime_totaltime`, `overtime_list`, `application_totaltime`, `application_list`, `is_valid`, `update_time`, `period_name`) VALUES (NEW.`id`, NEW.`unique_id`, NEW.`day`, NEW.`begin`, NEW.`end`, NEW.`work_hour`, NEW.`week`, NEW.`is_holiday`, NEW.`official_holiday`, NEW.`late`, NEW.`leave_early`, NEW.`work_time_enough`, NEW.`right_name`, NEW.`right_flag`, NEW.`overtime_totaltime`, NEW.`overtime_list`, NEW.`application_totaltime`, NEW.`application_list`, NEW.`is_valid`, NEW.`update_time`, NEW.`period_name`);END
Suffix: ins
Time  : AFTER
-----------------------------------------------------------
2024-11-01T02:07:42 Created triggers OK.
2024-11-01T02:07:42 Copying approximately 3135963 rows...
INSERT LOW_PRIORITY IGNORE INTO `hec_attendance`.`_attendance_20240115_new` (`id`, `unique_id`, `day`, `begin`, `end`, `work_hour`, `week`, `is_holiday`, `official_holiday`, `late`, `leave_early`, `work_time_enough`, `right_name`, `right_flag`, `overtime_totaltime`, `overtime_list`, `application_totaltime`, `application_list`, `is_valid`, `update_time`, `period_name`) SELECT `id`, `unique_id`, `day`, `begin`, `end`, `work_hour`, `week`, `is_holiday`, `official_holiday`, `late`, `leave_early`, `work_time_enough`, `right_name`, `right_flag`, `overtime_totaltime`, `overtime_list`, `application_totaltime`, `application_list`, `is_valid`, `update_time`, `period_name` FROM `hec_attendance`.`attendance_20240115` FORCE INDEX(`PRIMARY`) WHERE ((`id` >= ?)) AND ((`id` <= ?)) LOCK IN SHARE MODE /*pt-online-schema-change 8 copy nibble*/
SELECT /*!40001 SQL_NO_CACHE */ `id` FROM `hec_attendance`.`attendance_20240115` FORCE INDEX(`PRIMARY`) WHERE ((`id` >= ?)) ORDER BY `id` LIMIT ?, 2 /*next chunk boundary*/
Copying `hec_attendance`.`attendance_20240115`:   2% 24:19 remain
Copying `hec_attendance`.`attendance_20240115`:   2% 32:24 remain
Copying `hec_attendance`.`attendance_20240115`:   3% 38:47 remain
Copying `hec_attendance`.`attendance_20240115`:   4% 42:32 remain
.......
........
........
Copying `hec_attendance`.`attendance_20240115`:  98% 01:10 remain
Copying `hec_attendance`.`attendance_20240115`:  99% 00:44 remain
Copying `hec_attendance`.`attendance_20240115`:  99% 00:18 remain
2024-11-01T03:37:13 Copied rows OK.
2024-11-01T03:37:13 Swapping tables...
RENAME TABLE `hec_attendance`.`attendance_20240115` TO `hec_attendance`.`_attendance_20240115_old`, `hec_attendance`.`_attendance_20240115_new` TO `hec_attendance`.`attendance_20240115`
2024-11-01T03:37:13 Swapped original and new tables OK.
2024-11-01T03:37:13 Dropping old table...
DROP TABLE IF EXISTS `hec_attendance`.`_attendance_20240115_old`
2024-11-01T03:37:14 Dropped old table `hec_attendance`.`_attendance_20240115_old` OK.
2024-11-01T03:37:14 Dropping triggers...
DROP TRIGGER IF EXISTS `hec_attendance`.`pt_osc_hec_attendance_attendance_20240115_del`
DROP TRIGGER IF EXISTS `hec_attendance`.`pt_osc_hec_attendance_attendance_20240115_upd`
DROP TRIGGER IF EXISTS `hec_attendance`.`pt_osc_hec_attendance_attendance_20240115_ins`
2024-11-01T03:37:14 Dropped triggers OK.
Successfully altered `hec_attendance`.`attendance_20240115`.
````

通过日志可以清晰的看到，整个创建新表，创建触发器，同步数据，RENAME TABLE， 删除触发器等一系列操作过程