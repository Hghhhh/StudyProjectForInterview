# [java 线程方法join的简单总结](https://www.cnblogs.com/lcplcpjava/p/6896904.html)



虽然关于讨论线程join方法的博客已经很多了，不过个人感觉挺多都讨论得不够全面，所以我觉得有必要对其进行一个全面的总结。

　　一、作用

　　Thread类中的join方法的主要作用就是同步，它可以使得线程之间的并行执行变为串行执行。具体看代码：

　　

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public class JoinTest {
    public static void main(String [] args) throws InterruptedException {
        ThreadJoinTest t1 = new ThreadJoinTest("小明");
        ThreadJoinTest t2 = new ThreadJoinTest("小东");
        t1.start();
        /**join的意思是使得放弃当前线程的执行，并返回对应的线程，例如下面代码的意思就是：
         程序在main线程中调用t1线程的join方法，则main线程放弃cpu控制权，并返回t1线程继续执行直到线程t1执行完毕
         所以结果是t1线程执行完后，才到主线程执行，相当于在main线程中同步t1线程，t1执行完了，main线程才有执行的机会
         */
        t1.join();
        t2.start();
    }

}
class ThreadJoinTest extends Thread{
    public ThreadJoinTest(String name){
        super(name);
    }
    @Override
    public void run(){
        for(int i=0;i<1000;i++){
            System.out.println(this.getName() + ":" + i);
        }
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

上面程序结果是先打印完小明线程，在打印小东线程；　　

上面注释也大概说明了join方法的作用：在A线程中调用了B线程的join()方法时，表示只有当B线程执行完毕时，A线程才能继续执行。注意，这里调用的join方法是没有传参的，join方法其实也可以传递一个参数给它的，具体看下面的简单例子：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public class JoinTest {
    public static void main(String [] args) throws InterruptedException {
        ThreadJoinTest t1 = new ThreadJoinTest("小明");
        ThreadJoinTest t2 = new ThreadJoinTest("小东");
        t1.start();
        /**join方法可以传递参数，join(10)表示main线程会等待t1线程10毫秒，10毫秒过去后，
         * main线程和t1线程之间执行顺序由串行执行变为普通的并行执行
         */
        t1.join(10);
        t2.start();
    }

}
class ThreadJoinTest extends Thread{
    public ThreadJoinTest(String name){
        super(name);
    }
    @Override
    public void run(){
        for(int i=0;i<1000;i++){
            System.out.println(this.getName() + ":" + i);
        }
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

上面代码结果是：程序执行前面10毫秒内打印的都是小明线程，10毫秒后，小明和小东程序交替打印。

所以，join方法中如果传入参数，则表示这样的意思：如果A线程中掉用B线程的join(10)，则表示A线程会等待B线程执行10毫秒，10毫秒过后，A、B线程并行执行。需要注意的是，jdk规定，join(0)的意思不是A线程等待B线程0秒，而是A线程等待B线程无限时间，直到B线程执行完毕，即join(0)等价于join()。

　　

　　二、join与start调用顺序问题

　　上面的讨论大概知道了join的作用了，那么，入股 join在start前调用，会出现什么后果呢？先看下面的测试结果

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public class JoinTest {
    public static void main(String [] args) throws InterruptedException {
        ThreadJoinTest t1 = new ThreadJoinTest("小明");
        ThreadJoinTest t2 = new ThreadJoinTest("小东");
        /**join方法可以在start方法前调用时，并不能起到同步的作用
         */
        t1.join();
        t1.start();
        //Thread.yield();
        t2.start();
    }

}
class ThreadJoinTest extends Thread{
    public ThreadJoinTest(String name){
        super(name);
    }
    @Override
    public void run(){
        for(int i=0;i<1000;i++){
            System.out.println(this.getName() + ":" + i);
        }
    }
}
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

上面代码执行结果是：小明和小东线程交替打印。

所以得到以下结论：join方法必须在线程start方法调用之后调用才有意义。这个也很容易理解：如果一个线程都没有start，那它也就无法同步了。

 

　　三、join方法实现原理

　　有了上面的例子，我们大概知道join方法的作用了，那么，join方法实现的原理是什么呢？

　　其实，join方法是通过调用线程的wait方法来达到同步的目的的。例如，A线程中调用了B线程的join方法，则相当于A线程调用了B线程的wait方法，在调用了B线程的wait方法后，A线程就会进入阻塞状态，具体看下面的源码：

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (millis == 0) {
            while (isAlive()) {
                wait(0);
            }
        } else {
            while (isAlive()) {
                long delay = millis - now;
                if (delay <= 0) {
                    break;
                }
                wait(delay);
                now = System.currentTimeMillis() - base;
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

从源码中可以看到：join方法的原理就是调用相应线程的wait方法进行等待操作的，例如A线程中调用了B线程的join方法，则相当于在A线程中调用了B线程的wait方法，当B线程执行完（或者到达等待时间），B线程会自动调用自身的notifyAll方法唤醒A线程，从而达到同步的目的。