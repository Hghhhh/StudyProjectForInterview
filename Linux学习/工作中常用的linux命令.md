## 查看日志

### cat - 查看日志内容

```shell
cat error.log | grep -C 5 error //查看文件中error那行以及上下5行
(grep -B 5 error )  //查看文件中error那行及前5行
(grep -A 5 error )  //查看文件中error那行及后5行
```

### less - 分页查看日志

```shell
less error.log //分页查看日志
ps -ef | less //分页查看进程
history |less //分页查看命令历史记录
```

按q退出，按d翻一页，按空格翻半页，按b向后翻一页，输入‘/xxx’搜索，按n向后搜索，按N向前搜索，按G移动到日志最后，按g移动到日志最前

### tail - 查看文件结尾

```shell
tail file //显示文件最后10行
tail -n 100 file //显示文件最后100行
tail -f file //实时显示日志最后10行
tail +20 file //显示日志从20行到文件结尾
```



参考：

[查看线上日志常用命令](ttps://blog.csdn.net/xiaolyuh123/article/details/80365953)

[每天一个linux命令：less命令](https://www.cnblogs.com/peida/archive/2012/11/05/2754477.html)



## 查看服务器运行情况

### lsof - 查看打开的文件

```shell
lsof -i :80 //查看谁在使用80端口
lsof -p 1 //查看进程号为1打开的文件
lsof -c mysql //查看mysql进程打开的文件
lsof -u username //列出某个用户打开的文件信息
```

### df - 查看文件系统的磁盘空间占用情况

```shell
df -a //全部文件系统列表的磁盘情况
df -h //以方便阅读方式显示
df -i //显示inode信息
```

### du - 查看文件和目录的磁盘空间占用情况

```shell
du -ah file //文件目录都以方便阅读的方式显示显示
du -s //显示总和
du|sort -nr|more //按照空间大小排序查看
```

> du和df的区别：
>
>  du，disk usage,是通过搜索文件来计算每个文件的大小然后累加，du能看到的文件只是一些当前存在的，没有被删除的。他计算的大小就是当前他认为存在的所有文件大小的累加和。
>
>        df，disk free，通过文件系统来快速获取空间大小的信息，当我们删除一个文件的时候，这个文件不是马上就在文件系统当中消失了，而是暂时消失了，当所有程序都不用时，才会根据OS的规则释放掉已经删除的文件， df记录的是通过文件系统获取到的文件的大小，他比du强的地方就是能够看到已经删除的文件，而且计算大小的时候，把这一部分的空间也加上了，更精确了。
>         当文件系统也确定删除了该文件后，这时候du与df就一致了。

### top - 实时显示各个进程资源占用情况

```shell
top -c //显示完整的COMMAND
top -u www-data //显示www-data用户的进程
top -p 1 //显示pid为1的进程的情况
```

### free - 查看系统内存

```shell
free -h
free -s 10 //每10s执行一次
```

### netstat - 查看网络连接情况

```shell
netstat -a //查看所有连线中的socket
netstat -t //查看tcp协议连线情况
netstat -u //显示UDP传输协议的连线状况
netstat -p //显示正在使用Socket的程序识别码和程序名称。
```



参考：

[每天一个linux命令：lsof命令](https://www.cnblogs.com/peida/archive/2013/02/26/2932972.html)

[每天一个linux命令：du命令](https://www.cnblogs.com/peida/archive/2012/12/10/2810755.html)

[每天一个linux命令：top命令](https://www.cnblogs.com/peida/archive/2012/12/24/2831353.html)

[每天一个linux命令（45）：free 命令](https://www.cnblogs.com/peida/archive/2012/12/25/2831814.html)

[每天一个linux命令（56）：netstat命令](https://www.cnblogs.com/peida/archive/2013/03/08/2949194.html)

[阮一峰-理解inode（强烈推荐！！！）](http://www.ruanyifeng.com/blog/2011/12/inode.html)



## 权限相关

### chmod - 修改文件访问权限

```shell
chmod 775 file //给file所属用户设置读写执行权限，所在用户组所有用户设置读写执行权限，其他用户组用户设置读执行权限
数字顺序：
u(所属用户) , g(同组用户) , o (其他组用户) 
数字含义：
0 没有权限
1 可执行
2 可写
4 可读
```

### chown - 修改文件的所有者

```shell
chown mail:mail file //改变file的所属用户和组为mail mail
chown -R -v root:mail test6 //改变指定目录以及其子目录下的所有文件的拥有者和群组 
```

> Linux系统中的每个文件和目录都有访问许可权限，用它来确定谁可以通过何种方式对文件和目录进行访问和操作。
> 　　文件或目录的访问权限分为只读，只写和可执行三种。以文件为例，只读权限表示只允许读其内容，而禁止对其做任何的更改操作。可执行权限表示允许将该文件作为一个程序执行。文件被创建时，文件所有者自动拥有对该文件的读、写和可执行权限，以便于对文件的阅读和修改。用户也可根据需要把访问权限设置为需要的任何组合。
> 　　有三种不同类型的用户可对文件或目录进行访问：文件所有者，同组用户、其他用户。所有者一般是文件的创建者。所有者可以允许同组用户有权访问文件，还可以将文件的访问权限赋予系统中的其他用户。在这种情况下，系统中每一位用户都能访问该用户拥有的文件或目录。
> 　　每一文件或目录的访问权限都有三组，每组用三位表示，分别为文件属主的读、写和执行权限；与属主同组的用户的读、写和执行权限；系统中其他用户的读、写和执行权限。当用ls -l命令显示文件或目录的详细信息时，最左边的一列为文件的访问权限。 
>
> 例如：-rw-r--r-- 1 root root 296K 11-13 06:03 log2012.log
>
> 共有10个位置，第一个字符指定了文件类型。在通常意义上，一个目录也是一个文件。如果第一个字符是横线，表示是一个非目录的文件。如果是d，表示是一个目录。从第二个字符开始到第十个共9个字符，3个字符一组，分别表示了3组用户对文件或者目录的权限。权限字符用横线代表空许可，r代表只读，w代表写，x代表可执行。
>
> 例如：　　- rw- r-- r--
> 表示log2012.log是一个普通文件；log2012.log的属主有读写权限；与log2012.log属主同组的用户只有读权限；其他用户也只有读权限。
>
> 确定了一个文件的访问权限后，用户可以利用Linux系统提供的chmod命令来重新设定不同的访问权限。也可以利用chown命令来更改某个文件或目录的所有者。利用chgrp命令来更改某个文件或目录的用户组。 

### sudo 

```shell
sudo -s su -m root //切换为root用户
```



参考：

[每天一个linux命令（27）：linux chmod命令（强烈推荐！！！）](https://www.cnblogs.com/peida/archive/2012/11/29/2794010.html)

