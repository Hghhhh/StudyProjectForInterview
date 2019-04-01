## ReentrantLock特性（对比synchronized）

- 和synchronized一样是可重入的

- synchronized使用的使用jvm的原语monitorenter、monitorexit进行加锁，而reentrantlock使用的是lock和unlock方法

- 不过ReentrantLock提供了公平锁的功能，而且提供了读写锁的功能

- 支持tryLock，获取不到锁可以立即返回


## ReentrantLock(重入锁)

```
public class MyService {

    private Lock lock = new ReentrantLock();

    public void testMethod() {
        lock.lock();
        for (int i = 0; i < 5; i++) {
            System.out.println("ThreadName=" + Thread.currentThread().getName()
                    + (" " + (i + 1)));
        }
        lock.unlock();
    }

}
```

效果和synchronized一样，都可以同步执行，lock方法获得锁，unlock方法释放锁

## await方法

```
public class MyService {

    private Lock lock = new ReentrantLock();
    private Condition condition=lock.newCondition();
    public void testMethod() {
        
        try {
            lock.lock();
            System.out.println("开始wait");
            condition.await();
            for (int i = 0; i < 5; i++) {
                System.out.println("ThreadName=" + Thread.currentThread().getName()
                        + (" " + (i + 1)));
            }
        } catch (InterruptedException e) {
            // TODO 自动生成的 catch 块
            e.printStackTrace();
        }
        finally
        {
            lock.unlock();
        }
    }

}
```

- 通过创建Condition对象来使线程wait，必须先执行lock.lock方法获得锁

## signal方法

```
public void signal()
    {
        try
        {
            lock.lock();
            condition.signal();
        }
        finally
        {
            lock.unlock();
        }
    }
```

- condition对象的signal方法可以唤醒wait线程

## 创建多个condition对象

- 一个condition对象的signal（signalAll）方法和该对象的await方法是一一对应的，也就是一个condition对象的signal（signalAll）方法不能唤醒其他condition对象的await方法
- ReentrantLock类可以唤醒指定条件的线程，而object的唤醒是随机的

## Condition类和Object类

- Condition类的awiat方法和Object类的wait方法等效
- Condition类的signal方法和Object类的notify方法等效
- Condition类的signalAll方法和Object类的notifyAll方法等效

## Lock的公平锁和非公平锁

```
Lock lock=new ReentrantLock(true);//公平锁
Lock lock=new ReentrantLock(false);//非公平锁
```

- 公平锁指的是线程获取锁的顺序是按照加锁顺序来的，而非公平锁指的是抢锁机制，先lock的线程不一定先获得锁。

## ReentrantLock类的方法

- getHoldCount() 查询当前线程保持此锁的次数，也就是执行此线程执行lock方法的次数
- getQueueLength（）返回正等待获取此锁的线程估计数，比如启动10个线程，1个线程获得锁，此时返回的是9
- getWaitQueueLength（Condition condition）返回等待与此锁相关的给定条件的线程估计数。比如10个线程，用同一个condition对象，并且此时这10个线程都执行了condition对象的await方法，那么此时执行此方法返回10
- hasWaiters(Condition condition)查询是否有线程等待与此锁有关的给定条件(condition)，对于指定contidion对象，有多少线程执行了condition.await方法
- hasQueuedThread(Thread thread)查询给定线程是否等待获取此锁
- hasQueuedThreads()是否有线程等待此锁
- isFair()该锁是否公平锁
- isHeldByCurrentThread() 当前线程是否保持锁锁定，线程的执行lock方法的前后分别是false和true
- isLock()此锁是否有任意线程占用
- lockInterruptibly（）如果当前线程未被中断，获取锁
- tryLock（）尝试获得锁，仅在调用时锁未被线程占用，获得锁
- tryLock(long timeout TimeUnit unit)如果锁在给定等待时间内没有被另一个线程保持，则获取该锁

## tryLock和lock和lockInterruptibly的区别

- tryLock能获得锁就返回true，不能就立即返回false，tryLock(long timeout,TimeUnit unit)，可以增加时间限制，如果超过该时间段还没获得锁，返回false
- lock能获得锁就返回true，不能的话一直等待获得锁
- lock和lockInterruptibly，如果两个线程分别执行这两个方法，但此时中断这两个线程，前者不会抛出异常，而后者会抛出异常

## 读写锁

- ReentrantLock是完全互斥排他的，这样其实效率是不高的。

```
public void read() {
        try {
            try {
                lock.readLock().lock();
                System.out.println("获得读锁" + Thread.currentThread().getName()
                        + " " + System.currentTimeMillis());
                Thread.sleep(10000);
            } finally {
                lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
    }
```

- 读锁，此时多个线程可以或得读锁

```
public void write() {
        try {
            try {
                lock.writeLock().lock();
                System.out.println("获得写锁" + Thread.currentThread().getName()
                        + " " + System.currentTimeMillis());
                Thread.sleep(10000);
            } finally {
                lock.writeLock().unlock();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

- 写锁，此时只有一个线程能获得写锁

```
public void read() {
        try {
            try {
                lock.readLock().lock();
                System.out.println("获得读锁" + Thread.currentThread().getName()
                        + " " + System.currentTimeMillis());
                Thread.sleep(10000);
            } finally {
                lock.readLock().unlock();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public void write() {
        try {
            try {
                lock.writeLock().lock();
                System.out.println("获得写锁" + Thread.currentThread().getName()
                        + " " + System.currentTimeMillis());
                Thread.sleep(10000);
            } finally {
                lock.writeLock().unlock();
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

- 结果是读锁释放后才执行写锁的方法，说明读锁和写锁是互斥的

## 总结

- Lock类也可以实现线程同步，而Lock获得锁需要执行lock方法，释放锁需要执行unLock方法
- Lock类可以创建Condition对象，Condition对象用来是线程等待和唤醒线程，需要注意的是Condition对象的唤醒的是用同一个Condition执行await方法的线程，所以也就可以实现唤醒指定类的线程
- Lock类分公平锁和不公平锁，公平锁是按照加锁顺序来的，非公平锁是不按顺序的，也就是说先执行lock方法的锁不一定先获得锁
- Lock类有读锁和写锁，读读共享，写写互斥，读写互斥