**内嵌Lua的执行**

万幸 Redis 内嵌了 Lua 执行环境，支持 Lua 脚本的执行，通过执行 Lua 脚本，我们可以把多个命令复合为一个 Lua 脚本，通过 Lua 脚本来实现上文中提到的 Redis 命令的次序性和 Redis 服务端计算。

**Lua**

Lua 是一个简洁、轻量、可扩展的脚本语言，它的特性有：

- 轻量：源码包只有核心库，编译后体积很小。
- 高效：由 ANSI C 写的，启动快、运行快。
- 内嵌：可内嵌到各种编程语言或系统中运行，提升静态语言的灵活性。如 OpenResty 就是将 Lua 嵌入到 nginx 中执行。

而且完全不需要担心语法问题，Lua 的语法很简单，分分钟使用不成问题。

**执行步骤**

Redis 在 2.6 版本后，启动时会创建 Lua 环境、载入 Lua 库、定义 Redis 全局表格、存储 redis.pcall 等 Redis 命令，以准备 Lua 脚本的执行。

一个典型的 Lua 脚本执行步骤如下：

1. 检查脚本是否执行过，没执行过使用脚本的 sha1 校验和生成一个 Lua 函数；
2. 为函数绑定超时、错误处理勾子；
3. 创建一个伪客户端，通过这个伪客户端执行 Lua 中的 Redis 命令；
4. 处理伪客户端的返回值，最终返回给客户端；

虽然 Lua 脚本使用的是伪客户端，但 Redis 处理它会跟普通客户端一样，也会将执行的 Redis 命令进行 rdb aof 主从复制等操作。

**使用**

Lua 脚本的使用可以通过 Redis 的 EVAL 和 EVALSHA 命令。

EVAL 适用于单次执行 Lua 脚本，执行脚本前会由脚本内容生成 sha1 校验和，在函数表内查询函数是否已定义，如未定义执行成功后 Redis 会在全局表里缓存这个脚本的校验和为函数名，后续再次执行此命令就不会再创建新的函数了。

而要使用 EVALSHA 命令，就得先使用 SCRIPT LOAD 命令先将函数加载到 Redis，Redis 会返回此函数的 sha1 校验和， 后续就可以直接使用这个校验和来执行命令了。

以下是使用上述命令的例子：

```redis
127.0.0.1:6379> EVAL "return 'hello'" 0 0
"hello"
 
127.0.0.1:6379> SCRIPT LOAD "return redis.pcall('GET', ARGV[1])"
"20b602dcc1bb4ba8fca6b74ab364c05c58161a0a"
 
127.0.0.1:6379> EVALSHA 20b602dcc1bb4ba8fca6b74ab364c05c58161a0a 0 test
"zbs"
```

EVAL 命令的原型是 `EVAL script numkeys key [key ...] arg [arg ...] `，在 Lua 函数内部可以使用 `KEYS[N]`和 `ARGV[N] `引用键和参数，需要注意 KEYS 和 ARGV 的参数序号都是从 1 开始的。

还需要注意在 Lua 脚本中，Redis 返回为空时，结果是 false，而 不是 nil；

**Lua 脚本实例**

下面写几个 Lua 脚本的实例，用来介绍语法的，仅供参考。

Redis 里 hashSet A 的 字段 B 的值是 C，取出 Redis 里键为 C 的值。

```
`// 使用: EVAL script 2 A B` `local tmpKey = redis.call(``'HGET'``, KEYS[1], KEYS[2]); ``return` `redis.call(``'GET'``, tmpKey);`
```

一次 lpop 出多个值，直到值为 n，或 list 为空（pipeline 也可轻易实现）；

```
`// 使用: EVAL script 2 list count``local list = {};``local item = ``false``;``local num = tonumber(KEYS[2]);``while` `(num > 0)``do``  ``item = redis.call(``'LPOP'``, KEYS[1]);``  ``if` `item == ``false` `then``    ``break``;``  ``end;``  ``table.insert(list, item);``  ``num = num - 1;``end;``return` `list;`
```

获取 zset 内 score 最多的 n 个元素 对应 hashset 中的详细信息；

```
`local elements = redis.call(``'ZRANK'``, KEYS[1], 0, KEY[2]);``local detail = {};``for` `index,ele in elements ``do``  ``local info = redis.call(``'HGETALL'``, ele);``  ``table.insert(detail, info);``end;``return` `detail;`
```

基本使用语法就是如此，更多应用就看各个具体场景了。、



**错误处理**

前面的命令介绍部分说过， `redis.call()` 和 `redis.pcall()` 的唯一区别在于它们对错误处理的不同。

当 `redis.call()` 在执行命令的过程中发生错误时，脚本会停止执行，并返回一个脚本错误，错误的输出信息会说明错误造成的原因：

```redis
redis> lpush foo a
(integer) 1

redis> eval "return redis.call('get', 'foo')" 0
(error) ERR Error running script (call to f_282297a0228f48cd3fc6a55de6316f31422f5d17): ERR Operation against a key holding the wrong kind of value
```

和 `redis.call()` 不同， `redis.pcall()` 出错时并不引发(raise)错误，而是返回一个带 `err` 域的 Lua 表(table)，用于表示错误：

```redis
redis 127.0.0.1:6379> EVAL "return redis.pcall('get', 'foo')" 0
(error) ERR Operation against a key holding the wrong kind of value
```



**小结：**

- 初始化 Lua 脚本环境需要一系列步骤，其中最重要的包括：
  - 创建 Lua 环境。
  - 载入 Lua 库，比如字符串库、数学库、表格库，等等。
  - 创建 `redis` 全局表格，包含各种对 Redis 进行操作的函数，比如 `redis.call` 和 `redis.log` ，等等。
  - 创建一个无网络连接的伪客户端，专门用于执行 Lua 脚本中的 Redis 命令。
- Reids 通过一系列措施保证被执行的 Lua 脚本无副作用，也没有有害的写随机性：对于同样的输入参数和数据集，总是产生相同的写入命令。
- [EVAL](http://redis.readthedocs.org/en/latest/script/eval.html#eval) 命令为输入脚本定义一个 Lua 函数，然后通过执行这个函数来执行脚本。
- [EVALSHA](http://redis.readthedocs.org/en/latest/script/evalsha.html#evalsha) 通过构建函数名，直接调用 Lua 中已定义的函数，从而执行相应的脚本。

参考：

http://redisdoc.com/script/eval.html

https://www.jb51.net/article/136116.htm