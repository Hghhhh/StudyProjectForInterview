##Redis常用命令

| 命令               | 解释                  | 时间复杂度 |
| ------------------ | --------------------- | ---------- |
| keys *             | 查找所有的key         | o(n)       |
| dbsize             | key的数量             | o(1)       |
| exits key          | 判断key是否存在       | o(1)       |
| del key            | 删除key               | o(1)       |
| expire key seconds | key在seconds秒后过期  | o(1)       |
| ttl key            | 查看key剩余的过期时间 | o(1)       |
| persist key        | 去掉key的过期时间     | o(1)       |
| type key           | 返回key的类型         | o(1)       |

## 数据结构和内存编码

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84%E5%92%8C%E5%86%85%E9%83%A8%E7%BC%96%E7%A0%81.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redisObject.jpg)

## 单线程

redis是单线程的

单线程为什么这么快？

- 纯内存（主要）
- 非阻塞IO
- 避免线程的切换和竞态切换

单线程要注意什么？

- 一次只运行一条命令
- 拒绝慢（长）命令：keys,flushall,fluashdb,slow lua script,mutil/exex,operate big value(collection)
- 其实不是单线程: fysnc file descriptor , close file descriptor

## Redis API的使用和理解

api的具体例子可以查看http://doc.redisfans.com/index.html

###string

value最大为512M

场景：

- 缓存
- 计数器
- 分布式锁

| API                               | 解释                                       |
| --------------------------------- | ------------------------------------------ |
| get key                           | 获取key对应的value                         |
| set key value                     | 设置key-value，不管key存不存在都设置       |
| del key                           | 删除key-value                              |
| incr key                          | key的值自增1                               |
| decr key                          | key的值自减1                               |
| incrby key k                      | key的值自增k（k为整数）                    |
| decrby key k                      | key的值自减k                               |
| incrbyfloat key k                 | k可以为浮点数，负数（故没有decrbyfloat）   |
| setnx key value                   | key不存在才设置                            |
| set key value xx                  | key存在才设置                              |
| mget key1 key2 key3 ..            | 批量获取key                                |
| mset key1 values1 key2 value2 ... | 批量设置key-value                          |
| getset key newvalue               | 设置新值并返回旧值,没有旧值返回1           |
| append key value                  | 将value追加到旧value后                     |
| strlen key                        | 返回字符串的长度，key不存在返回0，注意中文 |
| getrange key start end            | 获取字符串指定下标的所有值                 |
| setrange key index value          | 设置指定下标的所有对应的值                 |

实战：

- 缓存用户个人页面的访问数 incr viewNum
- 缓存视频地址伪代码
- 分布式id生成器

### hash

结构：

| key  | field | value        |
| ---- | ----- | ------------ |
| user | email | 11321@qq.com |
|      | name  | 231          |
|      | sex   | man          |

特点：

- Mapmap？
- small redis
- field不能相同，value可以相同

| API                                                    | 解释                               | 时间复杂度 |
| ------------------------------------------------------ | ---------------------------------- | ---------- |
| hget key field                                         | 获取hash key对应得filed的value     | o(1)       |
| hset key field value                                   | 设置hash key对应的field的value     | o(1)       |
| hsetnx key filed value                                 |                                    |            |
| hdel key field                                         | s删除hash key对应field的value      | o(1)       |
| hexists key filed                                      | 判断hash key是否存在field          | o(1)       |
| hlen key                                               | 获取hash key field的数量           | o(1)       |
| hmget key field1 field2 field3 ...                     | 批量获取hash key的一批field        | o(n)       |
| hmset key field1 value1 field2 value2 filed3 value3 .. | 批量设置一批field value            | o(n)       |
| hgetall key                                            | 返回hash key对应所以的field和value | o(n)       |
| hvals key                                              | 返回hash key对应所有field的value   | o(n)       |
| hkeys key                                              | 返回所有hash key对应所有field      | o(n)       |
| hincrby key field k                                    | k可以为正、负、小数                | o(1)       |
| hincrbyfloat key field k                               | k为小数                            |            |

实战：

- 记录每个网站每个用户的页面访问量：hincrby user pageview count

- 缓存视频的基本信息（数据源在mysql中）：

  ```java
  public VideoInfo get (long id){
      String redisKey = redisPrefix+id;
      Map<String,String> hashMap =redis.hgetAll(redisKey);
      VideoInfo videoInfo = tranferMapToVideo(hashMap);
      if(videoInfo==null){
          videoInfo = mysql.get(id);
          if(videoInfo!=null){
              redis.hmset(redisKey,transferVideoToMap(videoInfo);
          }                  
      }
                          return videoInfo;
  }
  ```

### list

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%88%97%E8%A1%A8%E7%BB%93%E6%9E%84.jpg)



![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%88%97%E8%A1%A8%E7%BB%93%E6%9E%842.jpg)

特定：

- 有序
- 可重复
- 左右两边弹出

| API                                      | 解释                                                         | 时间复杂度 |
| ---------------------------------------- | ------------------------------------------------------------ | ---------- |
| lpush key value1 value2 ...valueN        | 从列表左端插入值（1-N）                                      | o(1-n)     |
| rpush key value1 value2 ...valueN        | 从列表右端插入值（1-N）                                      | o(1-n)     |
| linsert key before\|after value newValue | 在list指定的值前\|后插入newValue                             | o(n)       |
| lpop key                                 | 从列表左边弹出一个value                                      | o(1)       |
| rpop key                                 | 从列表右边弹出一个value                                      | o(1)       |
| lrem key count value                     | 根据count值，从列表中删除所有value相等的项（1）count>0,从左到右删除最多count个value相等的元素（2）count<0，从右到左，删除最多Math.abs(count)个value相等的元素（3）count=0,删除所有value相等的项 | o(n)       |
| ltrim key start end                      | 按照范围修减列表，eg:ltrim listkey 1 4 ,表示只保留1-4的元素，其余删除 | o(n)       |
| lrang key start end（包含end）           | 获取列表指定索引范围的所有item                               | o(n)       |
| lindex key index                         | 获取对应key下标的元素                                        | o(n)       |
| llen key                                 | 获取列表的长度                                               | o(1)       |
| lset key index newValue                  | 设置列表指定索引值为newValue                                 | o(n)       |
| blpop  key timeout                       | lpop阻塞版本，timeout是阻塞超时时间，timeout=0为永远不阻塞   | o(1)       |
| brpop  key timeout                       | rpop阻塞版本，timeout是阻塞超时时间，timeout=0为永远不阻塞   | o(1)       |
|                                          |                                                              |            |

实战：

- 你关注的人微博更新，lpush

tips:

- lpush + lpop = stack
- lpush + rpop = queue
- lpush + ltrim = Capped Collection
- lpush + brpop = Message Queue

### set

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/set1.jpg)

特定：

- 无序
- 无重复
- 支持集合间存在：交、差、并集

| API                               | 解释                                                         | 时间复杂度             |
| --------------------------------- | ------------------------------------------------------------ | ---------------------- |
| sadd key element                  | 向集合key添加element（如果element已经存在则添加失败）        | o(1)                   |
| srem key element                  | 向集合key删除element                                         | o(1)                   |
| scard key                         | 计算集合的大小                                               | o(1)                   |
| sismember key element             | p判断element是否在key中                                      | o(n)                   |
| srandmember key [count]           | 如果没有count则从集合中随机返回一个element，如果有count，如果cout为正数，返回不重复的元素，cout为负数，返回的元素可能重复 | o(1)（o(n)如果有cout） |
| smembers key                      | 返回所有elements                                             | o(n)                   |
| sdiff key1 key2                   | key1和key2的差集                                             | o(n)                   |
| sinter key1 key2                  | 交集                                                         | o(n)                   |
| sunion key1 key2                  | 并集                                                         | o(n)                   |
| sdiffstore destination key1 key2  | 这个命令的作用和 [*SDIFF*](http://doc.redisfans.com/set/sdiff.html#sdiff) 类似，但它将结果保存到 `destination` 集合 | o(n)                   |
| sinterstore destination key1 key2 | 同上                                                         | o(n)                   |
| sunion destination key1 key2      | 同上                                                         | o(n)                   |
| **SPOP key**                      | 移除并返回集合中的一个随机元素。                             | O(1)                   |

tips:
[*SMEMBERS*](http://doc.redisfans.com/set/smembers.html#smembers) 命令被用于处理一个大的集合键时， 它们可能会阻塞服务器达数秒之久。因此可以使用[scan](http://doc.redisfans.com/key/scan.html)命令族，支持增量式迭代， 它们每次执行都只会返回少量元素。

实战：

- sadd = tagging
- spop/srandmember = Random item
- sadd +sinter = social Graph

### zset/sortedSet

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/ZSET.jpg)

特点：

- 无重复
- 有序
- element+score

| API                                              | 解释                                                         | 时间复杂度                                            |
| ------------------------------------------------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| zadd key score element(可以多对)                 | 添加score和element                                           | o(longN)                                              |
| zrem key element                                 | 删除元素                                                     | o(1)                                                  |
| zscore key element                               | 返回元素的分数                                               | o(1)                                                  |
| zincrby key increScore element                   | 增加或减少元素的分数                                         | o(1)                                                  |
| zcard key                                        | 返回元素的个数                                               | o(1)                                                  |
| zrange key start end [withscores]                | 返回指定索引范围内的升序元素                                 | o(log(N)+m),`N` 为有序集的基数，而 `M` 为结果集的基数 |
| zrangebyscore key minscore maxscore [withscores] | 返回指定分数范围内的升序元素                                 | o(log(N)+m)                                           |
| zcount key minScore maxScore                     | 返回有序集合内在指定分数范围内的个数                         | o(log(N)+m)                                           |
| zremrangebyrank key start end                    | 删除指定排名内的升序元素                                     | o(log(N)+m)                                           |
| zremrangebyscore key start end                   | 删除指定分数内的升序元素                                     | o(log(N)+m)                                           |
| **ZRANK key member**                             | 返回有序集 `key` 中成员 `member` 的排名。其中有序集成员按 `score` 值递增(从小到大)顺序排列。 | O(log(N))                                             |
| **ZREVRANK key member**                          | 返回有序集 `key` 中成员 `member` 的排名。其中有序集成员按 `score` 值递减(从大到小)排序。 | O(log(N))                                             |

实战：

- 排行榜

## Jedis and JedisPool

导包：

```me&#39;v
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.0.1</version>
</dependency>
```

### jedis

```java
Jedis jedis = new Jedis("localhost",6379);
        jedis.set(key,value);
		System.out.println(jedis.get(key));
        jedis.close();
```

### jedisPool

```java
JedisPool jp = new JedisPool("localhost",6379);
Jedis jedis = jp.getResource();
jedis.get(key);
jedis.close();
//这里的close并不是关闭jedis连接，而是将jedis放回连接池
```

### spring整合redis

https://blog.csdn.net/plei_yue/article/details/79362372

## 慢查询



Redis客户端执行一条命令分为4个部分：

1）发送命令

2）排队

3）执行命令

4）返回结果

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.jpg)



两个配置：

**slowlog-max-len**:决定 slow log *最多*能保存多少条日志， **slow log 本身是一个 FIFO 队列，当队列大小超过 `slowlog-max-len` 时，最旧的一条日志将被删除，而最新的一条日志加入到 slow log ，以此类推。**

**slowlog-log-slower-than:**决定要对执行时间大于多少**微秒**(microsecond，1秒 = 1,000,000 微秒)的查询进行记录。

如何配置：
config get slowlog-max-len

config get slowlog-log-slower-than

config set slowlog-max-len 1000

config set slowlog-log-slower-than 1000

相关命令：
**slowlog get [number]**:获取慢查询队列

**slowlog len**: 获取慢查询条数

**slowlog reset** : 重置慢查询队列

```redis
127.0.0.1:6379> slowlog get
# 可以看到每个慢查询日志有4个属性组成，分别是慢查询日志的识别id、发生时间戳、命令耗时、执行命令和参数。
1) 1) (integer) 1
   2) (integer) 1513709400
   3) (integer) 11
   4) 1) "slowlog"
      2) "get"
2) 1) (integer) 0
   2) (integer) 1513709398
   3) (integer) 4
   4) 1) "config"
      2) "set"
      3) "slowlog-log-slower-than"
      4) "2"
```

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/1260387-20171221013836443-819021027.png)

运维经验：

- slowlog-max-len不要设置过小，通常1000左右
- slowlog-log-slower-than 不要设置过大，通常1ms
- 理解命令生命周期：慢查询仅包括执行阶段
- 定期执行slowlog get命令将慢查询日志持久化到其他存储中（例如，MySQL）



## pipeline

简单来说，如果每条命令都需要发起一起网络请求，那么效率很低，管道（pipeline）可以一次性发送多条命令并在执行完后一次性将结果返回，pipeline通过减少客户端与redis的通信次数来实现降低往返延时时间。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/%E6%B5%81%E6%B0%B4%E7%BA%BF.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/%E6%B5%81%E6%B0%B4%E7%BA%BF%E4%BD%9C%E7%94%A8.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/%E6%B5%81%E6%B0%B4%E7%BA%BF%E4%BD%9C%E7%94%A8%E7%BB%AD.jpg)

### pipeline-Jedis实现

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%BD%BF%E7%94%A8pipeline.jpg)

注意：

- mset等m操作是原子的，但是pipeline不是原子的，在执行时拆分成各条命令执行
- 注意每次pipeline携带的数据流
- pipeline每次只能作用在一个redis节点
- [Pipeline源码分析](https://blog.csdn.net/ouyang111222/article/details/50942893):pipeline实现了自己的输入输出流，一次性将输出流输出到redis server



## 发布订阅

角色：

- 发布者（publisher）
- 订阅者（subscriber）
- 频道（channel）

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%8F%91%E5%B8%83%E8%AE%A2%E9%98%85%E6%A8%A1%E5%9E%8B.jpg)

类型生产者消费者模型，发布者发布消息后，订阅者会收到信息

发布-订阅命令：

publish channel message：发布消息

subscribe [channel...] :订阅频道，一个或多个

unsubscribe  [channel...] ：取消订阅

其他API：

psubscribe [pattern...] :订阅模式，每个模式以 `*` 作为匹配符，比如 `it*` 匹配所有以 `it` 开头的频道( `it.news` 、 `it.blog` 、 `it.tweets` 等等)， `news.*` 匹配所有以 `news.` 开头的频道( `news.it` 、 `news.global.today` 等等)，诸如此类。

punsubscribe [pattern...]：退订指定的模式

pubsub channels :列出至少有一个订阅者的频道

pubsub numsub [channel...] ：给出指定频道的订阅者数量

pubsub numpat ：列出被订阅模式的数量，注意， 这个命令返回的不是订阅模式的客户端的数量， 而是客户端订阅的所有模式的数量总和



## bitmap（位图）

所谓的Bit-map就是用一个bit位来标记某个元素对应的Value， 而Key即是该元素。由于采用了Bit为单位来存储数据，因此在存储空间方面，可以大大节省。

| api                               | 解释                                                         | 时间复杂度 |
| --------------------------------- | ------------------------------------------------------------ | ---------- |
| setbit key offset value           | 给位图指定索引设置值                                         | o(1)       |
| getbit key offset                 | 获取位图指定索引的值                                         | o(1)       |
| bitcount key [start end]          | 获取位图指定范围位值为1的个数                                | o(n)       |
| bitop op destkey key [key...]     | 做多个Bitmap的and（交集）、or(并集)、not（非）、xor（异或）操作并将结果保存在destkey中 | o(n)       |
| bitpos key targetbit [start- end] | 计算位图指定范围（start到end，单位为字节，不指定就是获取全部）第一个偏移量对应的值等于targetBit的位置 | o(n)       |
|                                   |                                                              |            |
|                                   |                                                              |            |

使用经验：

- type=string,最大512mb
- 注意setbit时的偏移量，可能有较大耗时
- 位图不是绝对好

实战：

[用bitmap统计上亿访问量的周活跃用户](https://www.jianshu.com/p/62cf39db5c2f)

## HyperLogLog

[HyperLogLog](https://www.cnblogs.com/ysuzhaixuefei/p/4052110.html) 可以接受多个元素作为输入，并给出输入元素的基数估算值：

- 基数：集合中不同元素的数量。比如 {'apple', 'banana', 'cherry', 'banana', 'apple'} 的基数就是 3 。 

-  估算值：算法给出的基数并不是精确的，可能会比实际稍微多一些或者稍微少一些，但会控制在合理的范围之内

HyperLogLog 的优点是，即使输入元素的数量或者体积非常非常大，计算基数所需的空间总是固定的、并且是很小的。
在 Redis 里面，每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算接近 2^64 个不同元素的基数。这和计算基数时，元素越多耗费内存就越多的集合形成鲜明对比。
但是，因为 HyperLogLog 只会根据输入元素来计算基数，而不会储存输入元素本身，所以HyperLogLog 不能像集合那样，返回输入的各个元素。

| API                                       | 解释                      | 时间复杂度                     |
| ----------------------------------------- | ------------------------- | ------------------------------ |
| pfcount key [key ..]                      | 计算hyperloglog的独立总数 | o(n)，n为被计算的hyperloglog数 |
| pfadd key element [element ...]           | 向hyperloglog添加元素     | o(n),n为被添加的元素数         |
| pfmerge destkey sourcekey [sourcekey ...] | 合并多个hyperloglog       | o(n)                           |
|                                           |                           |                                |

使用经验：

- 是否能容忍错误？（错误率：0.81%）
- 是否需要单条数据？



## GEO（地理信息定位)

Redis3.2版本提供了[GEO](https://blog.csdn.net/xiangnan10/article/details/80225929) (地理位置定位)功能(注意：只有3.2以上的Redis版本才能使用)，支持存储地理位置信息来实现诸如附近位置、摇一摇这类依赖于地理位置信息的功能，对于需要实现这些功能的开发者来说是一大福音。GEO功能是Redis的另一做着Matt Stancliff借鉴NoSQL数据库Ardb实现的，Ardb的作者是一名中国人，它提供了优秀的GEO功能。

存储经纬度，计算两地距离，范围计算等

应用场景：
微信摇一摇

周边餐饮等

| API                                                          | 解释                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| geoadd key longitude latitude member [longitude latitude member...] | 添加地理位置信息,longitude: 经度,latitude: 纬度,例子：geoadd points 104.074977 30.560872 世纪城地铁站 |
| geodist key member1 member2 [unit]                           | 返回两个给定位置之间的距离,指定单位的参数 unit 必须是以下单位的其中一个：**m** 表示单位为米(默认)。 **km** 表示单位为千米。 **mi** 表示单位为英里。 **ft** 表示单位为英尺。 |
| geopos key member [member...]                                | 从key里返回所有给定位置元素的位置（经度和纬度）              |
| [georadius](http://www.redis.net.cn/order/3689.html)         | 以给定的经纬度为中心， 找出某一半径内的元素                  |
| [georadiusbymember](http://www.redis.net.cn/order/3690.html) | 找出位于指定范围内的元素，中心点是由给定的位置元素决定       |
| zrem key member                                              | 删除地理位置信息                                             |

注意：
geo的key的type是zset



## Redis持久化

redis所有数据保存在内存中，对数据的更新将异步地保存在磁盘上。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E6%8C%81%E4%B9%85%E5%8C%96%E6%96%B9%E5%BC%8F.jpg)

快照（Snapshot）的定义是：关于指定数据集合的一个完全可用拷贝，该拷贝包括相应数据在某个时间点（拷贝开始的时间点）的映像。快照可以是其所表示的数据的一个副本，也可以是数据的一个复制品。

### RDB

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%BB%80%E4%B9%88%E6%98%AFRDB.jpg)

触发机制的三种方式

- save：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/save.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/save2.jpg)

- bgsave：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/bgsave.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/bgsave2.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/saveandbgsave.jpg)

- 自动：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E8%87%AA%E5%8A%A8%E7%94%9F%E6%88%90rdb.jpg)

一些配置：

```redis
save 900 1  //在900s内改变了1条就写入rdb文件
save 300 10
save 60 10000
dbfilename dump.rdb//rdb文件的名字
dir ./   //日志文件存在哪里
stop-writes-on-bgsave-error yes //如果bgsave发生了错误是否停止写入
rdbcompression yes  //rdb是否采用压缩格式
rdbchecksum yes  //是否对rdb文件进行检验
```

最佳配置：

```redis
dbfilename dump-${port}.rdb
dir /bigdiskpath
stop-writes-on-bgsave-error yes
rdbcompression yes
```

触发机制-不容忽略方式：

- 全量复制
- debug reload
- shutdown

总结：

- RDB是redis内存到硬盘的快照，，用于持久化
- save通常会阻塞redis
- bgsave不会阻塞redis，但是会fork新进程
- save自动配置满足任一就会被执行
- 有些触发机制不容忽视：主从复制、shutdown、reload

### AOF

RDB有什么问题：

- 耗时耗性能：o(n)数据，fork()消耗内存，disk i/o性能消耗
- 不可控、丢失数据

什么是AOF：

redis会将每一个收到的写命令都通过write函数追加到文件中(默认是 appendonly.aof)。

当redis重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。

当然由于os会在内核中缓存 write做的修改，所以可能不是立即写到磁盘上。这样aof方式的持久化也还是有可能会丢失部分修改。不过我们可以通过配置文件告诉redis我们想要 通过fsync函数强制os写入到磁盘的时机。

AOF的三种策略：

- always：每次收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用

- everysec：每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐

- no：完全依赖os，操作系统决定什么时候刷到硬盘就什么时候，性能最好,持久化没保证

| 命令 | always     | everysec      | no     |
| ---- | ---------- | ------------- | ------ |
| 优点 | 不丢失数据 | 每秒一次fsync | 不用管 |
| 缺点 | IO开销大   | 丢1秒数据     | 不可控 |

AOF重写：去掉不必要的写命令

AOF重写作用：

- 减少硬盘占用空间
- 加速恢复速度

AOF重写的两种实现方式：

- bgrewriteaof（手动）:

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/bgrewriteaof.jpg)

- AOF重写配置（自动）：

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/aof%E9%87%8D%E5%86%99.jpg)

最佳配置：

```redis
appendonly yes //启用aof持久化方式
appendfilename "appendonly-${port}.aof"
appendfsync everysec
dir /bigdiskpath
no-appendfsync-on-rewrite yes //#yes : 在日志重写时，不进行命令追加操作，而只是将其放在缓冲区里，避免与命令的追加造成DISK IO上的冲突。#no : 在日志重写时，命令追加操作照常进行
auto-aof-rewrite-percentage 100 //aof重写触发，文件大小增加100%
auto-aof-rewrite-min-size 64mb //aof重写触发，文件大小到达64mb
```

注意：
no-appendfsync-on-rewrite 的解释：

在默认情况下 当aof进行重写的时候，aof的同步信息不是关闭的。在这种情况下 。子进程rewrite在写硬盘 主进程 aof也在写硬盘。在rewrite的过程中 子进程对主进程造成了磁盘阻塞（disk io冲突），导致了报警信息的产生。但是这个参数修改成 yes之后 ，又会有安全上的问题。当rewrite的过程中 要是redis down掉的话 丢失的数据 就不是之前appendfsync 定下的策略。



### RDB和AOF的抉择

对比:

| 命令       | RDB    | AOF      |
| ---------- | ------ | -------- |
| 启动优先级 | 低     | 高       |
| 体积       | 小     | 大       |
| 恢复速度   | 快     | 慢       |
| 数据安全性 | 丢数据 | 根据策略 |
| 轻重       | 重     | 轻       |

RDB最佳策略：

- 平时关闭，主从复制等的时候触发才使用
- 集中管理：集中备份的时候可以用，因为文件小
- 主从，从开？

AOF最佳策略：

- 大部分情况（缓存和存储）下开
- AOF重写不应集中
- everysec

最佳策略：

- 小分片
- 根据缓存或存储来决定使用哪种持久化
- 监控（硬盘、内存、负载、网络）
- 足够的内存（不要全部给redis，留一些给fork）

###总结：

**RDB：**
1）RDB文件用于保存和还原Redis服务器所有数据库中所有键值对数据

2）SAVA命令由服务器进程直接执行保存操作，所以该命令会阻塞服务器

3）BGSAVE由子进程执行保存操作，所以该命令不会阻塞服务器

4）服务器状态中会保存所有用save配置的条件，当满足任何一个保存条件时，服务器自动执行BGSAVE

5）RDB文件是一个经过压缩的二进制文件，由多个部分组成

**AOF:**

1）AOF通过保存所有修改数据库的写命令请求来记录服务器的数据库状态

2）命令请求会先保存到AOF缓冲区，之后在定期写入并同步到AOF文件

3）根据appendfsync选项的（always、everysec、no）对AOF持久化功能的安全性以及Redis服务器的性能有很大影响

4）AOF重写可以产生一个新的AOF文件，这个新文件和原来的AOF文件保存的数据库状态一样，但体积更小

5）**AOF重写的实现是通过读取数据库中的键值对来实现的，无需对现有AOF进行读入、分析等操作。**

6）在执行BGREWRITEAOF命令时，Redis服务器会维护一个**AOF重写缓冲区**，该缓冲区**会在子进程创建新AOF文件期间，记录服务器的所有写命令**。当子进程完成创建新的AOF文件之后，服务器会将重写缓冲区中的内容**追加到新AOF文件的末尾**，**使得新旧AOF文件所保存的数据库状态一致**，最后新的AOF替换旧的AOF文件。

7）根据AOF文件恢复数据库的时候，会创建一个伪客户端来执行AOF中的每一条命令

**AOF和RDB的不同：**

1）存储的方式不同，RDB存储的是redis数据库的键值对，而AOF存储的是写命令

2）AOF的更新频率通常比RDB高，RBD很可能会丢数据，AOF是根据策略来保证会不会丢数据，所以如果AOF持久化功能开启了，服务器会优先选择根据AOF去还原数据库状态

3）AOF的恢复速度没有RDB快

## 开发运维常见问题

### fork操作

1. 同步操作
2. 与内存量息息相关:redis占用的内存越大，耗时越长（与机器类型有关）
3. info: latest_fork_uses（查看生成fork使用的时间）

4. 改善fork：

- 机器硬件加强
- 控制redis实例最大可用内存：maxmemory
- 合理配置linux内存分配策略：vm.overcommit_memory=1
- 降低fork频率：例如放宽AOF重写自动触发时机，不必要的全量复制

### 子进程开销和优化

1. cpu 

- 开销：RDN和AOF文件生成，属于CPU密集型
- 优化：不做CPU绑定，不和CPU密集型部署

2. 内存

- 开销：fork内存开销，copy-on-write
- 优化： echo never > /svs/kernel/mm/transparent_hugepage/enabled

3. 硬盘

   - 开销：AOF和RDB文件写入，可以结合iostat，iotop做分析

   - 优化：

     （1）不要和高硬盘负载服务器部署在一起：存储服务、信息队列等；

     （2）no-appendfsync-on-rewirte=yes

     （3）根据写入量决定磁盘类型：例如ssd

     （4）单机多实例持久化文件目录可以考虑分盘

### AOF追加阻塞

使用everysec(每秒)刷盘策略的流程：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/aof%E8%BF%BD%E5%8A%A0%E9%98%BB%E5%A1%9E.jpg)

1.主线程负责AOF缓冲区

2.AOF线程负责每秒一次同步磁盘操作,并记录最近一次同步时间.

3.主线程对比AOF同步时间:

3.1如果距离上次同步时间在两秒内,主线程直接返回。

3.2如果距离上次同步时间超过两秒(意思是现在还在同步),主线程将会被阻塞, 直到同步完成。

AOF阻塞定位：

- redis日志
- info Persistence ：aof_delayed_fsync阻塞次数.
- 通过硬盘判断

解决方案

1.打开no-appendfsync-on-rewrite参数, 默认关闭,表示AOF重写期间不做sync操作, 并不能根本解决问题, 因为故障转移前没有发生AOF重写。

2.关闭AOF, 如果一组(主-从) 同时宕机, 会丢失5分钟数据，启动redis时如果没有发现AOF文件，redis 会选择RDB来恢复数据,rdb copy-on-write到磁盘的频率5分钟一次。

3.提升磁盘写入速度。

## Redis主从复制

### 配置

redis.conf:

```
slaveof masterIp masterPort
```

```
masterauth password
```

```redis
slave-read-only yes
```

或者命令行直接配置，无需重启，消除slave命令行：slave of no one

info replication：查看master、slave信息（类似mysql的show master status和show slave status）



### 全量复制

将master之前的数据也传到salve

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%85%A8%E9%87%8F%E5%A4%8D%E5%88%B6.jpg)

全量复制开销

1. bgsave时间
2. RDB文件网络传输时间
3. 从节点清空数据时间
4. 从节点加载RDB时间
5. 可能的AOF重写时间

### 部分复制

slave前面已经有了master数据了，再次同步master的数据

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E9%83%A8%E5%88%86%E5%A4%8D%E5%88%B6.jpg)

### 主从复制常见问题

1. 读写分离：读流量分摊到从节点

   - 可能遇到的问题：

     （1）复制数据延迟

     （2）读到过期数据

     （3）从节点故障

2. 配置不一致

   - 例如maxmemory不一致：丢失数据
   - 例如数据结构优化参数（例如hash-max-ziplist-entries）：内存不一致

3. 规避全量复制

   - 第一次全量复制不可避免：办法是使用小主节点、低峰

   - 节点运行ID不匹配：（1）主节点重启了（runId改变了）（2）进行了故障转移，例如哨兵或集群

   - 复制积压缓冲区不足：

     （1）网络中断了，部分复制无法满足

     （2）增大复制缓冲区配置rel_backlog_size，网络增强“

4. 规避复制风暴：

   - 单主节点复制风暴：原因是主节点重启多从节点负载，解决方法是更换复制拓扑
   - 单机器复制风暴：解决办法是主节点分散多机器

##Sentinel（哨兵）

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/sentinel.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/sentinel%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB.jpg)

sentinel主要配置

```redis
port ${port}
dir "/usr/redis/data"
logfile "${port}.log"
sentinel monitor mymaster 127.0.0.1 7000 2 
//用两个哨兵监视mymaster这个节点，要给出节点的ip和端口

sentinel auth-pass <master-name> <password>

sentinel down-after-milliseconds mymaster 30000 
//30000ms连接不上就认为mymaster down掉了

sentinel paraller-syncs mymaster 1 
//这个配置项指定了在发生failover主备切换时最多可以有多少个slave同时对新的master进行 同步，这个数字越小，完成failover所需的时间就越长，但是如果这个数字越大，就意味着越 多的slave因为replication而不可用。可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。

sentinel failover-timeout mymaster 180000
//哨兵有一个规则：如果哨兵投票给另一个哨兵来进行给定主机的故障转移，它将等待一段时间尝试再次进行同一主机的故障转移。此延迟是您可以在sentinel.conf中配置的故障转移超时。这意味着sentinel不会同时尝试故障转移同一主机，第一个请求授权的人将尝试，如果失败，另一个将在一段时间后尝试，依此类推。
```



### 客户端实现原理

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%AE%9E%E7%8E%B0%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/sentinel%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/sentinel%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%862.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/sentinel%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%863.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/sentinel%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%864.jpg)

### Jedis客户端

```java
Set sentinelSet = new HashSet();
sentinelSet.add("120.77.208.81:26379");
sentinelSet.add("120.77.208.81:26380");
sentinelSet.add("120.77.208.81:26381");
JedisSentinelPool sentinelPool = new JedisSentinelPool(masterName,sentinelSet,poolConfig,timeOut);
Jedis jedis = null;
try{
    jedis = redisSentinelPool.getResource();
    
}catch(Exception e){
    logger.error(e.getMessage(),e);
} finally{
    if(jedis!=null)
        jedis.close();
}
```

###三个定时任务

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%B8%89%E4%B8%AA%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E7%AC%AC%E4%BA%8C%E4%B8%AA%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E7%AC%AC%E4%B8%89%E4%B8%AA%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1.jpg)

### 主观下线和客观下线

主观下线：每个sentinel节点对Redis节点连接失败的“偏见”

客观下线：所有sentinel节点对Redis节点失败“达成共识”（超过quorum个统一）

sentinel is-master-down-by-addr

当前哨兵一旦监测到某个主节点实例主观下线之后，就会向其他哨兵发送”is-master-down-by-addr”命令，询问其他哨兵是否也认为该主节点主观下线了。如果有超过quorum个哨兵（包括当前哨兵）反馈，都认为该主节点主观下线了，则当前哨兵就将该主节点实例标记为客观下线。



### 领导者选举

- 原因：只有一个sentinel节点完成故障转移
- 选举：通过sentinel is-master-down-by-addr命令希望成为领导者
  1. 每个做主观下线的Sentinel节点向其他Sentinel节点发送命令，要求将它设置为领导者
  2. 收到命令的Sentinel节点如果没有同意通过其他Sentinel节点发送的命令，那么将同意该请求，否则拒绝
  3. 如果该Sentinel节点发现自己的票已经超过半数且超过quorum，那么它将成为领导者
  4. 如果此过程有多个Sentinel节点成为了领导者，那么将等待一段时间重新进行选举

### 故障转移

1. 从slave节点中选择一个合适的节点作为新master节点
2. 对上面的slave节点执行slaveof no one 命令让其成为master节点
3. 向剩余的slave节点发送命令，让它们成为新master节点的slave节点，复制规则和parallel-syncs参数有关
4. 更新对原来master节点配置为slave，并保持对其关注，当其恢复后命令它去复制新的master节点

三个消息：

- +switch-master：将原来的一个从节点切换为主节点
- +convert-to-slave：将原来的主节点将为从节点
- +sdown：主观下线



**注意：什么是合适的slave节点？**

1. 先选择slave-priority最高的slave节点，如果存在则返回，不存在则继续

2. 再选择复制偏移量最大的slave节点，如果存在则返回，不存在则继续

3. 最后没办法才选择runId最小的slave节点



###节点运维：

1. 手动下线master：

客户端发送命令`sentinel failover <masterName>`

2. 下线从节点：slaveof即可，sentinel会自动感知

3. sentinel节点启动上线，其他sentinel也会自动感知到

### 总结：

- redis sentinel是Redis的高可用实现方案：有故障发现、故障自动转移、配置中心、客户端通知等功能。

- sentinel个数最好为奇数个且大于3个。
- 客户端初始化时连接的是sentinel节点集合，不再是具体的Redis节点，但Sentinel只是配置中心不是代理
- Redis Sentinel通过三个定时任务实现了Sentinel节点对于主节点、从节点、其余Sentinel节点的监控
- Redis Sentinel在对节点做失败判断时分为主观下线和客观下线
- 看懂Redis Sentinel故障转移日志对于问题排查非常有帮助
- Redis Sentinel实现读写分离高可用可用依赖Sentinel节点的消息通知，获取Redis数据节点的状态变化



## Redis Cluster

 redis提供的一个分布式集群

### 数据分布概论

顺序分布：例如1-100，1-33分在一个区，34-66分在一个区，67-100分到一个区

哈希分布：例如节点取模 ，hash（key）%3

对比

| 分布方式 | 特点                                                         | 产品                                |
| -------- | ------------------------------------------------------------ | ----------------------------------- |
| 哈希分布 | 数据分散度高；键值分布与业务无关；无法顺序访问；支持批量操作 | 一致性哈希Mencache、Redis Cluster等 |
| 顺序分布 | 数据分散度低；键值与业务相关；可顺序访问；不支持批量操作     | BigTable、HBase                     |

### 哈希分布

#### 节点取余

hash(key)%nodes

节点伸缩：数据节点关系变化，导致数据迁移

迁移数量和添加节点数量有关：建议翻倍扩容

#### 一致性哈希

对于任何的哈希函数，都有其取值范围。我们可以用环形结构来标识范围。通过哈希函数，每个节点都会被分配到环上的一个位置，每个键值也会被映射到环上的一个位置，然后顺时针找到相邻的节点。如下图所示，例如key分布在range1内，那么数据存储在node2上。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C.jpg)

- 客户端分片：哈希+顺时针（优化取余）
- 节点伸缩：只影响邻近节点，但是还是有数据迁移
- 翻倍伸缩：保证最小迁移数据和负载均衡

#### 虚拟槽分区

在redis cluster中使用槽来存储一定范围内的数据集。每个redis节点上有一定数量的槽。当客户端提交数据时，要先根据**CRC16(key)&16383**来计算出数据要落到哪个虚拟槽内。

- 预设虚拟槽：每个槽映射一个数据子集，一般比节点数大
- 良好的哈希函数：例如CRC16
- 服务端管理节点、槽、数据：例如Redis Cluster

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E8%99%9A%E6%8B%9F%E6%A7%BD.jpg)

与节点取余和一致性哈希分区不同，虚拟槽分区是服务端分区。客户端可以将数据提交到任意一个redis cluster节点上，如果存储该数据的槽不在这个节点上，则返回给客户端目标节点信息，告知客户端向目标节点提交数据。

### 基本架构

Redis-Cluster采用无中心结构，每个节点保存数据和整个集群状态,每个节点都和其他所有 节点连接。其redis-cluster架构图如下：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/redis%20cluster%E7%BB%93%E6%9E%84.jpg)

其结构特点：

​     1、所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。
​     2、节点的fail是通过集群中超过半数的节点检测失效时才生效。
​     3、客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可。
​     4、redis-cluster把所有的物理节点映射到[0-16383]slot上（不一定是平均分配）,cluster 负责维护node<->slot<->value。

​     5、Redis集群预分好**16384**个桶，当需要在 Redis 集群中放置一个 key-value 时，根据 CRC16(key) mod 16384的值，决定将一个key放到哪个桶中。

### 安装配置

原生命令安装

1. 配置开启redis cluster

```redis
cluster-enabled yes //开启cluster
cluster-config-file nodes-${port}.conf  //cluster配置文件位置
```

2. 启动各个redis

3. meet：cluster meet ip port

4. 其他主要配置：

```redis
cluster-node-timeout 15000
cluster-require-full-coverage yes //如果有一个节点失效了，整个cluster都失效
```

5. 分配槽：

cluster addslots slot [slot...]

eg: redis-cli 127.0.0.1 -p 7000 cluster addslots {0...5461}

6. 设置主从

cluster replicate node-id

eg: redis-cli - 127.0.0.1 -p 7003 cluster replicate ${node-id-7000}

7.  cluster nodes  ：查看节点情况
8. cluster info：查看集群信息

使用ruby工具安装：

1. 下载、编译、安装Ruby：

   wget https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.1.tar.gz

   tar -xvf ruby-2.3.1.tar.gz 

   cd ruby-2.3.1

   ./configure -prefix=/usr/local/ruby

   make

   make install

   cd /usr/local/ruby

   cp bin/ruby /usr/local/bin

2. 安装rubygem redis

   wget http://rubygems.orgs/downloads/redis-3.3.0.gem

   gem install -l redis-3.3.0.gem

   gem list --check redis gem

3. 安装redis-trib.rb

   cp ${REDIS_HOME}/src/redis-trib.rb /usr/local/bin

4. 使用redis-trib.rb创建集群:
   ./redis-trib.rb create --replicas 1 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 127.0.0.1:7006  

   使用create命令 --replicas 1 参数表示为每个主节点创建一个从节点，其他参数是实例的地址集合。

   5.redis-trib.rb其他命令：

（1）检查命令：./redis-trib.rb check 127.0.0.1:7002

（2）连接命令：./redis-cli -c -p 7001    

（3）添加集群节点：./redis-trib.rb add-node 127.0.0.1:7007 127.0.0.1:7002  

（ add-node是加入集群节点，127.0.0.1:7007为要加入的节点，127.0.0.1:7002 表示加入的集群的一个节点，用来辨识是哪个集群，理论上那个集群的节点都可以。）

（4）在新增节点后要重新分配卡槽 ： redis-trib.rb reshard 127.0.0.1:7005

（这个命令是用来迁移slot节点的，后面的127.0.0.1:7005是表示是哪个集群，端口填［7000-7007］都可以）

（5）添加从节点：./redis-trib.rb add-node --slave --master-id $[nodeid] 127.0.0.1:7008 127.0.0.1:7000  

 （nodeid为要加到master主节点的node id，127.0.0.1:7008为新增的从节点，127.0.0.1:7000为集群的一个节点（集群的任意节点都行），用来辨识是哪个集群；如果没有给定那个主节点--master-id的话，redis-trib将会将新增的从节点随机到从节点较少的主节点上。）

参考：[Redis Cluster集群](https://www.cnblogs.com/lykxqhh/p/5690923.html)

###客户端重定向

cluster根据槽的位置重定向客户端到某个节点

####moved重定向

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/moved%E9%87%8D%E5%AE%9A%E5%90%91.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/moved%E5%BC%82%E5%B8%B8.jpg)

#### ask重定向

如果客户端发送命令的时候，数据刚好在迁移，就需要ask重定向来解决

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/ask%E9%87%8D%E5%AE%9A%E5%90%91.jpg)

moved和ask

- 两者都是客户单重定向
- moved：槽已经确定迁移
- ask：槽正在迁移

#### smart客户端

追求性能

原理：

1. 从集群中选一个可运行节点，使用cluster slots初始化槽和节点映射
2. 将cluster slots的结果映射到本地，为每个节点创建JedisPool
3. 准备执行命令：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/smart%E5%8E%9F%E7%90%86%E6%89%A7%E8%A1%8C%E5%91%BD%E4%BB%A4.jpg)

#### JedisCluster(smart客户端)

基本使用：

```java
Set<HostAndPort> nodeList = new HashSet<HostAndPort>();
nodeList.add(new HostAndPort(host1,port1));
nodeList.add(new HostAndPort(host2,port2));
nodeList.add(new HostAndPort(host3,port3));
nodeList.add(new HostAndPort(host4,port4));
JedisCluster redisCluster = new JedisCluster(nodeList,timeOut,poolConfig);
redisCluster.set(key,value);
redisCluster.get(key);
...redisCluster.command...
```

使用技巧：

1. 单列：内置了所有节点的连接池
2. 无需手动借还连接池
3. 合理设置commons-pool

### 批量操作

cluster苛刻的要求：mget和mset操作的数据必须在同一个节点上

四种批量优化的方法：

1. 串行mget：将mget拆分成多个get，n次网络时间
2. 串行IO：将mget中的数据，在客户端计算槽，按照在不同节点拆分成多个mget，nodes次网络时间
3. 并行IO：在串行IO的基础上改成多个进程来发起这个多mget命令，nodes/线程数网络时间
4. hash_tag：将mget的key加上{tag}，使他们在同一个node上

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%9B%9B%E7%A7%8D%E6%96%B9%E6%A1%88%E4%BC%98%E7%BC%BA%E7%82%B9%E5%88%86%E6%9E%90.jpg)

### 故障转移

#### 故障发现

- 通过ping/pong信息实现故障发现

- 主观下线和客观下线：

  （1）主观下线：某个节点认为另一个节点不可用，”偏见“。流程如下：

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%B8%BB%E8%A7%82%E4%B8%8B%E7%BA%BF.jpg)

  （2）客观下线：当半数以上持有槽的主节点都标记某节点主观下线。流程如下：

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%AE%A2%E8%A7%82%E4%B8%8B%E7%BA%BF.jpg)

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%B0%9D%E8%AF%95%E5%AE%A2%E8%A7%82%E4%B8%8B%E7%BA%BF.jpg)

####故障恢复

发现故障后，进行故障恢复

- 资格检查：检查每个从节点与故障主节点的断线时间，如果超过cluster-node-timeout*cluster-slave-validity-factor则取消资格，cluster-slave-validity-factor默认是10

- 准备选举时间:偏移量大的先选举

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%87%86%E5%A4%87%E9%80%89%E4%B8%BE%E6%97%B6%E9%97%B4.jpg)

- 选举投票:

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E9%80%89%E4%B8%BE%E6%8A%95%E7%A5%A8.jpg)

- 替换主节点:

  (1)当前从节点取消复制变为主节点（slaveof  no one）

  (2)执行clusterDelSlot撤销故障主节点负责的槽，并执行clusterAddSlot把这些槽分配给自己

  (3)向集群广播自己的pong信息，表明已经替换了故障从节点

###Redis Cluster开发运维常见问题

- 集群完整性：

  clusetr-require-full-coverage默认为yes，有一个节点down了或正在进行数据转移：

  (error)CLUSTERDOWN The cluster is down。大多数业务无法容忍，建议设为yes

- 带宽消耗：

  官方建议1000个节点，Ping/Pong带宽消耗不容忽视

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%B8%A6%E5%AE%BD%E6%B6%88%E8%80%973%E4%B8%AA%E6%96%B9%E9%9D%A2.jpg)

- Pub/Sub广播:
  问题：publish在集群每个节点广播：加重带宽

  解决：单独走一套redis sentinel

- 数据倾斜：内存不均

  （1）节点和槽分配不均：redis-trib.rb info ip:port来查看节点、槽、键值分布； redis-trib.rb rebalance ip:port来进行均衡（谨慎使用）

  （2）不同槽对应键值数量差异大：可能存在hash_tag;cluster countkeysinslot {slot}获取槽对应键值个数

  （3）包含bigkey：bigkey内存大的键值；redis-cli --bigkeys（建议在从节点查看）；优化数据结构

  （4）内存相关配置不一致：hash-max-ziplist-value、set-max-intset-entries等；定期检查配置一致性

- 请求倾斜：热点key大量请求

  优化：避免bigkey；热键不用hash_tag；当一致性不高时，可以使用本地缓存+MQ

- 读写分离：不建议使用cluster去做读写分离

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E8%AF%BB%E5%86%99%E5%88%86%E7%A6%BB.jpg)

- 数据迁移

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E6%95%B0%E6%8D%AE%E8%BF%81%E7%A7%BB.jpg)

- 集群vs单机

  (1)集群限制：

  ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E9%9B%86%E7%BE%A4%E9%99%90%E5%88%B6.jpg)

  分布式redis不一定好，很多业务不需要，大多数客户端性能会降低，维护更复杂

  很多场景redis sentinel已经够用

### 集群总结

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E9%9B%86%E7%BE%A4%E6%80%BB%E7%BB%931.jpg)

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E9%9B%86%E7%BE%A4%E6%80%BB%E7%BB%932.jpg)

## 缓存

好处：快、降低后端存储负载

缺点：

-  数据不一致
- 代码维护成本
- 运维成本

### 缓存更新策略

1.LRU/LFU/FIFIO算法剔除：例如maxmemory-policy

2.超时剔除:例如expire

3.主动更新：开发控制生命周期

| 策略                  | 一致性 | 维护成本 |
| --------------------- | ------ | -------- |
| LRU/LFU/FIFIO算法剔除 | 最差   | 低       |
| 超时剔除              | 差     | 低       |
| 主动更新              | 高     | 高       |

两条建议：
1.低一致性：最大内存和淘汰策略

2.高一致性：超时提出和主动更新结合，最大内存和淘汰策略兜底

### 缓存粒度问题

参考：[缓存在高并发场景下的常见问题](https://www.cnblogs.com/wajika/p/6683372.html)

三个角度来选择缓存粒度

1. 通用性：全量属性更好
2. 占用空间：部分属性更好
3. 代码维护：表面上全量属性更好

### 缓存穿透问题-大量请求不命中

**缓存穿透是指查询一个一定不存在的数据，由于缓存不命中时需要从数据库查询，查不到数据则不写入缓存，这将导致这个不存在的数据每次请求都要到数据库去查询，造成缓存穿透。**

原因：

1. 业务代码自身问题
2. 恶意攻击、爬虫等等

如何发现：

1. 业务的相应时间
2. 业务本身问题
3. 相关指标：总调用数、缓存层命中数、存储层命中数

**解决方法：**

1. 缓存空对象：

   对查询结果为空的对象也进行缓存，如果是集合，可以缓存一个空的集合（非null），如果是缓存单个对象，可以通过字段标识来区分。这样避免请求穿透到后端数据库。同时，也需要保证缓存数据的时效性。这种方式实现起来成本较低，比较适合命中不高，但可能被频繁更新的数据。

   ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E7%BC%93%E5%AD%98%E7%A9%BA%E5%AF%B9%E8%B1%A1.jpg)

   两个问题：

   - 需要更多的键，所以需要设置过期时间
   - 缓存层和存储层数据“短期”不一致

2. 布隆过滤器拦截

   对所有可能对应数据为空的key进行统一的存放，并在请求前做拦截，这样避免请求穿透到后端数据库。这种方式实现起来相对复杂，比较适合命中不高，但是更新不频繁的数据。

   ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E5%B8%83%E9%9A%86%E8%BF%87%E6%BB%A4%E5%99%A8.jpg)

### 缓存雪崩

缓存雪崩就是指由于缓存的原因，导致大量请求到达后端数据库，从而导致数据库崩溃，整个系统崩溃，发生灾难。导致这种现象的原因有很多种，上面提到的“缓存并发”，“缓存穿透”，“缓存颠簸”等问题，其实都可能会导致缓存雪崩现象发生。这些问题也可能会被恶意攻击者所利用。还有一种情况，例如某个时间点内，系统预加载的缓存周期性集中失效了，也可能会导致雪崩。为了避免这种周期性失效，可以通过设置不同的过期时间，来错开缓存过期，从而避免缓存集中失效。

从应用架构角度，我们可以通过限流、降级、熔断等手段来降低影响，也可以通过多级缓存来避免这种灾难。

此外，从整个研发体系流程的角度，应该加强压力测试，尽量模拟真实场景，尽早的暴露问题从而防范。

### 无底洞

该问题由 facebook 的工作人员提出的， facebook 在 2010 年左右，memcached 节点就已经达3000 个，缓存数千 G 内容。

他们发现了一个问题---memcached 连接频率，效率下降了，于是加 memcached 节点，

添加了后，发现因为连接频率导致的问题，仍然存在，并没有好转，称之为”无底洞现象”。

问题关键点：

- 更多机器！=更高性能 ，=多次网络时间
- 批量接口需求（mget,mset）等
- 数据增长与水平扩展需求

解决方法主要是优化IO：

- 命令本身优化：例如优化慢查询keys、hgetall、bigkey，串行mget、串行IO、并行IO、hash_tag
- 减少网络通信次数
- 降低接入成本：例如客户端长连接/连接池、NIO等



### 热点key的重建优化

问题：热点key重建的时候因为请求多次而重建了多次

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E7%83%AD%E7%82%B9key%E9%87%8D%E5%BB%BA.jpg)

三个目标和两个解决方法：

1. 三个目标：
   - 减少重缓存的次数
   - 数据尽可能一致
   - 减少潜在危险
2. 两个解决方法
   - 互斥锁（mutex key）
   - 永不过期

#### 互斥锁

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%BA%92%E6%96%A5%E9%94%81.jpg)

示例代码：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E4%BA%92%E6%96%A5%E9%94%81%E5%AE%9E%E4%BE%8B%E4%BB%A3%E7%A0%81.jpg)

#### 永不过期

1. 真的永不过期

2. 在value中设置逻辑过期时间，如果取出来发现已经过期了，再用互斥锁去执行一个线程去重建：
   ![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E6%B0%B8%E4%B8%8D%E8%BF%87%E6%9C%9F.jpg)

示例代码：
![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/redis/%E6%B0%B8%E4%B8%8D%E8%BF%87%E6%9C%9F%E5%AE%9E%E4%BE%8B%E4%BB%A3%E7%A0%81.jpg)

两种方案对比

| 方案     | 优点                    | 缺点                                             |
| -------- | ----------------------- | ------------------------------------------------ |
| 互斥锁   | 思路简单；保证一致性    | 代码复杂度增加；存在死锁风险                     |
| 永不过期 | 基本杜绝热点key重建问题 | 不保证一致性；逻辑过期时间增加维护成本和内存成本 |





# Redis的内存上限和内存回收策略

###内存上限
Redis可以通过` maxmemory `参数来限制最大可用内存，主要为了避免Redis内存超过操作系统内存，从而导致服务器响应变慢甚至死机的情况。

maxmemory 参数限制的是Redis的对象内存大小，也就是 used_memory 对应的内存大小。由于内存碎片的存在，所以Redis服务器实际占用的内存是要超过 maxmemory 的。

所以我们在设置Redis内存上限的时候要预留一部分内存出来，比如说一台32GB内存的机器，可以启动 3 台8GB内存的Redis，预留8GB给机器其他进程、内存碎片、fork子进程等。

可以通过 `config set maxmemory `命令来动态修改Redis内存上限：

`config set maxmemory 2GB`

###内存回收策略
Redis的内存回收策略主要体现在两个方面： 

- **删除到达过期时间的键对象** 
- **内存达到 maxmemory 后的淘汰机制**

###删除过期键对象
由于Redis进程内保存了大量的键，维护每个键的过期时间去删除键会消耗大量的CPU资源，对于单线程的Redis来说成本很高。所以**Redis采用惰性删除 + 定时任务删除机制来实现过期键的内存回收。**

惰性删除：当客户端读取键时，如果键带有过期时间并且已经过期，那么会执行删除操作并且查询命令返回空。这种机制是为了节约CPU成本，不需要单独维护一个TTL链表来处理过期的键。但是这种删除机制会导致内存不能及时得到释放，所以将结合下面的定时任务删除机制一起使用。
定时任务删除：Redis内部维护一个定时任务，用于随机获取一些带有过期属性的键，并将其中过期的键删除。来删除一些过期的冷数据。
在兼顾CPU和内存的的考虑下，Redis使用惰性删除 + 定时任务删除机制相结合，来删除过期键对象。

###淘汰机制
当Redis所使用的内存达到 maxmemory 之后会触发相应的溢出控制策略，Redis支持 6 种策略： 

- noeviction：当内存使用达到阈值的时候，所有引起申请内存的命令会报错。 
- allkeys-lru：在所有键中采用lru算法删除键，直到腾出足够内存为止。 
- volatile-lru：在设置了过期时间的键中采用lru算法删除键，直到腾出足够内存为止。 
- allkeys-random：在所有键中采用随机删除键，直到腾出足够内存为止。 
- volatile-random：在设置了过期时间的键中随机删除键，直到腾出足够内存为止。 
- volatile-ttl：在设置了过期时间的键空间中，具有更早过期时间的key优先移除。

lru是Least Recently Used的缩写，即最近最少使用。

内存的溢出控制策略可以采用 config set maxmemory-policy {policy} 命令来动态配置：

`config set maxmemory-policy volatile-lru`

频繁执行回收内存成本很高，每次都要去查找可回收键和删除键，所以合理设置Redis的 maxmenory 很重要，不合理的Redis溢出控制策略可能会导致一些不可预知的问题。





**Redis怎么限制ip访问？**

**开启redis 允许外网IP 访问**

在 Linux 中安装了redis 服务，当在客户端通过远程连接的方式连接时，报could not connect错误。

错误的原因很简单，就是没有连接上redis服务，由于redis采用的安全策略，默认会只准许本地访问。

需要通过简单配置，完成允许外网访问。

修改redis的配置文件，将所有bind信息全部屏蔽。

```
`# bind 192.168.1.100 10.0.0.1` `# bind 192.168.1.8` `# bind 127.0.0.1`
```

　　

修改完成后，需要重新启动redis服务。

```
`redis-server redis.conf`
```

　　

如果iptables 没有开启6379端口，用这个方法开启端口

```
`命令：/sbin/iptables -I INPUT -p tcp --dport 6379 -j ACCEPT` `保存防火墙修改命令：/etc/rc.d/init.d/iptables save`
```

　 

通过iptables 允许指定的外网ip访问

修改 [Linux](http://www.2cto.com/os/linux/) 的防火墙(iptables)，开启你的redis服务端口，默认是6379。

```
`//只允许127.0.0.1访问6379``iptables -A INPUT -s 127.0.0.1 -p tcp --dport 6379 -j ACCEPT``//其他ip访问全部拒绝``iptables -A INPUT -p TCP --dport 6379 -j REJECT`
```

