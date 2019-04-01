Explain命令是查看查询优化器如何决定执行查询的主要方法。学会解释EXPLAIN将帮助你了解Mysql优化器是如何工作的。

## 调用EXPLAIN

要使用EXPLAIN只需要在查询中的SELECT关键字之前加上EXPLAIN这个词。

一个简单的EXPLAIN结果：

![EXPLAIN](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/EXPLAIN1.png)

要意识到EXPLAIN只是个近似结果，而且有时候可能会与真相相差甚远，下面是一些相关的限制：

- EXPLAIN不会告诉你触发器、存储过程或UDF会如何影响查询
- 不支持存储过程，尽管可以手动抽取查询并单独地对其进行EXPLAIN操作
- 并不会显示关于查询的执行计划的所有信息
- 并不区分具有相同名字的事务。例如它对内存排序和临时文件都使用“filesort”，并且对于磁盘上和内存中的临时表都显示“Using temporary”
- 可能会误导。例如，它会对一个有着很小LIMIT的查询显示全索引扫描。（Mysql5.1的EXPLAIN关于检查的行数会显示更精确的信息，但早期版本并不考虑）

*注意：Mysql会在查询上设置一个标记。当执行查询时，这个标记会使其返回关于在执行计划中的每一步信息，而不是执行它。它会返回一行或多行信息，显示出执行计划中的每一部分和执行的次序。认为增加EXPLAIN时MYSQL不会执行查询，这是一个常见的错误。事实上，如果查询在FROM子句中包括子查询，那么MySQL实际上会执行子查询，将其结果放在一个临时表中，然后完成外层查询优化。它必须在可以完成外层查询优化之前处理所有类似的子查询，这对于EXPLAIN来说是必须要做的。这意味着如果语句包含开销较大的子查询或使用临时表算法的视图，实际上会给服务器带来大量工作。*

##重写非SELECT查询

Mysql5.5及以下EXPLAIN只支持SELECT查询，不支持INSERT、UPDATE、DELETE或其它语句做解释。Mysql5.6已经允许解释非SELECT查询，故这部分省略。

## EXPLAIN中的列

### id列

标记SELECT所属的层，对应于该行在SELECT语句中的位置。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/EXPLAIN2.png)

例如上面的语句，id为2的层指的是`select 1 from tb_vip`

### select_type列

这一列显示了对应列是简单还是复杂SELECT。SIMPLE意味值不包括子查询和UNION。如果查询有任何复杂的子部分，则最外层部分标记为PRIMARY，其他部分标记如下：

**SUBQUERY**：包含在SELECT列表中的子查询中SELECT标记为SUBQUERY，如上面的例子。

**DERIVED**：表示包含在FROM子句中的子查询，Mysql会递归执行并将其结果放到一个临时表中。

**UNION**：在UNION中的第二个和随后的SELECT被标记为UNION。第一个SELECT的标记不一定是PRIMARY，例如如果这个UNION语句被FROM子句中的子查询包含，那么它的第一个SELECT会被标记为DEIVED。

**UNION RESULT**：用来从UNION的匿名临时表检索结果的SELECT被标记为UNION RESULT。 

除了这些值，SUBQUERY和UNION还可以被标记为DEPENDENT和UNCACHFABIE。DEPENDENT意味着SELECT依赖于外层查询中发现的数据；UNCACHEABLE意味着SELECT中的某些特性阻止结果被缓存于一个Itemcache中。（Itemcache未被文档记载；它与查询缓存不是一回事，尽管它可以被一些相同类型的构件否定，例如RAND()函数。）

MYSQL5.6支持UPDATE、INSERT、DELETE的select_type发别也是UPDATE、INSERT、DELETE



## table列

这一列显示了对应行正在访问的表。通常情况下，它相当明了，就是访问的那个表的名字或别名。

当在FORM子句中有子查询时，table列是`<derivedN>`的形式,其中N是子查询的id。

当有UNION时，UNION RESULT的table列包含一个参与UNION的id列表。

**一个复杂的SELECT类型的例子：**

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/EXPLAIN3.png)

执行结果：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/EXPLAIN4.png)

### type列

这一列显示了访问类型——就是Mysql决定如何查找表中行。下面是最重要的访问方法，依次从最差到最优：

**ALL**：全表扫描，意味着mysql扫描整张表，去找到需要的行。（这里也有个例外，例如在查询里使用了LIMIT，或者在Extra列中显示“Using distinct/not exists”）

**index**：这个跟全表扫描一样，只是Mysql扫描表的时候按索引次序进行而不是行。它的主要优点是避免了排序；最大的缺点是要承担按索引次序读取整个表的开销。这通常意味着若是按照随机次序访问行，开销将会非常大。

如果在Extra列中看到“Using index”，说明Mysql正在使用覆盖索引，它只扫描索引中的数据，而不是按索引次序的每一行。它会比按索引次序全表扫描的开销要少很多。

**range**：范围扫描就是一个有限制的索引扫描，它开始于索引里的某一点，返回匹配这个值域的某一行，这比全索引扫描要好一些。显而易见的范围扫描是带有BETWEEN或在WHERE子句里带有'>'的查询。当mysql使用索引去查找一系列值时，例如IN()和OR列表，也会显示为范围扫描。

![range](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/range.png)

**ref**：这是一种索引访问，它返回所有匹配某个单个值的行。把它叫ref是因为索引要跟某个参考值相比较，这个参考值或者是一个常数，或者是来自多表查询前一个表的结果值。此类索引访问只有当使用非唯一性索引或唯一性索引的非唯一性前缀是才会发生。

**eq_ref**：使用这种索引查找，mysql知道最多只返回一条符号条件的记录。这种访问方法可以在mysql使用主键或唯一性索引查找的时候看到。

**const,system**：当mysql能对查询某部分进行优化并将其转化为一个常量是，就会使用这些访问类型。

**NULL**：这种访问方式意味着Mysql能在优化阶段分解查询语句，在执行阶段设置不用访问表或索引。

### possible_keys列

这一列显示了查询可能使用哪些索引，这是基于查询访问的列和使用的比较操作符来判断的，这个列表是在优化过程的早期创建的，有些可能后续优化过程是没有用的。

### key列

这一列显示了Mysql决定采用哪个索引来优化对该表的访问。该key可能没有出现在possible_keys列表中，例如，虽然没有没有where子句，但mysql选择了一个覆盖索引来查询结果。

###key_len列

该列显示了Mysql在索引里使用的字节数。

### ref列

这一列显示了之前的表在key列记录的索引中查找值所用的列或常量。

eg:

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/ref.png)

### row列

这一列是Mysql为了找到所需的行而需要读取的平均行数。

### filtered列

这一列是在Mysql5.1时加上去的，显示的是针对表里符合某个条件的记录数的百分比所做的一个悲观估计。如果你把rows列和这个百分比相乘，就能看到Mysql估算它将和查询计划里前一个表关联的行数。

###Extra列

这一列包含的是不适合在其他列显示的额外信息。

最常见的值如下：

**Using index**：表示Mysql将使用覆盖索引。

**Using where**：这意味值Mysql服务器将在存储引擎检索行后再进行过滤。许多Where条件里面涉及索引中的列，会被存储引擎检验，因此不少所有待Where子句的查询都会显示”Using where“

**Using temporary**：意味着Mysql在对查询结果排序时会使用一个临时表

**Using filesort**：意味着Mysql会对结果使用一个外部索引排序，而不是按索引次序从表中读取行。Mysql有两种文件排序算法，两种方式都可以在内存或磁盘上完成。EXPLAIN不会告诉你使用的是哪种，也不会告诉你是内存还是磁盘上完成排序。

**Range checked for each record（index map：N）**：这个值意味着没有好用的索引，新的索引将在联接的每一行上重新估算。N是显示在possible_keys列中的索引的位图，并且是冗余的。