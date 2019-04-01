Callable接口：



```
public interface Callable<V> {
    V call() throws Exception;
}
```

 

Runnable接口：



```
public interface Runnable {
    public abstract void run();
}
```

 

相同点：

1. 两者都是接口；（废话）
2. 两者都可用来编写多线程程序；
3. 两者都需要调用Thread.start()启动线程；

 

不同点：

1. 两者最大的不同点是：实现Callable接口的任务线程能返回执行结果；而实现Runnable接口的任务线程不能返回结果；
2. Callable接口的call()方法允许抛出异常；而Runnable接口的run()方法的异常只能在内部消化，不能继续上抛；

 

注意点：

- Callable接口支持返回执行结果，此时需要调用FutureTask.get()方法实现，此方法会阻塞主线程直到获取‘将来’结果；当不调用此方法时，主线程不会阻塞！

 

Callable工作的Demo：

[![复制代码](https://common.cnblogs.com/images/copycode.gi](javascript:void(0);)

```
package com.callable.runnable;

import java.util.concurrent.Callable;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;

/**
 * Created on 2016/5/18.
 */
public class CallableImpl implements Callable<String> {

    public CallableImpl(String acceptStr) {
        this.acceptStr = acceptStr;
    }

    private String acceptStr;

    @Override
    public String call() throws Exception {
        // 任务阻塞 1 秒
        Thread.sleep(1000);
        return this.acceptStr + " append some chars and return it!";
    }


    public static void main(String[] args) throws ExecutionException, InterruptedException {
        Callable<String> callable = new CallableImpl("my callable test!");
        FutureTask<String> task = new FutureTask<>(callable);
        long beginTime = System.currentTimeMillis();
        // 创建线程
        new Thread(task).start();
        // 调用get()阻塞主线程，反之，线程不会阻塞
        String result = task.get();
        long endTime = System.currentTimeMillis();
        System.out.println("hello : " + result);
        System.out.println("cast : " + (endTime - beginTime) / 1000 + " second!");
    }
}
```