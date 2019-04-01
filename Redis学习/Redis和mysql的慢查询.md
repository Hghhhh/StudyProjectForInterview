#Redis的慢查询开启方式：

Redis的慢查询日志是一个FIFO的队列，有最大长度，超过长度之后就要移除前面的日志

配置两个参数：

slowlog-max-len：配置日志队列的最大长度

slowlog-log-slower-than:配置执行命令的阈值，单位是微秒，1秒=1000,000微秒，用微秒做单位，可见Redis有多快。

日志结构：

分别是慢查询日志的识别id、发生时间戳、命令耗时、执行命令和参数。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/1260387-20171221013836443-819021027.png)



#MySQL的慢查询日志

MySQL的慢查询日志是直接持久化保存在文件的

配置参数：

slow_query_log：ON/OFF,开启/关闭慢查询日志 

slow_query_log_file：慢查询日志文件的位置

long_query_time : 配置执行命令的阈值，单位是秒

慢日志的内容：

```sqlite
# Time: 2019-03-18T13:47:13.838290Z
# User@Host: root[root] @ localhost [127.0.0.1]  Id:  2029
# Query_time: 1.374994  Lock_time: 0.000000 Rows_sent: 1  Rows_examined: 2127
SET timestamp=1552916833;
SELECT count(*) FROM tb_user WHERE name like '%%';
```

Time:发生的时间

User：执行命令的用户

Id:该行日志的标识

Query_time: 语句执行的时间

Lock_time: 等待锁的时间

Rows_sent返回的行数

Rows_examined：扫描的行数

最后是执行的语句