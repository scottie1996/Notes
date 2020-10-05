Vert.x的核心库定义了编写异步网络应用的基本API，可以为应用程序选择有用的模块（如数据库链接、监控、认证、日志、服务发现、集群支持等）。**Vert.x基于Netty项目**，通常情况下开发者会受益于Vert.x提供的高级API，而且与原始Netty相比并不牺牲性能。
Vert.x不强制要求打包方式或者构建环境。由于Vert.x Core本身只是一个普通的JAR库，所以它可以嵌入到应用程序内部，无论这些应用是打包为一组JAR、一个包含所有依赖的单独的JAR、或者即使被部署到流行的组件和应用容器。
由于Vert.x设计用于异步通信，因此它可以处理更多的并发网络连接，且比诸如Java Servlet或者java.net.socket类等同步API使用更少的线程。Vert.x可以用于广泛的应用程序：大容量消息/事件处理、微服务、API网关、移动应用HTTP API，等。Vert.x及其生态提供了构建端到端响应式应用的各种技术工具。



### 线程和编程模型

许多网络库和框架依靠一个简单的线程策略：每个网络客户端在链接时被分配一个线程，这个线程处理这个客户端的请求，直到链接断开。这是Servlet以及使用java.io和java.net包编写网络代码的情况。虽然这种“同步I/O”线程模型具有简单易懂的优点，但是当存在太多并发链接时，它会损害可伸缩性，因为系统线程并非廉价，并且在高负载的情况下，操作系统内核在线程调度管理上花费显著的时间。在这种情况下，我们需要转移到“异步I/O”，Vert.x则为“异步I/O”提供了坚实的基础。

Vert.x中的部署单元称为Verticle,即程序的入口。一个Verticle通过一个事件循环（EventLoop）处理接收到的事件，这些事件可以是任何事情，如接收网络缓冲、调度事件或由其它Verticle发送的消息。事件循环（EventLoop）是异步编程模型中是特有的：

<img src="https://img-blog.csdn.net/20180519121629687?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VsaW5lc3BhY2U=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="这里写图片描述" style="zoom: 33%;" />

每个事件应在合理的时间内进行处理，以免阻塞事件循环（EventLoop）。这意味着当在事件循环（EventLoop）中执行时，不能进行线程阻塞操作，类似在一个GUI中处理事件（如通过执行一个缓慢的网络请求可以冻结Java/Swing界面）。如我们在本指南接下来看到的，Vert.x在事件循环（EventLoop）的外部提供了处理阻塞操作的机制。在任何情况下，事件循环（EventLoop）执行事件的时间过长时，Vert.x会在日志中输出警告，这是可配置的，以满足特定应用需求（如当工作于较慢的IoT ARM板时）。

每个事件循环（EventLoop）连接到一个线程。默认情况下，Vert.x为每个CPU内核线程赋予2个事件循环（EventLoop）。这直接导致了常规的Verticle总是在同一个线程中处理事件，因此不需要使用线程协调机制来操作Verticle状态（如Java类属性）。

### 事件总线

在Vert.x中，Verticle构成了代码部署的技术单元。Vert.x事件总线是在不同Verticle之间通过异步消息传递进行通讯的主要工具。例如，假设我们有一个Verticle用于处理HTTP请求，一个Verticle用于管理数据库访问。事件总线允许**HTTP Verticle**发送一个请求到**数据库Verticle**，执行一个SQL查询，并响应HTTP Verticle：

<img src="https://img-blog.csdn.net/20180519121713990?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2VsaW5lc3BhY2U=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70" alt="这里写图片描述" style="zoom: 25%;" />



事件总线允许传递任何类型的数据，然而JSON是首选的交换格式，因为它允许不同语言编写的Verticle之间进行通讯，并且更一般地，JSON是一种流行的通用半结构化数据封送处理文本格式。

消息可以发送到任意自由字符串组成的目的地。事件总线支持以下通讯方式：

1. 端到端的消息
2. 请求——响应消息
3. 发布/订阅用于广播消息

事件总线允许Verticle之间透明的通讯，不只是在同一个JVM进程中：

- 当网络集群启动时，事件总线是分布式的，因此消息可以被发送到运行于其它应用节点的Verticle。
- 事件总线可以通过一个简单的TCP协议访问，以便第三方应用进行通讯。

