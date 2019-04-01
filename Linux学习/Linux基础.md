## 1.Linux中怎么查看内存？

- 查看`/proc/meminfo`文件，这个文件列出了所有你想了解的内存的使用情况：

  ```shell
  cat /proc/meminfo
  MemTotal:        2048124 kB
  MemFree:          456656 kB
  MemAvailable:    1466260 kB
  Buffers:          204280 kB
  Cached:           894080 kB
  SwapCached:            0 kB
  Active:          1007412 kB
  Inactive:         457220 kB
  Active(anon):     366800 kB
  Inactive(anon):     2216 kB
  Active(file):     640612 kB
  Inactive(file):   455004 kB
  Unevictable:           0 kB
  Mlocked:               0 kB
  SwapTotal:             0 kB
  SwapFree:              0 kB
  Dirty:               192 kB
  Writeback:             0 kB
  AnonPages:        366268 kB
  Mapped:            81448 kB
  Shmem:              2748 kB
  Slab:             104128 kB
  SReclaimable:      91336 kB
  SUnreclaim:        12792 kB
  KernelStack:        4416 kB
  PageTables:         4212 kB
  NFS_Unstable:          0 kB
  Bounce:                0 kB
  WritebackTmp:          0 kB
  CommitLimit:     1024060 kB
  Committed_AS:    1113440 kB
  VmallocTotal:   34359738367 kB
  VmallocUsed:           0 kB
  VmallocChunk:          0 kB
  HardwareCorrupted:     0 kB
  AnonHugePages:         0 kB
  CmaTotal:              0 kB
  CmaFree:               0 kB
  HugePages_Total:       0
  HugePages_Free:        0
  HugePages_Rsvd:        0
  HugePages_Surp:        0
  Hugepagesize:       2048 kB
  DirectMap4k:       42880 kB
  DirectMap2M:     2054144 kB
  DirectMap1G:           0 kB
  ```

- 通过`free`命令,`free`命令是一个快速查看内存使用情况的方法，它是对 `/proc/meminfo `收集到的信息的一个概述。

```shell
free
              total        used        free      shared  buff/cache   available
Mem:        2048124      389072      456592        2748     1202460     1466200
Swap:             0           0           0

free -h
              total        used        free      shared  buff/cache   available
Mem:           2.0G        379M        446M        2.7M        1.1G        1.4G
Swap:            0B          0B          0B
```

- 通过`top`命令：top命令提供了实时的运行中的程序的资源使用统计

- 通过`ps`命令，ps命令可以实时的显示各个进程的内存使用情况，最常用的是`ps aux`来察看包含了其他使用者的详细信息：

```shell
ps -au
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root       683  0.0  0.1  15752  2156 ttyS0    Ss+  Feb10   0:00 /sbin/agetty --keep-baud 1152
root       685  0.0  0.0  15936  1880 tty1     Ss+  Feb10   0:00 /sbin/agetty --noclear tty1 l
root     25462  0.0  0.2  22628  5228 pts/0    Ss   09:09   0:00 -bash
root     26148  0.0  0.1  37364  3288 pts/0    R+   09:57   0:00 ps au
```

另外使用 `ps aux | grep [name]` 可以用来找到包含name的启动命令行的程序



## 2.常用的Linux命令？

- cd：`cd ..`,``cd ~`，`cd /`
- pwd:查看当前目录的路径
- mkdir：创建目录；加上 -p 会把不存在的目录也创建了
- ls:`ls -a`查看包括.开头的隐藏文件；`ls -l`查看文件的详细信息；`ls -s`按大小排序，`ls -t`按时间排序
- rm:

```shell
-f ：就是force的意思，忽略不存在的文件，不会出现警告消息  
-i ：互动模式，在删除前会询问用户是否操作  
-r ：递归删除，最常用于目录删除 
         实例：
         （1）删除任何.log文件；删除前逐一询问确认
         rm -i *.log
         （2）删除test子目录及子目录中所有档案删除,并且不用一一确认
         rm -rf test
         （3）删除以-f开头的文件
         rm -- -f*
```

- rmdir:删除空目录，加上-p，如果parent子目录被删除后使它也成为空目录的话，则顺便一并删除
- mv：修改文件名或移动文件

```shell
         移动文件或修改文件名，根据第二参数类型（如目录，则移动文件；如为文件则重命令该文件）。      
         当第二个参数为目录时，可刚多个文件以空格分隔作为第一参数，移动多个文件到参数2指定的目录中
         实例：
        （1）将文件test.log重命名为test1.txt
         mv test.log test1.txt
         （2）将文件log1.txt,log2.txt,log3.txt移动到根的test3目录中
         mv llog1.txt log2.txt log3.txt /test3
         （3）将文件file1改名为file2，如果file2已经存在，则询问是否覆盖
         mv -i log1.txt log2.txt
         （4）移动当前文件夹下的所有文件到上一级目录
         mv * ../
```

- cp:用来复制文件的命令

```shell
         将源文件复制至目标文件，或将多个源文件复制至目标目录。
         注意：命令行复制，如果目标文件已经存在会提示是否覆盖，而在shell脚本中，如果不加-i参数，则不会提示，而是直接覆盖！
         -i 提示
         -r 复制目录及目录内所有项目
         -a 复制的文件与原文件时间一样
         实例：
         （1）复制a.txt到test目录下，保持原文件时间,如果原文件存在提示是否覆盖
         cp -ai a.txt test
         （2）为a.txt建议一个链接（快捷方式）
         cp -s a.txt link_a.txt
```

- vi：编辑文件的工具
- cat：查看文件
- ps：查看进程运行情况
- top:实时显示进程运行情况
- free：显示内存信息
- wget：下载东西
- tar：解压、压缩文件
- kill：`kill -9`表示强制杀死进程，而 kill 则有局限性，例如后台进程，守护进程等；
- grep：文本搜索命令
- chmod命令:用于改变linux系统文件或目录的访问权限。

```shel
该命令有两种用法。一种是包含字母和操作符表达式的文字设定法；另一种是包含数字的数字设定法。
         每一文件或目录的访问权限都有三组，每组用三位表示，分别为文件属主的读、写和执行权限；与属主同组的用户的读、写和执行权限；系统中其他用户的读、写和执行权限。可使用ls -l test.txt查找
         以文件log2012.log为例：
         -rw-r--r-- 1 root root 296K 11-13 06:03 log2012.log
         第一列共有10个位置，第一个字符指定了文件类型。在通常意义上，一个目录也是一个文件。如果第一个字符是横线，表示是一个非目录的文件。如果是d，表示是一个目录。从第二个字符开始到第十个共9个字符，3个字符一组，分别表示了3组用户对文件或者目录的权限。权限字符用横线代表空许可，r代表只读，w代表写，x代表可执行。
         常用参数：
         -c 当发生改变时，报告处理信息
         -R 处理指定目录以及其子目录下所有文件
         权限范围：
         u ：目录或者文件的当前的用户
         g ：目录或者文件的当前的群组
         o ：除了目录或者文件的当前用户或群组之外的用户或者群组
         a ：所有的用户及群组
         权限代号：
         r ：读权限，用数字4表示
         w ：写权限，用数字2表示
         x ：执行权限，用数字1表示
         - ：删除权限，用数字0表示
         s ：特殊权限
         实例：
         （1）增加文件t.log所有用户可执行权限
         chmod a+x t.log
         （2）撤销原来所有的权限，然后使拥有者具有可读权限,并输出处理信息
        chmod u=r t.log -c
         （3）给file的属主分配读、写、执行(7)的权限，给file的所在组分配读、执行(5)的权限，给其他用户分配执行(1)的权限
        chmod 751 t.log -c（或者：chmod u=rwx,g=rx,o=x t.log -c)
         （4）将test目录及其子目录所有文件添加可读权限
         chmod u+r,g+r,o+r -R text/ -c
```

- df:查看磁盘使用情况

```shell
显示磁盘空间使用情况。获取硬盘被占用了多少空间，目前还剩下多少空间等信息，如果没有文件名被指定，则所有当前被挂载的文件系统的可用空间将被显示。
         -a 全部文件系统列表
         -h 以方便阅读的方式显示信息
         -i 显示inode信息
         -k 区块为1024字节
         -l 只显示本地磁盘
         -T 列出文件系统类型
         实例：
         （1）显示磁盘使用情况
         df -l
         （2）以易读方式列出所有文件系统及其类型
         df -haT 
```

- du:查看文件或目录所占磁盘空间

  ```shel
           命令格式：
  
           du [选项] [文件]
  
           常用参数：
  
           -a 显示目录中所有文件大小
  
           -k 以KB为单位显示文件大小
  
           -m 以MB为单位显示文件大小
  
           -g 以GB为单位显示文件大小
  
           -h 以易读方式显示文件大小
  
           -s 仅显示总计
  
           -c或--total  除了显示个别目录或文件的大小外，同时也显示所有目录或文件的总和
  
           实例：
  
           （1）以易读方式显示文件夹内及子文件夹大小
  
           du -h scf/
  
           （2）以易读方式显示文件夹内所有文件大小
  
           du -ah scf/
  
           （3）显示几个文件或目录各自占用磁盘空间的大小，还统计它们的总和
  
           du -hc test/ scf/
  
           （4）输出当前目录下各个子目录所使用的空间
  
           du -hc --max-depth=1 scf/
  ```




参考[Linux常用命令](https://www.cnblogs.com/gaojun/p/3359355.html)