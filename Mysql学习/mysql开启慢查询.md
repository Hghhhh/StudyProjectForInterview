## 简介

开启慢查询日志，可以让MySQL记录下查询超过指定时间的语句，通过定位分析性能的瓶颈，才能更好的优化数据库系统的性能。



## 配置

方法一：全局变量设置
将 slow_query_log 全局变量设置为“ON”状态

```
mysql> set global slow_query_log='ON'; 
```

设置慢查询日志存放的位置

```
mysql> set global slow_query_log_file='/usr/local/mysql/data/slow.log';
```

查询超过1秒就记录

```
mysql> set global long_query_time=1;
```

方法二：配置文件设置
修改配置文件my.cnf，在[mysqld]下的下方加入

```
[mysqld]
slow_query_log = ON
slow_query_log_file = /usr/local/mysql/data/slow.log
long_query_time = 1
```

