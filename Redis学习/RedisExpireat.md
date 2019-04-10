# Redis Expireat 命令 

EXPIREAT 的作用和 EXPIRE 类似，都用于为 key 设置过期时间。 不同在于 EXPIREAT 命令接受的时间参数是 UNIX 时间戳(unix timestamp)。

Redis Expireat 命令用于以 UNIX 时间戳(unix timestamp)格式设置 key 的过期时间。key 过期后将不再可用。

### 语法

redis Expireat 命令基本语法如下：

```
redis 127.0.0.1:6379> Expireat KEY_NAME TIME_IN_UNIX_TIMESTAMP
```

### 可用版本

\>= 1.0.0

### 返回值

设置成功返回 1 。 当 key 不存在或者不能为 key 设置过期时间时(比如在低于 2.1.3 版本的 Redis 中你尝试更新 key 的过期时间)返回 0 。

### 实例

首先创建一个 key 并赋值：

```
redis 127.0.0.1:6379> SET w3ckey redisOK
```

为 key 设置过期时间：

```
redis 127.0.0.1:6379> EXPIREAT w3ckey 1293840000(integer) 1EXISTS w3ckey(integer) 0
```



##用在哪里？

当我们在进行主从复制的时候，如果使用`expire key time`，计算得到的过期时间是当前的timesteamp+time，从主传到从的时候，主从的过期时间其实会有偏差的，特别是从正在进行全量复制的时候。而是要expireat命令就可以保证主从的过期时间的一样的。