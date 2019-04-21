参考博客：[消息队列之 RabbitMQ](https://www.jianshu.com/p/79ca08116d57)

[springboot(八)：RabbitMQ详解](https://www.cnblogs.com/ityouknow/p/6120544.html)

# 什么叫消息队列

消息（Message）是指在应用间传送的数据。消息可以非常简单，比如只包含文本字符串，也可以更复杂，可能包含嵌入对象。

消息队列（Message Queue）是一种应用间的通信方式，消息发送后可以立即返回，由消息系统来确保消息的可靠传递。消息发布者只管把消息发布到 MQ 中而不用管谁来取，消息使用者只管从 MQ 中取消息而不管是谁发布的。这样发布者和使用者都不用知道对方的存在。

# 为何用消息队列

从上面的描述中可以看出消息队列是一种应用间的异步协作机制，那什么时候需要使用 MQ 呢？

以常见的订单系统为例，用户点击【下单】按钮之后的业务逻辑可能包括：扣减库存、生成相应单据、发红包、发短信通知。在业务发展初期这些逻辑可能放在一起同步执行，随着业务的发展订单量增长，需要提升系统服务的性能，这时可以将一些不需要立即生效的操作拆分出来异步执行，比如发放红包、发短信通知等。这种场景下就可以用 MQ ，在下单的主流程（比如扣减库存、生成相应单据）完成之后发送一条消息到 MQ 让主流程快速完结，而由另外的单独线程拉取MQ的消息（或者由 MQ 推送消息），当发现 MQ 中有发红包或发短信之类的消息时，执行相应的业务逻辑。

以上是用于业务解耦的情况，其它常见场景包括最终一致性、广播、错峰流控等等。



#RabbitMQ

* RabbitMQ 作用：异步处理，业务解耦，错峰流控，消息分发。
* RabbitMQ 主要分为3个部分，生产者，交换机和队列，消费者。

##为什么RabbitMQ？

1. 从社区活跃度高

2. 持久化消息比较：

   `ZeroMq` 不支持，`ActiveMq` 和`RabbitMq` 都支持。持久化消息主要是指我们机器在不可抗力因素等情况下挂掉了，消息不会丢失的机制。

3. 综合技术实现：

   可靠性、灵活的路由、集群、事务、高可用的队列、消息排序、问题追踪、可视化管理工具、插件系统等等。

4. 高并发：

   毋庸置疑，`RabbitMQ` 最高，原因是它的实现语言是天生具备高并发高可用的`erlang` 语言。

5. 比较关注的比较， RabbitMQ 和 Kafka：

   `RabbitMq` 比`Kafka` 成熟，在可用性上，稳定性上，可靠性上，  [RabbitMq](http://www.sojson.com/tag_rabbitmq.html)  胜于  [Kafka](http://www.sojson.com/tag_kafka.html)  （理论上）。

   另外，`Kafka` 的定位主要在日志等方面， 因为`Kafka` 设计的初衷就是处理日志的，可以看做是一个日志（消息）系统一个重要组件，针对性很强，所以 如果业务方面还是建议选择 `RabbitMq` 。

   还有就是，`Kafka` 的性能（吞吐量、`TPS` ）比`RabbitMq` 要高出来很多。

# RabbitMQ 中的概念模型

##### 消息模型

所有 MQ 产品从模型抽象上来说都是一样的过程：
 消费者（consumer）订阅某个队列。生产者（producer）创建消息，然后发布到队列（queue）中，最后将消息发送到监听的消费者。



![img](https:////upload-images.jianshu.io/upload_images/5015984-066ff248d5ff8eed.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/401/format/webp)

消息流

##### RabbitMQ 基本概念

上面只是最简单抽象的描述，具体到 RabbitMQ 则有更详细的概念需要解释。上面介绍过 RabbitMQ 是 AMQP 协议的一个开源实现，所以其内部实际上也是 AMQP 中的基本概念：



![img](https:////upload-images.jianshu.io/upload_images/5015984-367dd717d89ae5db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/554/format/webp)

RabbitMQ 内部结构

1. Message
    消息，消息是不具名的，它由消息头和消息体组成。消息体是不透明的，而消息头则由一系列的可选属性组成，这些属性包括routing-key（路由键）、priority（相对于其他消息的优先权）、delivery-mode（指出该消息可能需要持久性存储）等。
2. Publisher
    消息的生产者，也是一个向交换器发布消息的客户端应用程序。
3. Exchange
    交换器，用来接收生产者发送的消息并将这些消息路由给服务器中的队列。
4. Binding
    绑定，用于消息队列和交换器之间的关联。一个绑定就是基于路由键将交换器和消息队列连接起来的路由规则，所以可以将交换器理解成一个由绑定构成的路由表。
5. Queue
    消息队列，用来保存消息直到发送给消费者。它是消息的容器，也是消息的终点。一个消息可投入一个或多个队列。消息一直在队列里面，等待消费者连接到这个队列将其取走。
6. Connection
    网络连接，比如一个TCP连接。
7. Channel
    信道，多路复用连接中的一条独立的双向数据流通道。信道是建立在真实的TCP连接内地虚拟连接，AMQP 命令都是通过信道发出去的，不管是发布消息、订阅队列还是接收消息，这些动作都是通过信道完成。因为对于操作系统来说建立和销毁 TCP 都是非常昂贵的开销，所以引入了信道的概念，以复用一条 TCP 连接。
8. Consumer
    消息的消费者，表示一个从消息队列中取得消息的客户端应用程序。
9. Virtual Host
    虚拟主机，表示一批交换器、消息队列和相关对象。虚拟主机是共享相同的身份认证和加密环境的独立服务器域。每个 vhost 本质上就是一个 mini 版的 RabbitMQ 服务器，拥有自己的队列、交换器、绑定和权限机制。vhost 是 AMQP 概念的基础，必须在连接时指定，RabbitMQ 默认的 vhost 是 / 。
10. Broker
     表示消息队列服务器实体。

##### AMQP 中的消息路由

AMQP 中消息的路由过程和 Java 开发者熟悉的 JMS 存在一些差别，AMQP 中增加了 Exchange 和 Binding 的角色。生产者把消息发布到 Exchange 上，消息最终到达队列并被消费者接收，而 Binding 决定交换器的消息应该发送到那个队列。



![img](https:////upload-images.jianshu.io/upload_images/5015984-7fd73af768f28704.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/484/format/webp)

AMQP 的消息路由过程

##### Exchange 类型

Exchange分发消息时根据类型的不同分发策略有区别，目前共四种类型：direct、fanout、topic、headers 。headers 匹配 AMQP 消息的 header 而不是路由键，此外 headers 交换器和 direct 交换器完全一致，但性能差很多，目前几乎用不到了，所以直接看另外三种类型：

1. direct



   ![img](https:////upload-images.jianshu.io/upload_images/5015984-13db639d2c22f2aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/385/format/webp)

   direct 交换器

   消息中的路由键（routing key）如果和 Binding 中的 binding key 一致， 交换器就将消息发到对应的队列中。路由键与队列名完全匹配，如果一个队列绑定到交换机要求路由键为“dog”，则只转发 routing key 标记为“dog”的消息，不会转发“dog.puppy”，也不会转发“dog.guard”等等。它是完全匹配、单播的模式。

2. fanout



   ![img](https:////upload-images.jianshu.io/upload_images/5015984-2f509b7f34c47170.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/463/format/webp)

   fanout 交换器

   每个发到 fanout 类型交换器的消息都会分到所有绑定的队列上去。fanout 交换器不处理路由键，只是简单的将队列绑定到交换器上，每个发送到交换器的消息都会被转发到与该交换器绑定的所有队列上。很像子网广播，每台子网内的主机都获得了一份复制的消息。fanout 类型转发消息是最快的。

3. **topic**



   ![img](https:////upload-images.jianshu.io/upload_images/5015984-275ea009bdf806a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/558/format/webp)

   topic 交换器

    topic 交换器通过模式匹配分配消息的路由键属性，将路由键和某个模式进行匹配，此时队列需要绑定到一个模式上。它将路由键和绑定键的字符串切分成单词，这些单词之间用点隔开。它同样也会识别两个通配符：符号“`#`”和符号“`*`”。`#`匹配0个或多个单词，“`*`”匹配不多不少一个单词。

#Springboot+Rabbit

### 简单使用

1、配置pom包，主要是添加spring-boot-starter-amqp的支持

```
<dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

2、配置文件

配置rabbitmq的安装地址、端口以及账户信息

```
spring.application.name=spirng-boot-rabbitmq

spring.rabbitmq.host=192.168.0.86
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=123456
```

3、队列配置

```
@Configuration
public class RabbitConfig {

    @Bean
    public Queue Queue() {
        return new Queue("hello");
    }

}
```

3、发送者

rabbitTemplate是springboot 提供的默认实现

```
public class HelloSender {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send() {
        String context = "hello " + new Date();
        System.out.println("Sender : " + context);
        this.rabbitTemplate.convertAndSend("hello", context);
    }

}
```

4、接收者

```
@Component
@RabbitListener(queues = "hello")
public class HelloReceiver {

    @RabbitHandler
    public void process(String hello) {
        System.out.println("Receiver  : " + hello);
    }

}
```

5、测试

```
@RunWith(SpringRunner.class)
@SpringBootTest
public class RabbitMqHelloTest {

    @Autowired
    private HelloSender helloSender;

    @Test
    public void hello() throws Exception {
        helloSender.send();
    }

}
```

> 注意，发送者和接收者的queue name必须一致，不然不能接收

具体其他exchange模式的代码请看最上面的博客地址

##可能出现的问题：

###消息持久化

在生产环境中，我们需要考虑万一生产者挂了，消费者挂了，或者 rabbitmq 挂了怎么样。一般来说，如果生产者挂了或者消费者挂了，其实是没有影响，因为消息就在队列里面。那么万一 rabbitmq 挂了，之前在队列里面的消息怎么办，其实可以做消息持久化，RabbitMQ 会把信息保存在磁盘上。

做法是可以先从 Connection 对象中拿到一个 Channel 信道对象，然后再可以通过该对象设置消息持久化。



参考：[RabbitMq持久化机制](https://blog.csdn.net/u013256816/article/details/60875666/)

###生产者或者消费者断线重连

这里 Spring 有自动重连机制。

###ACK 确认机制

每个Consumer可能需要一段时间才能处理完收到的数据。如果在这个过程中，Consumer出错了，异常退出了，而数据还没有处理完成，那么 非常不幸，这段数据就丢失了。因为我们采用no-ack的方式进行确认，也就是说，每次Consumer接到数据后，而不管是否处理完 成，RabbitMQ Server会立即把这个Message标记为完成，然后从queue中删除了。

如果一个Consumer异常退出了，它处理的数据能够被另外的Consumer处理，这样数据在这种情况下就不会丢失了（注意是这种情况下）。
为了保证数据不被丢失，RabbitMQ支持消息确认机制，即acknowledgments。为了保证数据能被正确处理而不仅仅是被Consumer收到，那么我们不能采用autoack。而应该是在处理完数据后发送手动ack。

在处理数据后发送的ack，就是告诉RabbitMQ数据已经被接收，处理完成，RabbitMQ可以去安全的删除它了。
如果Consumer退出了但是没有发送ack，那么RabbitMQ就会把这个Message发送到下一个Consumer。这样就保证了在Consumer异常退出的情况下数据也不会丢失。

参考：[RabbitMQ入门教程信息确认ACK](https://blog.csdn.net/vbirdbest/article/details/78699913)

##Redis事务+confirm模式

https://blog.csdn.net/u013256816/article/details/55515234

