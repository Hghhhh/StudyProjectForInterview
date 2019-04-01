## 1.mysql大数据量分页

分页语句：`select <colmoun> from <table> limit <num*(pageNum-1)>,<num>`

当页数太多的时候，比如`select id,name from tb_goods limit 10000000,10`

mysql是从头开始读的，计数到10000000条数据的时候，往后读10条，然后丢弃掉前面10000000条，速度会非常慢。

优化方法：延迟关联

上面的语句改成`select id,name from tb_goods inner join (select id from tb_goods limit 10000000,10) as tmp using (id)`

这里的id是主键，利用覆盖索引，先找出id，再根据10条id去找id和name，速度明显加快了。

## 2.巧用变量

1:用变量排名

例: 以ecshop中的商品表为例,计算每个栏目下的商品数,并按商品数排名.

`select cat_id,count(*)  as cnt from  goods group by cat_id order by cnt desc;`

并按商品数计算这些栏目的名次

```sql
set @curr_cnt := 0,@prev_cnt := 0, @rank := 0;

select cat_id, (@curr_cnt := cnt) as cnt,

(@rank := if(@curr_cnt <> @prev_cnt,@rank+1,@rank)) as rank,

@prev_cnt := @curr_cnt

 from ( select cat_id,count(*) as cnt from shop. goods group by shop. goods.cat_id order by cnt desc) as tmp;
```

2:用变量计算真正影响的行数

当插入多条,当主键重复时,则自动更新,这种效果,可以用**insert on duplicate key for update**

要统计真正”新增”的条目, 如下图,我们想得到的值是”1”,即被更新的行数.

![img](file:///C:/Users/ADMINI~1/AppData/Local/Temp/msohtmlclip1/01/clip_image001.png)

 

```sql
insert into user (uid,uname) values (4,’ds’),(5,'wanu'),(6,’safdsaf’)

 on duplicate key update  uid=values(uid)+(0*(@x:=@x+1)) , uname=values(uname);

mysql> set @x:=0;

Query OK, 0 rows affected (0.00 sec)

```

![img](file:///C:/Users/ADMINI~1/AppData/Local/Temp/msohtmlclip1/01/clip_image003.jpg)

总影响行数-2*实际update数, 即新增的行数.


3: 简化union

比如有新闻表,news , news_hot, 

new_hot是一张内存表,非常快,用来今天的热门新闻.

首页取新闻时,逻辑是这样的:

先取hot, 没有 再取news,为了省事,用一个union来完成.

```sql
select nid,title from news_hot where nid=xxx

union 

select nid,title from news where nid=xxx;
```

![img](file:///C:/Users/ADMINI~1/AppData/Local/Temp/msohtmlclip1/01/clip_image004.png)

 

如何利用变量让后半句select不执行 ,

```sql
set @find := null
select id,content,(@find := 1) from news where id=1

union

select id,content,(@find :=1) from news2 where id=1  and (@find <= 0)

union 1,1,1 where (@find :=null) is not null;
```



 