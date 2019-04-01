优化的理由：

- SQL语句是对数据库进行操作的唯一途径

- SQL语句消耗了70%-90%的数据库资源
- SQL语句独立于程序设计逻辑，相对于对程序源代码的优化，对SQL语句的优化在时间成本和风险上的代价都很低。
- SQL语句可以有不同的写法。

##1.not in 子查询优化

案例一：子查询性能没有join性能好，应尽可能将in、exists子查询改为join

```sql
->select SQL_NO_CACHE count(*) from test1 where id not in (select id from test2);
1 row in set(5.81sec)
->select SQL_NO_CACHE count(*) from test1 where not exists (select * from test2 where test2.id=test1.id);
1 row in set(5.25sec)
#换成下面这个之后：
->select SQL_NO_CACHE count(*) from test1 left join test2 on test1.id=test2.id where test2.id=null;
1 row in set(4.63sec)
```

案例二：mysql error （1093）： You can't specify target table '表名' for update in FROM clause（它的意思是说，不能先select出同一表中的某些值，再update这个表(在同一语句中)，即不能依据某字段值做判断再来更新某字段的值。）

```sql
->delete from tb_user where id in (select id from tb_user where id<5);
ERROR 1093(HY000): You can't specify target table 'tb_user' for update in FROM clause
#对于这种错误，将SELECT出的结果再通过中间表SELECT一遍
->delete from tb_user where id in (select * from (select id from tb_user where id<5) tmp);
#再优化，改为表连接模式
->delete from tb_user join (select id from tb_user where id<5) tmp on tmp.id=tb_user.id;
```

## 2.模式匹配like '%xxx%'优化

在Mysql里，like 'xxx%'可以用到索引，like ‘%xxx%’却不行。如下:

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/like%E7%9A%84%E4%BC%98%E5%8C%96.JPG)



这种情况可以通过覆盖索引来优化：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/like%E4%BC%98%E5%8C%962.JPG)

这里id是主键（聚集索引），叶子节点上保存了数据（InnoDB引擎），从索引中就能取得select的id列，不必读取数据行，通过覆盖索引，可以减少I/O，提供性能。（这里userId是索引，通过userId索引去找id）

优化案例：

```sql
->select count(*) from artist where name like '%Queen%';
1 row in set(17.12 sec)
#使用覆盖索引优化如下：
->select count(*) from artist a join (select artist_id from artis where name like '%Queen%') b on a.artist_id = b.artist_id;
1 row in set (8.31 sec)
```



## 3.limit分页优化

案例一：

```sql
select  SQL_NO_CACHE * from test1 order by id limit 99999,10;
#上面的语句要从第一行定位到第99999行再扫描出十行，显然效率不高

优化：
select SQL_NO_CACHE * FROM test1 where id>=100000 order by id limit 10;
```

案列二：

```sql
select id,title,createdate from test order by creatdate asc limit 355555,10;

优化：
select a.id,a.title,a.createdate from test join (select id from test order by createdate asc limit 355555,1) b on a.id >= b.id limit 10;
思路：先取出第35555行，再根据id去找后面的9行
```

## 4.count(*)如何加快速度

count(*)在innoDb里面一般是比较慢的，所以通过sql语句的变通来优化。

案例一：count(辅助索引)快于count(*)

```sql
select count(*) from tb_user

优化
select count(*) from tb_user where name>'0';
```

案例二：count（distinct）优化

```sql
select count(distinct k） from test;
       
优化：
select count(*) from (select distinct k from test);
```



注意：InnoDB不想MYISAM那样内置了一个计数器，可在count(*)的时候直接从计数器中那数据。InnoDB必须要全表扫描一下才能得到总的数据，而且会锁表。



## 5.or条件优化

**在SQL语句里面有or条件，则会用不到索引**。

```sql
select * from test where name='1' or age=41;
#这条语句不会使用索引
优化：使用union
select * from test where name='1' union select * from test where age=41;
```



## 6.使用ON DUPLICATE KEY UPDATE子句

eg:

```sql
insert into tb_user values('1','2','3') ON DUPLICATE KEY UPDATE name='2';
```

意思是插入数据，数据主键冲突了，就执行update操作。

## 7.用where子句代替having子句

having子句只会在检索出所有记录之后才会对数据集进行过滤。如果通过where子句限制记录的数目，可以减少很多开销。

# 合理使用索引

适当的索引对应用的性能来说至关重要，索引的速度是极快的。但是索引也有相关的开销，每次向表写入（INSERT,UPDATE，DELETE）时要维护索引。此外索引还增加了数据库的规模。只有当某列被用于where子句时，才能享受到索引性能提升带来的好处。

1.考虑使用联合索引

2.字段使用函数，将不能用到索引

3.使用字符类型的查找，用来查找的数据不加引号，如果不加不会走索引

4.当取出的数据量超过表中数据的20%时，优化器就不会使用索引，而是全表扫描。

5.考虑不为某些列建立索引，比如性别，state等

6.order by 优化：

```sql
select * from tb_user where id>10 order by name;
#上面的语句id和name都是索引，但一条sql只能使用一个索引，故优化器会选择最优的那个，所以可以考虑为id字段和name字段建立一个联合索引。
#再运行后可以看到Using filesort没有了，group by 也是同样的优化方法。
```

注意：如果order by后面有多个字段排序，排序顺序要一致，如果一个是asc一个是desc，也会出现Using filesort。



