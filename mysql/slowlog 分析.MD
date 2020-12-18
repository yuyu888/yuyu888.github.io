## mysql 慢日志分析
mysqlsla是hackmysql.com推出的一款MySQL的日志分析工具，功能非常强大. 数据报表,非常有利于分析慢查询的原因, 包括执行频率, 数据量, 查询消耗等

### 安装
以centos为例
```
yum install perl-DBI perl-DBD-MySQL perl-devel -y #安装依赖包
wget  ftp://ftp.tw.freebsd.org/pub/distfiles/mysqlsla-2.03.tar.gz #下载安装包
tar -xvzf mysqlsla-2.03.tar.gz #解压
cd mysqlsla-2.03 
perl Makefile.PL #预编译
make #编译
make install #安装
```
PS:最好把此工具安装在本地的测试服务器上，然后将你的慢查询日志拷贝至本地进行分析，以免影响生产环境数据库。

### 运行
```
mysqlsla -lt slow slowlog_20201211.log >fx.log
```

### 结果

```
cat fx.log
```

```
Report for slow logs: slowlog_20201211.log
9.54k queries total, 182 unique
Sorted by 't_sum'
Grand Totals: Time 61.93k s, Lock 1.33k s, Rows sent 62.51k, Rows Examined 283.37M


______________________________________________________________________ 001 ___
Count         : 369  (3.87%)
Time          : 8912.785912 s total, 24.153891 s avg, 257.548 ms to 87.60141 s max  (14.39%)
  95% of Time : 7590.545687 s total, 21.687273 s avg, 257.548 ms to 59.900178 s max
Lock Time (s) : 2.9123 s total, 7.892 ms avg, 21 �s to 499.899 ms max  (0.22%)
  95% of Lock : 26.049 ms total, 74 �s avg, 21 �s to 633 �s max
Rows sent     : 2 avg, 1 to 2 max  (1.14%)
Rows examined : 135.69k avg, 135.69k to 135.69k max  (17.67%)
Database      :
Users         :
	star@ 10.3.1.4 : 52.30% (193) of query, 46.34% (4423) of all users
	star@ 10.3.1.12 : 47.70% (176) of query, 53.34% (5091) of all users

Query abstract:
SET timestamp=N; SELECT level FROM star_contribute WHERE (user_id, idol_id) IN ((N, N), (N, N));

Query sample:
SET timestamp=1607651870;
SELECT `level` FROM `star_contribute` WHERE (`user_id`, `idol_id`) IN ((1006, 3), (1006, 1));

______________________________________________________________________ 002 ___
Count         : 353  (3.70%)
Time          : 8108.992887 s total, 22.971651 s avg, 256.376 ms to 113.197409 s max  (13.09%)
  95% of Time : 6799.351968 s total, 20.296573 s avg, 256.376 ms to 56.802259 s max
Lock Time (s) : 6.894704 s total, 19.532 ms avg, 30 �s to 797.193 ms max  (0.52%)
  95% of Lock : 726.917 ms total, 2.17 ms avg, 30 �s to 99.387 ms max
Rows sent     : 9 avg, 5 to 10 max  (5.16%)
Rows examined : 135.69k avg, 135.69k to 135.69k max  (16.90%)
Database      :
Users         :
	star@ 10.3.1.12 : 52.97% (187) of query, 53.34% (5091) of all users
	star@ 10.3.1.4 : 47.03% (166) of query, 46.34% (4423) of all users

Query abstract:
SET timestamp=N; SELECT * FROM star_contribute WHERE (user_id, idol_id) IN ((N, N), (N, N), (N, N), (N, N), (N, N), (N, N), (N, N), (N, N), (N, N), (N, N));

Query sample:
SET timestamp=1607651870;
SELECT * FROM `star_contribute` WHERE (`user_id`, `idol_id`) IN ((8152, 4), (144876, 3), (129426, 3), (20472, 6), (20472, 4), (48486, 3), (143176, 3), (100004, 4), (143176, 4), (100001, 1));

```

