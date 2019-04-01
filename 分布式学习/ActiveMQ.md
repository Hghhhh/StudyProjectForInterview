转自https://segmentfault.com/a/1190000014958916

## JMS(Java Message Service)

3.1什么是jms?

> JMS即Java消息服务（Java Message Service）应用程序接口，是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。Java消息服务是一个与具体平台无关的API，绝大多数MOM提供商都对JMS提供支持。

3.2 JMS 消息传送模式

![clipboard.png](https://segmentfault.com/img/bVbanlX?w=469&h=251)

- 客户端 A、C 和 D之间的消息传送说明了点对点模式(**P2P**)。客户端使用此模式向队列目的地发送一条消息，只有一个接收者能够从该目的地获得该消息。访问该目的地的其他任何接收者都不能获得该消息。
- 客户端 B、E 和 F之间的消息传送说明了发布/订阅模式(**publish-subscribe)**。客户端使用此广播模式向主题目的地发送一条消息，任意数量的使用方订户都可以从该目的地检索此消息。每个订户都获得此消息的一个副本。

3.3 JMS 消息传送对象

```
Provider/MessageProvider：生产者
Consumer/MessageConsumer：消费者
PTP：Point To Point，点对点通信消息模型
Pub/Sub：Publish/Subscribe，发布订阅消息模型
Queue：队列，目标类型之一，和PTP结合
Topic：主题，目标类型之一，和Pub/Sub结合
ConnectionFactory：连接工厂，JMS用它创建连接
Connnection：JMS Client到JMS Provider的连接
Destination：消息目的地，由Session创建
Session：会话，由Connection创建，实质上就是发送、接受消息的一个线程，因此生产者、消费者都是Session创建的
```

![clipboard.png](https://segmentfault.com/img/bVbanoM?w=407&h=470)

## MQ (Message Queue)

> MQ全称为Message Queue,消息队列(MQ)是正确而又完整的 JMS 实现,消息队列（MQ）是一种应用程序对应用程序的通信方法。应用程序通过写和检索出入列队的针对应用程序的数据（消息）来通信，而无需专用连接来链接它们。消息传递指的是程序之间通过在消息中发送数据进行通信，而不是通过直接调用彼此来通信，直接调用通常是用于诸如远程过程调用的技术。

## JMS实现--ActiveMQ

> ActiveMQ是Apache软件基金下的一个开源软件，它遵循JMS1.1规范（Java Message Service），是消息驱动中间件软件（MOM）。它为企业消息传递提供高可用，出色性能，可扩展，稳定和安全保障。
>
>

#### 5.1 中间件、JMS、MQ、ActiveMQ之间的关系

![clipboard.png](https://segmentfault.com/img/bVbaHrv?w=538&h=358)

#### 5.2 ActiveMQ的消息传递模式

![clipboard.png](https://segmentfault.com/img/bVbaHPh?w=715&h=339)

> P2P （点对点）消息域使用 queue 作为 Destination，消息可以被同步或异步的发送和接收，每个消息只会给一个 Consumer 传送一次。
> Pub/Sub（发布/订阅，Publish/Subscribe）消息域使用 topic 作为 Destination，发布者向 topic 发送消息，订阅者注册接收来自 topic 的消息。发送到 topic 的任何消息都将自动传递给所有订阅者。接收方式（同步和异步）与 P2P 域相同。

#### 5.3 ActiveMQ简单案例

*消息生产者*

```
       //创建session会话
        ConnectionFactory factory = new ActiveMQConnectionFactory("tcp://192.168.187.13:61616");
        Connection connection = factory.createConnection();
        connection.start();
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);

        //创建一个消息队列 session.createQueue("jms.test.topic")--P2P模式
        Destination destination = session.createTopic("jms.test.topic");

        //创建消息生产者
        MessageProducer producer = session.createProducer(destination);

        //消息持久化
        producer.setDeliveryMode(DeliveryMode.PERSISTENT);

        for (int i = 0; i < messageNum; i++) {
            producer.send(session.createTextMessage("Message Producer:" + i));
        }

        //提交会话
        session.commit();
```

*消息消费者*

```java
       //创建session会话
        ConnectionFactory factory = new ActiveMQConnectionFactory("tcp://192.168.187.13:61616");
        Connection connection = factory.createConnection();
        connection.start();
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);

        //创建一个消息队列 session.createQueue("jms.test.topic")--P2P模式
        Destination destination = session.createTopic("jms.test.topic");

        //创建消息消费者
        MessageConsumer consumer = session.createConsumer(destination);

        while (true) {
            TextMessage message = (TextMessage) consumer.receive();
            if (message != null){
                System.out.println("Message Consumer:"+message.getText());
            }else {
                break;
            }
        }
        session.commit();
```

#### 5.4 ActiveMQ的消息存储机制

- KahaDB

> ActiveMQ 5.3 版本起的默认存储方式。**KahaDB存储是一个基于文件的快速存储消息**，设计目标是易于使用且尽可能快。它使用基于文件的消息数据库意味着没有第三方数据库的先决条件。

```
<broker brokerName="broker" persistent="true" useShutdownHook="false">
        <persistenceAdapter>
                <kahaDB directory="${activemq.data}/kahadb" journalMaxFileLength="32mb"/>
        </persistenceAdapter>
</broker>
```

- AMQ

> MQ存储使用户能够快速启动和运行，因为它不依赖于第三方数据库。**AMQ 消息存储库是可靠持久性和高性能索引的事务日志组合**，当消息吞吐量是应用程序的主要需求时，该存储是最佳选择。但因为它为每个索引使用两个分开的文件，并且每个 Destination 都有一个索引，所以当你打算在代理中使用数千个队列的时候，不应该使用它。

```
<persistenceAdapter>
        <amqPersistenceAdapter
                directory="${activemq.data}/kahadb"
                syncOnWrite="true"
                indexPageSize="16kb"
                indexMaxBinSize="100"
                maxFileLength="10mb" />
</persistenceAdapter>
```

- JDBC

> 选择关系型数据库，通常的原因是企业已经具备了管理关系型数据的专长，但是它在性能上绝对不优于上述消息存储实现。事实是，许多企业使用关系数据库作为存储，是因为他们更愿意充分利用这些数据库资源。

```
<beans>
        <broker brokerName="test-broker" persistent="true" xmlns="http://activemq.apache.org/schema/core">
                <persistenceAdapter>
                        <jdbcPersistenceAdapter dataSource="#mysql-ds"/>
                </persistenceAdapter>
        </broker>
        <bean id="mysql-ds" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
                <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://127.0.0.1/jms?relaxAutoCommit=true"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
                <property name="maxActive" value="200"/>
                <property name="poolPreparedStatements" value="true"/>
        </bean>
</beans>
```

- 内存存储

> 内存消息存储器将所有持久消息保存在内存中。在仅存储有限数量 Message 的情况下，内存消息存储会很有用，因为 Message 通常会被快速消耗。在 activema.xml 中将 broker 元素上的 persistent 属性设置为 false 即可。

```
<broker brokerName="test-broker" persistent="false" xmlns="http://activemq.apache.org/schema/core">
        <transportConnectors>
                <transportConnector uri="tcp://192.168.187.13:61616"/>
        </transportConnectors>
</broker>
```

------

#### 1.KahaDB存储的目录结构及简单说明

```
    -rw-rw-r--. 1 lvmama01 lvmama01 32M 5月  18 09:47 db-1.log
    -rw-rw-r--. 1 lvmama01 lvmama01 32K 5月  18 09:47 db.data
    -rw-rw-r--. 1 lvmama01 lvmama01 33K 5月  18 09:47 db.redo
    -rw-rw-r--. 1 lvmama01 lvmama01 0   5月  16 19:31 lock
```

可以看出，上面directory一共有四个文件：

①db.data

它是消息的索引文件。本质上是B-Tree的实现，使用B-Tree作为索引指向db-*.log里面存储的消息。

②db.redo

主要用来进行消息恢复。

③db-*.log 存储消息的内容。对于一个消息而言，不仅仅有消息本身的数据(message data)，而且还有(Destinations、订阅关系、事务...),data log以日志形式存储消息，而且新的数据总是以APPEND的方式追加到日志文件末尾。因此，消息的存储是很快的。比如，对于持久化消息，Producer把消息发送给Broker，Broker先把消息存储到磁盘中(enableJournalDiskSyncs配置选项)，然后再向Producer返回Acknowledge。Append方式在一定程度上减少了Broker向Producer返回Acknowledge的时间。

④lock文件

#### 2.KahaDB存储底层原理简单分析

*KahaDB内部分为：data logs, 按照Message ID高度优化的索引，memory message cache。*

![clipboard.png](https://segmentfault.com/img/bVbaKES?w=380&h=404)

①在内存(cache)中的那部分B-Tree是Metadata Cache

通过将索引缓存到内存中，可以加快查询的速度(quick retrival of message data)。但是需要定时将 Metadata Cache 与 Metadata Store同步。这个同步过程就称为：check point。由checkpointInterval选项 决定每隔多久时间进行一次checkpoint操作。

②BTree Indexes则是保存在磁盘上的，称为Metadata Store，它对应于文件db.data，它就是对Data Logs以B树的形式索引。

有了它，Broker（消息服务器）可以快速地重启恢复，因为它是消息的索引，根据它就能恢复出每条消息的location。如果Metadata Store被损坏，则只能扫描整个Data Logs来重建B树了。

③Data Logs则对应于文件 db-*.log，默认是32MB

Data Logs以日志形式存储消息，它是生产者生产的数据的真正载体。

④Redo Log则对应于文件 db.redo,redo log的原理用到了“Double Write”。

> 简要记录下自己的理解：因为磁盘的页大小与操作系统的页大小不一样，磁盘的页大小一般是16KB，而OS的页大小是4KB。而数据写入磁盘是以磁盘页大小为单位进行的，即一次写一个磁盘页大小，这就需要4个OS的页大小（4*4=16）。如果在写入过程中出现故障(突然断电)就会导致只写入了一部分数据(partial page write)
> 而采用了“Double Write”之后，将数据写入磁盘时，先写到一个Recovery Buffer中，然后再写到真正的目的文件中。在ActiveMQ的源码PageFile.java中有相应的实现。

```
public void unload() throws IOException {
     //load时创建writeFile(db.data)和 recoveryFile(db.redo)
    writeFile = new RecoverableRandomAccessFile(file, "rw", false);

    ........

    if (enableRecoveryFile) {
         recoveryFile = new RecoverableRandomAccessFile(getRecoveryFile(), "rw");
    }
}
private void writeBatch() throws IOException {
    .......
    //将数据写入磁盘时，先写到一个Recovery Buffer中(db.data)
    for (PageWrite w : batch) {
        try {
            checksum.update(w.getDiskBound(), 0, pageSize);
        } catch (Throwable t) {
            throw IOExceptionSupport.create("Cannot create recovery file. Reason: " + t, t);
        }
        recoveryFile.writeLong(w.page.getPageId());
        recoveryFile.write(w.getDiskBound(), 0, pageSize);
    }
    .......
    //写入真正的目的文件中(db.redo)
    for (PageWrite w : batch) {
        writeFile.seek(toOffset(w.page.getPageId()));
        writeFile.write(w.getDiskBound(), 0, pageSize);
        w.done();
    }
}
```

#### 5.4 ActiveMQ的部署模式

*1.默认的单机部署（kahadb）*

略......

------

2.共享存储主从模式(基于数据库)

![clipboard.png](https://segmentfault.com/img/bVbaMA1?w=486&h=409)

------

*3.共享存储主从模式(基于文件系统)*

![clipboard.png](https://segmentfault.com/img/bVbaMBQ?w=457&h=405)

------

*4.基于zookeeper的主从（levelDB Master/Slave\**详细说明**）*

第一步:zookeeper集群搭建

```
server.1=lvmama01:2888:3888
server.2=lvmama02:2888:3888
server.3=lvmama03:3888:3888
```

第二步:activemq集群搭建修改activemq.xml文件:

```
 <persistenceAdapter>
         <replicatedLevelDB 
            　　directory="${activemq.data}/leveldb"
            　　replicas="3"
            　　bind="tcp://0.0.0.0:0"
            　　//zookeeper集群地址
            　　zkAddress="192.168.187.11:2181,192.168.187.12:2181,192.168.187.13:2181"
            　　//本地ip
            　　hostname="192.168.187.11"
            　　sync="local_disk"
            　　zkPath="/activemq/leveldb-stores"
        　　/>
   </persistenceAdapter>
```

第三步:分别启动三台activemq(**仔细查看日志**):

1.启动第一台机器(lvmama01:192.168.187.11)

![clipboard.png](https://segmentfault.com/img/bVbaMY1?w=1840&h=589)

2.启动第二台机器(lvmama02:192.168.187.12)

![clipboard.png](https://segmentfault.com/img/bVbaMZS?w=1826&h=703)

![clipboard.png](https://segmentfault.com/img/bVbaM1p?w=1820&h=338)

3.第三台启动同第二台

第三步:查看是否启动成功(没成功可以查看activemq.log日志)

![clipboard.png](https://segmentfault.com/img/bVbaMIa?w=984&h=186)

启动成功后通过zkCli.sh可以看到已创建leveldb-stores如下:

![clipboard.png](https://segmentfault.com/img/bVbaMJO?w=630&h=111)

第四步：通过流量器访问web管理页面(注意只有master机器可以访问)

![clipboard.png](https://segmentfault.com/img/bVbaMKz?w=1836&h=680)

第五步:测试

```
        String userName = ActiveMQConnectionFactory.DEFAULT_USER;
        String password = ActiveMQConnectionFactory.DEFAULT_PASSWORD;
        String brokerURL = "failover:(tcp://192.168.187.11:61616,tcp://192.168.187.12:61616,tcp://192.168.187.13:61616)?Randomize=false";

        //2. 通过ConnectionFactory建立一个Connection连接，并且调用start方法开启
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(userName, password, brokerURL);
        Connection connection = connectionFactory.createConnection();
        connection.start();

        //3. 通过Connection创建Session，用于接收消息[第一个参数：是否启用事务；第二个参数：设置签收模式]
        Session session = connection.createSession(false, Session.CLIENT_ACKNOWLEDGE);

        //4. 通过Session创建Destination对象
        Destination destination = session.createQueue("cluster-queue");

        //5. 通过Session创建发送或接受对象
        MessageProducer messageProducer = session.createProducer(null);
        
```

运行结果(**此时发送的目标为192.168.187.11**):

```
Connected to the target VM, address: '127.0.0.1:12266', transport: 'socket'
 INFO | Successfully connected to tcp://192.168.187.11:61616
生产者：Hello MQ:1
生产者：Hello MQ:2
生产者：Hello MQ:3
生产者：Hello MQ:4
生产者：Hello MQ:5
生产者：Hello MQ:6
生产者：Hello MQ:7
生产者：Hello MQ:8
生产者：Hello MQ:9
```

**此时将activemq master服务停止,集群自动重新选举 lvmama02(192.168.187.12)成为Master**

![clipboard.png](https://segmentfault.com/img/bVbaM3X?w=1778&h=394)

我们再试运行测试用例发现消息任然可以发送，**只不过发送的目标变为192.168.187.12**



```
Connected to the target VM, address: '127.0.0.1:12400', transport: 'socket'
 INFO | Successfully connected to tcp://192.168.187.12:61616
生产者：Hello MQ:1
生产者：Hello MQ:2
生产者：Hello MQ:3
生产者：Hello MQ:4
生产者：Hello MQ:5
生产者：Hello MQ:6
生产者：Hello MQ:7
生产者：Hello MQ:8
生产者：Hello MQ:9
```