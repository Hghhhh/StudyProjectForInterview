#如何用Redis保存一段时间的热点数据？

当redis使用的内存超过了设置的最大内存时，会触发redis的key淘汰机制，在redis 3.0中有6种淘汰策略：

noeviction: 不删除策略。当达到最大内存限制时, 如果需要使用更多内存,则直接返回错误信息。（redis默认淘汰策略）
allkeys-lru: 在所有key中优先删除最近最少使用(less recently used ,LRU) 的 key。
allkeys-random: 在所有key中随机删除一部分 key。
volatile-lru: 在设置了超时时间（expire ）的key中优先删除最近最少使用(less recently used ,LRU) 的 key。
volatile-random: 在设置了超时时间（expire）的key中随机删除一部分 key。
volatile-ttl: 在设置了超时时间（expire ）的key中优先删除剩余时间(time to live,TTL) 短的key。
场景：

数据库中有1000w的数据，而redis中只有50w数据，如何保证redis中10w数据都是热点数据？

方案：

 限定 Redis 占用的内存，Redis 会根据自身数据淘汰策略，留下热数据到内存。所以，计算一下 50W 数据大约占用的内存，然后设置一下 Redis 内存限制即可，并将淘汰策略为volatile-lru或者allkeys-lru。  

设置Redis最大占用内存：

打开redis配置文件，设置maxmemory参数，maxmemory是bytes字节类型

```redis
#In short... if you have slaves attached it is suggested that you set a lower
#limit for maxmemory so that there is some free RAM on the system for slave
#output buffers (but this is not needed if the policy is 'noeviction')
#maxmemory <bytes>
maxmemory 268435456
```


设置过期策略：

```redis
maxmemory-policy volatile-lru
```



作者：代码搬运工. 
来源：CSDN 
原文：https://blog.csdn.net/u013308490/article/details/87737810 
版权声明：本文为博主原创文章，转载请附上博文链接！





所以我们要缓存一天内的热点数据，可以为数据设置1天的过期时间，然后设置redis的淘汰策略为volatile-lru。