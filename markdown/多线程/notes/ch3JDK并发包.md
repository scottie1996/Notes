[TOC]



## 第3章　JDK并发包	70

### 3.1　多线程的团队协作：同步控制	70

#### Lock API

Lock是一个接口，方法定义如下

```java
void lock() // 如果锁可用就获得锁，如果锁不可用就阻塞直到锁释放
void lockInterruptibly() // 和 lock()方法相似, 但阻塞的线程可中断，抛出 java.lang.InterruptedException异常
boolean tryLock() // 非阻塞获取锁;尝试获取锁，如果成功返回true
boolean tryLock(long timeout, TimeUnit timeUnit) //带有超时时间的获取锁方法
void unlock() // 释放锁
```

##### Lock的实现

实现Lock接口的类有很多，以下为几个常见的锁实现

- ReentrantLock：表示重入锁，它是唯一一个实现了Lock接口的类。重入锁指的是线程在获得锁之后，再次获取该锁不需要阻塞，而是直接关联一次计数器增加重入次数
- ReentrantReadWriteLock：重入读写锁，它实现了ReadWriteLock接口，在这个类中维护了两个锁，一个是ReadLock，一个是WriteLock，他们都分别实现了Lock接口。读写锁是一种适合读多写少的场景下解决线程安全问题的工具，基本原则是：`读和读不互斥、读和写互斥、写和写互斥`。也就是说涉及到影响数据变化的操作都会存在互斥。
- StampedLock： stampedLock是JDK8引入的新的锁机制，可以简单认为是读写锁的一个改进版本，读写锁虽然通过分离读和写的功能使得读和读之间可以完全并发，但是读和写是有冲突的，如果大量的读线程存在，可能会引起写线程的饥饿。stampedLock是一种乐观的读策略，使得乐观锁完全不会阻塞写线程

##### ReentrantLock的简单实用

如何在实际应用中使用ReentrantLock呢？我们通过一个简单的demo来演示一下

```java
public class Demo {
    private static int count=0;
    static Lock lock=new ReentrantLock();
    public static void inc(){
        lock.lock();
        try {
            Thread.sleep(1);
            count++;
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally{
            lock.unlock();
        }
    }
```

这段代码主要做一件事，就是通过一个静态的`incr()`方法对共享变量`count`做连续递增，在没有加同步锁的情况下多线程访问这个方法一定会存在线程安全问题。所以用到了`ReentrantLock`来实现同步锁，并且在finally语句块中释放锁。
**那么我来引出一个问题，大家思考一下**

> 多个线程通过lock竞争锁时，当竞争失败的锁是如何实现等待以及被唤醒的呢?





#### 3.1.1　synchronized的功能扩展：重入锁`ReentrantLock`	71

- `ReentrantLock`是`java.util.concurrent.locks.ReentrantLock`下面的包，`ReentrantLock `实现了 `Lock `接口，`ReentrantLock` 只是 `Lock `接口的一个实现而已。它们都是` java.util.concurrent `包里面的内容（俗称 JUC、并发包），也都是 JDK 1.5 开始加入的。

  

- 可重入锁代表着**一个线程**可以多次获得同一把锁

  #### 可重入锁重入锁最重要的几个方法:

  #### 1.中断响应，**lockInterruptibly()**

  对于`synchronized`来说,一个线程要么获取锁,要么等待锁,但是`ReentrantLock提供了另外一种可能:线程被中断(线程在等待锁的过程中,可以取消对锁的请求)下面的例子是对于死锁的处理.

  ```java
  
  import java.util.concurrent.locks.ReentrantLock;
  
  public class IntLock implements Runnable {
      public static ReentrantLock lock1 = new ReentrantLock();
      public static ReentrantLock lock2 = new ReentrantLock();
      int lock;
  
      public IntLock(int lock) {
          this.lock = lock;
      }
  
  
      @Override
      public void run() {
          try {
              if (this.lock == 1) {
                  lock1.lockInterruptibly();
                  Thread.sleep(500);
                  lock2.lockInterruptibly();
              } else {
                  lock2.lockInterruptibly();
                  Thread.sleep(500);
                  lock1.lockInterruptibly();
              }
          } catch (InterruptedException e) {
              e.printStackTrace();
          } finally {
              if (lock1.isHeldByCurrentThread())
                  lock1.unlock();
              if (lock2.isHeldByCurrentThread())
                  lock2.unlock();
              System.out.println(this.lock + "线程退出");
          }
      }
  
      public static void main(String[] args) throws InterruptedException {
          IntLock r1 = new IntLock(1);
          IntLock r2 = new IntLock(2);
          Thread t1 = new Thread(r1);
          Thread t2 = new Thread(r2);
          t1.start();
          t2.start();
          Thread.sleep(1000);
          t2.interrupt();
      }
  }
  ```

- #### 锁申请等待限时，**tryLock()**

  `ReentrantLock.tryLock`接收两个参数,一个表示等待时长,一个表示计时单位.由于其中一个线程获取锁后会等待六秒,另一个线程无法获取锁,所以请求会失败.

- ```java
  public class TimeLock implements Runnable {
      public static ReentrantLock lock = new ReentrantLock();
  
      public void run() {
          try {
              if (lock.tryLock(5, TimeUnit.SECONDS)) {
                  Thread.sleep(6000);
              } else {
                  System.out.println("Get lock failed");
              }
          } catch (InterruptedException e) {
              e.printStackTrace();
          } finally {
              if(lock.isHeldByCurrentThread())
                  lock.unlock();
          }
      }
  
      public static void main(String [] args){
          TimeLock lock1 = new TimeLock();
          Thread t1 = new Thread(lock1);
          Thread t2 = new Thread(lock1);
          t1.start();
          t2.start();
      }
  }
  ```

- #### 不带参数的**tryLock()**

- 这种方法也可以不带参数运行,这时如果没请求成功,会立即返回失败.

- #### 公平锁

- **公平锁需要维护一个有序队列**，成本高，默认是非公平锁，[FairLock](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s1/FairLock.java)

重入锁的实现主要包含三个要素：

- 原子状态，CAS
- 等待队列
- 阻塞原语park()和unpark()

#### 3.1.2　重入锁的好搭档：Condition条件	80

- wait和notify是和synchronize合作使用的，**newCondition()**方法是和重入锁相关联的，方法返回一个这个锁的 Condition 实例，可以实现 `synchronized` 关键字类似 `wait/ notify` 实现多线程通信的功能，不过这个比 `wait/ notify` 要更灵活，更强大！

- ```java
  import java.util.concurrent.locks.Condition;
  import java.util.concurrent.locks.ReentrantLock;
  
  public class ReenterLockCondition implements Runnable {
      public static ReentrantLock lock = new ReentrantLock();
      public static Condition condition = lock.newCondition();
  
      public void run() {
          try{
              lock.lock();
              condition.await();
              System.out.println("Thread is going on");
          } catch (InterruptedException e) {
              e.printStackTrace();
          }finally {
              lock.unlock();
          }
      }
  
      public static void main(String [] args) throws InterruptedException {
          ReenterLockCondition r1 = new ReenterLockCondition();
          Thread t1= new Thread(r1);
          t1.start();
          Thread.sleep(2000);
          lock.lock();
          condition.signal();
          lock.unlock();
      }
  }
  ```

Condition接口提供的基本方法

- await()方法会使当前线程等待，**同时释放当前锁**，当其他线程中使用signal()或者 signalAll()方法时，线程会重新获得锁并继续执行。或者当线程被中断时，也能跳出等待。这和Object.wait()方法很相似。
- awaitUninterruptibly()方法与await()方法基本相同，但是它并**不会在等待过程中响应中断。**
- singal()方法用于唤醒一个在等待中的线程。相对的singalAll()方法会唤醒所有在等待中 的线程。这和Obejct.notify()方法很类似。



#### 3.1.3　允许多个线程同时访问：信号量（Semaphore）	类 83

无论是synchronized还是reentrantLock都只允许一个线程访问一个资源,信号量却可以指定多个线程同时访问某个资源.

Semaphore类位于`java.util.concurrent`包下，它提供了2个构造器：

**构造函数：**

```java
public Semaphore(int permits) //第一个参数指定同时访问的线程数
public Semaphore(int permits, boolean fair) //第二个参数可以指定是否公平
```

**主要方法：**

```java
public void acquire() //尝试获取准入许可，如没有获得，则线程会等待，直到其他线程释放一个许可或者当前线程被中断
public void acquireUninterruptibly() //一样，但是不响应中断
public void release () //释放一个许可
public void release(int permits) { }    //释放permits个许可
```

这4个方法都会被阻塞，如果想要立即返回的方法，可以用

```java
public boolean tryAcquire() { };   //尝试获取一个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire( long timeout, TimeUnit unit)  throws InterruptedException { }; //尝试获取一个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
public boolean tryAcquire(int permits) { };  //尝试获取permits个许可，若获取成功，则立即返回true，若获取失败，则立即返回false
public boolean tryAcquire( int permits,  long timeout, TimeUnit unit)  throws InterruptedException { }; //尝试获取permits个许可，若在指定的时间内获取成功，则立即返回true，否则则立即返回false
```

```java

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

public class SemapDemo implements Runnable {
    //5个一组输出
    final Semaphore semp = new Semaphore(5);

    public void run() {
        try {
            semp.acquire();
            //模拟耗时操作
            Thread.sleep(2000);
            System.out.println(Thread.currentThread().getId() + " done!");
            semp.release();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String []args){
        ExecutorService exec = Executors.newFixedThreadPool(20); //线程池
        final SemapDemo demo = new SemapDemo();
        for(int i=0;i<20;i++){
            exec.submit(demo);
        }
    }
}
```



#### 3.1.4　ReadWriteLock读写锁 ReentrantReadWriteLock()	85



```java

import java.util.Random;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteLockDemo {
    private static Lock lock = new ReentrantLock();
    private static ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private static Lock readLock = readWriteLock.readLock();
    private static Lock writeLock = readWriteLock.writeLock();
    private int value;

    public Object handleRead(Lock lock) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000); //模拟读操作
            System.out.println("read success");
            return value;
        } finally {
            lock.unlock();
        }
    }

    public void handleWrite(Lock lock, int index) throws InterruptedException {
        try {
            lock.lock();
            Thread.sleep(1000);//模拟写操作
            value = index;
            System.out.println("write success");
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        final ReadWriteLockDemo demo = new ReadWriteLockDemo();
        Runnable readRunnable = new Runnable() {
            public void run() {
                try {
                    demo.handleRead(readLock);
                    //demo.handleRead(lock);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        Runnable writeRunnable = new Runnable() {

            public void run() {
                try {
                    demo.handleWrite(writeLock, new Random().nextInt());
                    //demo.handleWrite(lock, new Random().nextInt());
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        for (int i = 0; i < 18; i++) {
            new Thread(readRunnable).start();
        }

        for (int i = 18; i < 20; i++) {
            new Thread(writeRunnable).start();
        }
    }
}
```

- 实践了一下果然速度差距相当大
- [ReadWriteLockDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s1/ReadWriteLockDemo.java)

#### 3.1.5　倒计时器：CountDownLatch	87

CountDownLatch类位于java.util.concurrent包下，利用它可以实现类似计数器的功能。比如有一个任务A，它要等待其他4个任务执行完毕之后才能执行，此时就可以利用CountDownLatch来实现这种功能了。

CountDownLatch类只提供了一个构造器：

```java
public  CountDownLatch( int count) { }; //参数count为计数值
```

 　然后下面这3个方法是CountDownLatch类中最重要的方法：

```java
public void await()  throws InterruptedException { }; //调用await()方法的线程会被挂起，它会等待直到count值为0才继续执行 public boolean await(long timeout, TimeUnit unit) throws InterruptedException { };//和await()类似，只不过等待一定的时间后count值还没变为0的话就会继续执行
public void countDown() { }; //将count值减1
```

 　

#### 3.1.6　循环栅栏：CyclicBarrier	89

字面意思回环栅栏，通过它可以实现让一组线程等待至某个状态之后再全部同时执行。叫做回环是因为当所有等待线程都被释放以后，CyclicBarrier可以被重用。我们暂且把这个状态就叫做barrier，当调用await()方法之后，线程就处于barrier了。

　　CyclicBarrier类位于java.util.concurrent包下，CyclicBarrier提供2个构造器：

```java
public CyclicBarrier( int parties, Runnable barrierAction) {
}
public CyclicBarrier( int parties) {
}
```

　　参数parties指让多少个线程或者任务等待至barrier状态；参数barrierAction为当这些线程都达到barrier状态时会执行的内容。

　　然后CyclicBarrier中最重要的方法就是await方法，它有2个重载版本：

```java
public int await()  throws InterruptedException, BrokenBarrierException { };
public int await( long timeout, TimeUnit unit) throws InterruptedException,BrokenBarrierException,TimeoutException { };
```

 　第一个版本比较常用，用来挂起当前线程，直至所有线程都到达barrier状态再同时执行后续任务；

　第二个版本是让这些线程等待至一定的时间，如果还有线程没有到达barrier状态就直接让到达barrier的线程执行后续任务。

#### 3.1.7　线程阻塞工具类：LockSupport	92

- LockSupport的park和unpark函数，和suspend和resume函数比，不存在先后顺序，也就不会由于先resume后suspend导致程序死掉
- 和信号量的区别是它只有一个许可，而信号量可以有多个
- [LockSupportDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s1/LockSupportDemo.java)
- [LockSupportIntDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s1/LockSupportIntDemo.java)
          
### 3.2　线程复用：线程池	95

#### 3.2.1　什么是线程池	96

1. 使用线程池的目的∶避免线程的创建和销毁带来的性能开销。
2. 免大量线程间因互相抢占系统资源导致的阻塞现象。
3. 能对线程进行简单管理，提供定时执行、间隔执行的功能。



#### 3.2.2　不要重复发明轮子：JDK对线程池的支持	97

- 类图要好好看看，建议阅读jdk源代码了解下关系，整理如下：
    - `ExecutorService`继承了线程池顶级接口`Executor`
    - `AbstractExecutorService`实现了`ExecutorService`接口
    - `Executors`生成了实现类`ThreadPoolExecutor`
    
- Executors中包含的一部分线程池类型：
    1. newFixedThreadPool
    
        固定大小的线程池，每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小，线程池的大小一旦达到最大值就会保持不变。
        如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。可以控制线程最大并发量（核心线程数等于最大线程数）
    
        队列大小没有限制
    
        <img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201029172710520.png" alt="image-20201029172710520" style="zoom: 33%;" />
    
    2. newSingleThreadExecutor
    
        一个单线程池，池中只有一个线程在工作，串行执行所有任务。保证任务执行顺序按照任务的提交顺序执行。
    
        这个线程因为异常结束，会有一个新的线程来代替它。
    
        ![image-20201029173505702](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201029173505702.png)
    
        
    
    3. newCachedThreadPool
    
        可根据需要创建新线程，线程数没有限制。
    
        如果以前创建的线程可用，那么先会重用以前的线程；如果没有可用的，则会创建新的线程添加入池。
    
        线程池中已有60秒钟未被使用的线程将被终止和移除。
    
        队列拿到任务就给线程池加线程。
    
        ![image-20201029173555326](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201029173555326.png)
    
    4. newSingleThreadScheduledExecutor
    
        
    
    5. newScheduledThreadPool 
    
        创建一个大小无限的线程池，支持定时以及周期性执行任务的需求。
    
        ![image-20201029173640963](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201029173640963.png)
    
    6. - FixedRate是从上一个任务开始后计时，[ScheduledExecutorServiceDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s2/ScheduledExecutorServiceDemo.java)
        - FixedDelay是从上一个任务结束后计时

#### 3.2.3　刨根究底：核心线程池的内部实现	102

- 均使用ThreadPoolExecutor实现，最全的构造函数如下

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          RejectedExecutionHandler handler) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), handler);
}
```

1. **int corePoolSize**

> 线程池的核心线程数最大值。
>
> 一般无论是否有任务核心线程会在池中一直存活。
>
> ThreadPoolExecutor 中的方法 allowCoreThreadTimeOut(boolean value) 设置为 true 时，闲置的核心线程会存在超时机制，如果在指定时间没有新任务来时，核心线程也会被终止。
>
> 这个指定时间由后面的keepAliveTime指定。

**核心线程：**

**线程池新建线程时，如果当前线程数小于*corePoolSize**，则新建的是核心线程，如果超过**corePoolSize**，则新建的是非核心线程。

2. **int maximumPoolSize**

池中能容纳的最大线程数，达到该值后的后续任务被阻塞。

**线程数**=核心线程数+非核心线程数

3. **Long keepAliveTime**

非核心线程闲置时的超时时长。

但如果allowCoreThreadTimeOut(boolean value) 设置为 true 时，也作用于核心线程。

4. **TimeUnit unit**

指定`keepAliveTime`的时间单位，`TimeUnit是enum枚举类型`，常用的有：TimeUnit.HOURS(小时)、TimeUnit.MINUTES(分钟)、TimeUnit.SECONDS(秒) 和 TimeUnit.MILLISECONDS(毫秒)等。

5. **BlockingQueue<Runnable> workQueue**

线程池的任务队列，维护着等待执行的Runnable对象。

通过线程池的 execute(Runnable command) 方法会将任务 Runnable 存储在队列中。

6. **ThreadFactory threadFactory**

线程工厂，它是一个接口，用来为线程池创建新线程的。

7. **RejectedExecutionHandler handler**

抛出异常专用

workQueue和handler需要特别理解一下，核心执行代码如下：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

> **线程池的执行策略：**
>
> **线程池中线程数量没有达到核心线程数，就新建一个核心线程执行任务；**
>
> **线程池中核心线程数已满，就在任务队列中排队；**
>
> **任务队列中已满时，就新建非核心线程执行任务；**
>
> **当线程数量达到最大线程数时会抛出异常。**

#### 常用的workQueue类型：

**1、SynchronousQueue：**这个队列接收到任务的时候，会直接提交给线程处理，而不保留它，如果所有线程都在工作怎么办？那就新建一个线程来处理这个任务！所以**为了保证不出现<线程数达到了maximumPoolSize而不能新建线程>的错误**，使用这个类型队列的时候，maximumPoolSize一般指定成Integer.MAX_VALUE，即无限大

**策略：不在队列中等，而是在线程池中新建线程。**

**2、LinkedBlockingQueue：**这个队列接收到任务的时候，如果当前线程数小于核心线程数，则新建线程(核心线程)处理任务；**如果当前线程数等于核心线程数，则进入队列等待**。由于这个队列没有最大值限制，即所有超过核心线程数的任务都将被添加到队列中，这也就**导致了maximumPoolSize的设定失效**，因为总线程数永远不会超过corePoolSize

**策略：线程数少于核心线程数，那就创建核心线程；线程数等于核心线程数，入队等待，队列无限。**

**3、ArrayBlockingQueue：**可以限定队列的长度，接收到任务的时候，如果没有达到corePoolSize的值，则新建线程(核心线程)执行任务，如果达到了，则入队等候，如果队列已满，则新建线程(非核心线程)执行任务，又如果总线程数到了maximumPoolSize，并且队列也满了，则发生错误

**策略：线程数少于核心线程数，那就创建核心线程；线程数等于核心线程数，入队等待，队列满就在线程池中创建非核心线程**

**4、DelayQueue：**队列内元素必须实现Delayed接口，这就意味着你传进去的任务必须先实现Delayed接口。这个队列接收到任务时，首先先入队，只有达到了指定的延时时间，才会执行任务

**策略：所有任务必须延时执行。**



#### 3.2.4　超负载了怎么办：拒绝策略	106

- 四种系统定义的拒绝策略代码

```java
 public static class CallerRunsPolicy implements RejectedExecutionHandler {
    public CallerRunsPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            r.run();
        }
    }
}

public static class AbortPolicy implements RejectedExecutionHandler {
    public AbortPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        throw new RejectedExecutionException("Task " + r.toString() +
                                             " rejected from " +
                                             e.toString());
    }
}

public static class DiscardPolicy implements RejectedExecutionHandler {
    public DiscardPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
    }
}

public static class DiscardOldestPolicy implements RejectedExecutionHandler {
    public DiscardOldestPolicy() { }
    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            e.getQueue().poll();
            e.execute(r);
        }
    }
}
```

- [RejectThreadPoolDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s2/RejectThreadPoolDemo.java)

#### 3.2.5　自定义线程创建：ThreadFactory	109

- [MyThreadFactory](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s2/MyThreadFactory.java)

#### 3.2.6　我的应用我做主：扩展线程池	110

- 比较了一下ExecuteService中submit和execute函数的区别，通过观察发现，sumit最终也会执行execute函数，具体分析可见这篇[文章](https://www.jianshu.com/p/e05b1d060d7d)
- 顺着execute函数的位置，列一下这几个类的关系，代码如下：

```java
//最上层Executor接口
public interface Executor {
    void execute(Runnable command);
}

//继承了Executor接口的ExecutorService接口
public interface ExecutorService extends Executor {
    //中间各种函数，但不包含execute函数
}

//实现了ExecutorService接口的AbstractExecutorService抽象类
public abstract class AbstractExecutorService implements ExecutorService {
    //省略其他函数...
    
    //submit函数内调用了execute，但该抽象类没有execute函数的具体实现
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);//调用了Executor的execute方法
        return ftask;
    }
}

//实现了AbstractExecutorService抽象类的ThreadPoolExecutor类
public class ThreadPoolExecutor extends AbstractExecutorService {
    //省略其他函数...
    
    //execute函数的具体实现
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
}
```

- [ExtThreadPool](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s2/ExtThreadPool.java)


#### 3.2.7　合理的选择：优化线程池线程数量	112

请见书中公式

#### 3.2.8　堆栈去哪里了：在线程池中寻找堆栈	113

- 程序的本质意思就是写一个类重写execute和submit方法，加一个包装，让该包装可以抛出异常信息
- [TraceThreadPoolExecutor](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s2/TraceThreadPoolExecutor.java)

#### 3.2.9　分而治之：Fork/Join框架	117

- 本质就是一种递归的调用，然后不断缩小规模直到可以计算，最后将结果加起来
- [CountTask](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s2/CountTask.java)

### 3.3　不要重复发明轮子：JDK的并发容器	121

- 本节代码都是`java.util.concurrent;`包中的源代码，故而不修改每个文件的包引入行，保持代码原有的样子，方便读者观看

#### 3.3.1　超好用的工具类：并发集合简介	121

注：此处使用[JDK1.7](https://github.com/guanpengchn/JDK/tree/master/JDK1.7)的源代码

- [ConcurrentHashMap](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/ConcurrentHashMap.java)，线程安全的HahsMap
- [CopyOnWriteArrayList](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/CopyOnWriteArrayList.java)，线程安全的ArrayList一族
- [ConcurrentLinkedQueue](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/ConcurrentLinkedQueue.java)，线程安全的LinkedList
- [BlockingQueue](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/BlockingQueue.java)，阻塞队列
- [ConcurrentSkipListMap](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/ConcurrentSkipListMap.java)，跳表，用于快速查找

#### 3.3.2　线程安全的HashMap	122

- 将HashMap变为线程安全的，可用以下方法，但并发级别不高

```java
public static Map m=Collections.synchronizedMap(new HashMap());
```

- 更加专业的并发HashMao是ConcurrentHashMap

#### 3.3.3　有关List的线程安全	123

- 将List变为线程安全的，可用以下方法

```java
public static List<String> l=Collections.synchronizedList(new LinkedList<String>());
```

- 更加专业的并发HashMap是[ConcurrentHashMap](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s3/ConcurrentHashMap.java)

#### 3.3.4　高效读写的队列：深度剖析ConcurrentLinkedQueue	123

- 这里很难，还要再读一遍再做笔记！！！！！

#### 3.3.5　高效读取：不变模式下的CopyOnWriteArrayList	129

- 读读不冲突，读写不冲突，只有写写冲突
- 在写的时候先做一次数据复制，将修改的内容写入副本中，再讲副本替换原来的数据
- [CopyOnWriteArrayList](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s3/CopyOnWriteArrayList.java)

#### 3.3.6　数据共享通道：BlockingQueue	130
#### 3.3.7　随机数据结构：跳表（SkipList）	134

### 3.4　参考资料	136