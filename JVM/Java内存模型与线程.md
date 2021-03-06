---
title: Java内存模型与线程
date: 2018-09-13 19:56:56
tags: JVM
---

## 硬件的效率与一致性

在正式了解Java虚拟机并发相关知识前，需要先花费一点时间去了解下物理计算机中的并发问题。

由于计算机的存储设备与处理器的运算速度存在几个数量级的差距，于是加入了高速缓存来作为内存与处理器之间的缓冲：**将运算需要使用的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回内存中，这样处理器就无须等待缓慢的内存读写了。**但这也引入了缓存一致性的问题，即每个处理器都有自己的一个高速缓存，那么同步相同数据回主内存的时候以谁为准。这里需要使用一些缓存一致性协议来处理。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/%E9%AB%98%E9%80%9F%E7%BC%93%E5%AD%98%E3%80%81%E4%B8%BB%E5%86%85%E5%AD%98.jpg)



## Java内存模型（JMM）

让Java在各种平台都能达到一致的内存访问效果。

### 主内存和工作内存

JMM规定了所有变量都存储在主内存，每个线程有自己的工作内存，线程对变量的操作必须在工作内存，而不能直接读写主内存中的变量，不同线程之间的工作内存是隔离的。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/jMM.jpg)

### 内存间交互操作

JMM定义了8种操作来完成工作内存和主内存的交互，每一个操作都是原子的、不可再分的：

- lock（锁定）：作用于主内存的变量，它把一个变量标识为一个线程独占的状态
- unlock（解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
- read（读取）：作用于主内存的变量，它把一个变量的值从主内存传送到线程中的工作内存，以便随后的load动作使用
- load（载入）：作用于工作内存的变量，它把read操作从主内存中得到的变量值放入工作内存的变量副本中
- use（使用）：作用于工作内存的变量，它把工作内存中一个变量的值传递给执行引擎
- assign（赋值）：作用于工作内存的变量，它把一个从执行引擎接收到的值赋值给工作内存中的变量
- store（存储）：作用于工作内存的变量，它把工作内存中的一个变量的值传送到主内存中，以便随后的write操作
- write（写入）：作用于主内存的变量，它把store操作从工作内存中得到的变量的值写入主内存的变量中

**Java内存模型还规定了执行上述8种基本操作时必须满足如下规则**

​     1、不允许read和load、store和write操作之一单独出现，以上两个操作必须按顺序执行，但没有保证必须连续执行，也就是说，read与load之间、store与write之间是可插入其他指令的。

​     2、不允许一个线程丢弃它的最近的assign操作，即变量在工作内存中改变了之后必须把该变化同步回主内存。

​     3、不允许一个线程无原因地（没有发生过任何assign操作）把数据从线程的工作内存同步回主内存中。

​     4、一个新的变量只能从主内存中“诞生”，不允许在工作内存中直接使用一个未被初始化（load或assign）的变量，换句话说就是对一个变量实施use和store操作之前，必须先执行过了assign和load操作。

​    5、一个变量在同一个时刻只允许一条线程对其执行lock操作，但lock操作可以被同一个条线程重复执行多次，多次执行lock后，只有执行相同次数的unlock操作，变量才会被解锁。

​    6、如果对一个变量执行lock操作，将会清空工作内存中此变量的值，在执行引擎使用这个变量前，需要重新执行load或assign操作初始化变量的值。

​    7、如果一个变量实现没有被lock操作锁定，则不允许对它执行unlock操作，也不允许去unlock一个被其他线程锁定的变量。

​    8、对一个变量执行unlock操作之前，必须先把此变量同步回主内存（执行store和write操作）。

### volatile型变量的特殊规则

volatile保证了可见性和有序性，但不保证原子性。

####为什么不保证原子性？

因为修改一个volatile变量的值分为多步，先从主内存中read和load最新的值到工作内存，赋值修改后再store和write回去主内存。所以如果运算结果依赖当前值还是需通过加锁来保证原子性。

####volatile保证可见性、有序性？

通过禁止指令重排序来保证。

在修改完volatile变量的值后，会在后面多执行一条字节码指令，`lock addl $0x0,(%esp)`，这个操作相当于一个内存屏障（意味着之前的所有操作都已经执行完了，重排序的时候内存屏障之后的指令不能重排序到内存屏障之前）。lock的作用是使得本CPU的Cache写入了内存，该写入动作会导致其他CPU无效化其Cache，通过这样一个空操作，可让前面volatile变量的修改对其他CPU立即可见。

####JMM对volatile的特殊规则

- 简单来说，要求工作内存使用Volatile变量之前必须先从主内存刷新最新的值
- 修改volatile变量的值需要立刻写回主内存
- volatile修饰的变量不会被指令从排序优化。

### sychronized的原子性、可见性、有序性是怎么实现的？

- 原子性：通过字节码指令`monitorenter`、`monitorexit`来保证，这两条指令隐式调用了lock和unlock操作，它们之间的代码块只能一次进入一个线程，故保证了原子性
- 可见性：syschronized的可见性是由”对一个变量执行unlock操作之前，必须先把此变量同步回主内存中“这条规则得到的。
- 有序性：一个变量在同一时刻只允许一条线程对其进行lock操作，持有同一个锁的两个同步块只能串行地进入。

###先行发生原则（happen-before）

Java内存模型中定义的两项操作之间的偏序关系，如果说操作A先于操作B，其实就是说发送操作B之前，操作A产生的影响能被操作B观察到，影响包括了修改内存中共享变量的值、发送了信息、调用了方法等。

JMM有这些”天生的“先行发生原则：

**1、程序次序规则**。**在一个线程内，书写在前面的代码先行发生于后面的。确切地说应该是，按照程序的控制流顺序，因为存在一些分支结构。**

**2、Volatile变量规则**。**对一个volatile修饰的变量，对他的写操作先行发生于读操作。**

**3、线程启动规则。Thread对象的start()方法先行发生于此线程的每一个动作。**

**4、线程终止规则。线程的所有操作都先行发生于对此线程的终止检测。**

**5、线程中断规则。对线程interrupt()方法的调用先行发生于被中断线程的代码所检测到的中断事件。**

**6、对象终止规则。一个对象的初始化完成（构造函数之行结束）先行发生于发的finilize()方法的开始。**

**7、传递性**。**A先行发生B，B先行发生C，那么，A先行发生C。**

**8、管程锁定规则。一个usnlock操作先行发生于后面对同一个锁的lock操作。**

 

以上就是Java无需任何的同步手段就能成立的先行发生规则。其他情况下就没有顺序保障，虚拟机可以对它们随意地进行重排序。

































































-+**++++++++++++++++++++++++++