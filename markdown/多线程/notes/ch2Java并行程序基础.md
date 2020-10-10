[TOC]



## 第2章　Java并行程序基础	29

### 2.1　有关线程你必须知道的事	29

- 线程状态

```java
public enum State{
NEW,
RUNNABLE,
BLOCKED,
WAITING,
TIMED_WAITING,
TERMINATED
}
```

### 2.2　初始线程：线程的基本操作	32

#### 2.2.1　新建线程	32

1. 继承Thread类或实现Runnable接口，Thread中的构造方法`public Thread(Runnable target)`，其中run函数如下：

```java
public void run(){
    if(target != null){
        target.run();
    }
}
```

2. 创建FutureTask对象，创建Callable子类对象，复写call(相当于run)方法，将其传递给FutureTask对象（相当于一个Runnable）。  创建Thread类对象，将FutureTask对象传递给Thread对象。调用start方法开启线程。这种方式可以获得线程执行完之后的返回值。该方法使用Runnable功能更加强大的一个子类.这个子类是具有返回值类型的任务方法。



#### 2.2.2　终止线程	34

- 终止线程的stop方法不建议使用，可能会导致数据不一致，可以使用内部设置函数来解决问题

#### 2.2.3　线程中断	38

```java
public void Thread.interrupt() //中断线程
public boolean Thread.isInterrupted() //判断线程是否中断
public static boolean Thread.interrupted() //判断是否中断，并清除中断状态
```

在sleep时，如果来临中断，会进入中断异常，同时也会将标记位清空，所以异常处理那里经常要再次中断自己

#### 2.2.4　等待（wait）和通知（notify）	41

- wait会释放掉锁
- [SimpleWN](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch2/SimpleWN.java)

#### 2.2.5　挂起（suspend）和继续执行（resume）线程	44

- suspend和resume不推荐使用，因为不会释放锁资源，查看jstack时可以使用jps查看pid
- [BadSuspend](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch2/BadSuspend.java)

#### 2.2.6　等待线程结束（join）和谦让（yield）	48

##### 面试题： 为什么wait方法在object类中，sleep方法在Thread类中？

这两者的**施加者是有本质区别的**. 
sleep()是让某个线程暂停运行一段时间,其控制范围是由**当前线程决定**,也就是说,在线程里面决定.好比如说,我要做的事情是

 `点火->烧水->煮面`但是当我点完火之后我不立即烧水,我想要休息一段时间再烧.**对于运行的主动权是由所属线程的流程来控制.** 所以sleep方法在Thread类中


而wait(),首先,这是**由某个确定的对象来调用的**,将这个对象理解成一个传话的人,当这个人在某个线程里面说"暂停!",也是 thisObj.wait(),这里的暂停是**阻塞**,`"点火->烧水->煮饭"`,thisObj就好比一个监督我的人站在我旁边,本来该线 程应该执行1后执行2,再执行3,**而在2处被那个对象喊暂停,那么我就会一直等在这里而不执行3,但正个流程并没有结束,我一直想去煮饭,但还没被允许**, 直到那个对象在某个地方说"通知暂停的线程启动!",也就是thisOBJ.**notify()**的时候,那么我就可以煮饭了,这个被暂停的线程就会从暂停处 继续执行. 

其实两者都可以让线程暂停一段时间,但是本质的区别**是sleep是线程的运行状态控制**,**wait则是线程之间的通讯的问题**
在`java.lang.Thread`类中，提供了`sleep()`，而`java.lang.Object`类中提供了`wait()， notify()和notifyAll()`方法来操作线程
sleep()可以将一个线程睡眠，参数可以指定一个时间。而wait()可以将一个线程挂起，直到超时或者该线程被唤醒。

##### sleep和wait的区别总结

1. 这两个方法来自不同的类分别是Thread和Object
2. 最主要是sleep方法没有释放锁，而wait方法释放了锁，使得其他线程可以使用同步控制块或者方法。
3. `wait，notify和notifyAll`是一种线程间的通信手段，**只能在同步控制方法或者同步控制块里面使用**，而sleep可以在任何地方使用

```java
 synchronized(x){
   x.notify()
   //或者wait()
  }
```

 

4. sleep必须捕获异常，而wait，notify和notifyAll不需要捕获异常

### 2.3　volatile与Java内存模型（JMM）	50

- volatile无法保证原子性
- [NoVisibility](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch2/NoVisibility.java)

### 2.4　分门别类的管理：线程组	52

- [ThreadGroupName](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch2/ThreadGroupName.java)

### 2.5　驻守后台：守护线程（Daemon）	54

与用户线程相对，用户线程可以理解为完成程序的业务操作，而守护现成则为这些操作提供支持，区别为虚拟机要等待用户线程执行完才会退出，但是不需要等待守护线程（因为如果用户线程结束，就意味着程序实际上无事可做了，守护线程要守护的对象也就不存在了，虚拟机自然就会退出。）当系统只有守护线程时，系统就会退出。垃圾回收就是一个典型的守护线程。**不能把正在运行的常规线程设置为守护线程。**

- [DaemonDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch2/DaemonDemo.java)

  ```java
  public class DeamonDemo {
  
      public static class DeamonT extends Thread{
  
  
          public void run() {
              while (true){
                  System.out.println("I am alive");
                  try {
                      Thread.sleep(1000);
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
          }
      }
  
      public static void main(String[] args) throws InterruptedException {
          Thread t = new DeamonT();
          t.setDaemon(true);//设置守护线程需要在start之前，否则会抛出IllegalThreadStateException（但是这个线程会被
          				//当成用户线程正常执行）
          t.start();
          Thread.sleep(2000);
      }
  }
  ```

### 2.6　先干重要的事：线程优先级	55

线程可以被设置优先级，优先级高的线程在抢占资源时更有优势（只是概率更高）

- [PriorityDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch2/PriorityDemo.java)

- ```java
  package ch2;
  
  public class PriorityDemo {
      public static class HighPriority extends Thread{
          static int count = 0;
  
          public void run() {
              while (true){
                  synchronized (PriorityDemo.class){
                      count ++ ;
                      if(count > 10000000){
                          System.out.println("HighPriority finished");
                          break;
                      }
                  }
              }
          }
      }
  
      public static class LowPriority extends Thread{
          static int count = 0;
  
          public void run() {
              while (true){
                  synchronized (PriorityDemo.class){
                      count ++ ;
                      if(count > 10000000){
                          System.out.println("LowPriority finished");
                          break;
                      }
                  }
              }
          }
      }
  
      public static void main(String []args){
          Thread high = new HighPriority();
          Thread low = new LowPriority();
          high.setPriority(Thread.MAX_PRIORITY);//设置为最高优先级(10)
          low.setPriority(Thread.MIN_PRIORITY);//设置为最低优先级（1）
          high.start();
          low.start();
      }
  }
  ```

### 2.7　线程安全的概念与synchronized,volatile	57

线程安全是并行执行程序的基本保证。

> 当多个线程访问同一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替运行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获取正确的结果，那这个对象是线程安全的。--深入理解JVM

### synchronized关键字

在前面一章提到过,线程安全的保证是原子性,有序性和可见性,满足这三个条件的*Synchronized*关键字是Java中解决并发问题的一种最常用的方法，也是最简单的一种方法。*Synchronized*的作用主要有三个

1. 确保线程**互斥的访问**同步代码
2. 保证**共享变量的修改能够及时可见**
3. 有效解决重排序问题。从语法上讲，*Synchronized*总共有三种用法：

#### synchronized的用法

- 没有同步的情况：

线程1和线程2同时进入执行状态，线程2执行速度比线程1快，所以线程2先执行完成，这个过程中线程1和线程2是同时执行(抢占cpu执行权)的。

```java
public class SynchronizedTest {
    public void method1(){
        System.out.println("Method 1 start");
        try {
            System.out.println("Method 1 execute");
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 1 end");
    }

    public void method2(){
        System.out.println("Method 2 start");
        try {
            System.out.println("Method 2 execute");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 2 end");
    }

    public static void main(String[] args) {
        final SynchronizedTest test = new SynchronizedTest();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method1();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method2();
            }
        }).start();
    }
}
```



- 对普通方法同步：(锁是对象)

跟代码段一比较，可以很明显的看出，线程2需要等待线程1的method1执行完成才能开始执行method2方法。

```java
package com.paddx.test.concurrent;

public class SynchronizedTest {
    public synchronized void method1(){
        System.out.println("Method 1 start");
        try {
            System.out.println("Method 1 execute");
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 1 end");
    }

    public synchronized void method2(){
        System.out.println("Method 2 start");
        try {
            System.out.println("Method 2 execute");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 2 end");
    }

    public static void main(String[] args) {
        final SynchronizedTest test = new SynchronizedTest();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method1();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method2();
            }
        }).start();
    }
}
```



- 静态方法（类）同步:(锁是类)

对静态方法的同步本质上**是对类的同步**（静态方法本质上是属于类的方法，而不是对象上的方法），所以即使test和test2属于不同的对象，但是它们都属于SynchronizedTest类的实例，所以也只能顺序的执行method1和method2，不能并发执行。

```java

package com.paddx.test.concurrent;

 public class SynchronizedTest {
     public static synchronized void method1(){
         System.out.println("Method 1 start");
         try {
             System.out.println("Method 1 execute");
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         System.out.println("Method 1 end");
     }

     public static synchronized void method2(){
         System.out.println("Method 2 start");
         try {
             System.out.println("Method 2 execute");
             Thread.sleep(1000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         System.out.println("Method 2 end");
     }

     public static void main(String[] args) {
         final SynchronizedTest test = new SynchronizedTest();
         final SynchronizedTest test2 = new SynchronizedTest();

         new Thread(new Runnable() {
             @Override
             public void run() {
                 test.method1();
             }
         }).start();

         new Thread(new Runnable() {
             @Override
             public void run() {
                 test2.method2();
             }
         }).start();
     }
 }
```



- 代码块同步(锁可以自己指定)

虽然线程1和线程2都进入了对应的方法开始执行，但是线程2在进入同步块之前，需要等待线程1中同步块执行完成。

```java
package com.paddx.test.concurrent;

public class SynchronizedTest {
    public void method1(){
        System.out.println("Method 1 start");
        try {
            synchronized (this) {
                System.out.println("Method 1 execute");
                Thread.sleep(3000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 1 end");
    }

    public void method2(){
        System.out.println("Method 2 start");
        try {
            synchronized (this) {
                System.out.println("Method 2 execute");
                Thread.sleep(1000);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("Method 2 end");
    }

    public static void main(String[] args) {
        final SynchronizedTest test = new SynchronizedTest();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method1();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                test.method2();
            }
        }).start();
    }
}
```



#### Synchronized 原理

#### 同步代码块

我们先通过反编译下面的代码来看看Synchronized是如何实现对代码块进行同步的,以及反编译的结果:

```java
public class SynchronizedDemo {
    public void method() {
        synchronized (this) {
            System.out.println("Method 1 start");
        }
    }
}

```

<img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/synchronized%E5%8F%8D%E7%BC%96%E8%AF%91.png" alt="synchronized反编译" style="zoom: 50%;" />

JVM规范中对monitorenter 指令的描述为

每个对象有一个监视器锁（monitor）。当(且仅当)monitor被占用时会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1. 如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
2. 如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
3. 如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

对monitorexit指令的描述为

1. 执行monitorexit的线程必须是objectref所对应的monitor的所有者。
2. 指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。 

通过这两段描述，我们应该能很清楚的看出Synchronized的实现原理，Synchronized的语义底层是通过一个monitor的对象来完成，其实**wait/notify等方法也依赖于monitor对象**，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。

#### 同步方法

我们再来看一下同步方法的反编译结果：

```java
package com.paddx.test.concurrent;

public class SynchronizedMethod {
    public synchronized void method() {
        System.out.println("Hello World!");
    }
}


```

<img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/synchronized%E5%8F%8D%E7%BC%96%E8%AF%912.png" alt="synchronized反编译2" style="zoom: 50%;" />

从反编译的结果来看，方法的同步**并没有通过指令monitorenter和monitorexit来完成**（理论上其实也可以通过这两条指令来实现），不过相对于普通方法，其常量池中多了ACC_SYNCHRONIZED标示符。JVM就是根据该标示符来实现方法的同步的：当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。在方法执行期间，其他任何线程都无法再获得同一个monitor对象。 其实本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。

总结一下,synchronized是对通过多个线程一段内存进行互斥访问来实现**可见性**的

**原子性**，JMM一共定义了8条原子操作，其中六条可以满足**基本数据类型的访问读写具备原子性**，还剩下lock和unlock两条原子操作。如果我们需要更大范围的原子性操作就可以使用lock和unlock原子操作。尽管jvm没有把lock和unlock开放给我们使用，但jvm以更高层次的指令monitorenter和monitorexit指令开放给我们使用，反应到java代码中就是---synchronized关键字，也就是说**synchronized满足原子性**。

**有序性**:synchronized语义表示锁在同一时刻只能由一个线程进行获取，当锁被占用后，其他线程只能等待.因此，synchronized语义就要求线程在访问读写共享变量时只能“串行”执行，因此**synchronized具有有序性**。



### volatile关键字

通常情况下我们可以通过Synchronized关键字来解决这些个问题，不过如果对Synchronized原理有了解的话，应该知道Synchronized是一个比较重量级的操作，对系统的性能有比较大的影响，所以，如果有其他解决方案，我们通常都避免使用Synchronized来解决问题。而volatile关键字就是Java中提供的另一种解决可见性和有序性问题的方案。对于原子性，需要强调一点，也是大家容易误解的一点：对volatile变量的单次读/写操作可以保证原子性的，如long和double类型变量，但是并不能保证i++这种操作的原子性，因为本质上i++是读、写两次操作。

#### **volatile的使用**

1. **防止重排序**(实现有序性)

   们从一个最经典的例子来分析重排序问题。大家应该都很熟悉单例模式的实现，而在并发环境下的单例实现方式，我们通常可以采用双重检查加锁（DCL）的方式来实现。

   ```java
   public class Singleton {
       public static volatile Singleton singleton;
   
   /**
   
    * 构造函数私有，禁止外部实例化
      */
      private Singleton() {};
   
   public static Singleton getInstance() {
       if (singleton == null) {
           synchronized (singleton) {
               if (singleton == null) {
                   singleton = new Singleton();
               }
           }
       }
       return singleton;
   }
   
   }
   ```

   现在我们分析一下为什么要在变量singleton之间加上volatile关键字。要理解这个问题，先要了解对象的构造过程，实例化一个对象其实可以分为三个步骤：

   　　（1）分配内存空间。

   　　（2）初始化对象。

   　　（3）将内存空间的地址赋值给对应的引用。

   但是由于操作系统可以对指令进行重排序，所以上面的过程也可能会变成如下过程：

   　　（1）分配内存空间。

   　　（2）将内存空间的地址赋值给对应的引用。

   　　（3）初始化对象

   如果是这个流程，**多线程环境下就可能将一个未初始化的对象引用暴露出来，从而导致不可预料的结果**。因此，为了防止这个过程的重排序，我们需要将变量设置为volatile类型的变量。

   **Volatile是怎么保证不会被执行重排序的**

   java编译器会在生成指令系列时在适当的位置会插入**内存屏障指令**来禁止特定类型的处理器重排序。它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

   ![volatile规则](C:\Users\zhouz\Documents\笔记\java-concurrent-programming\notes\pic\volatile规则.jpg)

需要注意的是：volatile写是在前面和后面**分别插入内存屏障**，而volatile读操作是在**后面插入两个内存屏障**。

**写volatile时的内存屏障**

![内存屏障1(写)](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C1(%E5%86%99).jpg)

**读volatile时的内存屏障**

![内存屏障2(读)](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/%E5%86%85%E5%AD%98%E5%B1%8F%E9%9A%9C2(%E8%AF%BB).jpg)

2. 实现可见性

可见性问题主要指一个线程修改了共享变量值，而另一个线程却看不到。引起可见性问题的主要原因是每个线程拥有自己的一个线程工作内存。volatile关键字能有效的解决这个问题,原理为**它会强制将对缓存的修改操作立即写入主存；**

```java
public class VolatileTest {
    int a = 1;
    int b = 2;

    public void change(){
        a = 3;
        b = a;
    }

    public void print(){
        System.out.println("b="+b+";a="+a);
    }

    public static void main(String[] args) {
        while (true){
            final VolatileTest test = new VolatileTest();
            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.change();
                }
            }).start();

            new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    test.print();
                }
            }).start();

        }
    }
}
```

3. 关于原子性

   volatile只能保证**对单次读/写的原子性**。比如说,long和double两种数据类型的操作可分为高32位和低32位两部分，因此普通的long或double类型读/写可能**不是原子的**。因此，鼓励大家将共享的long和double变量设置为volatile类型，这样能保证任何情况下对long和double的单次读/写操作都具有原子性。

### 2.8　程序中的幽灵：隐蔽的错误	61

#### 2.8.1　无提示的错误案例	61
#### 2.8.2　并发下的ArrayList	62
#### 2.8.3　并发下诡异的HashMap	63
#### 2.8.4　初学者常见问题：错误的加锁	66

### 2.9　参考文献	68