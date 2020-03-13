RocketMQ官网<http://rocketmq.apache.org/docs/quick-start/>

视频地址<https://www.bilibili.com/video/av66702383?p=24>

## 消息样例

![流程分析](https://s2.ax1x.com/2020/03/08/3z6yid.png)

### 基本样例

#### 消息发送

##### 1）发送同步消息

```java
/**
 * 发送同步消息
 */
public class SynProducer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer producer = new DefaultMQProducer("SynProducer");
        producer.setNamesrvAddr("120.77.208.81:9876");
        producer.start();
        for(int i=0; i<10; i++){
            Message msg = new Message("TopicTest", "TagA",
                    ("Hello RocketMQ").getBytes());
            SendResult sendResult = producer.send(msg);
            System.out.println("返回结果："  + sendResult);
        }
        producer.shutdown();
    }
}
```

##### 2）发送异步消息

通常用在对响应时间敏感的业务场景，即发送端不能容忍长时间地等待Broker的响应。

```java
public class AsyncProducer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer producer = new DefaultMQProducer("SynProducer");
        producer.setNamesrvAddr("120.77.208.81:9876");
        producer.start();
        for(int i=0; i<10; i++){
            Message msg = new Message("TopicTest", "Tag2",
                    ("Hello RocketMQ").getBytes());
            producer.send(msg, new SendCallback() {
                public void onSuccess(SendResult sendResult) {
                    System.out.println("发送结果：" + sendResult);
                }
                public void onException(Throwable e) {
                    System.out.println("发送异常：" + e);
                }
            });
            TimeUnit.MILLISECONDS.sleep(300);
        }
        producer.shutdown();
    }
}
```

##### 3) 发送单向信息

这种方法主要用在不特别关心结果的场景，例如发送日志。

```java
public class OneWayProducer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer producer = new DefaultMQProducer("SynProducer");
        producer.setNamesrvAddr("120.77.208.81:9876");
        producer.start();
        for(int i=0; i<10; i++){
            Message msg = new Message("TopicTest", "Tag3",
                    ("Hello 单项信息").getBytes());
            producer.sendOneway(msg);
        }
        producer.shutdown();
    }
}
```

#### 消费信息

```java
public class Comsumer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
        consumer.setNamesrvAddr("120.77.208.81:9876");
        consumer.subscribe("TopicTest", "Tag2");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs){
                    System.out.println(new String(msg.getBody()));
                }
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
       consumer.start();
    }
}
```

##### 1） 负载均衡模式

默认就是这种模式

```java
       consumer.setMessageModel(MessageModel.CLUSTERING);

```

##### 2） 广播模式

```java
        consumer.setMessageModel(MessageModel.BROADCASTING);

```

### 顺序消息

默认情况下，消息发送会采用轮询方式发送到不同的queue；而消费消息时从多个queue拉取消息是不能保证顺序的。但是如果控制发送的顺序消息只依次发送到同一个queue中，消息消费的时候只从这个queue上依次拉取，就保证了顺序。当发生和消费参与的queue只有一个，则是全局有序；如果多个queue参与，则为分区有序，即相对于每个queue，消息都是有序的。

![顺序消息分析](https://s2.ax1x.com/2020/03/08/3z7gAI.png)

#### 顺序消息生产

```java
public class Producer {

    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer producer = new DefaultMQProducer("SynProducer");
        producer.setNamesrvAddr("120.77.208.81:9876");
        producer.start();
        List<Order> orders = Order.buildOrders();
        for(Order order : orders){
            Message msg = new Message("OrderTopic", "Order",
                    (order.toString()).getBytes());
            /**
             * 参数一： 消息对象
             * 参数二：消息队列选择器
             * 参数三： 选择队列的业务标识（订单ID）
             */
            SendResult sendResult = producer.send(msg,
                    new MessageQueueSelector() {
                        /**
                         * @param mqs 队列集合
                         * @param msg 消息对象
                         * @param arg 业务标识
                         * @return
                         */
                        public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                            int id = (Integer)arg;
                            int chooseQueue = id % mqs.size();
                            return mqs.get(chooseQueue);
                        }
                    }, order.getOrderId());
            System.out.println("返回结果："  + sendResult);
        }
        producer.shutdown();
    }

}
```

#### 顺序消息消费

```java
public class Comsumer {
    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group1");
        consumer.setNamesrvAddr("120.77.208.81:9876");
        consumer.subscribe("OrderTopic", "Order");
        consumer.setMessageModel(MessageModel.CLUSTERING);
        //MessageListenerOrderly
        consumer.registerMessageListener(new MessageListenerOrderly() {
            public ConsumeOrderlyStatus consumeMessage(List<MessageExt> msgs, ConsumeOrderlyContext context) {
                for(MessageExt msg : msgs){
                    System.out.println(Thread.currentThread().getName() + " " + new String(msg.getBody()));
                }
                return ConsumeOrderlyStatus.SUCCESS;
            }
        });
        consumer.start();
    }
}
```



### 延时消息

目前只支持下面1-18个延时等级

```java
private String messageDelayLevel = "1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h"
```

#### 发送延时消息

```java
public class DelayProducer {

    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        DefaultMQProducer producer = new DefaultMQProducer("SynProducer");
        producer.setNamesrvAddr("120.77.208.81:9876");
        producer.start();
        for(int i=0; i<10; i++){
            Message msg = new Message("DelayTopic", "Tag1",
                    ("Hello RocketMQ" + i).getBytes());
            //设置延迟时间
            msg.setDelayTimeLevel(2);
            SendResult sendResult = producer.send(msg);
        }
        producer.shutdown();
    }

}
```

#### 消费延迟消息

```java
   
//其他一样
System.out.println("延迟时间：" + (System.currentTimeMillis() - msg.getStoreTimestamp()));

```

### 批量发送消息

Producer直接send(List<Message>)即可，**批量消息不能超过4M**

如果信息总长度大于4M，这时候要把信息进行分割

```java
/**
 * 4M分隔器
 * 使用方法： Producer批量发送的时候
 * List<Message> messages = new ArrayList<>();
 * messages.add(message1);
 * messages.add(message2);
 * messages.add(message3); 
 * ...
 * ListSpliter listSpliter = new ListSpliter(messages);
 * while(listSpliter.hasNext()){
 *     producer.send(listSpliter.next());
 * }
 */
public class ListSpliter implements Iterator<List<Message>> {
    private final int SIZE_LIMIT = 1024 * 1024 * 4;
    private final List<Message> messages;
    private int curIndex;

    public ListSpliter(List<Message> messages){
        this.messages = messages;
    }

    public boolean hasNext() {
        return curIndex < messages.size();
    }

    public List<Message> next() {
        int nextIndex = curIndex;
        int totalSize = 0;
        for(; nextIndex < messages.size(); nextIndex ++){
            Message message = messages.get(nextIndex);
            int tmpSize = message.getTopic().length() + message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for(Map.Entry<String, String> entry : properties.entrySet()){
                tmpSize += entry.getKey().length() + entry.getValue().length();
            }
            //增加日志的开销20字节
            tmpSize += 20;
            if(tmpSize > SIZE_LIMIT){
                //单个消息超过了最大限制
                //忽略，否则会阻塞分裂的进程
                if(nextIndex - curIndex == 0){
                    //假如下一个子列表没有元素，则添加这个子列表然后退出循环
                    nextIndex ++ ;
                }
                break;
            }
            if(tmpSize + totalSize > SIZE_LIMIT){
                break;
            }
            else{
                totalSize += tmpSize;
            }
        }
        List<Message> subList = messages.subList(curIndex, nextIndex);
        curIndex = nextIndex;
        return subList;
    }

    public void remove() {
    }
}
```



### 过滤信息

#### Tag过滤

```java
consumer.subscribe("BatchTopic", "tag1"); //接收tag1的消息
consumer.subscribe("BatchTopic", "tag1 || tag2"); //接收tag1和tag2的消息
consumer.subscribe("BatchTopic", "*"); //接收所有tag的消息
```

#### SQL语句过滤

使用sql基本语法来设置过滤条件

![sql语句过滤](https://s2.ax1x.com/2020/03/09/8SSAw8.png)

```java
//Producer中给Message设置属性
msg.putUserProperty("i", String.valueOf(i));

//Consumer：
consumer.subscribe("BatchTopic", MessageSelector.bySql("i>5")); //接收message中i>5的
```



### 事务消息

#### 事务流程分析

![事务流程分析](https://s2.ax1x.com/2020/03/09/8SSKln.png)

#### 发送事务消息

```java
public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        TransactionListener transactionListener = new TransactionListenerImpl();
        TransactionMQProducer producer = new TransactionMQProducer("please_rename_unique_group_name");
        ExecutorService executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });

        producer.setExecutorService(executorService);
        producer.setTransactionListener(transactionListener);
        producer.start();

        String[] tags = new String[] {"TagA", "TagB", "TagC", "TagD", "TagE"};
        for (int i = 0; i < 10; i++) {
            try {
                Message msg =
                    new Message("TopicTest1234", tags[i % tags.length], "KEY" + i,
                        ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                SendResult sendResult = producer.sendMessageInTransaction(msg, null);
                System.out.printf("%s%n", sendResult);

                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        for (int i = 0; i < 100000; i++) {
            Thread.sleep(1000);
        }
        producer.shutdown();
    }
}

public class TransactionListenerImpl implements TransactionListener {
       private AtomicInteger transactionIndex = new AtomicInteger(0);
   
       private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();
   
       @Override
       public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
           int value = transactionIndex.getAndIncrement();
           int status = value % 3;
           localTrans.put(msg.getTransactionId(), status);
           return LocalTransactionState.UNKNOW;
       }
   
       @Override
       public LocalTransactionState checkLocalTransaction(MessageExt msg) {
           Integer status = localTrans.get(msg.getTransactionId());
           if (null != status) {
               switch (status) {
                   case 0:
                       return LocalTransactionState.UNKNOW;
                   case 1:
                       return LocalTransactionState.COMMIT_MESSAGE;
                   case 2:
                       return LocalTransactionState.ROLLBACK_MESSAGE;
               }
           }
           return LocalTransactionState.COMMIT_MESSAGE;
       }
   }
```



## SpringBoot集成RocketMQ

#### 生产者

```javascript
//1.添加依赖
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.1.0</version>
</dependency>

//2.配置文件
#application.properties
rocketmq.name-server=xxx.xxx.xxx:9876
rocketmq.producer.group=my-group

//3.使用RocketMQTemplate发送消息
@Autowired
private RocketMQTemplate rocketMQTemplate；
rocketMQTemplate.convertAndSend("topic", "message");
```

#### 消费者

```java
//1.导入依赖，同上

//2.配置文件
#application.properties
rocketmq.name-server=xxx.xxx.xxx:9876
rocketmq.consumer.group=my-group

//3. 信息监听器
@Component
@RocketMQMessageListener(topic="topic", consumerGroup= "consumer-group-1")
public class Consumer implements RocketMQListener<String>{
    @Override
    public void onMessage(String message){
        System.out.print(message);
    }
}
```



## 信息存储

![信息存储流程](https://s2.ax1x.com/2020/03/10/8FArOf.png)

### 存储介质

目前业界比较常用的几款产品均采用的是消息刷盘至所部署的虚拟机/物理机的文件系统来做持久化（刷盘一般可以分为异步刷盘和同步刷盘两种模式）。消息刷盘为消息存储提供了一种高效率、高可靠性和高性能的数据持久化方式。除非部署MQ机器或是本地磁盘挂了，否则一般不会出现无法持久化的故障问题。

### 信息存储和发送

#### 1） 信息存储

目前高性能的磁盘，顺序写速度可以达到600MB/S。但是随机写的速度只有大概100KB/S。RocketMQ的消息用顺序写，保证了消息存储的速度。

#### 2）信息发送

RocketMQ通过使用mmap的方式，可以省去用户态的内存复制，提高速度。这种机制在Java中是通过MappedByteBuffer实现的。

> 这里需要注意的是，采用MappedByteBuffer这种内存映射的方式有几个限制，其中之一是一次只能映射1.5-2G的文件至用户态的虚拟内存，这也是为何RocketMQ默认设置单个CommitLog日志数据文件只有1G的原因了。

### 信息存储结构

RocketMQ消息的存储是由ConsumerQueue和CommitLog配合完成的，消息真正的物理存储文件是CommitLog，ConsumeQueue是消息的逻辑队列，类似数据库的索引文件，存储的是指向物理存储的地址。每个Topic下的每个MessageQueue都有一个对应的ConsumeQueue文件。

![信息存储结构](https://s2.ax1x.com/2020/03/10/8FmGIU.png)

- CommitLog：存储消息的元数据
- ConsumerQueue：存储消息在CommitLog的索引
- IndexFile：为了消息查询提供了一种通过key或时间区间来查询消息的方法，这种通过IndexFile来查找消息的方法不影响发送与消费信息的主流程。



### 刷盘机制

![刷盘机制](https://s2.ax1x.com/2020/03/10/8Fmoo8.png)

#### 配置

通过Broker配置文件里面的`flushDiskType`参数配置的，这个参数可配置为`SYNC_FLUSH`或`ASYNC_FLUSH`中的一种。



## 高可用性机制

Master支持读写，Slave只支持读。

Consumer可连接Master也可连接Slava来消费信息。

### 消息消费高可用

在Consumer配置文件中，并不需要设置从Master读还是从Slave读，当Master不可用或繁忙时，Consumer会被自动切换到Slave读。有了自动切换，当一个Master角色的机器出现故障后，Consumer仍然可以从Slave读取消息。这就达到了消费端的高可用性。

### 消息发送高可用

在创建Topic的时候，把Topic的多个Message Queue创建在多个Broker组上（相同Broker名称，不同BrokerId的机器组成一个Broker组），这样当一个Broker组的Master不可用后，其他组的Master仍然可用，Producer仍然可以发送消息。RocketMQ目前还不支持把Slave自动转换为Master，如果机器资源不足，需要把Salve转Master，则要手动停止Slave的Broker，更改配置文件重启。

### 主从复制

![主从复制](https://s2.ax1x.com/2020/03/10/8FKbWD.png)



## 负载均衡

### Producer负载均衡

![Producer负载均衡](https://s2.ax1x.com/2020/03/10/8FQpu9.png)

### Consumer负载均衡

#### 1） 集群模式

![1](https://s2.ax1x.com/2020/03/10/8FQy2F.png)

![2](https://s2.ax1x.com/2020/03/10/8FQ6v4.png)

需要注意的是，集群模式下，queue都是只允许分配一个实例，这是由于如果多个实例同时消费一个queue消息，由于拉取哪些消息是consumer主动控制的，那样会导致同一个消息在不同实例下被多次消费，所以算法上都是一个queue只分给一个consumer实例，一个consumer实例可以允许同时分到不同的queue。

#### 2） 广播模式

由于广播模式下要求每一条信息都被投递到一个消费者组下面的所有消费者实例，所以也就没有负载均衡的说法。

在实现上其中一个不同就是Consumer分配queue的时候，所以Consumer都分到所有的queue。



## 消息重试

### 顺序消息的重试

对于顺序消息，当消费者消费失败后，消息队列RocketMQ会自动不断进行消息重试（每次间隔1S），这时，应用会出现消息消费被阻塞的去情况。因此，在使用顺序消息时，务必保证应用能够及时监控并处理消费失败的情况，避免阻塞现象发生。

### 无序消息的重试

对于无序消息（普通、定时、延时、事务消息），当消费者消费消息失败时，可以通过设置返回状态达到消息重试的结果。无序消息的重试只针对集群模式，广播模式不提供重试，即消费失败后，失败消息不再重试，继续消费新的消息。

#### 1）重试次数

![重试次数](https://s2.ax1x.com/2020/03/10/8F3k40.png)

#### 2）配置方式

##### 消费失败后，重试配置方式

集群消费模式下，消费失败后期望重试，需要在信息监听器接口中：

- 返回NULL
- 返回Action.Reconsumer（推荐）
- 抛出异常

```java
public class MessageListenerImpl implements MessageListenr{
    @Override
    public Action consume(Message message, ConsumeContext context){
         //获取重试次数
  		System.out.print(message.getReconsumerTimes());  
        return Action.Reconsumer;
    }
}
```

##### 自定义消息最大重试次数

自定义重试间隔时间按照以下策略：

- 最大重试次数小于等于16次，则重试时间间隔同上表
- 最大重试次数大于16次，超过16次的重试时间间隔均为每次2小时

```java
Properties properties = new Properties();
properties.put(PropertyKeyConst.MaxReconsumerTimes,"20");
Consumer consumer = ONSFactory.createConsumer(properties);
```

> 注意：
>
> - 消息最大重试次数的设置对相同Group ID下所以Consumer实例有效，例如只对相同Group ID下两个Consumer实例中的其中一个设置了MaxReconsumerTimes，那么该配置对两个Consumer实例均生效。
> - 配置采用覆盖的方式生效，即最后启动的Consumer实例会覆盖之前启动实例的配置。



## 死信队列

![死信队列](https://s2.ax1x.com/2020/03/10/8FYNWt.md.png)

可通过管理界面查看死信队列，并选择重新发送。



## 消息幂等性

处理方式：

![处理方式](https://s2.ax1x.com/2020/03/10/8FtZnS.png)

