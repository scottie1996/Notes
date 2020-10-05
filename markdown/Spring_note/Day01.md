[TOC]



## 程序之间的耦合

- 程序耦合：程序之间的依赖关系，包括类之间的依赖和方法之间的依赖。

- 解耦：降低程序间的依赖关系，实际开发中应做到**编译期间不依赖，运行时才依赖**。

  ```java
  public class JdbcDemo1 {
      public static void main(String[] args) throws  Exception{
          //1.注册驱动
  //      DriverManager.registerDriver(new com.mysql.jdbc.Driver());
          Class.forName("com.mysql.jdbc.Driver");
  
          //2.获取连接
          Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/eesy","root","qqqqqq960310");
  
          //3.获取操作数据库的预处理对象
          PreparedStatement pstm = conn.prepareStatement("select * from accont");
  
          //4.执行SQL，得到结果集
          ResultSet rs = pstm.executeQuery();
  
          //5.遍历结果集
          while(rs.next()){
              System.out.println(rs.getString("name"));
          }
  
          //6.释放资源
          rs.close();
          pstm.close();
          conn.close();
      }
  }
  ```

- 在注册驱动时，如果没有在pom里加上依赖，使用*DriverManager.registerDriver(new com.mysql.jdbc.Driver());*的形式，在编译时就会产生error，因为寻找不到相应的jar包，这时耦合性就是比较高的(编译期间有依赖)

- 使用反射方法*Class.forName("com.mysql.cj.jdbc.Driver");*这句话中Driver不是*以一个类的形式存在而是以一个字符串的形式*，所以不会出现编译时异常，但若此时没有加入mysql的jar包依赖就会报运行时异常，因为找不到Driver这个类(运行时才有依赖)。

### 解耦的思路：

1. 第一步：使用反射来创建对象，而避免使用new关键字

   **区别**：new依赖一个**具体的驱动类**，而反射则**依赖一个字符串**。

   DriverManager.registerDriver(new com.mysql.cj.jdbc.Driver());

   Class.forName("com.mysql.cj.jdbc.Driver");

   但是，如果把这个字符串限制死，那么只能获取mysql的驱动，显然不太合适，所以

2. 第二步：通过读取配置文件来获取要创建的对象全限定类名

## 工厂模式解耦

工厂类Factory是一个**创建Bean对象**的工厂

> Bean：在计算机英语中，有可**重用组件**的含义。比如持久层的Dao可以被多个业务层Service调用，它就是可重用组件；业务层的Service可以被多个表现层的Servlet调用，它也是可重用组件。
>
> JavaBean：是指用java语言编写的可重用组件。
>
> 注意：经常将JavaBean和实体类划等号，这是错误的。JavaBean的范围要远大于实体类。

所以工厂模式就是创建service和dao对象的工厂。

1. 需要一个配置文件来配置service和dao；

   配置的内容至少包括：**唯一标识=全限定类名**(key value 的形式)

2. 通过读取配置文件中配置的内容，反射创建对象。

配置文件可以是xml，也可以是properties

### 读取properties文件

*Properites props  = new Properties();*获取properites文件的流对象

*InputStream in = new FileInputStream()；*这里**不要用FileInputStream**，因为不知道里面的路径参数该如何填写，项目发布后src的相对路径都不存在了，绝对路径又不能保证有C盘D盘，所以用**类加载器**完成

*InputStream in = BeanFactory.class.getClassLoader().getResourceAsStream(bean.properties);*

*props.load(in)；*

```java
public class BeanFactory {
    //定义一个Properties对象
    private static Properties props;

    //定义一个Map,用于存放我们要创建的对象。我们把它称之为容器
    private static Map<String,Object> beans;

    //使用静态代码块为Properties对象赋值
    static {
        try {
            //实例化对象
            props = new Properties();
            //获取properties文件的流对象
            //用类加载器完成
            InputStream in = BeanFactory.class.getClassLoader().getResourceAsStream("bean.properties");
            props.load(in);
            //实例化容器
            beans = new HashMap<String,Object>();
            //取出配置文件中所有的Key
            Enumeration keys = props.keys();
            //遍历枚举
            while (keys.hasMoreElements()){
                //取出每个Key
                String key = keys.nextElement().toString();
                //根据key获取value
                String beanPath = props.getProperty(key);
                //反射创建对象
                Object value = Class.forName(beanPath).newInstance();
                //把key和value存入容器中
                beans.put(key,value);
            }
        }catch(Exception e){
            throw new ExceptionInInitializerError("初始化properties失败！");
        }
    }

    //调整后的getBean
	public static Object getBean(String beanName){
		return beans.get(beanName);
	}
 
	//调整前的getBean
	public static Object getBean(String beanName){
		Object bean = null;
		try{
			String beanPath = props.getProperty(beanName);
			//单例和多例的问题
			//单例容易出现线程问题，Servlet就是单例对象，不能轻易处理它的类成员变量
			//多例影响执行效率
			//service和dao可以使用单例模式，但不会出现单例的问题，一般都把“类成员变量“放在方法内部，每次都会重新初始化
			//newInstance都会调用默认构造函数进行初始化，而长时间不使用java的垃圾回收机制就会收回，再去创建，对象就不再是之前的；所以下面生成的一个bean就把它放在一个”容器“中——单例
			bean = Class.forName(beanPath).newInstance();
		}catch(Exception e){
			e.printStackTrace();
		}
		return bean;
	}
}

```

## IOC控制翻转(inverse of control )

<img src="C:\Users\zhouz\Documents\Spring_note\pic\IOC.png" alt="控制反转" style="zoom:50%;" />

**控制反转（Inversion of Control）**是一种是面向对象编程中的一种设计原则，用来减低计算机代码之间的耦合度。其基本思想是：**借助于“第三方”实现具有依赖关系的对象之间的解耦**。

![软件的耦合关系](C:\Users\zhouz\Documents\Spring_note\pic\软件中的耦合对象.png)



![IOC解耦](C:\Users\zhouz\Documents\Spring_note\pic\IOC解耦.png)

由于引进了中间位置的“第三方”，也就是IOC容器，使得A、B、C、D这4个对象没有了耦合关系，齿轮之间的传动全部依靠“第三方”了，全部对象的控制权全部上缴给“第三方”IOC容器，所以，IOC容器成了整个系统的关键核心，它起到了一种类似“粘合剂”的作用，把系统中的所有对象粘合在一起发挥作用，如果没有这个“粘合剂”，对象与对象之间会彼此失去联系，这就是有人把IOC容器比喻成“粘合剂”的由来。

### 控制翻转的由来

1. 软件系统在没有引入IOC容器之前，如图1所示，对象A依赖于对象B，那么对象A在初始化或者运行到某一点的时候，自己必须主动去创建对象B或者使用已经创建的对象B。无论是创建还是使用对象B，**控制权都在自己手上。**

2. 软件系统在引入IOC容器之后，这种情形就完全改变了，如图2所示，由于IOC容器的加入，**对象A与对象B之间失去了直接联系**，所以，当对象A运行到需要对象B的时候，**IOC容器会主动创建一个对象B注入到对象A需要的地方。**
   通过前后的对比，我们不难看出来：对象A获得依赖对象B的过程,由主动行为变为了被动行为，控制权颠倒过来了，这就是“控制反转”这个名称的由来。

控制反转不只是软件工程的理论，在生活中我们也有用到这种思想。再举一个现实生活的例子：

海尔公司作为一个电器制商需要把自己的商品分销到全国各地，但是发现，不同的分销渠道有不同的玩法，于是派出了各种销售代表玩不同的玩法，随着渠道越来越多，发现，每增加一个渠道就要新增一批人和一个新的流程，严重耦合并依赖各渠道商的玩法。实在受不了了，于是制定业务标准，开发分销信息化系统，只有符合这个标准的渠道商才能成为海尔的分销商。让各个渠道商反过来依赖自己标准。反转了控制，倒置了依赖。

我们把海尔和分销商当作软件对象，分销信息化系统当作IOC容器，可以发现，在没有IOC容器之前，分销商就像图1中的齿轮一样，增加一个齿轮就要增加多种依赖在其他齿轮上，势必导致系统越来越复杂。开发分销系统之后，所有分销商只依赖分销系统，就像图2显示那样，可以很方便的增加和删除齿轮上去。



## Spring框架的使用(三个实现类)

### 配置文件xml：

1. 要导入约束

   > ```
   > <?xml version="1.0" encoding="UTF-8"?>
   > <beans xmlns="http://www.springframework.org/schema/beans"
   >        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   >        xsi:schemaLocation="http://www.springframework.org/schema/beans
   >         http://www.springframework.org/schema/beans/spring-beans.xsd">
   > ```

2. 把对象的创建交给Spring，用<bean>标签说明id（唯一标识）和class（全限定类名）。

### 模拟表现层的使用:

<img src="C:\Users\zhouz\Documents\Spring_note\pic\Spring工厂类结构图.png" style="zoom:60%;" />

```java
public class Client{
	//获取spring的IoC核心容器，并根据id获取对象
	public static void main(String[] args){
		//1、创建容器，获取到核心容器对象
		ApplicationContext ac = new ClassPathXmlApplicationContext("bean.xml");//参数为路径，将xml放在了resources根目录下，直接写文件名即可
		//2、根据id获取Bean对象
		IAccountService as = (IAccountService)ac.getBean("accountService");
		IAccountDao adao = ac.getBean("accountDao",IAccountDao.class);		
}

```

本质上还是通过工厂模式创建对象，只不过读取配置文件，创建对象存入map，这些操作都交给spring来做了。我们做的是写好配置文件，然后获取核心容器，通过传入参数获取对象。

### ApplicationContext三个常用实现类：

1. *ClassPathXmlApplicationContext*: 它可以加载类路径下的配置文件，要求配置文件必须在类路径下，不在此路径下的就加载不了。
2. *FileSystemXmlApplicationContext*: 它可以加载磁盘**任意路径下的配置文件**（必须有访问权限）
3. *AnnotationConfigApplicationContext:*它是用于读取注解创建容器的。

### 核心容器的两个接口引发出的问题：

ApplicationContext:     **单例**对象适用（bean单例对象适用）

*      它在构建核心容器时，创建对象采取的策略是采用**立即加载**的方式。也就是说，**只要一读取完配置文件马上就创建配置文件中配置的对象。** 实际开发中更多采用此接口。

BeanFactory: **多例**对象使用（bean多例对象适用）

*      它在构建核心容器时，创建对象采取的策略是采用**延迟加载的方式**。也就是说，**什么时候根据id获取对象了，什么时候才真正的创建对象。**

## Spring对Bean的管理细节：

### Bean的三种创建方式：

1. 使用默认构造函数创建。

   在Spring的配置文件中使用bean标签，配以id和class属性后，且没有其他属性和标签时，采用这种方法创建。

   **此时如果工厂类中没有默认构造函数，那么bean对象无法创建。**

   ```java
   <bean id="accountService" class="   com.itheima.service.impl.AccountServiceImpl"></bean>
   ```

2. 使用普通工厂中的方法创建对象（使用某个类中的方法创建对象，并存入spring容器）

   类存在于jar包中，无法修改，给出默认构造函数

```java
<bean id="instanceFactory" class="com.itheima.factory.InstanceFactory"></bean>
<bean id="accountService" factory-bean="instanceFactory" factory-method="getAccountService"></bean>
```

3. 使用工厂中的静态方法创建对象（使用某个类中的静态方法创建对象，并存入spring容器）

```java
<bean id="accountService" class="com.itheima.factory.StaticFactory" factory-method="getAccountService"></bean>
```

### bean的作用范围调整：

Spring创建bean默认是单例的。但是可以通过bean标签的scope属性指定bean的作用范围。

- **singleton**：单例（默认值）
- **prototype：多例**
- request：作用于web应用的请求范围
- session：作用于web应用的会话范围
- global-session：集群环境的会话范围（全局会话）

### bean对象的生命周期：

单例对象 

- 生命周期和容器相同。

多例对象

- 使用对象时Spring框架为我们创建
- 对象在使用过程中一直活着
- 当对象长时间不用，且没有别的对象引用时，由java垃圾回收机制回收