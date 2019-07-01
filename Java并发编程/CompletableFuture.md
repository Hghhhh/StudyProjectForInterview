## Future的局限性

JDK5新增了Future接口，用于描述一个异步计算的结果。虽然 Future 以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。阻塞的方式显然和我们的异步编程的初衷相违背，轮询的方式又会耗费无谓的 CPU 资源，而且也不能及时地得到计算结果。

Future接口可以构建异步应用，但依然有其局限性。它很难直接表述多个Future 结果之间的依赖性。实际开发中，我们经常需要达成以下目的：

1. 将多个异步计算的结果合并成一个
2. 等待Future集合中的所有任务都完成
3. Future完成事件（即，任务完成以后触发执行动作）
4. ......

## CompletableFuture介绍

- 在Java8中，CompletableFuture提供了非常强大的Future的扩展功能，可以帮助我们简化异步编程的复杂性，并且提供了函数式编程的能力，可以通过回调的方式处理计算结果，也提供了转换和组合 CompletableFuture 的结果的方法。
- 它可能代表一个明确完成的Future，也有可能代表一个完成阶段（ CompletionStage ），它支持在计算完成以后触发一些函数或执行某些动作。
- 它实现了Future和CompletionStage接口

## CompletableFuture的基本用法

### 1.创建CompletableFuture对象

```java
public static CompletableFuture<Void> runAsync(Runnable runnable);
public static CompletableFuture<Void> runAsync(Runnable runnable,Executor executor);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,Executor executor);
```

在上面所列的这些方法中，以“Async”结尾并且没有指定Executor方法的，将会默认使用ForkJoinPooll.commonPool()作为它的线程池执行异步代码。

### 2.调用完成的处理方法

```java
CompletableFuture<Void> thenAccept(Consumer<? super T> block);
CompletableFuture<Void> thenAccept(Runnable action);
```

### 3.输出转换

由于实现了非阻塞的异步回调方法，我们并不需要等待一个调用完成，再通过阻塞或轮训的方式得到结果，而是只要告诉CompletableFuture,当调用完成的时候执行某个Funtion就可以了，同时还可以输出转换。

```java
public CompletableFuture thenApply(Function fn);
public CompletableFuture thenApplyAsync(Function fn);
public CompletableFuture thenApplyAsync(Function fn,Executor executor);
```

这一组函数的功能是在原来的CompletableFuture调用完成之后，将结果传递给函数“fn”，然后将“fn”的结果作为新的CompletableFuture调用结果，相当于将`CompletableFuture<T>`

转换为`CompletableFuture<U>`.

这三个函数中不以“Async”结尾的方法，由原来的线程池执行，以“Async”结尾的方法将由默认的ForkJoinPool.commonPool()或者指定的线程池executor执行。

eg:

```java
public CompletableFuture<ModelAndView> findById(Long id){
  return orderFuture.findById(id).thenApply(o->
         new ModelAndView("order/show","order","o"));
}
```

### 4.链接两个调用

```java
<U> CompletableFuture<U> thenCompose(Function<? super T,CompletableFuture<U> fn);
```

eg:

```java
CompletabelFuture.supplyAsync(()->goodService.findById(id)).thenCompose(goods->CompletableFuture.supplyAsync()->{
    GoodsQo goodsQo = new Gson().fromJson(goods,goodsQo.class);
    String sorts = sortsService.findById(goodsQo.getSortsId());
    return sorts;
});
```

### 5.结合两个转换

将两个转换的结果结合起来，提供给另一个调用使用：

```java
<U,V> CompletabelFuture<V> thenCombine(CompletableFuture<? extends U> other,BiFunction<? super T,? super U,?extends V> fn)
```

eg:

```java
public CompletableFuture<String> getMessage(){
    return CompletableFuture.supplyAsync(()->{
        return "I am here";
    }).thenCombine(CompletableFuture.supplyAsync(()->"with you"),(ret,msg)->ret+msg).
        thenCompose(p->CompletableFuture.supplyAsync(()->p+"and everyone"));
}
```

通过结合转换，再进行链接，最后输出的结果是“I am here with you and everyone”;

### 6.多种结合的调用

```java
static CompletableFuture<Void> allOf(CompletableFuture<?>...cfs);
static CompletableFuture<Object> anyOf(CompletableFuture<?>...cfs);
```

最后需要指出的是：不管在创建CompletableFuture的是时候，我们使用了多少层级的组合调用，这些调用并不是嵌套的，而是扁平的，即每一个调用都在非阻塞的情况下自主地进行处理。同时，我们仅仅需要一步操作，就能获得所有这些组合的调用结果。

### 7.使用CompletableFuture实现Service类

```java
@Component
public class OrderFuture{
    @Autowired
    private OrderRestService orderService;
    @AsyncTimed
    public CompletableFuture<String> findById(long id){
        return CompletableFuture.supplyAsync(()->{
            return orderService.findById(id);
        });
    }
    
        @AsyncTimed
    public CompletableFuture<String> findByPage(OrderQo orderQo){
        return CompletableFuture.supplyAsync(()->{
            return orderService.findPage(orderQo);
        });
    }
    
}
```

在这个组件设计中，使用了CompletableFuture的一些函数式调用方法，实现了对服务层的订单服务的高并发调用。





参考：

《Spring Cloud与Docker高并发微服务架构设计与实施》

[CompletableFuture基本用法](https://www.cnblogs.com/cjsblog/p/9267163.html)

