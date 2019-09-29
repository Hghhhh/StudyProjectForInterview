[Vert.x中文文档](http://vertxchina.github.io/vertx-translation-chinese/start/Start.html)笔记

# FAQ

此处会列出常见的关于Vert.x 各个组件的常见问题以及相应的注意事项和解决方案。

### 问：怎样正确理解Vert.x机制？

答：Vert.x其实就是建立了一个Verticle内部的线程安全机制，让用户可以排除多线程并发冲突的干扰，专注于业务逻辑上的实现，用了Vert.x，您就不用操心多线程和并发的问题了。Verticle内部代码，除非声明Verticle是Worker Verticle，否则Verticle内部环境全部都是线程安全的，不会出现多个线程同时访问同一个Verticle内部代码的情况。

> 请注意：*一般情况下，用了Vert.x的Verticle之后，原则上synchronized，Lock，volatile，static对象，java.util.concurrent, HashTable, Vector, Thread, Runnable, Callable, Executor, Task, ExecutorService等这些并发和线程相关的东西就不再需要使用了，可以由Verticle全面接管，如果您不得不在Vert.x代码中使用上述内容，则多少暗示着您的设计或者使用Vert.x的姿势出现了问题，建议再斟酌商榷一下。*

### 问：Verticle对象和处理器（Handler）是什么关系？Vert.x如何保证Verticle内部线程安全？

答：Verticle对象往往包含有一个或者多个处理器（Handler），在Java代码中，后者经常是以Lambda也就是匿名函数的形式出现，比如：

```java
vertx.createHttpServer().requestHandler(req->{
    //blablabla            
}).listen(8080);
```

这里的Lambda/匿名函数/req->{//blablabla}就是一个处理器（Handler），在随后的例子中，我们用1stHandler以及2ndHandler来指代具体的匿名函数，让代码更加清晰明了，放在Verticle中类似：

```java
public class MyVerticle extends AbstractVerticle {
    public void start() throws Exception {
        vertx.createHttpServer().requestHandler(1stHandler).listen(8080);
    vertx.createHttpServer().requestHandler(2ndHandler).listen(8081);
    }
}
```

Java中，Lambda本身也是一个对象，是一个@FunctionalInterface的对象，Verticle对象中包含了一个或者多个处理器（Handler）对象，比如上述例子中MyVerticle中就包含有两个handler。在Vert.x中，完成Verticle的部署之后，真正调用处理逻辑的入口往往是处理器（Handler），Vert.x保证同一个普通Verticle（也就是EventLoop Verticle，非Worker Verticle）内部的所有处理器（Handler）都只会由同一个EventLoop线程调用，由此保证Verticle内部的线程安全。所以我们可以放心地在Verticle内部声明各种线程不安全的属性变量，并在handler中分享他们，比如：

```java
public class MyVerticle extends AbstractVerticle {
    int i = 0;//属性变量

    public void start() throws Exception {
        vertx.createHttpServer().requestHandler(req->{
            i++;
        req.response().end();//要关闭请求，否则连接很快会被占满
        }).listen(8080);

        vertx.createHttpServer().requestHandler(req->{
            System.out.println(i);
        req.response().end(""+i);
        }).listen(8081);
    }
}
```

访问 [http://localhost:8080](http://localhost:8080/) 就会使计数器加1，访问 [http://localhost:8081](http://localhost:8081/) 将会看到具体的计数。同理，也可以将i替换成HashMap等线程不安全对象，不需要使用ConcurrentHashMap或HashTable，可在Verticle内部安全使用。

Vert.x的Handler内部是atomic/原子操作，Verticle内部是thread safe/线程安全的，Verticle之间传递的数据是immutable/不可改变的。

一个vert.x实例/进程内有多个Eventloop和Worker线程，每个线程会部署多个Verticle对象并对应执行Verticle内的Handler，每个Verticle内有多个Handler，普通Verticle会跟Eventloop绑定，而Worker Verticle对象则会被Worker线程所共享，会依次顺序访问，但不会并发同时访问，如果声明为Multiple Threaded Worker Verticle则没有此限制，需要开发者手工处理并发冲突，我们并不推荐这类操作。

### 问：什么是显著执行时间？什么是异步？如何正确理解文档中说的不要阻塞Eventloop？

答：初次接触Vert.x的开发人员往往会在异步，显著执行时间，阻塞等概念理解上遇到一定的困难，在此一并做个解释和说明。

*注意：我们会尽量将原理讲得通俗易懂，但还是要求读者具备有线程，进程，内存，CPU，操作系统，数据库，算法等专业基础知识。*

计算机本质上是计算的机器，任何一个指令的执行，都需要一定的时间予以完成，这个时间可能长可能短，而这些指令的执行，都会分配给一个具体的线程，并在线程中执行完成。在Vert.x中，黄金原则是不阻塞Eventloop线程，而Eventloop顾名思义，是一个事件的循环，可以认为是一个在Vert.x生命周期内，不停轮询事件是否发生，并将发生的事件交给Handler予以处理的无限调度循环线程。那么为了不停地检查事件是否发生，该线程需要在短时间内完成一个调度循环。如果Eventloop在完成一个调度循环的时间过长，就有可能导致新发生的事件得不到及时的处理，进而导致单次事件响应时间过长，影响客户体验。

为了在短时间内完成调度循环，就需要用户正确估算出，哪些程序代码会相对长时间地占用Eventloop线程的执行时间，然后将该部分代码的执行交由**其它线程**去处理。值得注意的是，这里所说的其它线程，可能是内核线程，也就是操作系统的线程，也有可能是用户线程，用户线程中包括了其它应用程序的线程，也就是其它进程中的线程，或者是我们用户自定义的线程。

一个简单粗暴的判断标准：任何涉及到IO操作的代码，都可以认为是可能造成阻塞的代码，**纯粹内存操作的代码，只要执行时间没有明显超长（例如执行循环几万次的处理便可认为是执行时间超长），都可以认为是非阻塞代码**。

我们来看一个简单的例子：某个程序要求，当前线程发送一个请求给网络上另外一个服务器，然后获取到结果之后作出相应的处理。那么此时有一个明显的IO操作，就是发送网络请求并等待对方返回结果。因为网络的速度要远远慢于内存处理的速度，所以此时的操作便是非纯粹内存操作，就有可能造成线程的阻塞，那么此时应该将这个操作交给其它线程予以处理，在处理完成之前，释放当前线程，等处理完成之后，再由当前线程执行回调函数。Vert.x自带的网络客户端（NetClient，HttpClient等）已经帮您包装好了这部分逻辑代码，直接使用NetClient等客户端，Vert.x就会将发送请求并等待返回结果这部分代码交给另外一个线程予以执行，此时这另外一个线程是内核线程，这部分的异步处理由JVM以及操作系统完成，您不需要自己定义一个线程并执行。类似的，数据库的处理同样涉及IO操作，所以Vert.x自带的JDBC客户端（JDBCClient）会帮您完成这部分的封装，您只需要直接调用JDBCClient的各种API便可完成操作，此时有可能是其它进程中的线程，比如PostgresQL会使用进程池来建立连接池，但是该线程亦不需要您去创建，Vert.x帮您完成了这些操作。类似的，硬盘上的操作，比如文件系统的API，也有可能造成阻塞，所以Vert.x的文件系统API提供了非阻塞API，但是值得说明的是，硬盘上的操作，如果只是少量操作，执行时间上也不会明显超长，所以Vert.x同时提供了硬盘操作的阻塞和非阻塞两种API。最后我们来看一个纯粹内存操作同时又是阻塞的例子。

假设我们拿到两个数万个节点的链表（LinkedList），要求删除两个链表的交集，那么在没有任何算法优化的前提下，该操作的时间复杂度是O(n^2)，又因为内存中该链表节点数庞大，多达数万个节点，所以如果在Eventloop中执行该操作，将有可能使得执行时间超长，此时需要将这部分代码交由其它线程予以执行，Vert.x提供了除了Eventloop线程池以外的线程池，名曰Worker线程池。此时就需要用户自行将该部分代码包装成Worker线程执行的代码，并交给Worker线程予以执行，执行完成之后再由Eventloop线程执行回调函数处理其结果。*注意：Vert.x中将代码交给Worker线程执行的方式有两种，一种是通过executeBlocking函数包装，另外一种是写入Worker Verticle中。*

### 问：为什么Verticle之间传递的消息要求是immutable（不可变）的？

答：因为immutable（不可变的）的东西线程安全，可以被多个线程安全地并发访问，线程在使用的时候拷贝一份也不会有并发问题，Java里面字符串（String）对象是immutable（不可变）的，所以缺省情况下事件总线（eventbus）上传递的消息是字符串，vert.x也实现了事件总线上消息类型的编解码器，除了字符串以外，还支持少量的其它类型，比如原始数据类型及其包装类，比如字节流（byte[]），比如Json对象（JsonObject, JsonArray）还有Buffer，这些对象在传递过程中是不可变的。

Vert.x线程模型保证Verticle内部代码线程安全，同时要求在Verticle之间传递的消息是不可变的，通过此方法保证Verticle之间传递的消息也是线程安全的，从而进一步保证Vert.x内部整体是线程安全的，从而将开发人员从繁琐的，容易错的各种多线程并发问题中解脱出来。

### 问：我刚拿到一个第三方类库（lib/jars），怎么判断这个类库中的方法是异步还是同步的？有没有简单粗暴的方法可以一眼看出来？

答：严格说来，要认真看代码文档，比如JavaDoc，来判断方法是异步还是同步的，如果文档中没有明确说明，**则默认是同步的**，异步API往往出现在IO相关的方法中，所谓IO一般认为是涉及到硬盘（比如在硬盘上存取文件），网络（通过网络发送一个请求并获取相应结果），其它进程的操作（操作数据库）等，一般纯粹的，本进程内的内存操作并不被认为是涉及IO操作，这个时候将方法或API制作成异步的并无实际意义，所以一般非IO相关操作都被认为是同步的，同时并不会占用显著执行时间，所以不会阻塞线程。

> 请注意：*Vert.x中绝大多数涉及IO操作除非有明确说明，例如以Sync后缀命名的方法，否则均是异步的，另外在Vert.x中，EventBus的相关操作也被认为是涉及IO操作。*

有一个粗暴简单的判断同步还是异步的方法，就看给出的API或者方法中，是否有回调函数，如果有回调函数，且这个回调函数的参数是AsyncResult，则可以判断该API或方法是异步的。

### 问：Vert.x中各种Client该如何正确使用，用完是否需要关闭？

答：Vert.x中提供了各种预设客户端，例如HttpClient，JDBCClient，WebClient，MongoClient等，一般情况下，建议将客户端与Verticle对象绑定，一个Verticle对象内保留一个特定客户端的引用，并在start方法中将其实例化，这样Vert.x会在执行deployVerticle的时候执行start方法，实例化并保存该对象，在Verticle生命周期内，不需要频繁创建和关闭同类型的客户端，建议在Verticle的生命周期内对于特定领域，只创建一个客户端并复用该客户端，例如：

```java
import io.vertx.core.AbstractVerticle;
import io.vertx.core.http.HttpClient;
import io.vertx.core.json.JsonObject;
import io.vertx.ext.jdbc.JDBCClient;
import io.vertx.ext.web.client.WebClient;

/**
 * Created by chengen on 21/04/2017.
 */
public class TheVerticle extends AbstractVerticle{
    //将客户端对象与Verticle对象绑定，这里选取了三种不同的客户端作为示范
    HttpClient httpClient;
    WebClient webClient;
    JDBCClient jdbcClient;

    public void start(){
          //创建客户端
        httpClient = vertx.createHttpClient();

        webClient = WebClient.wrap(httpClient);

        JsonObject config = new JsonObject()
                .put("jdbcUrl", "...")
                .put("maximumPoolSize", 30)
                .put("username", "db user name")
                .put("password", "***")
                .put("provider_class", "...");

        jdbcClient = JDBCClient.createShared(vertx, config);

          //using clients 
    }
}
```

每次使用客户端完之后，**无需**调用client.close();方法关闭客户端，频繁创建销毁客户端会在一定程度上消耗系统资源，降低性能，同时增加开发人员的负担，Vert.x提供客户端的目的就在于复用连接以减少资源消耗，提升性能，同时简化代码，减轻开发人员的负担。如您关闭客户端，在下一次使用该客户端的时候，需要重新创建客户端。

### 问：Vert.x中Future该如何正确使用，怎样规避回调地狱？

答：Future对象提供了一种异步结果的包装，用户可使用Future类中的setHandler方法来保存回调函数，然后在原先使用该回调函数的地方用completer方法予以填充，这样便可将回调函数从原参数中取出，以减少回调缩进，从而规避回调地狱，来看一个简单的例子：

```java
//未使用future时，回调函数嵌在send方法内部，以匿名函数的形式作为send的参数
vertx.eventBus().send("address","message", asyncResult->{
    System.out.println(asyncResult.result().body()); 
});
```

以上是未使用future时的代码，以下是使用future改造后的代码：

```java
Future<Message<String>> future = Future.future();
//将回调函数存入future中，从而实现代码的扁平化
future.setHandler(asyncResult -> {
    System.out.println(asyncResult.result().body());
});
//使用future之后，用completer方法填充参数
vertx.eventBus().send("address","message", future.completer());
```

一个复杂一点的例子：

```java
//以下程序先向address1发送一个message，然后等address1回复之后，将address1的回复消息发送给address2，最后将address2的回复打印到控制台上
vertx.eventBus().send("address1","message", asyncResult->{
    vertx.eventBus().send("address2", asyncResult.result().body(), asyncResult2->{
        System.out.println(asyncResult2.result().body());
    });
});
```

可以看到此时为了保证顺序结构，产生了两层缩进，回调金字塔开始形成，多次缩进之后便会出现所谓的回调地狱，这便是异步开发中为了保证顺序所可能会遇到的问题，那么我们可以通过以下方式解决代码过多缩进的问题：

```java
Future<Message<String>> future1 = Future.future();
Future<Message<String>> future2 = Future.future();

future1.setHandler(asyncResult -> {
   vertx.eventBus().send("address2", asyncResult.result().body(), future2);
});

future2.setHandler(asyncResult-> {
    System.out.println(asyncResult.result().body());
});

vertx.eventBus().send("address1","message", future1);
```

可以看到，使用了future之后，原先的两层缩进被抽取出来，变成了最多单层的缩进，从而使得代码可读性更强，更加美观。

值得注意的是，示范代码可能会抛出NullPointerException，因为当操作失败时，asyncResult.result()方法返回值为null，此时调用.body()方法会抛出空指针异常，在生产环境中正确写法应该是：

```
if(asyncResult.succeeded()){
    System.out.println(asyncResult.result().body());
}else if(asyncResult.failed()){
    System.out.println(asyncResult.cause());
}else{
    System.out.println("Why am I here?");
}
```

示范代码中为了解释future的用法简写了代码。

#### 使用 Future 来包装异步代码块 (3.4.0+)

早期版本的 Future，针对每一个异步的过程，需要在代码中声明中间变量 future 来对应异步回调，例如上文例子中的：

```java
Future<Message<String>> future1 = Future.future();
Future<Message<String>> future2 = Future.future();
```

在最新的版本中(3.4.0+)，可以通过 [Future.future](http://vertx.io/docs/apidocs/io/vertx/core/Future.html#future-io.vertx.core.Handler-) 方法以函数式的方式直接用 Future 来包装一个异步的代码块，例如：

```java
Future.<Message<String>>future((future) ->
  vertx.eventBus().send("address1", "someMessage", future)
);
```

该方法的输入参数是一个 `Function`，该 `Function` 会以一个新的 `Future` 实例为参数被调用。由于 `Future` 自身实现了 `Handler<AsyncResult>`，因此你可以将它直接作为回调的 `Handler` 传入到异步方法里。该方法的返回值是提供给异步调用使用的 `Future` 实例。由此可以避免为嵌套的多个异步操作定义不同的 Future 变量，使代码更为简洁。

以下两种写法是等效的：

```java
vertx.eventBus().send("address","message", future.completer());
//或
vertx.eventBus().send("address","message", future);
```

复杂一点的例子：

```java
future.compose(message ->
  Future.<Message<String>>future(f ->
    vertx.eventBus().send("address", message.body(), f)
  )
);
//以上和以下两种写法是等效的
future.compose(message ->{
  Future<Message<String>> f = Future.<Message<String>>future();//可简写为Future<Message<String>> f = Future.future();
  vertx.eventBus().send("address",message.body(),f.completer());//可简写为vertx.eventBus().send("address",message.body(),f);
  return f;
});
```

#### 使用组合来实现链式调用 (3.4.0+)

Future 接口提供了 `compose` 方法来链式地组合多个异步操作。在介绍这个方法的用途时，我们先考虑一个传统的同步操作和异常处理的方式。**假设**我们有一个同步的方法 `send` 会抛出一些异常（注意，以下代码只是同步代码的示例，和 vert.x 无关）：

```java
try {
    String msg1 = send('address1', 'message');
    String msg2 = send('address2', msg1);
    String msg3 = send('address3', msg2);
    //deal with result
} catch (Exception e) {
    //deal with exception
}
```

在这个例子里，如果任意一个 `send` 方法抛出异常，则会立即跳转到 catch 的代码块中。

下面我们将这个 `send` 方法通过 `compose` 替换为 vert.x 的异步版本，如下：

```java
Future.<Message<String>>future(f ->
  vertx.eventBus().send("address1", "message", f)
).compose((msg) ->
  Future.<Message<String>>future(f ->
    vertx.eventBus().send("address2", msg.body(), f)
  )
).compose((msg) ->
  Future.<Message<String>>future(f ->
    vertx.eventBus().send("address3", msg.body(), f)
  )
).setHandler((res) -> {
  if (res.failed()) {
    //deal with exception
    return;
  }
  //deal with the result
});
```

每一个 `compose` 方法需要传入一个 `Function`，`Function` 的输入是前一个异步过程的返回值（此处的返回值不是 AsyncResult，而是具体的返回内容），需要返回一个新的需要链接的 `Future`。 该 `Function`方法当且仅当前一个异步流程执行成功时才会被调用。

上述的例子描述了这样一个流程：

1. 首先通过 eventbus 发送消息 `message` 到 `address1`
2. 如果第一步成功，则发送第一步的消息的返回值到 `address2`
3. 如果第二步成功，则发送第二部的消息的返回值到 `address3`
4. **如果以上任何一步失败**，则不会继续执行下一个异步流程，直接执行最终的 Handler ，并且 `res.succeeded()` 为 `false`，可以通过 `res.cause()` 来获得异常对象
5. 如果以上三步全都成功，则同样执行 Handler，`res.succeeded()` 为 `true`，可以通过 `res.result()`获取最后一步的结果。

通过 `compose` 方法来组织代码最大的价值在于可以 **让异步代码的执行顺序和代码的编写顺序看起来一致**，并在任何一步抛出异常时直接退出到最后一个 handler 来处理，**不需要针对每一个异步操作都编写异常处理的逻辑**。这对于编写复杂的异步流程时是非常有用的。

[Future.compose()](http://vertx.io/docs/apidocs/io/vertx/core/Future.html#compose-java.util.function.Function-) 这个方法的行为现在非常接近于 JDK1.8 提供的 [CompletableFuture.thenCompose()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCompose-java.util.function.Function-)，也很接近于 EcmaScript6 的 [Promise API](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise) 的的接口约定，其实都是关于 Promise 模式的应用。关于更多 Promise 模式的信息还可以参考这里 https://en.wikipedia.org/wiki/Futures_and_promises

### 问：Vert.x Web中如何实现Servlet和JSP中的forward和redirect方法？我想将根目录/自动映射到index.html文件该如何做？

答：需要用到其它的handler予以配合，例如我们想将URI：/static/index.html定位到/webroot/index.html文件，则需定义而StaticHandler：

```java
router.route("/static/*").handler(StaticHandler.create());
```

随后便可将根路径/映射为/static/index.html，从而映射到文件夹webroot下的index.html文件。

forward方法用reroute方法：

```java
router.route("/").handler(ctx->ctx.reroute(HttpMethod.GET,"/static/index.html"));//加上HttpMethod.GET参数原因见下文
router.route("/static/*").handler(StaticHandler.create());
```

reroute方法将会保留原Http方法，而StaticHandler只接受GET和HEAD方法，所以如果希望将POST方法reroute到一个静态文件，则需要改变Http方法：

```java
router.post("/").handler(ctx->ctx.reroute(HttpMethod.GET,"/static/index.html"));
router.route("/static/*").handler(StaticHandler.create());
```

redirect方法本质上是设置响应状态码为302，同时设置响应头Location值，根据该原理便可实现：

```java
router.route("/").handler(ctx->ctx.response().putHeader("Location", "/static/index.html").setStatusCode(302).end());
router.route("/static/*").handler(StaticHandler.create());
```

### 问：我之前有过Spring，Akka，Node.js或Go的经验，请问Vert.x的概念有我熟悉的吗？

答：严格说来，不同框架和语言之间的概念无法一一对应，但如果我们不那么严格地去深究细节，Vert.x定义的概念可以从其它框架以及语言中找到一些痕迹，以下是Vert.x中定义概念跟其它框架和语言定义概念的比较，同一行中的概念可被认为是相似的：

|              Vert.x               | Akka  |     Spring      |          EJB           | Node.js |    Go     |
| :-------------------------------: | :---: | :-------------: | :--------------------: | :-----: | :-------: |
|         Standard Verticle         |   -   |        -        |           -            | Reactor |     -     |
|          Worker Verticle          |   -   |        -        | Stateless Session Bean |    -    |     -     |
| Multiple Threaded Worker Verticle |   -   | Bean(Singleton) |           -            |    -    |     -     |
|              Handler              | Actor |        -        |           -            |    -    |     -     |
|             Coroutine             |   -   |        -        |           -            |    -    | Goroutine |



# Vert.x Core

## 组件介绍

**Vert.x 的核心 Java API 被我们称为 Vert.x Core**。

Vert.x Core 提供了下列功能:

- 编写 TCP 客户端和服务端
- 编写支持 WebSocket 的 HTTP 客户端和服务端
- 事件总线
- 共享数据 —— 本地的Map和分布式集群Map
- 周期性、延迟性动作
- 部署和撤销 Verticle 实例
- 数据报套接字
- DNS客户端
- 文件系统访问
- 高可用性
- 集群

## 故事从 Vert.x 开始

除非您拿到 [`Vertx`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html) 对象，否则在Vert.x领域中您做不了太多的事情。它是 Vert.x 的控制中心，也是您做几乎一切事情的基础，包括创建客户端和服务器、获取事件总线的引用、设置定时器等等。

那么如何获取它的实例呢？

如果您用嵌入方式使用Vert.x，可通过以下代码创建实例：

```java
Vertx vertx = Vertx.vertx();
```

> 请注意：*大部分应用将只会需要一个Vert.x实例，但如果您有需要也可创建多个Vert.x实例，如：隔离的事件总线或不同组的客户端和服务器。*

### 创建 Vertx 对象时指定配置项

如果缺省的配置不适合您，可在创建 `Vertx` 对象的同时指定配置项：

```java
Vertx vertx = Vertx.vertx(new VertxOptions().setWorkerPoolSize(40));
```

[`VertxOptions`](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html)对象有很多配置，包括集群、高可用、池大小等。在Javadoc中描述了所有配置的细节。

### 创建集群模式的 Vert.x 对象

如果您想创建一个集群模式的 `Vertx` 对象（参考 [Event Bus](http://vertxchina.github.io/vertx-translation-chinese/core/Core.html#event_bus) 章节了解更多事件总线集群细节），那么通常情况下您将需要使用另一种异步的方式来创建 `Vertx` 对象。

这是因为让不同的 Vert.x 实例组成一个集群需要一些时间（也许是几秒钟）。在这段时间内，我们不想去阻塞调用线程，所以我们将结果异步返回给您。

> 译者注：这里给个示例：
>
> ```java
> // 注意要添加对应的集群管理器依赖，详情见集群管理器章节
> VertxOptions options = new VertxOptions();
> Vertx.clusteredVertx(options, res -> {
>   if (res.succeeded()) {
>     Vertx vertx = res.result(); // 获取到了集群模式下的 Vertx 对象
>     // 做一些其他的事情
>   } else {
>     // 获取失败，可能是集群管理器出现了问题
>   }
> });
> ```

## 是流式的吗？

您也许注意到前边的例子里使用了一个流式（Fluent）的API。

一个流式的API表示将多个方法的调用链在一起。例如：

```java
request.response().putHeader("Content-Type", "text/plain").write("some text").end();
```

这是贯穿 Vert.x API 中的一个通用模式，所以请适应这种代码风格。

## Don’t call us, we’ll call you

Vert.x 的 API 大部分都是事件驱动的。这意味着当您感兴趣的事情发生时，它会以事件的形式发送给您。

以下是一些事件的例子：

- 触发一个计时器
- Socket 收到了一些数据
- 从磁盘中读取了一些数据
- 发生了一个异常
- HTTP 服务器收到了一个请求

您提供处理器给Vert.x API来处理事件。例如每隔一秒发送一个事件的计时器：

```java
vertx.setPeriodic(1000, id -> {
  // This handler will get called every second
  // 这个处理器将会每隔一秒被调用一次
  System.out.println("timer fired!");
});
```

又或者收到一个HTTP请求：

```java
server.requestHandler(request -> {
  // This handler will be called every time an HTTP request is received at the server
  // 服务器每次收到一个HTTP请求时这个处理器将被调用
  request.response().end("hello world!");
});
```

稍后当Vert.x有一个事件要传给您的处理器时，它会 **异步地** 调用这个处理器。

由此引入了下面一些Vert.x中的重要概念。

## 不要阻塞我！

除了很少的特例（如以 "Sync" 结尾的某些文件系统操作），Vert.x中的所有API都不会阻塞调用线程。

如果可以立即提供结果，它将立即返回，否则您需要提供一个处理器（`Handler`）来接收稍后回调的事件。

因为Vert.x API不会阻塞线程，所以通过Vert.x您可以只使用少量的线程来处理大量的并发。

## Reactor 模式和 Multi-Reactor 模式

我们前边提过 Vert.x 的 API 都是事件驱动的，当有事件时 Vert.x 会将事件传给处理器来处理。

在多数情况下，Vert.x使用被称为 **Event Loop** 的线程来调用您的处理器。

由于Vert.x或应用程序的代码块中没有阻塞，**Event Loop** 可以在事件到达时快速地分发到不同的处理器中。

由于没有阻塞，Event Loop 可在短时间内分发大量的事件。例如，一个单独的 **Event Loop** 可以非常迅速地处理数千个 HTTP 请求。

我们称之为 [Reactor 模式](https://en.wikipedia.org/wiki/Reactor_pattern)（译者注：Reactor Pattern 翻译成了[反应器模式](https://zh.wikipedia.org/wiki/反应器模式)）。

每个 `Vertx` 实例维护的是 **多个Event Loop 线程**。默认情况下，我们会根据机器上可用的核数量来设置 Event Loop 的数量，您亦可自行设置。

这意味着 Vertx 进程能够在您的服务器上扩展，与 Node.js 不同。

我们将这种模式称为 **Multi-Reactor 模式**（多反应器模式），区别于单线程的 Reactor 模式（反应器模式）。

>  请注意：*即使一个 Vertx 实例维护了多个 Event Loop，任何一个特定的处理器永远不会被并发执行。大部分情况下（除了 Worker Verticle 以外）它们总是在同一个 Event Loop 线程中被调用。*



## 黄金法则：不要阻塞Event Loop

 Event Loop 在被阻塞时就不能做任何事情。如果您阻塞了 `Vertx` 实例中的所有 Event Loop，那么您的应用就会完全停止！

所以不要这样做！**这是一个警告!**

这些阻塞做法包括：

- `Thead.sleep()`
- 等待一个锁
- 等待一个互斥信号或监视器（例如同步的代码块）
- 执行一个长时间数据库操作并等待其结果
- 执行一个复杂的计算，占用了可感知的时长
- 在循环语句中长时间逗留

如果您的应用程序没有响应，可能这是一个迹象，表明您在某个地方阻塞了Event Loop。为了帮助您诊断类似问题，若 Vert.x 检测到 Event Loop 有一段时间没有响应，将会自动记录这种警告。若您在日志中看到类似警告，那么您需要检查您的代码。比如：

```
Thread vertx-eventloop-thread-3 has been blocked for 20458 ms
```

Vert.x 还将提供堆栈跟踪，以精确定位发生阻塞的位置。

如果想关闭这些警告或更改设置，您可以在创建 `Vertx` 对象之前在 [`VertxOptions`](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html) 中完成此操作。

## 运行阻塞式代码

如之前讨论，您不能在 Event Loop 中直接调用阻塞式操作，因为这样做会阻止 Event Loop 执行其他有用的任务。那您该怎么做？

可以通过调用 [`executeBlocking`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#executeBlocking-io.vertx.core.Handler-boolean-io.vertx.core.Handler-) 方法来指定阻塞式代码的执行以及阻塞式代码执行后处理结果的异步回调。

```java
vertx.executeBlocking(future -> {
  // 调用一些需要耗费显著执行时间返回结果的阻塞式API
  String result = someAPI.blockingMethod("hello");
  future.complete(result);
}, res -> {
  System.out.println("The result is: " + res.result());
});
```

默认情况下，如果 `executeBlocking` 在同一个上下文环境中（如：同一个 Verticle 实例）被调用了多次，那么这些不同的 `executeBlocking` 代码块会 **顺序执行**（一个接一个）。

若您不需要关心您调用 [`executeBlocking`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#executeBlocking-io.vertx.core.Handler-boolean-io.vertx.core.Handler-) 的顺序，可以将 `ordered` 参数的值设为 `false`。这样任何 `executeBlocking` 都会在 Worker Pool 中并行执行。

另外一种运行阻塞式代码的方法是使用 [Worker Verticle](http://vertx.io/docs/vertx-core/java/#worker_verticles)。

一个 Worker Verticle 始终会使用 Worker Pool 中的某个线程来执行。

默认的阻塞式代码会在 Vert.x 的 Worker Pool 中执行，通过 [`setWorkerPoolSize`](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#setWorkerPoolSize-int-) 配置。

可以为不同的用途创建不同的池：

```java
WorkerExecutor executor = vertx.createSharedWorkerExecutor("my-worker-pool");
executor.executeBlocking(future -> {
  // 调用一些需要耗费显著执行时间返回结果的阻塞式API
  String result = someAPI.blockingMethod("hello");
  future.complete(result);
}, res -> {
  System.out.println("The result is: " + res.result());
});
```

Worker Executor 在不需要的时候必须被关闭：

```java
executor.close();
```

当使用同一个名字创建了许多 worker 时，它们将共享同一个 pool。当所有的 worker executor 调用了 `close` 方法被关闭过后，对应的 worker pool 会被销毁。

如果 Worker Executor 在 Verticle 中创建，那么 Verticle 实例销毁的同时 Vert.x 将会自动关闭这个 Worker Executor。

Worker Executor 可以在创建的时候配置：

```java
int poolSize = 10;

// 2分钟
long maxExecuteTime = 120000;

WorkerExecutor executor = vertx.createSharedWorkerExecutor("my-worker-pool", poolSize, maxExecuteTime);
```

> 请注意：*这个配置信息在 worker pool 创建的时候设置。*

## 异步协调

Vert.x 中的 [`Future`](http://vertx.io/docs/apidocs/io/vertx/core/Future.html) 可以用来协调多个异步操作的结果。它支持并发组合（并行执行多个异步调用）和顺序组合（依次执行异步调用）。

> 译者注：Vert.x 中的 [`Future`](http://vertx.io/docs/apidocs/io/vertx/core/Future.html) 即异步开发模式中的 Future/Promise 模式的实现。

### 并发合并

[`CompositeFuture.all`](http://vertx.io/docs/apidocs/io/vertx/core/CompositeFuture.html#all-io.vertx.core.Future-io.vertx.core.Future-) 方法接受多个 `Future` 对象作为参数（最多6个，或者传入 `List`）。当所有的 `Future` 都成功完成，该方法将返回一个 *成功的* `Future`；当任一个 `Future` 执行失败，则返回一个 *失败的* `Future`：

```java
Future<HttpServer> httpServerFuture = Future.future();
httpServer.listen(httpServerFuture.completer());

Future<NetServer> netServerFuture = Future.future();
netServer.listen(netServerFuture.completer());

CompositeFuture.all(httpServerFuture, netServerFuture).setHandler(ar -> {
  if (ar.succeeded()) {
    // 所有服务器启动完成
  } else {
    // 有一个服务器启动失败
  }
});
```

所有被合并的 `Future` 中的操作同时运行。当组合的处理操作完成时，该方法返回的 `Future` 上绑定的处理器（[`Handler`](http://vertx.io/docs/apidocs/io/vertx/core/Handler.html)）会被调用。当一个操作失败（其中的某一个 `Future` 的状态被标记成失败），则返回的 `Future` 会被标记为失败。当所有的操作都成功时，返回的 `Future` 将会成功完成。

您可以传入一个 `Future` 列表（可能为空）：

```java
CompositeFuture.all(Arrays.asList(future1, future2, future3));
```

不同于 `all` 方法的合并会等待所有的 Future 成功执行（或任一失败），`any` 方法的合并会等待第一个成功执行的Future。[`CompositeFuture.any`](http://vertx.io/docs/apidocs/io/vertx/core/CompositeFuture.html#any-io.vertx.core.Future-io.vertx.core.Future-) 方法接受多个 `Future` 作为参数（最多6个，或传入 `List`）。当任意一个 `Future` 成功得到结果，则该 `Future` 成功；当所有的 `Future` 都执行失败，则该 `Future` 失败。

```java
CompositeFuture.any(future1, future2).setHandler(ar -> {
  if (ar.succeeded()) {
    // 至少一个成功
  } else {
    // 所有的都失败
  }
});
```

它也可使用 `Future` 列表传参：

```java
CompositeFuture.any(Arrays.asList(f1, f2, f3));
```

`join` 方法的合并会等待所有的 `Future` 完成，无论成败。[`CompositeFuture.join`](http://vertx.io/docs/apidocs/io/vertx/core/CompositeFuture.html#join-io.vertx.core.Future-io.vertx.core.Future-) 方法接受多个 `Future`作为参数（最多6个），并将结果归并成一个 `Future` 。当全部 `Future` 成功执行完成，得到的 `Future`是成功状态的；当至少一个 `Future` 执行失败时，得到的 `Future` 是失败状态的。

```java
CompositeFuture.join(future1, future2, future3).setHandler(ar -> {
  if (ar.succeeded()) {
    // 所有都成功
  } else {
    // 至少一个失败
  }
});
```

它也可使用 `Future` 列表传参：

```java
CompositeFuture.join(Arrays.asList(future1, future2, future3));
```

### 顺序合并

和 `all` 以及 `any` 实现的并发组合不同，[`compose`](http://vertx.io/docs/apidocs/io/vertx/core/Future.html#compose-io.vertx.core.Handler-io.vertx.core.Future-) 方法作用于顺序组合 `Future`。

```java
FileSystem fs = vertx.fileSystem();
Future<Void> startFuture = Future.future();

Future<Void> fut1 = Future.future();
fs.createFile("/foo", fut1.completer());

fut1.compose(v -> {
  // fut1中文件创建完成后执行
  Future<Void> fut2 = Future.future();
  fs.writeFile("/foo", Buffer.buffer(), fut2.completer());
  return fut2;
}).compose(v -> {
  // fut2文件写入完成后执行
  fs.move("/foo", "/bar", startFuture.completer());
},
  // 如果任何一步失败，将startFuture标记成failed
  startFuture);
```

这里例子中，有三个操作被串起来了：

1. 一个文件被创建（`fut1`）
2. 一些东西被写入到文件（`fut2`）
3. 文件被移走（`startFuture`）

如果这三个步骤全部成功，则最终的 `Future`（`startFuture`）会是成功的；其中任何一步失败，则最终 `Future` 就是失败的。

例子中使用了：

- [`compose(mapper)`](http://vertx.io/docs/apidocs/io/vertx/core/Future.html#compose-io.vertx.core.Handler-io.vertx.core.Future-)：当前 `Future` 完成时，执行相关代码，并返回 `Future`。当返回的 `Future` 完成时，组合完成。
- [`compose(handler, next)`](http://vertx.io/docs/apidocs/io/vertx/core/Future.html#compose-io.vertx.core.Handler-io.vertx.core.Future-)：当前 `Future` 完成时，执行相关代码，并完成下一个 `Future` 的处理。

对于第二个例子，处理器需要完成 `next` future，以此来汇报处理成功或者失败。

您可以使用 [`completer`](http://vertx.io/docs/apidocs/io/vertx/core/Future.html#completer--) 方法来串起一个带操作结果的或失败的 `Future` ，它可使您避免用传统方式编写代码：如果成功则完成 `Future`，否则就标记为失败。（译者注：3.4.0 以后不需要再使用 `completer` 方法）

## Verticle

Verticle 是由 Vert.x 部署和运行的代码块。默认情况一个 Vert.x 实例维护了N（默认情况下N = CPU核数 x 2）个 Event Loop 线程。Verticle 实例可使用任意 Vert.x 支持的编程语言编写，而且一个简单的应用程序也可以包含多种语言编写的 Verticle。

### 编写 Verticle

Verticle 的实现类必须实现 [`Verticle`](http://vertx.io/docs/apidocs/io/vertx/core/Verticle.html) 接口。

如果您喜欢的话，可以直接实现该接口，但是通常直接从抽象类 [`AbstractVerticle`](http://vertx.io/docs/apidocs/io/vertx/core/AbstractVerticle.html) 继承更简单。

这儿有一个例子：

```java
public class MyVerticle extends AbstractVerticle {

  // Called when verticle is deployed
  // Verticle部署时调用
  public void start() {
  }

  // Optional - called when verticle is undeployed
  // 可选 - Verticle撤销时调用
  public void stop() {
  }

}
```

通常您需要像上边例子一样重写 `start` 方法。

当 Vert.x 部署 Verticle 时，它的 `start` 方法将被调用，这个方法执行完成后 Verticle 就变成已启动状态。

您同样可以重写 `stop` 方法，当Vert.x 撤销一个 Verticle 时它会被调用，这个方法执行完成后 Verticle 就变成已停止状态了。 

### Verticle 异步启动和停止

有些时候您的 Verticle 启动会耗费一些时间，您想要在这个过程做一些事，并且您做的这些事并不想等到Verticle部署完成过后再发生。如：您想在 `start` 方法中部署其他的 Verticle。

您不能在您的 `start` 方法中阻塞等待其他的 Verticle 部署完成，这样做会破坏 [黄金法则](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#%E9%BB%84%E9%87%91%E6%B3%95%E5%88%99%E4%B8%8D%E8%A6%81%E9%98%BB%E5%A1%9Eevent-loop)。

所以您要怎么做？

您可以实现 **异步版本** 的 `start` 方法来做这个事。这个版本的方法会以一个 `Future` 作参数被调用。方法执行完时，Verticle 实例**并没有**部署好（状态不是 deployed）。稍后，您完成了所有您需要做的事（如：启动其他Verticle），您就可以调用 `Future` 的 `complete`（或 `fail` ）方法来标记启动完成或失败了。

这儿有一个例子：

```java
public class MyVerticle extends AbstractVerticle {

  public void start(Future<Void> startFuture) {
    // 现在部署其他的一些verticle
    vertx.deployVerticle("com.foo.OtherVerticle", res -> {
      if (res.succeeded()) {
        startFuture.complete();
      } else {
        startFuture.fail(res.cause());
      }
    });
  }
}
```

同样的，这儿也有一个异步版本的 `stop` 方法，如果您想做一些耗时的 Verticle 清理工作，您可以使用它。

```java
public class MyVerticle extends AbstractVerticle {

  public void start() {
    // 做一些事
  }

  public void stop(Future<Void> stopFuture) {
    obj.doSomethingThatTakesTime(res -> {
      if (res.succeeded()) {
        stopFuture.complete();
      } else {
        stopFuture.fail();
      }
    });
  }
}
```

> 请注意：您不需要在一个 Verticle 的 `stop` 方法中手工去撤销启动时部署的子 Verticle，当父 Verticle 在撤销时 Vert.x 会自动撤销任何子 Verticle。

### Verticle 种类

这儿有三种不同类型的 Verticle：

- **Stardand Verticle**：这是最常用的一类 Verticle —— 它们永远运行在 Event Loop 线程上。稍后的章节我们会讨论更多。
- **Worker Verticle**：这类 Verticle 会运行在 Worker Pool 中的线程上。一个实例绝对不会被多个线程同时执行。
- **Multi-Threaded Worker Verticle**：这类 Verticle 也会运行在 Worker Pool 中的线程上。一个实例可以由多个线程同时执行（译者注：因此需要开发者自己确保线程安全）。

### Standard Verticle

当 Standard Verticle 被创建时，它会被分派给一个 Event Loop 线程，并在这个 Event Loop 中执行它的 `start` 方法。当您在一个 Event Loop 上调用了 Core API 中的方法并传入了处理器时，Vert.x 将保证用与调用该方法时相同的 Event Loop 来执行这些处理器。

这意味着我们可以保证您的 Verticle 实例中 **所有的代码都是在相同Event Loop中执行**（只要您不创建自己的线程并调用它！）

同样意味着您可以将您的应用中的所有代码用单线程方式编写，让 Vert.x 去考虑线程和扩展问题。您不用再考虑 synchronized 和 volatile 的问题，也可以避免传统的多线程应用经常会遇到的竞态条件和死锁的问题。

### Worker Verticle

Worker Verticle 和 Standard Verticle 很像，但它并不是由一个 Event Loop 来执行，而是由Vert.x中的 Worker Pool 中的线程执行。

Worker Verticle 被设计来调用阻塞式代码，它不会阻塞任何 Event Loop。

如果您不想使用 Worker Verticle 来运行阻塞式代码，您还可以在一个Event Loop中直接使用 [内联阻塞式代码](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#%E8%BF%90%E8%A1%8C%E9%98%BB%E5%A1%9E%E5%BC%8F%E4%BB%A3%E7%A0%81)。

若您想要将 Verticle 部署成一个 Worker Verticle，您可以通过 [`setWorker`](http://vertx.io/docs/apidocs/io/vertx/core/DeploymentOptions.html#setWorker-boolean-) 方法来设置：

```java
DeploymentOptions options = new DeploymentOptions().setWorker(true);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
```

Worker Verticle 实例绝对不会在 Vert.x 中被多个线程同时执行，但它可以在不同时间由不同线程执行。

### Multi-threaded Worker Verticle

一个 Multi-threaded Worker Verticle 近似于普通的 Worker Verticle，但是它可以由不同的线程同时执行。

> 警告：*Multi-threaded Worker Verticle 是一个高级功能，大部分应用程序不会需要它。由于这些 Verticle 是并发的，您必须小心地使用标准的Java多线程技术来保持 Verticle 的状态一致性。*

### 编程方式部署Verticle

您可以指定一个 Verticle 名称或传入您已经创建好的 Verticle 实例，使用任意一个 [`deployVerticle`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#deployVerticle-io.vertx.core.Verticle-) 方法来部署Verticle。

> 请注意：通过 Verticle **实例** 来部署 Verticle 仅限Java语言。

```java
Verticle myVerticle = new MyVerticle();
vertx.deployVerticle(myVerticle);
```

您同样可以指定 Verticle 的 **名称** 来部署它。

这个 Verticle 的名称会用于查找实例化 Verticle 的特定 [`VerticleFactory`](http://vertx.io/docs/apidocs/io/vertx/core/spi/VerticleFactory.html)。

不同的 Verticle Factory 可用于实例化不同语言的 Verticle，也可用于其他目的，例如加载服务、运行时从Maven中获取Verticle实例等。

这允许您部署用任何使用Vert.x支持的语言编写的Verticle实例。

这儿有一个部署不同类型 Verticle 的例子：

```java
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle");

// 部署JavaScript的Verticle
vertx.deployVerticle("verticles/myverticle.js");

// 部署Ruby的Verticle
vertx.deployVerticle("verticles/my_verticle.rb");
```

### Verticle名称到Factory的映射规则

当使用名称部署Verticle时，会通过名称来选择一个用于实例化 Verticle 的 Verticle Factory。

Verticle 名称可以有一个前缀 —— 使用字符串紧跟着一个冒号，它用于查找存在的Factory，参考例子。

```
js:foo.js // 使用JavaScript的Factory
groovy:com.mycompany.SomeGroovyCompiledVerticle // 用Groovy的Factory
service:com.mycompany:myorderservice // 用Service的Factory
```

如果不指定前缀，Vert.x将根据提供名字后缀来查找对应Factory，如：

```
foo.js // 将使用JavaScript的Factory
SomeScript.groovy // 将使用Groovy的Factory
```

若前缀后缀都没指定，Vert.x将假定这个名字是一个Java 全限定类名（FQCN）然后尝试实例化它。

### 如何定位Verticle Factory？

大部分Verticle Factory会从 classpath 中加载，并在 Vert.x 启动时注册。

您同样可以使用编程的方式去注册或注销Verticle Factory：通过 [`registerVerticleFactory`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#registerVerticleFactory-io.vertx.core.spi.VerticleFactory-) 方法和 [`unregisterVerticleFactory`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#unregisterVerticleFactory-io.vertx.core.spi.VerticleFactory-) 方法。

### 等待部署完成

Verticle的部署是异步方式，可能在 `deploy` 方法调用返回后一段时间才会完成部署。

如果您想要在部署完成时被通知则可以指定一个完成处理器：

```java
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", res -> {
  if (res.succeeded()) {
    System.out.println("Deployment id is: " + res.result());
  } else {
    System.out.println("Deployment failed!");
  }
});
```

如果部署成功，这个完成处理器的结果中将会包含部署ID的字符串。这个部署 ID可以在之后您想要撤销它时使用。

### 撤销Verticle

我们可以通过 [`undeploy`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#undeploy-java.lang.String-) 方法来撤销部署好的 Verticle。

撤销操作也是异步的，因此若您想要在撤销完成过后收到通知则可以指定另一个完成处理器：

```java
vertx.undeploy(deploymentID, res -> {
  if (res.succeeded()) {
    System.out.println("Undeployed ok");
  } else {
    System.out.println("Undeploy failed!");
  }
});
```

### 设置 Verticle 实例数

当使用名称部署一个 Verticle 时，您可以指定您想要部署的 Verticle 实例的数量。

```java
DeploymentOptions options = new DeploymentOptions().setInstances(16);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
```

这个功能对于跨多核扩展时很有用。例如，您有一个实现了Web服务器的Verticle需要部署在多核的机器上，您可以部署多个实例来利用所有的核。

### 向 Verticle 传入配置

可在部署时传给 Verticle 一个 JSON 格式的配置

```java
JsonObject config = new JsonObject().put("name", "tim").put("directory", "/blah");
DeploymentOptions options = new DeploymentOptions().setConfig(config);
vertx.deployVerticle("com.mycompany.MyOrderProcessorVerticle", options);
```

传入之后，这个配置可以通过 [`Context`](http://vertx.io/docs/apidocs/io/vertx/core/Context.html) 对象或使用 [`config`](http://vertx.io/docs/apidocs/io/vertx/core/AbstractVerticle.html#config--) 方法访问。

这个配置会以 JSON 对象（`JsonObject`）的形式返回，因此您可以用下边代码读取数据：

```java
System.out.println("Configuration: " + config().getString("name"));
```

### 在 Verticle 中访问环境变量

环境变量和系统属性可以直接通过 Java API 访问：

```java
System.getProperty("prop");
System.getenv("HOME");
```

### Verticle 隔离组

默认情况，当Vert.x部署Verticle时它会调用当前类加载器来加载类，而不会创建一个新的。大多数情况下，这是最简单、最清晰和最干净。

但是在某些情况下，您可能需要部署一个Verticle，它包含的类要与应用程序中其他类隔离开来。比如您想要在一个Vert.x实例中部署两个同名不同版本的Verticle，或者不同的Verticle使用了同一个jar包的不同版本。

当使用隔离组时，您需要用 [`setIsolatedClassed`](http://vertx.io/docs/apidocs/io/vertx/core/DeploymentOptions.html#setIsolatedClasses-java.util.List-) 方法来提供一个您想隔离的类名列表。列表项可以是一个Java 限定类全名，如 `com.mycompany.myproject.engine.MyClass`；也可以是包含通配符的可匹配某个包或子包的任何类，例如 `com.mycompany.myproject.*` 将会匹配所有 `com.mycompany.myproject` 包或任意子包中的任意类名。

请注意仅仅只有匹配的类会被隔离，其他任意类会被当前类加载器加载。

若您想要加载的类和资源不存在于主类路径（main classpath），您可使用 [`setExtraClasspath`](http://vertx.io/docs/apidocs/io/vertx/core/DeploymentOptions.html#setExtraClasspath-java.util.List-) 方法将额外的类路径添加到这里。

> 警告：*谨慎使用此功能，类加载器可能会导致您的应用难于调试，变得一团乱麻（can of worms）。*

以下是使用隔离组隔离 Verticle 的部署例子：

```java
DeploymentOptions options = new DeploymentOptions().setIsolationGroup("mygroup");
options.setIsolatedClasses(Arrays.asList("com.mycompany.myverticle.*",
                   "com.mycompany.somepkg.SomeClass", "org.somelibrary.*"));
vertx.deployVerticle("com.mycompany.myverticle.VerticleClass", options);
```

### 高可用性

Verticle可以启用高可用方式（HA）部署。在这种方式下，当其中一个部署在 Vert.x 实例中的 Verticle 突然挂掉，这个 Verticle 可以在集群环境中的另一个 Vert.x 实例中重新部署。

若要启用高可用方式运行一个 Verticle，仅需要追加 `-ha` 参数：

```
vertx run my-verticle.js -ha
```

当启用高可用方式时，不需要追加 `-cluster` 参数。

关于高可用的功能和配置的更多细节可参考 [高可用和故障转移](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#%E9%AB%98%E5%8F%AF%E7%94%A8%E5%92%8C%E6%95%85%E9%9A%9C%E8%BD%AC%E7%A7%BB) 章节。

### Context 对象

当 Vert.x 传递一个事件给处理器或者调用 Verticle 的 `start` 或 `stop` 方法时，它会关联一个 `Context` 对象来执行。通常来说这个 `Context` 会是一个 **Event Loop Context**，它绑定到了一个特定的 Event Loop 线程上。所以在该 `Context` 上执行的操作总是在同一个 Event Loop 线程中。对于运行内联的阻塞代码的 Worker Verticle 来说，会关联一个 Worker Context，并且所有的操作运都会运行在 Worker 线程池的线程上。

> 译者注：每个 `Verticle` 在部署的时候都会被分配一个 `Context`（根据配置不同，可以是Event Loop Context 或者 Worker Context），之后此 `Verticle` 上所有的普通代码都会在此 `Context` 上执行（即对应的 Event Loop 或Worker 线程）。一个 `Context` 对应一个 Event Loop 线程（或 Worker 线程），但一个 Event Loop 可能对应多个 `Context`。

您可以通过 [`getOrCreateContext`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#getOrCreateContext--) 方法获取 `Context` 实例：

```java
Context context = vertx.getOrCreateContext();
```

若已经有一个 `Context` 和当前线程关联，那么它直接重用这个 `Context` 对象，如果没有则创建一个新的。您可以检查获取的 `Context` 的类型：

```java
Context context = vertx.getOrCreateContext();
if (context.isEventLoopContext()) {
  System.out.println("Context attached to Event Loop");
} else if (context.isWorkerContext()) {
  System.out.println("Context attached to Worker Thread");
} else if (context.isMultiThreadedWorkerContext()) {
  System.out.println("Context attached to Worker Thread - multi threaded worker");
} else if (! Context.isOnVertxThread()) {
  System.out.println("Context not attached to a thread managed by vert.x");
}
```

当您获取了这个 `Context` 对象，您就可以在 `Context` 中异步执行代码了。换句话说，您提交的任务将会在同一个 `Context` 中运行：

```java
vertx.getOrCreateContext().runOnContext(v -> {
  System.out.println("This will be executed asynchronously in the same context");
});
```

当在同一个 `Context` 中运行了多个处理函数时，可能需要在它们之间共享数据。 `Context` 对象提供了存储和读取共享数据的方法。举例来说，它允许您将数据传递到 [`runOnContext`](http://vertx.io/docs/apidocs/io/vertx/core/Context.html#runOnContext-io.vertx.core.Handler-) 方法运行的某些操作中：

```java
final Context context = vertx.getOrCreateContext();
context.put("data", "hello");
context.runOnContext((v) -> {
  String hello = context.get("data");
});
```

您还可以通过 [`config`](http://vertx.io/docs/apidocs/io/vertx/core/Context.html#config--) 方法访问 Verticle 的配置信息。查看 [向 Verticle 传入配置](https://vertxchina.github.io/vertx-translation-chinese/core/Core.html#%E5%90%91-verticle-%E4%BC%A0%E5%85%A5%E9%85%8D%E7%BD%AE) 章节了解更多配置信息。

### 执行周期性/延迟性操作

在 Vert.x 中，想要延迟之后执行或定期执行操作很常见。

在 Standard Verticle 中您不能直接让线程休眠以引入延迟，因为它会阻塞 Event Loop 线程。取而代之是使用 Vert.x 定时器。定时器可以是一次性或周期性的，两者我们都会讨论到。

#### 一次性计时器

一次性计时器会在一定延迟后调用一个 Event Handler，以毫秒为单位计时。

您可以通过 [`setTimer`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#setTimer-long-io.vertx.core.Handler-) 方法传递延迟时间和一个处理器来设置计时器的触发。

```java
long timerID = vertx.setTimer(1000, id -> {
  System.out.println("And one second later this is printed");
});

System.out.println("First this is printed");
```

返回值是一个唯一的计时器id，该id可用于之后取消该计时器，这个计时器id会传入给处理器。

#### 周期性计时器

您同样可以使用 [`setPeriodic`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#setPeriodic-long-io.vertx.core.Handler-) 方法设置一个周期性触发的计时器。第一次触发之前同样会有一段设置的延时时间。

`setPeriodic` 方法的返回值也是一个唯一的计时器id，若之后该计时器需要取消则使用该id。传给处理器的参数也是这个唯一的计时器id。

请记住这个计时器将会定期触发。如果您的定时任务会花费大量的时间，则您的计时器事件可能会连续执行甚至发生更坏的情况：重叠。这种情况，您应考虑使用 [`setTimer`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#setTimer-long-io.vertx.core.Handler-) 方法，当任务执行完成时设置下一个计时器。

```java
long timerID = vertx.setPeriodic(1000, id -> {
  System.out.println("And every second this is printed");
});

System.out.println("First this is printed");
```

#### 取消计时器

指定一个计时器id并调用 [`cancelTimer`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#cancelTimer-long-) 方法来取消一个周期性计时器。如：

```java
vertx.cancelTimer(timerID);
```

#### Verticle 中自动清除定时器

如果您在 Verticle 中创建了计时器，当这个 Verticle 被撤销时这个计时器会被自动关闭。

### Verticle Worker Pool

Verticle 使用 Vert.x 中的 Worker Pool 来执行阻塞式行为，例如 [`executeBlocking`](http://vertx.io/docs/apidocs/io/vertx/core/Context.html#executeBlocking-io.vertx.core.Handler-boolean-io.vertx.core.Handler-) 或 Worker Verticle。

可以在部署配置项中指定不同的Worker 线程池：

```java
vertx.deployVerticle("the-verticle", new DeploymentOptions().setWorkerPoolName("the-specific-pool"));
```

## Event Bus

Event Bus 是 Vert.x 的神经系统。

每一个 Vert.x 实例都有一个单独的 Event Bus 实例。您可以通过 `Vertx` 实例的 [`eventBus`](http://vertx.io/docs/apidocs/io/vertx/core/Vertx.html#eventBus--) 方法来获得对应的 `EventBus` 实例。

您的应用中的不同部分通过 Event Bus 相互通信，无论它们使用哪一种语言实现，无论它们在同一个 Vert.x 实例中或在不同的 Vert.x 实例中。

甚至可以通过桥接的方式允许在浏览器中运行的客户端JavaScript在相同的Event Bus上相互通信。

Event Bus可形成跨越多个服务器节点和多个浏览器的点对点的分布式消息系统。

Event Bus支持发布/订阅、点对点、请求/响应的消息通信方式。

Event Bus的API很简单。基本上只涉及注册处理器、撤销处理器和发送和发布消息。

首先来看些基本概念和理论。

### 基本概念

#### 寻址

消息会被 Event Bus 发送到一个 **地址(address)**。

同任何花哨的寻址方案相比，Vert.x的地址格式并不麻烦。Vert.x中的地址是一个简单的字符串，任意字符串都合法。当然，使用某种模式来命名仍然是明智的。如：使用点号来划分命名空间。

一些合法的地址形如：`europe.news.feed1`、`acme.games.pacman`、`sausages`和`X`。

#### 处理器

消息在处理器（`Handler`）中被接收。您可以在某个地址上注册一个处理器来接收消息。

同一个地址可以注册许多不同的处理器，一个处理器也可以注册在多个不同的地址上。

#### 发布/订阅消息

Event Bus支持 **发布消息** 功能。

消息将被发布到一个地址中，发布意味着会将信息传递给 **所有** 注册在该地址上的处理器。这和 **发布/订阅模式** 很类似。

#### 点对点模式/请求-响应模式

Event Bus也支持 **点对点消息模式**。

消息将被发送到一个地址中，Vert.x将会把消息分发到某个注册在该地址上的处理器。若这个地址上有不止一个注册过的处理器，它将使用 **不严格的轮询算法** 选择其中一个。

点对点消息传递模式下，可在消息发送的时候指定一个应答处理器（可选）。

当接收者收到消息并且已经被处理时，它可以选择性决定回复该消息，若选择回复则绑定的应答处理器将会被调用。当发送者收到回复消息时，它也可以回复，这个过程可以不断重复。通过这种方式可以允许在两个不同的 Verticle 之间设置一个对话窗口。这种消息模式被称作 **请求-响应** 模式。

#### 尽力传输

Vert.x会尽它最大努力去传递消息，并且不会主动丢弃消息。这种方式称为 **尽力传输(Best-effort delivery)**。

但是，当 Event Bus 中的全部或部分发生故障时，则可能会丢失消息。

若您的应用关心丢失的消息，您应该编写具有幂等性的处理器，并且您的发送者可以在恢复后重试。

> 译者注：RPC通信通常情况下有三种语义：**at least once**、**at most once** 和 **exactly once**。不同语义情况下要考虑的情况不同。

#### 消息类型

Vert.x 默认允许任何基本/简单类型、`String` 或 [`Buffer`](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html) 作为消息发送。不过在 Vert.x 中的通常做法是使用 [JSON](http://json.org/) 格式来发送消息。

JSON 对于 Vert.x 支持的所有语言都是非常容易创建、读取和解析的，因此它已经成为了Vert.x中的通用语(*lingua franca*)。但是若您不想用 JSON，我们并不强制您使用它。

Event Bus 非常灵活，它支持在 Event Bus 中发送任意对象。您可以通过为您想要发送的对象自定义一个 [`MessageCodec`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageCodec.html) 来实现。



### Event Bus API

下面我们来看一下 API。

#### 获取Event Bus

您可以通过下面的代码获取 Event Bus 的引用：

```java
EventBus eb = vertx.eventBus();
```

对于每一个 Vert.x 实例来说它是单例的。

#### 注册处理器

最简单的注册处理器的方式是使用 [consumer](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#consumer-java.lang.String-io.vertx.core.Handler-) 方法，这儿有个例子：

```java
EventBus eb = vertx.eventBus();

eb.consumer("news.uk.sport", message -> {
  System.out.println("I have received a message: " + message.body());
});
```

当一个消息达到您的处理器，该处理器会以 [`message`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html) 为参数被调用。

调用 `consumer` 方法会返回一个 [`MessageConsumer`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html) 对象。该对象随后可用于撤销处理器、或将处理器用作流式处理。

您也可以不设置处理器而使用 [`consumer`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#consumer-java.lang.String-io.vertx.core.Handler-) 方法直接返回一个 `MessageConsumer`，之后再来设置处理器。如：

```java
EventBus eb = vertx.eventBus();

MessageConsumer<String> consumer = eb.consumer("news.uk.sport");
consumer.handler(message -> {
  System.out.println("I have received a message: " + message.body());
});
```

在集群模式下的Event Bus上注册处理器时，注册信息会花费一些时间才能传播到集群中的所有节点。

若您希望在完成注册后收到通知，您可以在 `MessageConsumer` 对象上注册一个 [`completion handler`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html#completionHandler-io.vertx.core.Handler-)。

```java
consumer.completionHandler(res -> {
  if (res.succeeded()) {
    System.out.println("The handler registration has reached all nodes");
  } else {
    System.out.println("Registration failed!");
  }
});
```

#### 注销处理器

您可以通过 [`unregister()`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html#unregister--) 方法来注销处理器。

若您在集群模式下的 Event Bus 中撤销处理器，则同样会花费一些时间在节点中传播。若您想在完成后收到通知，可以使用[`unregister(handler)`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html#unregister-io.vertx.core.Handler-) 方法注册处理器：

```java
consumer.unregister(res -> {
  if (res.succeeded()) {
    System.out.println("The handler un-registration has reached all nodes");
  } else {
    System.out.println("Un-registration failed!");
  }
});
```

#### 发布消息

发布消息很简单，只需使用 [`publish`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#publish-java.lang.String-java.lang.Object-) 方法指定一个地址去发布即可。

```java
eventBus.publish("news.uk.sport", "Yay! Someone kicked a ball");
```

这个消息将会传递给所有在地址 `news.uk.sport` 上注册过的处理器。

#### 发送消息

与发布消息的不同之处在于，发送(`send`)的消息只会传递给在该地址注册的其中一个处理器，这就是点对点模式。Vert.x 使用不严格的轮询算法来选择绑定的处理器。

您可以使用 [`send`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#send-java.lang.String-java.lang.Object-) 方法来发送消息：

```java
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball");
```

#### 设置消息头

在 Event Bus 上发送的消息可包含头信息。这可通过在发送或发布时提供的 [`DeliveryOptions`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html) 来指定。例如：

```java
DeliveryOptions options = new DeliveryOptions();
options.addHeader("some-header", "some-value");
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball", options);
```

#### 消息顺序

Vert.x将按照特定发送者发送消息的顺序来传递消息给特定处理器。

#### 消息对象

您在消息处理器中接收到的对象的类型是 [`Message`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html)。

消息的 [`body`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#body--) 对应发送或发布的对象。消息的头信息可以通过 [`headers`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#headers--) 方法获取。

#### 应答消息/发送回复

当使用 [`send`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#send-java.lang.String-java.lang.Object-) 方法发送消息时，Event Bus会尝试将消息传递到注册在Event Bus上的 [`MessageConsumer`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageConsumer.html)中。在某些情况下，发送者需要知道消费者何时收到消息并 *处理* 了消息。

消费者可以通过调用 [`reply`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#reply-java.lang.Object-) 方法来应答这个消息。

当这种情况发生时，它会将消息回复给发送者并且在发送者中调用应答处理器来处理回复的消息。

看这个例子会更清楚：

接收者：

```java
MessageConsumer<String> consumer = eventBus.consumer("news.uk.sport");
consumer.handler(message -> {
  System.out.println("I have received a message: " + message.body());
  message.reply("how interesting!");
});
```

发送者：

```java
eventBus.send("news.uk.sport", "Yay! Someone kicked a ball across a patch of grass", ar -> {
  if (ar.succeeded()) {
    System.out.println("Received reply: " + ar.result().body());
  }
});
```

在应答的消息体中可以包含有用的信息。

关于 *处理中* 的含义实际上是由应用程序来定义的。这完全取决于消费者如何执行，Event Bus 对此并不关心。

一些例子：

- 一个简单地实现了返回当天时间的服务，在应答的消息里会包含当天时间信息。
- 一个实现了持久化队列的消息消费者，当消息成功持久化到存储时，可以使用`true`来应答消息，或`false`表示失败。
- 一个处理订单的消息消费者也许会用`true`确认这个订单已经成功处理并且可以从数据库中删除。

#### 带超时的发送

当发送带有应答处理器的消息时，可以在 [`DeliveryOptions`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html) 中指定一个超时时间。如果在这个时间之内没有收到应答，则会以失败为参数调用应答处理器。默认超时是 **30 秒**。

#### 发送失败

消息发送可能会因为其他原因失败，包括：

- 没有可用的处理器来接收消息
- 接收者调用了 [`fail`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/Message.html#fail-int-java.lang.String-) 方法显式声明失败

发生这些情况时，应答处理器将会以这些失败为参数被调用。

#### 消息编解码器

您可以在 Event Bus 中发送任何对象，只要你为这个对象类型注册一个编解码器 [`MessageCodec`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/MessageCodec.html)。消息编解码器有一个名称，您需要在发送或发布消息时通过 [`DeliveryOptions`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html) 来指定：

```java
eventBus.registerCodec(myCodec);

DeliveryOptions options = new DeliveryOptions().setCodecName(myCodec.name());

eventBus.send("orders", new MyPOJO(), options);
```

若您总是希望某个类使用将特定的编解码器，那么您可以为这个类注册默认编解码器。这样您就不需要在每次发送的时候使用 [`DeliveryOptions`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/DeliveryOptions.html) 来指定了：

```java
eventBus.registerDefaultCodec(MyPOJO.class, myCodec);

eventBus.send("orders", new MyPOJO());
```

您可以通过 [`unregisterCodec`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBus.html#unregisterCodec-java.lang.String-) 方法注销某个消息编解码器。

消息编解码器的编码和解码不一定使用同一个类型。例如您可以编写一个编解码器来发送 MyPOJO 类的对象，但是当消息发送给处理器后解码成 MyOtherPOJO 对象。

#### 集群模式的 Event Bus

Event Bus 不仅仅存在于单个 Vert.x 实例中。通过您在网络上将不同的 Vert.x 实例集群在一起，它可以形成一个单一的、分布式的Event Bus。

#### 通过代码的方式启用集群模式

若您用编程的方式创建 Vert.x 实例（`Vertx`），则可以通过将 Vert.x 实例配置成集群模式来获取集群模式的Event Bus：

```java
VertxOptions options = new VertxOptions();
Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
    EventBus eventBus = vertx.eventBus();
    System.out.println("We now have a clustered event bus: " + eventBus);
  } else {
    System.out.println("Failed: " + res.cause());
  }
});
```

您需要确在您的 classpath 中（或构建工具的依赖中）包含 [`ClusterManager`](http://vertx.io/docs/apidocs/io/vertx/core/spi/cluster/ClusterManager.html) 的实现类，如默认的 `HazelcastClusterManager`。

#### 通过命令行启用集群模式

您可以通过以下命令以集群模式运行 Vert.x 应用：

```
vertx run my-verticle.js -cluster
```

### Verticle 中的自动清理

若您在 Verticle 中注册了 Event Bus 的处理器，那么这些处理器在 Verticle 被撤销的时候会自动被注销。

### 配置 Event Bus

Event Bus 是可以配置的，这对于以集群模式运行的 Event Bus 是非常有用的。Event Bus 使用 TCP 连接发送和接收消息，因此可以通过 [`EventBusOptions`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBusOptions.html) 对TCP连接进行全面的配置。由于 Event Bus 同时用作客户端和服务器，因此这些配置近似于 [`NetClientOptions`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetClientOptions.html) 和 [`NetServerOptions`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServerOptions.html)。

```java
VertxOptions options = new VertxOptions()
    .setEventBusOptions(new EventBusOptions()
        .setSsl(true)
        .setKeyStoreOptions(new JksOptions().setPath("keystore.jks").setPassword("wibble"))
        .setTrustStoreOptions(new JksOptions().setPath("keystore.jks").setPassword("wibble"))
        .setClientAuth(ClientAuth.REQUIRED)
    );

Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
    EventBus eventBus = vertx.eventBus();
    System.out.println("We now have a clustered event bus: " + eventBus);
  } else {
    System.out.println("Failed: " + res.cause());
  }
});
```

上边代码段描述了如何在Event Bus中使用SSL连接替换传统的TCP连接。

> **警告：** 若要在集群模式下保证安全性，您 **必须** 将集群管理器配置成加密的或强制安全的。参考集群管理器的文档获取更多细节。

Event Bus 的配置需要在所有集群节点中保持一致性。

[`EventBusOptions`](http://vertx.io/docs/apidocs/io/vertx/core/eventbus/EventBusOptions.html)还允许您指定 Event Bus 是否运行在集群模式下，以及它的主机信息和端口。您可使用 [`setClustered`](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#setClustered-boolean-)、[`getClusterHost`](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#getClusterHost--)和 [`getClusterPort`](http://vertx.io/docs/apidocs/io/vertx/core/VertxOptions.html#getClusterPort--) 方法来设置。

在容器中使用时，您也可以配置公共主机和端口号：

```java
VertxOptions options = new VertxOptions()
    .setEventBusOptions(new EventBusOptions()
        .setClusterPublicHost("whatever")
        .setClusterPublicPort(1234)
    );

Vertx.clusteredVertx(options, res -> {
  if (res.succeeded()) {
    Vertx vertx = res.result();
    EventBus eventBus = vertx.eventBus();
    System.out.println("We now have a clustered event bus: " + eventBus);
  } else {
    System.out.println("Failed: " + res.cause());
  }
});
```

## Buffer

在 Vert.x 内部，大部分数据被重新组织（shuffle，表意为洗牌）成 `Buffer` 格式。

一个 `Buffer` 是可以读取或写入的0个或多个字节序列，并且根据需要可以自动扩容、将任意字节写入 `Buffer`。您也可以将 `Buffer` 想象成字节数组（译者注：类似于 JDK 中的 `ByteBuffer`）。

### 创建 Buffer

可以使用静态方法 [`Buffer.buffer`](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#buffer--) 来创建 `Buffer`。

`Buffer` 可以从字符串或字节数组初始化，或者直接创建空的 `Buffer`。

这儿有一些创建 `Buffer` 的例子。

创建一个空的 `Buffer`：

```java
Buffer buff = Buffer.buffer();
```

从字符串创建一个 `Buffer`，这个 `Buffer` 中的字符会以 UTF-8 格式编码：

```java
Buffer buff = Buffer.buffer("some string");
```

从字符串创建一个 `Buffer`，这个字符串可以用指定的编码方式编码，例如：

```java
Buffer buff = Buffer.buffer("some string", "UTF-16");
```

从字节数组 `byte[]` 创建 `Buffer`：

```java
byte[] bytes = new byte[] {1, 3, 5};
Buffer buff = Buffer.buffer(bytes);
```

创建一个指定初始大小的 `Buffer`。若您知道您的 `Buffer` 会写入一定量的数据，您可以创建 `Buffer` 并指定它的大小。这使得这个 `Buffer` 初始化时分配了更多的内存，比数据写入时重新调整大小的效率更高。注意以这种方式创建的 `Buffer` 是 **空的**。它不会创建一个填满了 0 的Buffer。代码如下：

```java
Buffer buff = Buffer.buffer(10000);
```

### 向Buffer写入数据

向 `Buffer` 写入数据的方式有两种：追加和随机写入。任何一种情况下 `Buffer` 都会自动进行扩容，所以不可能在使用 `Buffer` 时遇到 `IndexOutOfBoundsException`。

#### 追加到Buffer

您可以使用 `appendXXX` 方法追加数据到 `Buffer`。`Buffer` 类提供了追加各种不同类型数据的追加写入方法。

因为 `appendXXX` 方法的返回值就是 Buffer 自身，所以它可以链式地调用:

```java
Buffer buff = Buffer.buffer();

buff.appendInt(123).appendString("hello\n");

socket.write(buff);
```

#### 随机访问写Buffer

您还可以指定一个索引值，通过 `setXXX` 方法写入数据到 `Buffer`，它也存在各种不同数据类型的方法。所有的 set 方法都会将索引值作为第一个参数 —— 这表示 `Buffer` 中开始写入数据的位置。`Buffer` 始终根据需要进行自动扩容。

```java
Buffer buff = Buffer.buffer();

buff.setInt(1000, 123);
buff.setString(0, "hello");
```

### 从Buffer中读取

可使用 `getXXX` 方法从 Buffer 中读取数据，它存在各种不同数据类型的方法，这些方法的第一个参数是从哪里获取数据的索引（获取位置）。

```java
Buffer buff = Buffer.buffer();
for (int i = 0; i < buff.length(); i += 4) {
  System.out.println("int value at " + i + " is " + buff.getInt(i));
}
```

### 使用无符号数

可使用 `getUnsignedXXX`、`appendUnsignedXXX` 和 `setUnsignedXXX` 方法将无符号数从 `Buffer` 中读取或追加/设置到 `Buffer` 里。这对以优化网络协议和最小化带宽消耗为目的实现的编解码器是很有用的。

下边例子中，值 200 被设置到了仅占用一个字节的特定位置：

```java
Buffer buff = Buffer.buffer(128);
int pos = 15;
buff.setUnsignedByte(pos, (short) 200);
System.out.println(buff.getUnsignedByte(pos));
```

控制台中显示 `200`。

### Buffer长度

可使用 [`length`](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#length--) 方法获取Buffer长度，Buffer的长度值是Buffer中包含的字节的最大索引 + 1。

#### 拷贝Buffer

可使用 [`copy`](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#copy--) 方法创建一个Buffer的副本。

#### 裁剪Buffer

裁剪得到的Buffer是基于原始Buffer的一个新的Buffer。它不会拷贝实际的数据。使用 [`slice`](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html#slice--) 方法裁剪一个Buffer。

#### Buffer 重用

将Buffer写入到一个Socket或其他类似位置后，Buffer就不可被重用了。

## 编写 TCP 服务端和客户端

Vert.x允许您很容易编写非阻塞的TCP客户端和服务器。

### 创建 TCP 服务端

最简单地使用所有默认配置项创建 TCP 服务端的方式如下：

```java
NetServer server = vertx.createNetServer();
```

### 配置 TCP 服务端

若您不想使用默认配置，可以在创建时通过传入一个 [`NetServerOptions`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServerOptions.html) 实例来配置服务器：

```java
NetServerOptions options = new NetServerOptions().setPort(4321);
NetServer server = vertx.createNetServer(options);
```

### 启动服务端监听

要告诉服务端监听传入的请求，您可以使用其中一个 [`listen`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#listen--) 方法。

让服务器监听配置项指定的主机和端口：

```java
NetServer server = vertx.createNetServer();
server.listen();
```

或在调用 `listen` 方法时指定主机和端口号，忽略配置项中的配置：

```java
NetServer server = vertx.createNetServer();
server.listen(1234, "localhost");
```

默认主机名是 `0.0.0.0`，它表示：监听所有可用地址。默认端口号是 `0`，这也是一个特殊值，它告诉服务器随机选择并监听一个本地没有被占用的端口。

实际的绑定也是异步的，因此服务器在调用了 `listen` 方法的一段时间之后才会实际开始监听。若您希望在服务器实际监听时收到通知，您可以在调用 `listen` 方法时提供一个处理器。例如：

```java
NetServer server = vertx.createNetServer();
server.listen(1234, "localhost", res -> {
  if (res.succeeded()) {
    System.out.println("Server is now listening!");
  } else {
    System.out.println("Failed to bind!");
  }
});
```

### 监听随机端口

若设置监听端口为`0`，服务器将随机寻找一个没有使用的端口来监听。

可以调用 [`actualPort`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#actualPort--) 方法来获得服务器实际监听的端口：

```java
NetServer server = vertx.createNetServer();
server.listen(0, "localhost", res -> {
  if (res.succeeded()) {
    System.out.println("Server is now listening on actual port: " + server.actualPort());
  } else {
    System.out.println("Failed to bind!");
  }
});
```

### 接收传入连接的通知

若您想要在连接创建完时收到通知，则需要设置一个 [`connectHandler`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#connectHandler-io.vertx.core.Handler-)：

```java
NetServer server = vertx.createNetServer();
server.connectHandler(socket -> {
  // 在这里处理传入连接
});
```

当连接成功时，您可以在回调函数中处理得到的 [`NetSocket`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html) 实例。这是一个代表了实际连接的套接字接口，它允许您读取和写入数据、以及执行各种其他操作，如关闭 Socket。

### 从Socket读取数据

您可以在Socket上调用 [`handler`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#handler-io.vertx.core.Handler-) 方法来设置用于读取数据的处理器。

每次 Socket 接收到数据时，会以 [`Buffer`](http://vertx.io/docs/apidocs/io/vertx/core/buffer/Buffer.html) 对象为参数调用处理器。

```java
NetServer server = vertx.createNetServer();
server.connectHandler(socket -> {
  socket.handler(buffer -> {
    System.out.println("I received some bytes: " + buffer.length());
  });
});
```

#### 向Socket中写入数据

您可使用 [`write`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#write-io.vertx.core.buffer.Buffer-) 方法写入数据到Socket：

```java
Buffer buffer = Buffer.buffer().appendFloat(12.34f).appendInt(123);
socket.write(buffer);

// 以UTF-8的编码方式写入一个字符串
socket.write("some data");

// 以指定的编码方式写入一个字符串
socket.write("some data", "UTF-16");
```

写入操作是异步的，可能调用 `write` 方法返回过后一段时间才会发生。

### 关闭处理器

若您想要在 Socket 关闭时收到通知，可以设置一个 [`closeHandler`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#closeHandler-io.vertx.core.Handler-)：

```java
socket.closeHandler(v -> {
  System.out.println("The socket has been closed");
});
```

### 处理异常

您可以设置一个 [`exceptionHandler`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#exceptionHandler-io.vertx.core.Handler-) 用以在发生任何异常的时候接收异常信息。

### Event Bus 写处理器

每个 Socket 会自动在Event Bus中注册一个处理器，当这个处理器中收到任意 `Buffer` 时，它会将数据写入到 Socket。

这意味着您可以通过向这个地址发送 `Buffer` 的方式，从不同的 Verticle 甚至是不同的 Vert.x 实例中向指定的 Socket 发送数据。

处理器的地址由 [`writeHandlerID`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#writeHandlerID--) 方法提供。

### 本地和远程地址

您可以通过 [localAddress](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#localAddress--)方法获取 [NetSocket](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html) 的本地地址，通过 [remoteAddress](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#remoteAddress--) 方法获取 [NetSocket](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html) 的远程地址（即连接的另一端的地址）。

### 发送文件或 Classpath 中的资源

您可以直接通过 [sendFile](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#sendFile-java.lang.String-) 方法将文件和 classpath 中的资源写入Socket。这种做法是非常高效的，它可以被操作系统内核直接处理。

请阅读 [从 Classpath 访问文件](http://vertxchina.github.io/vertx-translation-chinese/core/Core.html#从classpath访问文件) 章节了解类路径的限制或禁用它。

### 流式的Socket

[`NetSocket`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html) 接口继承了 [`ReadStream`](http://vertx.io/docs/apidocs/io/vertx/core/streams/ReadStream.html) 和 [`WriteStream`](http://vertx.io/docs/apidocs/io/vertx/core/streams/WriteStream.html) 接口，因此您可以将它套用（pump）到其他的读写流上。

有关更多信息，请参阅 [流和管道](http://vertxchina.github.io/vertx-translation-chinese/core/Core.html#流) 章节。

### 升级到 SSL/TLS 连接

一个非SSL/TLS连接可以通过[`upgradeToSsl`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetSocket.html#upgradeToSsl-io.vertx.core.Handler-)方法升级到SSL/TLS连接。

必须为服务器或客户端配置SSL/TLS才能正常工作。请参阅[SSL/TLS](http://vertx.io/docs/vertx-core/java/#ssl)章节来获取详细信息。

### 关闭 TCP 服务端

您可以调用 [`close`](http://vertx.io/docs/apidocs/io/vertx/core/net/NetServer.html#close--) 方法关闭服务端。关闭操作将关闭所有打开的连接并释放所有服务端资源。

关闭操作也是异步的，可能直到方法调用返回过后一段时间才会实际关闭。若您想在实际关闭完成时收到通知，那么您可以传递一个处理器。

当关闭操作完成后，绑定的处理器将被调用：

```java
server.close(res -> {
  if (res.succeeded()) {
    System.out.println("Server is now closed");
  } else {
    System.out.println("close failed");
  }
});
```

### Verticle中的自动清理

若您在 Verticle 内创建了 TCP 服务端和客户端，它们将会在Verticle 撤销时自动被关闭。

### 扩展 - 共享 TCP 服务端

任意一个 TCP 服务端中的处理器总是在相同的 Event Loop 线程上执行。这意味着如果您在多核的服务器上运行，并且只部署了一个实例，那么您的服务器上最多只能使用一个核。

为了利用更多的服务器核，您将需要部署更多的服务器实例。您可以在代码中以编程方式实例化更多（Server的）实例：

```java
for (int i = 0; i < 10; i++) {
  NetServer server = vertx.createNetServer();
  server.connectHandler(socket -> {
    socket.handler(buffer -> {
        //仅回传数据
      socket.write(buffer);
    });
  });
  server.listen(1234, "localhost");
}
```

如果您使用的是 Verticle，您可以通过在命令行上使用 `-instances` 选项来简单部署更多的服务器实例：

```
vertx run com.mycompany.MyVerticle -instances 10
```

或者使用编程方式部署您的 Verticle 时：

```java
DeploymentOptions options = new DeploymentOptions().setInstances(10);
vertx.deployVerticle("com.mycompany.MyVerticle", options);
```

一旦您这样做，您将发现echo服务器在功能上与之前相同，但是服务器上的所有核都可以被利用，并且可以处理更多的工作。

在这一点上，您可能会问自己：**如何让多台服务器在同一主机和端口上侦听？尝试部署一个以上的实例时真的不会遇到端口冲突吗？**

*Vert.x在这里有一点魔法。*

当您在与现有服务器相同的主机和端口上部署另一个服务器实例时，实际上它并不会尝试创建在同一主机/端口上侦听的新服务器实例。

相反，它内部仅仅维护一个服务器实例。当传入新的连接时，它以轮询的方式将其分发给任意一个连接处理器处理。

因此，Vert.x TCP 服务端可以水平扩展到多个核，并且每个实例保持单线程环境不变。



