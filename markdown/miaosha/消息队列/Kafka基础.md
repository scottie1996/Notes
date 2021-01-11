# Kafka



[TOC]



## 一、什么是Kafka？

官方文档翻译：

https://colobu.com/2014/08/06/kafka-quickstart/

学习笔记：

https://scala.cool/2018/03/learning-kafka-1/

### Kafka

在具体了解Kafka的细节前，我们先来看一下它的一些基本概念：

- Kafka是运行在一个集群上，所以它可以拥有一个或多个服务节点；
- Kafka集群将消息存储在特定的文件中，对外表现为Topics；
- 每条消息记录都包含一个key,消息内容以及时间戳；

从上面几点我们大致可以推测Kafka是一个分布式的消息存储系统，那么它就仅仅这么点功能吗，我们继续看下面。

Kafka为了拥有更强大的功能，提供了四大核心接口：

- Producer API允许了应用可以向Kafka中的topics发布消息；
- Consumer API允许了应用可以订阅Kafka中的topics,并消费消息；
- Streams API允许应用可以作为消息流的处理者，比如可以从topicA中消费消息，处理的结果发布到topicB中；
- Connector API提供Kafka与现有的应用或系统适配功能，比如与数据库连接器可以捕获表结构的变化；

<img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/kafka-apis.png" alt="kafka-apis" style="zoom: 50%;" />

Client和Server之间的交流通过一条简单、高性能并且不局限某种开发语言的TCP协议。除了Java Client外，还有非常多的其它编程语言的[Client](https://cwiki.apache.org/confluence/display/KAFKA/Clients)。

### 话题和日志 (Topic和Log)

Topics是一些主题的集合，更通俗的说Topic就像一个消息队列，生产者可以向其写入消息，消费者可以从中读取消息，一个Topic支持多个生产者或消费者同时订阅它，所以其扩展性很好。Topic又可以由一个或多个partition（分区）组成，比对于每一个Topic, Kafka集群维护这一个分区的log,就像下图中的示例：

![Kafka集群](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/log_anatomy.png)

- 每一个分区都是一个**顺序的、不可变的**消息队列， 并且可以持续的添加。分区中的消息都被分配了一个序列号，称之为偏移量(offset),在每个分区中此偏移量都是唯一的。

- 在以前版本的Kafka，这个`offset`是由Zookeeper来管理的，后来Kafka开发者认为Zookeeper不合适大量的删改操作，于是把`offset`在broker以内部topic(`__consumer_offsets`)的方式来保存起来。

  <img src="https://raw.githubusercontent.com/scottie1996/PicGo/master/img/log-consumer.png" alt="log-consumer" style="zoom: 25%;" />

- Kafka集群保持所有的消息，直到它们过期， 无论消息是否被消费了。

- 实际上**消费者只持有这个偏移量**，也就是消费者在这个log中的位置。 这个偏移量由消费者控制：正常情况当消费者消费消息的时候，偏移量也线性的的增加。但是实际偏移量由消费者控制，消费者可以将偏移量重置为更老的一个偏移量，重新读取消息。（**不同消费者对同一分区的消息读取互不干扰**，消费者可以通过设置消息位移（offset）来控制自己想要获取的数据，比如可以从头读取，最新数据读取，重读读取等功能。）

- 可以看到这种设计对消费者来说操作自如， **一个消费者的操作不会影响其它消费者对此log的处理。**
  再说说分区。Kafka中采用分区的设计有几个目的。一是可以处理更多的消息，不受单台服务器的限制。Topic拥有多个分区意味着它可以不受限的处理更多的数据。第二，分区可以作为并行处理的单元，稍后会谈到这一点。

### 分布式(Distribution)

- Log的分区被分布到集群中的多个服务器上,集群里的每个实例叫做Broker。每个Broker处理它分到的分区。 根据配置每个分区还可以复制到其它服务器作为备份容错。

- 每个分区有一个leader，零或多个follower。Leader处理此分区的所有的读写请求而follower被动的复制数据。生产者根据消息的**topic和key**值，确定了消息要发往哪个partition之后（假设是p1），会找到partition对应的leader(也就是broker2里的p1)，然后将消息发给leader，leader负责消息的写入，并与其余的replica进行同步。如果leader宕机，其它的一个follower会被推举为新的leader。

- 一台服务器可能同时是一个分区的leader，另一个分区的follower。 这样可以平衡负载，避免所有的请求都只让一台或者某几台服务器处理。

  ![img](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/v2-7fa0c347ac6811e06baea17eb2260c3d_1440w.jpg)

> 我们已经知道了往topic里边丢数据，实际上这些数据会分到不同的partition上，这些partition存在不同的broker上。分布式肯定会带来问题：“万一其中一台broker(Kafka服务器)出现网络抖动或者挂了，怎么办？”

Kafka是这样做的：数据存在不同的partition上，那kafka就把这些partition做**备份**。比如，现在我们有三个partition，分别存在三台broker上。每个partition都会备份，这些备份散落在**不同**的broker上。

![img](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/v2-3b028c081e2da3ea0830a29cbae613ab_1440w-20201127200325964.jpg)

红色块的partition代表的是**主**分区，紫色的partition块代表的是**备份**分区。生产者往topic丢数据，是与**主**分区交互，消费者消费topic的数据，也是与主分区交互。

**备份分区仅仅用作于备份，不做读写。**如果某个Broker挂了，那就会选举出其他Broker的partition来作为主分区，这就实现了**高可用**。

另外值得一提的是：当生产者把数据丢进topic时，我们知道是写在partition上的，那partition是怎么将其**持久化**的呢？（不持久化如果Broker中途挂了，那肯定会丢数据嘛)。

Kafka是将partition的数据写在**磁盘**的(消息日志)，不过Kafka只允许**追加写入**(顺序访问)，避免缓慢的随机 I/O 操作。

- Kafka也不是partition一有数据就立马将数据写到磁盘上，它会先**缓存**一部分，等到足够多数据量或等待一定的时间再批量写入(flush)。

上面balabala地都是讲生产者把数据丢进topic是怎么样的，下面来讲讲消费者是怎么消费的。既然数据是保存在partition中的，那么**消费者实际上也是从partition中取**数据。

### 生产者(Producers)

生产者往某个Topic上发布消息。生产者也负责选择发布到这此Topic上的哪一个分区。最简单的方式从分区列表中轮流选择。也可以根据某种算法依照权重选择分区。开发者负责如何选择分区的算法。

### 消费者(Consumers)

- 通常来讲，消息模型可以分为两种， **队列模式**和**发布-订阅模式**。 

  - 队列的处理方式是 **一组消费者从服务器读取消息，一条消息只有其中的一个消费者来处理。**

  - ![image-20201127152430735](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201127152430735.png)

  - 在发布-订阅模型中，消息**被广播给所有的消费者，接收到消息的消费者都可以处理此消息。![image-20201127152511321](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/image-20201127152511321.png)**

  - Kafka为这两种模型提供了单一的消费者抽象模型： 消费者组 （consumer group）。

    ![img](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/v2-7ab345fbbf37058b84903ba3c05d0090_1440w.jpg)

- 生产者可以有多个，消费者也可以有多个。像上面图的情况，是一个消费者消费三个分区的数据。多个消费者可以组成一个**消费者组**。

  ![img](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/v2-0c12d975f489dcb9f02aa72c2055b8ea_1440w-20201127200250906.jpg)

- 消费者用一个**消费者组名**标记自己。 一个发布在Topic上消息被分发给此消费者组中的一个消费者。

  - 假如所有的消费者都在一个组中，那么这就变成了queue模型。
  - 假如所有的消费者都在不同的组中，那么就完全变成了发布-订阅模型。

- 更通用的， 我们可以创建一些消费者组作为逻辑上的订阅者。消费者组是一种更高层次的的抽象，向Topic订阅消费消息的单位是消费者组，当然它其中也可以只有一个消费者（consumer）。下面是关于consumer的两条原则：

  - 假如所有消费者都在同一个消费者组中，那么它们将协同消费订阅Topic的部分消息（根据分区与消费者的数量分配），保存负载平衡；
  - 假如所有消费者都在不同的消费者组中，并且订阅了同个Topic，那么它们将可以消费Topic的所有消息；

![A two server Kafka cluster hosting four partitions (P0-P3) with two consumer groups. Consumer group A has two consumer instances and group B has four](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/consumer-groups.png)

上图中有两个Server节点，有一个Topic被分为四个分区（P0-P3)分别被分配在两个节点上，另外还有两个消费者组（GA，GB），其中GA有两个消费者实例，GB有四个消费者实例。

从图中我们可以看出，首先订阅Topic的单位是消费者组，另外我们发现Topic中的消息根据一定规则将消息推送给具体消费者，主要原则如下：

- 若消费者数小于partition数，且消费者数为一个，那么它就消费所有消息；
- 若消费者数小于partition数，假设消费者数为N，partition数为M，那么每个消费者能消费的分区数为M/N或M/N+1；
- 若消费者数等于partition数，那么每个消费者都会均等分配到一个分区的消息；
- 若消费者数大于partition数，则将会出现部分消费者得不到消息分区，出现空闲的情况；

总的来说，Kafka会根据消费者组的情况均衡分配消息，比如有消息着实例宕机，亦或者有新的消费者加入等情况。

正像传统的消息系统一样，Kafka**保证消息的顺序不变。**
传统的队列模型保持消息，并且保证它们的先后顺序不变。但是， 尽管服务器保证了消息的顺序，消息还是异步的发送给各个消费者，消费者收到消息的先后顺序不能保证了。这也意味着**并行消费将不能保证消息的先后顺序。**用过传统的消息系统的同学肯定清楚，消息的顺序处理很让人头痛。如果只让一个消费者处理消息，又违背了并行处理的初衷。
在这一点上Kafka做的更好，尽管并没有完全解决上述问题。 Kafka采用了一种分而治之的策略：分区。 因为Topic分区中消息只能由消费者组中的唯一一个消费者处理，所以消息肯定是按照先后顺序进行处理的。但是它也仅仅是保证Topic的一个分区顺序处理，不能保证跨分区的消息先后处理顺序。
所以，如果你想要顺序的处理Topic的所有消息，那就**只提供一个分区。**

### Kafka的保证(Guarantees)

- 生产者发送到一个特定的Topic的分区上的消息**将会按照它们发送的顺序依次加入这个分区，消费者收到的消息也是此顺序**，生产者越早向订阅的Topic发送的消息，会更早的被添加到Topic中，当然它们可能被分配到不同的分区；
- 如果一个Topic配置了复制因子( replication facto)为N， 那么可以允许N-1服务器宕机而不丢失任何已经增加的消息

### Kafka as a Storage System

存储消息也是消息系统的一大功能，Kafka相对普通的消息队列存储来说，它的表现实在好的太多，首先Kafka支持写入确认，保证消息写入的正确性和连续性，同时Kafka还会对写入磁盘的数据进行复制备份，来实现容错，另外Kafka对磁盘的使用结构是一致的，就是说不管你的服务器目前磁盘存储的消息数据有多少，它添加消息数据的效率是相同的。

Kafka的存储机制很好的支持消费者可以随意控制自身所需要读取的数据，在很多时候你也可以将Kafka作为一个高性能，低延迟的分布式文件系统。

前面讲解到了生产者往topic里丢数据是存在partition上的，而partition持久化到磁盘是IO顺序访问的，并且是先写缓存，隔一段时间或者数据量足够大的时候才批量写入磁盘的。

消费者在读的时候也很有讲究：正常的读磁盘数据是需要将内核态数据拷贝到用户态的，而Kafka 通过调用`sendfile()`直接从内核空间（DMA的）到内核空间（Socket的），**少做了一步拷贝**的操作。

![img](https://raw.githubusercontent.com/scottie1996/PicGo/master/img/v2-ecf911716b70f836b2a607157bbf11eb_1440w.jpg)

# 总结

> 使用消息队列不可能是单机的（必然是分布式or集群）

Kafka天然是分布式的，往一个topic丢数据，实际上就是往多个broker的partition存储数据

> 数据写到消息队列，可能会存在数据丢失问题，数据在消息队列需要**持久化**(磁盘？数据库？Redis？分布式文件系统？)

Kafka会将partition以消息日志的方式(落磁盘)存储起来，通过 顺序访问IO和缓存(等到一定的量或时间)才真正把数据写到磁盘上，来提高速度。

> 想要保证消息（数据）是有序的，怎么做？

Kafka会将数据写到partition，单个partition的写入是有顺序的。如果要保证全局有序，那只能写入一个partition中。如果要消费也有序，消费者也只能有一个。

> 为什么在消息队列中重复消费了数据

凡是分布式就无法避免网络抖动/机器宕机等问题的发生，很有可能消费者A读取了数据，还没来得及消费，就挂掉了。Zookeeper发现消费者A挂了，让消费者B去消费原本消费者A的分区，等消费者A重连的时候，发现已经重复消费同一条数据了。(各种各样的情况，消费者超时等等都有可能...)

如果业务上不允许重复消费的问题，最好消费者那端做业务上的校验（如果已经消费过了，就不消费了）