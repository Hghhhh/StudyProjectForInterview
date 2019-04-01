**KEYS pattern**

查找所有符合给定模式 `pattern` 的 `key` 。时间复杂度O(n)。

**SMEMBERS key**

返回集合 `key` 中的所有成员，不存在的 `key` 被视为空集合。时间复杂度O(n)。

**hgetall key**

返回hash key对应所以的field和value。时间复杂度O(n)。

**lrang key start end（包含end）**

获取列表指定索引范围的所有item。时间复杂度O(n)。

**zrange key start end [withscores]**

返回指定索引范围内的升序元素，  时间复杂度O(log(N)+m),`N` 为有序集的基数，而 `M` 为结果集的基数



因为Redis是单线程的，但在一个大的数据库中使用它仍然可能造成性能问题使用Keys、Smembers这样的命令，可能会阻塞Redis长达数秒之久。



替代这些命令的命令是**SCAN**

这是一个增量迭代的命令，一次只返回Count条命令，然后再根据游标去迭代后面的命令。

- `SCAN cursor [MATCH pattern][COUNT count]`

  [SCAN](http://doc.redisfans.com/key/scan.html#scan)命令用于迭代当前数据库中的数据库键。

  cursor是游标的位置，MATCH是返回匹配的模式，COUNT是一次迭代的数量

  ```redis
  eg：
  SCAN 0 MATCH *f* COUNT 100
  #表示返回从下标0开始，查找100个key，返回满足pattern的。
  1) "0"
  2)  1) "key:611"
      2) "key:711"
      3) "key:118"
      4) "key:117"
      5) "key:311"
      6) "key:112"
      7) "key:111"
      8) "key:110"
      9) "key:113"
     10) "key:211"
     11) "key:411"
     12) "key:115"
     13) "key:116"
     14) "key:114"
     15) "key:119"
     16) "key:811"
     17) "key:511"
     18) "key:11"
     
  SCAN 0
  #如果不指定COUNT的话，默认是10个左右
  1) "17"
  2)  1) "key:12"
      2) "key:8"
      3) "key:4"
      4) "key:14"
      5) "key:16"
      6) "key:17"
      7) "key:15"
      8) "key:10"
      9) "key:3"
      10) "key:7"
      11) "key:1"
  
  
  ```

- `SSCAN key cursor [MATCH pattern][COUNT count]`

  [*SSCAN*](http://doc.redisfans.com/set/sscan.html#sscan) 命令用于迭代集合键中的元素。

  SSCAN命令和下面的HSCAN,ZSACN都是在SCAN命令中加入了key而已

- [*HSCAN*](http://doc.redisfans.com/hash/hscan.html#hscan) 命令用于迭代哈希键中的键值对。

- [*ZSCAN*](http://doc.redisfans.com/sorted_set/zscan.html#zscan) 命令用于迭代有序集合中的元素（包括元素成员和元素分值）。



不过， 增量式迭代命令也不是没有缺点的： 举个例子， 使用 [*SMEMBERS*](http://doc.redisfans.com/set/smembers.html#smembers) 命令可以返回集合键当前包含的所有元素， 但是对于 [*SCAN*](http://doc.redisfans.com/key/scan.html#scan)这类增量式迭代命令来说， 因为在对键进行增量式迭代的过程中， 键可能会被修改， 所以增量式迭代命令只能对被返回的元素提供有限的保证 （offer limited guarantees about the returned elements）。