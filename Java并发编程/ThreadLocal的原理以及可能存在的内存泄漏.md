# [ThreadLocal 定义，以及是否可能引起的内存泄露(threadlocalMap的Key是弱引用，用线程池有可能泄露)](https://www.cnblogs.com/aspirant/p/8991010.html)



ThreadLocal 也可以跟踪一个请求，从接收请求，处理请求，到返回请求，只要线程不销毁，就可以在线程的任何地方，调用这个参数，这是百度二面的题目，参考：

[Threadlocal 传递参数(百度二面)](https://www.cnblogs.com/aspirant/p/9183920.html)

总结：

1. JVM利用设置ThreadLocalMap的Key为弱引用，来避免内存泄露。
2. JVM利用调用remove、get、set方法的时候，回收弱引用。
3. 当ThreadLocal存储很多Key为null的Entry的时候，而不再去调用remove、get、set方法，那么将导致内存泄漏。
4. 当使用static ThreadLocal的时候，延长ThreadLocal的生命周期，那也可能导致内存泄漏。因为，static变量在类未加载的时候，它就已经加载，当线程结束的时候，static变量不一定会回收。那么，比起普通成员变量使用的时候才加载，static的生命周期加长将更容易导致内存泄漏危机。<http://www.importnew.com/22039.html>

　

**那么如何有效的避免呢？**

**事实上，在ThreadLocalMap中的set/getEntry方法中，会对key为null（也即是ThreadLocal为null）进行判断，如果为null的话，那么是会对value置为null的。我们也可以通过调用ThreadLocal的remove方法进行释放！**

threadlocal里面使用了一个存在弱引用的map,当释放掉threadlocal的强引用以后,map里面的value却没有被回收.而这块value永远不会被访问到了. 所以存在着内存泄露. 最好的做法是将调用threadlocal的remove方法.

　　在threadlocal的生命周期中,都存在这些引用. 看下图: 实线代表强引用,虚线代表弱引用.

　　![img](https://images2018.cnblogs.com/blog/137084/201805/137084-20180504154502152-1165477841.jpg)

　　每个thread中都存在一个map, map的类型是ThreadLocal.ThreadLocalMap. Map中的key为一个threadlocal实例. 这个Map的确使用了弱引用,不过弱引用只是针对key. 每个key都弱引用指向threadlocal. 当把threadlocal实例置为null以后,没有任何强引用指向threadlocal实例,所以threadlocal将会被gc回收. 但是,我们的value却不能回收,因为存在一条从current thread连接过来的强引用. 只有当前thread结束以后, current thread就不会存在栈中,强引用断开, Current Thread, Map, value将全部被GC回收.

　　所以得出一个结论就是只要这个线程对象被gc回收，就不会出现内存泄露，但**在threadLocal设为null和线程结束这段时间不会被回收的，就发生了我们认为的内存泄露**。其实这是一个对概念理解的不一致，也没什么好争论的。最要命的是线程对象不被回收的情况，这就发生了真正意义上的内存泄露。**比如使用线程池的时候，线程结束是不会销毁的，会再次使用的。就可能出现内存泄露**。　　

　　PS.Java为了最小化减少内存泄露的可能性和影响，在ThreadLocal的get,set的时候都会清除线程Map里所有key为null的value。所以最怕的情况就是，threadLocal对象设null了，开始发生“内存泄露”，然后使用线程池，这个线程结束，线程放回线程池中不销毁，这个线程一直不被使用，或者分配使用了又不再调用get,set方法，那么这个期间就会发生真正的内存泄露。 

## 应用场景

最常见的ThreadLocal使用场景为 用来解决 数据库连接、Session管理等。如

```
private static ThreadLocal < Connection > connectionHolder = new ThreadLocal < Connection > () {
    public Connection initialValue() {
        return DriverManager.getConnection(DB_URL);
    }
};

public static Connection getConnection() {
    return connectionHolder.get();
}
private static final ThreadLocal threadSession = new ThreadLocal();

public static Session getSession() throws InfrastructureException {
    Session s = (Session) threadSession.get();
    try {
        if (s == null) {
            s = getSessionFactory().openSession();
            threadSession.set(s);
        }
    } catch (HibernateException ex) {
        throw new InfrastructureException(ex);
    }
    return s;
} 
```

## 一、目录

​     1、ThreadLocal是什么？有什么用？

​     2、ThreadLocal源码简要总结？

​     3、ThreadLocal为什么会导致内存泄漏？

## 二、ThreadLocal是什么？有什么用？

引入话题：在并发条件下，如何正确获得共享数据？举例：假设有多个用户需要获取用户信息，一个线程对应一个用户。在mybatis中，session用于操作数据库，那么设置、获取操作分别是session.set()、session.get()，如何保证每个线程都能正确操作达到想要的结果？

```
/**
 * 回顾synchronized在多线程共享线程的问题
 * @author qiuyongAaron
 */
public class ThreadLocalOne {
     volatile Person person=new Person();
 
     public  synchronized String setAndGet(String name){
          //System.out.print(Thread.currentThread().getName()+":");
           person.name=name;
           //模拟网络延迟
           try {
                TimeUnit.SECONDS.sleep(2);
           } catch (InterruptedException e) {
                e.printStackTrace();
           }
           return person.name;
     }
 
     public static void main(String[] args) {
           ThreadLocalOne  threadLocal=new ThreadLocalOne();
           new Thread(()->System.out.println(threadLocal.setAndGet("arron")),"t1").start();
           new Thread(()->System.out.println(threadLocal.setAndGet("tony")),"t2").start();
     }
}
 
class Person{
     String name="tom";
     public Person(String name) {
           this.name=name;
     }
 
     public Person(){}
}
 
运行结果：
无synchronized：
t1:tony
t2:tony
 
有synchronized：
t1:arron
t2:tony
```



步骤分析：

1. 无synchronized的时候，因为非原子操作，显然不是预想结果，可参考我关于synchronized的讨论。
2. 现在，我们的需求是：每个线程独立的设置获取person信息，不被线程打扰。
3. 因为，person是共享数据，用同步互斥锁synchronized，当一个线程访问共享数据的时候，其他线程堵塞，不再多余赘述。

 

通过举例问题，可能大家又会很疑惑？

mybatis、hibernate是如何实现的呢？

synchronized不会很消耗资源，当成千上万个操作的时候，承受并发不说，数据返回延迟如何确保用户体验？

 

ThreadLocal是什么？有什么用？

```
/**
 * 谈谈ThreadLocal的作用
 * @author qiuyongAaron
 */
public class ThreadLocalThree {
     ThreadLocal<Person> threadLocal=new ThreadLocal<Person>();
     public String setAndGet(String name){
           threadLocal.set(new Person(name));
           try {
                TimeUnit.SECONDS.sleep(2);
           } catch (InterruptedException e) {
                e.printStackTrace();
           }
           return threadLocal.get().name;
     }
 
     public static void main(String[] args) {
           ThreadLocalThree  threadLocal=new ThreadLocalThree();
           new Thread(()->System.out.println("t1:"+threadLocal.setAndGet("arron")),"t1").start();
           new Thread(()->System.out.println("t2:"+threadLocal.setAndGet("tony")),"t2").start();
     }
}
运行结果：
t1:arron
t2:tony
```

分析：

1、根据预期结果，那ThreadLocal到底是什么？

回顾Java内存模型：

　　![img](https://images2015.cnblogs.com/blog/1066658/201706/1066658-20170628211941414-1622377523.png)

​      在虚拟机中，堆内存用于存储共享数据（实例对象），堆内存也就是这里说的主内存。

​     每个线程将会在堆内存中开辟一块空间叫做线程的工作内存，附带一块缓存区用于存储共享数据副本。那么，共享数据在堆内存当中，线程通信就是通过主内存为中介，线程在本地内存读并且操作完共享变量操作完毕以后，把值写入主内存。

1. ThreadLocal被称为线程局部变量，说白了，他就是线程工作内存的一小块内存，用于存储数据。
2. 那么，ThreadLocal.set()、ThreadLocal.get()方法，就相当于**把数据存储于线程本地，取也是在本地内存读取**。**就不会像synchronized需要频繁的修改主内存的数据，再把数据复制到工作内存，也大大提高访问效率**。



2、ThreadLocal到底有什么用？

1. 回到最开始的举例，也就等价于mabatis、hibernate为什么要使用threadlocal来存储session？
2. 作用一**：因为线程间的数据交互是通过工作内存与主存的频繁读写完成通信，然而存储于线程本地内存，提高访问效率，避免线程阻塞造成cpu吞吐率下降**。
3. 作用二：**在多线程中，每一个线程都需要维护session，轻易完成对线程独享资源的操作**。

 

总结：

​     **Threadlocal是什么？在堆内存中，每个线程对应一块工作内存，threadlocal就是工作内存的一小块内存。**

​     **Threadlocal有什么用？threadlocal用于存取线程独享数据，提高访问效率。**

## 三、ThreadLocal源码简要总结？

那有同学可能还是有点云里雾里，感觉还是没有吃透？那线程内部如何去保证线程独享数据呢？

 

在这里，我只做简要总结，若有兴趣，可参考文章尾部的文章链接。重点看get、set方法。

```
 public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
 }
```

分析：

1. 一个线程对应一个ThreadLocalMap ，可以存储多个ThreadLocal对象。
2. ThreadLocal对象作为key、独享数据作为value。
3. ThreadLocalMap可参考HashMap，在ThreadMap里面存在Entry数组也就是一个Entry一个键值对。

```
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

分析：

1. 一个线程对应一个ThreadLocalMap，get()就是当前线程获取自己的ThreadLocalMap。
2. 线程根据使用那一小块的threadlocal，根据ThreadLocal对象作为key，去获取存储于ThreadLocalMap中的值。

 

总结：

​     回顾一下，我们在单线程中如何使用HashMap的？hashMap根据数组+链表来实现HashMap，一个key对应一个value。那么，我们抽象一下，Threadlocal也相当于在多线程中的一种HashMap用法，相当于对ThradLocal的操作也就如单线程操作一样。

​     总之，ThreadLocal就是堆内存的一块小内存，它用ThreadLocalMap维护ThreadLocal对象作为key，独享数据作为value的东西。

 

## 四、ThreadLocal为什么会导致内存泄漏？

**synchronized是用时间换空间(牺牲时间)、ThreadLocal是用空间换时间(牺牲空间)**，为什么这么说？

**因为synchronized操作数据，只需要在主存存一个变量即可，就阻塞等共享变量，而ThreadLocal是每个线程都创建一块小的堆工作内存**。显然，印证了上面的说法。

 

一个线程对应一块工作内存，线程可以存储多个ThreadLocal。那么假设，开启1万个线程，每个线程创建1万个ThreadLocal，也就是每个线程维护1万个ThreadLocal小内存空间，而且当线程执行结束以后，假设这些ThreadLocal里的Entry还不会被回收，那么将很容易导致堆内存溢出。

 

怎么办？难道JVM就没有提供什么解决方案吗？

ThreadLocal当然有想到，所以他们把ThreadLocal里的Entry设置为弱引用，当垃圾回收的时候，回收ThreadLocal。

什么是弱引用？

1. Key使用强引用：也就是上述说的情况，引用ThreadLocal的对象被回收了，ThreadLocal的引用ThreadLocalMap的Key为强引用并没有被回收，如果不手动回收的话，ThreadLocal将不会回收那么将导致内存泄漏。
2. Key使用弱引用：引用的ThreadLocal的对象被回收了，**ThreadLocal的引用ThreadLocalMap的Key为弱引用，如果内存回收，那么将ThreadLocalMap的Key将会被回收，ThreadLocal也将被回收。value在ThreadLocalMap调用get、set、remove的时候就会被清除**。
3. 比较两种情况，我们可以发现：由于`ThreadLocalMap`的生命周期跟`Thread`一样长，如果都没有手动删除对应`key`，都会导致内存泄漏，但是使用弱引用可以多一层保障：**弱引用ThreadLocal不会内存泄漏，对应的value在下一次ThreadLocalMap调用set,get,remove的时候会被清除**。

 

那按你这么说，既然JVM有保障了，还有什么内存泄漏可言？

ThreadLocalMap使用ThreadLocal对象作为弱引用，当垃圾回收的时候，ThreadLocalMap中Key将会被回收，也就是将Key设置为null的Entry。**如果线程迟迟无法结束，也就是ThreadLocal对象将一直不会回收，回顾到上面存在很多线程+TheradLocal，那么也将导致内存泄漏。(内存泄露的重点)**

 

其实，在ThreadLocal中，当调用remove、get、set方法的时候，会清除为null的弱引用，也就是回收ThreadLocal。

 ThreadLocal提供一个线程（Thread）局部变量，访问到某个变量的每一个线程都拥有自己的局部变量。说白了，ThreadLocal就是想在多线程环境下去保证成员变量的安全。 

**ThreadLocal提供的方法**

 

![img](https://upload-images.jianshu.io/upload_images/4943997-27579dffea7a65b1.png)

ThreadLocal API

> **对于ThreadLocal而言，常用的方法，就是get/set/initialValue方法。**

**我们先来看一个例子**

 

![img](https://upload-images.jianshu.io/upload_images/4943997-1ac9e79efbf24de4.png)

demo

**运行结果**

 

![img](https://upload-images.jianshu.io/upload_images/4943997-b77b24498ed0a445.png)

是你想象中的结果么？

**很显然，在这里，并没有通过ThreadLocal达到线程隔离的机制，可是ThreadLocal不是保证线程安全的么？这是什么鬼？**

**虽然，ThreadLocal让访问某个变量的线程都拥有自己的局部变量，但是如果这个局部变量都指向同一个对象呢？这个时候ThreadLocal就失效了。仔细观察下图中的代码，你会发现，threadLocal在初始化时返回的都是同一个对象a！**

 

# 看一看ThreadLocal源码

**我们直接看最常用的set操作：**

 

![img](https://upload-images.jianshu.io/upload_images/4943997-a29c9adf22113fb8.png)

set

 

 

![img](https://upload-images.jianshu.io/upload_images/4943997-586c3b6a3f8079e4.png)

线程局部变量

 

 

![img](https://upload-images.jianshu.io/upload_images/4943997-bbfde3b2a69b2268.png)

createMap

 

**你会看到，set需要首先获得当前线程对象Thread；**

**然后取出当前线程对象的成员变量ThreadLocalMap；**

**如果ThreadLocalMap存在，那么进行KEY/VALUE设置，KEY就是ThreadLocal；**

**如果ThreadLocalMap没有，那么创建一个；**

**说白了，当前线程中存在一个Map变量，KEY是ThreadLocal，VALUE是你设置的值。**

**看一下get操作：**

 

![img](https://upload-images.jianshu.io/upload_images/4943997-31a8fa43c397305f.png)

get

> **这里其实揭示了ThreadLocalMap里面的数据存储结构，从上面的代码来看，ThreadLocalMap中存放的就是Entry，Entry的KEY就是ThreadLocal，VALUE就是值。**

**ThreadLocalMap.Entry：**

 

![img](https://upload-images.jianshu.io/upload_images/4943997-b0f5157e62b1ed2d.png)

弱引用？

 

**在JAVA里面，存在强引用、弱引用、软引用、虚引用。这里主要谈一下强引用和弱引用。**

强引用，就不必说了，类似于：

A a = new A();

B b = new B();

考虑这样的情况：

**C c = new C(b);**

**b = null;**

考虑下GC的情况。要知道b被置为null，那么是否意味着一段时间后GC工作可以回收b所分配的内存空间呢？答案是否定的，因为即便b被置为null，但是c仍然持有对b的引用，而且还是强引用，所以GC不会回收b原先所分配的空间！既不能回收利用，又不能使用，这就造成了**内存泄露**。

那么如何处理呢？

**可以c = null;也可以使用弱引用！（WeakReference w = new WeakReference(b);）**

分析到这里，我们可以得到：

 

![img](https://upload-images.jianshu.io/upload_images/4943997-cac8c7ca612012f7.png)

内存结构图

**这里我们思考一个问题：ThreadLocal使用到了弱引用，是否意味着不会存在内存泄露呢？** 

**首先来说，如果把ThreadLocal置为null，那么意味着Heap中的ThreadLocal实例不在有强引用指向，只有弱引用存在，因此GC是可以回收这部分空间的，也就是key是可以回收的。但是value却存在一条从Current Thread过来的强引用链。因此只有当Current Thread销毁时，value才能得到释放。**

**因此，只要这个线程对象被gc回收，就不会出现内存泄露，但在threadLocal设为null和线程结束这段时间内不会被回收的，就发生了我们认为的内存泄露。最要命的是线程对象不被回收的情况，比如使用线程池的时候，线程结束是不会销毁的，再次使用的，就可能出现内存泄露。**

**那么如何有效的避免呢？**

**事实上，在ThreadLocalMap中的set/getEntry方法中，会对key为null（也即是ThreadLocal为null）进行判断，如果为null的话，那么是会对value置为null的。我们也可以通过调用ThreadLocal的remove方法进行释放！**

 

 

参考：[ThreadLocal可能引起的内存泄露](http://www.cnblogs.com/onlywujun/p/3524675.html)

参考：[对ThreadLocal实现原理的一点思考](https://www.jianshu.com/p/ee8c9dccc953)

参考：[并发编程（四）：ThreadLocal从源码分析总结到内存泄漏](http://www.cnblogs.com/qiuyong/p/7091689.html)

转自：https://www.cnblogs.com/aspirant/p/8991010.html

深度好文！强烈推荐！