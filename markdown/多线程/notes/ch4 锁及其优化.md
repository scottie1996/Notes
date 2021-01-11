[TOC]



## 第4章　锁的优化及注意事项	138

### 4.1　有助于提高“锁”性能的几点建议	139

#### 4.1.1　减小锁持有时间（只在必要的时候进行同步）	139

- JDK1.7源代码：[Pattern](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/regex/Pattern.java)

#### 4.1.2　减小锁粒度	140

- 对整个HashMap加锁粒度过大，对于ConcurrentHashMap内部细分若干个HashMap，称之为段，被分成16个段
- 现根据hashcode得到应该存放到哪个段中，然后对该段加锁
- JDK8中改变了实现方式，使用CAS来做实现，区别可见[文章](https://blog.csdn.net/Gavin__Zhou/article/details/76792071)

#### 4.1.3　读写分离锁来替换独占锁	142

- [ReadWriteLock](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch3/s1/ReadWriteLockDemo.java)

#### 4.1.4　锁分离	142

- 在LinkedBlockingQueue中，take和put函数分别作用于队列的前端和尾端，不冲突，如果使用独占锁就无法并发，所以实现中使用了两把锁
- JDK1.7源代码：[LinkedBlockingQueue](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/LinkedBlockingQueue.java)

#### 4.1.5　锁粗化	144

- 虚拟机在遇到一连串的锁请求和释放，会优化合并成一次，叫做锁的粗化
- 在循环内使用锁往往可以考虑优化成在循环外

### 4.2　Java虚拟机对锁优化所做的努力	146

#### Markword

markword是java对象数据结构中的一部分，markword数据的长度在32位和64位的虚拟机（未开启压缩指针）中分别为32bit和64bit，它的**最后2bit是锁状态标志位**，用来标记当前对象的状态，对象的所处的状态，决定了markword存储的内容，如下表所示:

![image-20201227171640560](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201227171640560.png)

32位虚拟机在不同状态下markword结构如下图所示：

![这里写图片描述](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/SouthEast.png)





#### 4.2.1　锁偏向	146

- 如果一个线程获得了锁，就进入了**偏向模式**，之后该线程再去连续申请就无须做其他操作
- 对于几乎没有锁竞争的场合，锁偏向有较好的优化效果，因为连续多次的请求极有可能来自同一个线程。
- 但是如果不同线程来回切换，效果反而差，不如不开启锁偏向
- Java虚拟机参数：-XX:+IseBiasedLocking

#### 4.2.2　轻量级锁	146

- 偏向锁请求失败时会转为请求轻量级锁，请求轻量级锁也失败时则升级为重量级锁
- 这里书中写的比较简略，可以看文章[java 中的锁 -- 偏向锁、轻量级锁、自旋锁、重量级锁](https://blog.csdn.net/zqz_zqz/article/details/70233767)

#### 4.2.3　自旋锁	146

- 锁膨胀后，虚拟机假设当前线程还可以获得锁，不马上挂起线程，让当前线程做几个空循环（也就是自旋），如果能获得就进入临界区，如果不行则挂起

#### 4.2.4　锁消除	146

- 在编译过程中，去掉不可能共享资源的锁，比如局部变量
- 锁消除的一项关键技术叫做逃逸分析
- 逃逸分析必须在-server模式下，虚拟机参数：-XX:+DoEscapeAnalysis打开逃逸分析，-XX:+EliminateLocks打开锁消除

### 4.3　人手一支笔：ThreadLocal	147

除了控制资源的策略之外，还可以采用增加资源的策略

#### 4.3.1　ThreadLocal的简单使用	148

- 这个demo测试没有ThreadLocal，仅仅是对象在run内部new出来也行呀，不是很懂
- [ThreadLocalDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s3/ThreadLocalDemo.java)
- 为每个线程人手分配一个对象并不是ThreadLoacl的工作，它只是起到了简单的容器作用，分配对象工作需要在应用层面保证

#### 4.3.2　ThreadLocal的实现原理	149

- **set方法**

  ```java
  public void set(T value) {
      Thread t = Thread.currentThread();//获取当前线程对象
      ThreadLocalMap map = getMap(t);// 拿到线程的ThreadLocalMap,实际上是WeakHashMap的实现
      if (map != null)
          map.set(this, value);
      else
          createMap(t, value); //设置到ThreadLocalMap中的值就是写入到这个Map里的，由于这个Map是定义在线程内部的，所以ThreadLocal就保存了当前线程所拥有的“局部变量”
  }
  ```

  官方文档的说明是《将当前线程的此线程局部变量的副本设置为指定的值。 大多数子类将无需重写此方法，仅依靠[`initialValue()`](https://www.matools.com/file/manual/jdk_api_1.8_google/java/lang/ThreadLocal.html#initialValue--)方法设置线程本地值的值》

  **get方法**

  ```java
  public T get() {
      Thread t = Thread.currentThread();//也是先获取当前线程的对象
      ThreadLocalMap map = getMap(t);
      if (map != null) {
          ThreadLocalMap.Entry e = map.getEntry(this);//将自己作为对象取出内部的实际数据
          if (e != null) {
              @SuppressWarnings("unchecked")
              T result = (T)e.value;
              return result;
          }
      }
      return setInitialValue();
  }
  ```

  - 清理ThreadLocalMap的工作只有在线程退出时才会做，所以在当前线程不会退出的情况下（例如使用固定大小的线程池时，线程总是存在），ThreadLocal中存放了一些较大的对象时，如果不使用它，就会导致内存溢出。所以，如果希望及时回收对象，最好用ThreadLocal.remove()这个方法将变量移除，让gc进行回收

- [ThreadLocalDemo_Gc](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s3/ThreadLocalDemo_Gc.java)

- WeakHashMap和HashMap的区别可以见该[文章](http://mzlly999.iteye.com/blog/1126049)和代码[WeakVsHashMap](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s3/WeakVsHashMap.java)

- TLAB

  我们知道，Java是一门面向对象的语言，我们在Java中使用的对象都需要被创建出来，在Java中，创建一个对象的方法有很多种，但是无论如何，对象在创建过程中，都需要进行内存分配。

  对象的内存分配过程中，主要是对象的引用指向这个内存区域，然后进行初始化操作。

  但是，因为堆是全局共享的，因此在同一时间，可能有多个线程在堆上申请空间，那么，在并发场景中，**如果两个线程先后把对象引用指向了同一个内存区域**，怎么办。

  <img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/tlab3.png" alt="img" style="zoom:50%;" />

为了解决这个并发问题，对象的内存分配过程就必须进行同步控制。但是我们都知道，**无论是使用哪种同步方案（实际上虚拟机使用的可能是CAS），都会影响内存的分配效率。**

而Java对象的分配是Java中的高频操作，所有，人们想到另外一个办法来提升效率。这里我们重点说一个HotSpot虚拟机的方案：

> 每个线程在Java堆中预先分配一小块内存，然后再给对象分配内存的时候，直接在自己这块”私有”内存中分配，当这部分区域用完之后，再分配新的”私有”内存。

这种方案被称之为TLAB分配，即Thread Local Allocation Buffer。这部分Buffer是从堆中划分出来的，但是是本地线程独享的。

所以说，因为有了TLAB技术，堆内存并不是完完全全的线程共享，其eden区域中还是有一部分空间是分配给线程独享的。

**这里值得注意的是，我们说TLAB是线程独享的，但是只是在“分配”这个动作上是线程独占的，至于在读取、垃圾回收等动作上都是线程共享的。而且在使用上也没有什么区别。**

<img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/tlab-20201128234031115.png" alt="img" style="zoom:67%;" />

**TLAB使用的相关参数**

TLAB功能是可以选择开启或者关闭的，可以通过设置-XX:+/-UseTLAB参数来指定是否开启TLAB分配。

TLAB默认是eden区的1%，可以通过选项-XX:TLABWasteTargetPercent设置TLAB空间所占用Eden空间的百分比大小。

默认情况下，TLAB的空间会在运行时不断调整，使系统达到最佳的运行状态。如果需要禁用自动调整TLAB的大小，可以使用-XX:-ResizeTLAB来禁用，并且使用-XX：TLABSize来手工指定TLAB的大小。

TLAB的refill_waste也是可以调整的，默认值为64，即表示使用约为1/64空间大小作为refill_waste(**“最大浪费空间”** , 当请求分配的内存大于refill_waste的时候，会选择在堆内存中分配。若小于refill_waste值，则会废弃当前TLAB，重新创建TLAB进行对象内存分配。)，使用参数：-XX：TLABRefillWasteFraction来调整。

如果想要观察TLAB的使用情况，可以使用参数-XX+PringTLAB 进行跟踪。

  

#### 4.3.3　对性能有何帮助	155

- 见下面demo可得ThreadLocal的效率还是很高的
- [ThreadLocalPerformance](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s3/ThreadLocalPerformance.java)

### 4.4　无锁	157

对于并发控制而言，锁是一种悲观的策略，它总是假设每一次对于临界区的操作都会产生冲突，而无锁是一种乐观的策略，它会假设并行的线程之间是没有冲突的。

- 无锁策略使用一种叫做比较交换的技术（CAS Compare And Swap）

#### 4.4.1　与众不同的并发策略：比较交换（CAS）	158

- 天生免疫死锁，没有锁竞争和线程切换的开销
- CAS(V,E,N)，V表示要更新的变量，E表示预期值，N表示新值，当V=E时，才会将V值设为N，如果不相等则该线程被告知失败，可以再次尝试
- 硬件层面现代处理器已经支持原子化的CAS指令

#### 4.4.2　无锁的线程安全整数：AtomicInteger	159

- atomic包中实现了直接使用**CAS的线程安全类型**，程序员可以直接调用

- 与Integer不同的地方是，他是可变的，并且是线程安全的，所有的操作都是用CAS指令来进行的。

- ```java
  package ch4.s4;
  
  import java.util.concurrent.atomic.AtomicInteger;
  
  public class AtomicIntegerDemo {
      static AtomicInteger i = new AtomicInteger();
      public static class AddThread implements Runnable{
  
          public void run() {
              for(int k=0;k<10000;k++){
                  i.incrementAndGet();
              }
          }
      }
  
      public static void main(String []args) throws InterruptedException {
          Thread []ts = new Thread[10];
          for(int k=0;k<10;k++){
              ts[k] = new Thread(new AddThread());
          }
          for(int k=0;k<10;k++){
              ts[k].start();
          }
          for(int k=0;k<10;k++){
              ts[k].join();
          }
          System.out.println(i);
      }
  }
  ```

  

#### 4.4.3　Java中的指针：Unsafe类	161

- native方法是不用java实现的
- [compareAndSet](https://github.com/guanpengchn/JDK/blob/master/JDK1.7/src/java/util/concurrent/atomic/AtomicInteger.java#L134-L136)
- Unsafe类在rt.jar中，jdk无法找到
- JDK开发人员并不希望大家使用Unsafe类，下面的代码，会检查调用getUnsafe函数的类，如果这个类的ClassLoader不为空（为空说明是Bootstrap加载的），直接抛出异常拒绝工作，这使得自己的程序无法直接调用Unsafe类，也就是说，UnSafe类是JDK的专属类

```java
public static Unsafe getUnsafe() {
    Class cc = Reflection.getCallerClass();
    if (cc.getClassLoader() != null)
        throw new SecurityException("Unsafe");
    return theUnsafe;
}
```

- 注意：根据Java类加载器的原理，应用程序的类由App Loader加载，而系统核心的类，如rt.jar中的由Bootstrap类加载器加载。Bootstrap加载器没有java对象的对象，因此试图获得该加载器会返回null，所以当一个类的类加载器为null时，说明是由Bootstrap加载的，这个类也极可能是rt.jar中的类

#### 4.4.4　无锁的对象引用：AtomicReference	162

- AtomicInteger是对整数的封装，AtomicReference是对对象的封装
- 运行下面demo可以见到错误，进行了多次充值，原因是状态可能不同，但是值却可能相同，所以不应该用值来判断状态
- [AtomicReferenceDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s4/AtomicReferenceDemo.java)

#### 4.4.5　带有时间戳的对象引用：AtomicStampedReference	165

- 对象值和时间戳必须都一样才能修改成功，所以只会充值一次
- [AtomicStampedReferenceDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s4/AtomicStampedReferenceDemo.java)

#### 4.4.6　数组也能无锁：AtomicIntegerArray	168

- [AtomicIntegerArrayDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s4/AtomicIntegerArrayDemo.java)

#### 4.4.7　让普通变量也享受原子操作：AtomicIntegerFieldUpdater	169

- 可以包装普通变量，让其也具有原子操作
- [AtomicIntegerFieldUpdaterDemo](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s4/AtomicIntegerFieldUpdaterDemo.java)
- 变量必须可见，不能为private
- 为确保变量被正确读取，需要有volatile
- CAS通过偏移量赋值，不支持static（Unsafe, objectFieldOffset不支持静态变量）

#### 4.4.8　挑战无锁算法：无锁的Vector实现	171

- N_BUCKET为30，相当于有30个数组，第一个数组大小FIRST_BUCKET_SIZE为8，但是Vector之后会不断翻倍，第二个数组就是16个，最终能2^33左右
- 这段比较复杂，多看

#### 4.4.9　让线程之间互相帮助：细看SynchronousQueue的实现	176

### 4.5　有关死锁的问题	179

- [DeadLock](https://github.com/guanpengchn/java-concurrent-programming/blob/master/src/main/java/ch4/s5/DeadLock.java)

### 4.6　参考文献	183