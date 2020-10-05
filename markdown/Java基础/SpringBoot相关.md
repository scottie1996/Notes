# SpringBoot优点

### ①良好的基因

因为SpringBoot是伴随着Spring 4.0而生的，boot是引导的意思，也就是它的作用其实就是在于帮助开发者快速的搭建Spring框架，因此SpringBoot继承了Spring优秀的基因，在Spring中开发更为方便快捷。

![img](https://img-blog.csdn.net/20180721100643551?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzMyNTk1NDUz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### ②简化编码

，比如我们要创建一个 web 项目，使用 Spring 的朋友都知道，在使用 Spring 的时候，需要在 pom 文件中添加多个依赖，而 Spring Boot 则会帮助开发着快速启动一个 web 容器，在 Spring Boot 中，我们只需要在 pom 文件中添加如下一个 starter-web 依赖即可。

```xml
<dependency>



    <groupId>org.springframework.boot</groupId>



    <artifactId>spring-boot-starter-web</artifactId>



</dependency>
```

我们点击进入该依赖后可以看到，Spring Boot 这个 starter-web 已经包含了多个依赖，包括之前在 Spring 工程中需要导入的依赖，我们看一下其中的一部分，如下：

```xml
<!-- .....省略其他依赖 -->



<dependency>



    <groupId>org.springframework</groupId>



    <artifactId>spring-web</artifactId>



    <version>5.0.7.RELEASE</version>



    <scope>compile</scope>



</dependency>



<dependency>



    <groupId>org.springframework</groupId>



    <artifactId>spring-webmvc</artifactId>



    <version>5.0.7.RELEASE</version>



    <scope>compile</scope>



</dependency>
```

由此可以看出，Spring Boot 大大简化了我们的编码，我们不用一个个导入依赖，直接一个依赖即可。

### ③简化配置

Spring 虽然使Java EE轻量级框架，但由于其繁琐的配置，一度被人认为是“配置地狱”。各种XML、Annotation配置会让人眼花缭乱，而且配置多的话，如果出错了也很难找出原因。Spring Boot更多的是采用 Java Config 的方式，对 Spring 进行配置。举个例子：

我新建一个类，但是我不用 @Service注解，也就是说，它是个普通的类，那么我们如何使它也成为一个 Bean 让 Spring 去管理呢？只需要@Configuration 和@Bean两个注解即可，如下：

```java
public class TestService {



    public String sayHello () {



        return "Hello Spring Boot!";



    }



}



 



import org.springframework.context.annotation.Bean;



import org.springframework.context.annotation.Configuration;



 



@Configuration



public class JavaConfig {



    @Bean



    public TestService getTestService() {



        return new TestService();



    }



}
```

@Configuration表示该类是个配置类，@Bean表示该方法返回一个 Bean。这样就把TestService作为 Bean 让 Spring 去管理了，在其他地方，我们如果需要使用该 Bean，和原来一样，直接使用@Resource注解注入进来即可使用，非常方便。

@Resource private TestService testService;

另外，部署配置方面，原来 Spring 有多个 xml 和 properties配置，在 Spring Boot 中只需要个 application.yml即可。

 

### ④简化部署

在使用 Spring 时，项目部署时需要我们在服务器上部署 tomcat，然后把项目打成 war 包扔到 tomcat里，在使用 Spring Boot 后，我们不需要在服务器上去部署 tomcat，因为 Spring Boot 内嵌了 tomcat，我们只需要将项目打成 jar 包，使用 java -jar xxx.jar一键式启动项目。

另外，也降低对运行环境的基本要求，环境变量中有JDK即可。

 

### ⑤简化监控

我们可以引入 spring-boot-start-actuator 依赖，直接使用 REST 方式来获取进程的运行期性能参数，从而达到监控的目的，比较方便。但是 Spring Boot 只是个微框架，没有提供相应的服务发现与注册的配套功能，没有外围监控集成方案，没有外围安全管理方案，所以在微服务架构中，还需要 Spring Cloud 来配合一起使用。

 

## 3.从未来发展趋势看

微服务是未来发展的趋势，项目会从传统架构慢慢转向微服务架构，因为微服务可以使不同的团队专注于更小范围的工作职责、使用独立的技术、更安全更频繁地部署。而 继承了 Spring 的优良特性，与 Spring 一脉相承，而且 支持各种REST API 的实现方式。Spring Boot 也是官方大力推荐的技术，可以看出，Spring Boot 是未来发展的一个大趋势

# spring boot 的常用注解使用

@RestController和@RequestMapping注解

4.0重要的一个新的改进是@RestController注解，它继承自@Controller注解。4.0之前的版本，Spring MVC的组件都使用@Controller来标识当前类是一个控制器servlet。使用这个特性，我们可以开发REST服务的时候不需要使用@Controller而专门的@RestController。

 当你实现一个RESTful web services的时候，response将一直通过response body发送。为了简化开发，Spring 4.0提供了一个专门版本的controller。下面我们来看看@RestController实现的定义：



```java
@Target(value=TYPE)  



 @Retention(value=RUNTIME)  



 @Documented  



 @Controller  



 @ResponseBody  



public @interface RestController 
```


@RequestMapping 注解提供路由信息。它告诉Spring任何来自"/"路径的HTTP请求都应该被映射到 home 方法。 @RestController 注解告诉Spring以字符串的形式渲染结果，并直接返回给调用者。



注： @RestController 和 @RequestMapping 注解是Spring MVC注解（它们不是Spring Boot的特定部分）

@EnableAutoConfiguration注解



第二个类级别的注解是 @EnableAutoConfiguration 。这个注解告诉Spring Boot根据添加的jar依赖猜测你想如何配置Spring。由于 spring-boot-starter-web 添加了Tomcat和Spring MVC，所以auto-configuration将假定你正在开发一个web应用并相应地对Spring进行设置。Starter POMs和Auto-Configuration：设计auto-configuration的目的是更好的使用"Starter POMs"，但这两个概念没有直接的联系。你可以自由地挑选starter POMs以外的jar依赖，并且Spring Boot将仍旧尽最大努力去自动配置你的应用。

你可以通过将 @EnableAutoConfiguration 或 @SpringBootApplication 注解添加到一个 @Configuration 类上来选择自动配置。
注：你只需要添加一个 @EnableAutoConfiguration 注解。我们建议你将它添加到主 @Configuration 类上。

如果发现应用了你不想要的特定自动配置类，你可以使用 @EnableAutoConfiguration 注解的排除属性来禁用它们。

```html
<pre name="code" class="java">import org.springframework.boot.autoconfigure.*;



import org.springframework.boot.autoconfigure.jdbc.*;



import org.springframework.context.annotation.*;



@Configuration



@EnableAutoConfiguration(exclude={DataSourceAutoConfiguration.class})



public class MyConfiguration {



}

```



@Configuration

Spring Boot提倡基于Java的配置。尽管你可以使用一个XML源来调用 SpringApplication.run() ，我们通常建议你使用 @Configuration 类作为主要源。一般定义 main 方法的类也是主要 @Configuration 的一个很好候选。你不需要将所有的 @Configuration 放进一个单独的类。 @Import 注解可以用来导入其他配置类。另外，你也可以使用 @ComponentScan 注解自动收集所有的Spring组件，包括 @Configuration 类。

如果你绝对需要使用基于XML的配置，我们建议你仍旧从一个 @Configuration 类开始。你可以使用附加的 @ImportResource 注解加载XML配置文件。

@Configuration注解该类，等价 与XML中配置beans；用@Bean标注方法等价于XML中配置bean

```java
@ComponentScan(basePackages = "com.hyxt",includeFilters = {@ComponentScan.Filter(Aspect.class)})
```



@SpringBootApplication

很多Spring Boot开发者总是使用 @Configuration ， @EnableAutoConfiguration 和 @ComponentScan 注解他们的main类。由于这些注解被如此频繁地一块使用（特别是你遵循以上最佳实践时），Spring Boot提供一个方便的 @SpringBootApplication 选择。
该 @SpringBootApplication 注解等价于以默认属性使用 @Configuration ， @EnableAutoConfiguration 和 @ComponentScan 。

```java
package com.example.myproject;



import org.springframework.boot.SpringApplication;



import org.springframework.boot.autoconfigure.SpringBootApplication;



@SpringBootApplication // same as @Configuration @EnableAutoConfiguration @ComponentScan



public class Application {



    public static void main(String[] args) {



        SpringApplication.run(Application.class, args);



    }



}
```


Spring Boot将尝试校验外部的配置，默认使用JSR-303（如果在classpath路径中）。你可以轻松的为你的@ConfigurationProperties类添加JSR-303 javax.validation约束注解：



```java
@Component



@ConfigurationProperties(prefix="connection")



public class ConnectionSettings {



@NotNull



private InetAddress remoteAddress;



// ... getters and setters



}
```



@Profiles

Spring Profiles提供了一种隔离应用程序配置的方式，并让这些配置只能在特定的环境下生效。任何@Component或@Configuration都能被@Profile标记，从而限制加载它的时机。



```java
@Configuration



@Profile("production")



public class ProductionConfiguration {



// ...



}
```

@ResponseBody



表示该方法的返回结果直接写入HTTP response body中

一般在异步获取数据时使用，在使用@RequestMapping后，返回值通常解析为跳转路径，加上
@responsebody后返回结果不会被解析为跳转路径，而是直接写入HTTP response body中。比如
异步获取json数据，加上@responsebody后，会直接返回json数据。



@Component：

泛指组件，当组件不好归类的时候，我们可以使用这个注解进行标注。一般公共的方法我会用上这个注解

@AutoWired

byType方式。把配置好的Bean拿来用，完成属性、方法的组装，它可以对类成员变量、方法及构
造函数进行标注，完成自动装配的工作。
当加上（required=false）时，就算找不到bean也不报错。

@RequestParam：
用在方法的参数前面。

```java
@RequestParam String a =request.getParameter("a")。
```


@PathVariable:
路径变量。

```java
RequestMapping("user/get/mac/{macAddress}")



public String getByMacAddress(@PathVariable String macAddress){



//do something;



}
```

参数与大括号里的名字一样要相同。

以上注解的示范

```java
/**



 * 用户进行评论及对评论进行管理的 Controller 类；



 */



@Controller



@RequestMapping("/msgCenter")



public class MyCommentController extends BaseController {



    @Autowired



    CommentService commentService;



 



    @Autowired



    OperatorService operatorService;



 



    /**



     * 添加活动评论；



     *



     * @param applyId 活动 ID；



     * @param content 评论内容；



     * @return



     */



    @ResponseBody



    @RequestMapping("/addComment")



    public Map<String, Object> addComment(@RequestParam("applyId") Integer applyId, @RequestParam("content") String content) {



        ....



        return result;



    }



}
```



```html
 @RequestMapping("/list/{applyId}")



    public String list(@PathVariable Long applyId, HttpServletRequest request, ModelMap modelMap) {



}
```

全局处理异常的：

@ControllerAdvice：

包含@Component。可以被扫描到。
统一处理异常。

@ExceptionHandler（Exception.class）：
用在方法上面表示遇到这个异常就执行以下方法。

```java
/**



 * 全局异常处理



 */



@ControllerAdvice



class GlobalDefaultExceptionHandler {



    public static final String DEFAULT_ERROR_VIEW = "error";



 



    @ExceptionHandler({TypeMismatchException.class,NumberFormatException.class})



    public ModelAndView formatErrorHandler(HttpServletRequest req, Exception e) throws Exception {



        ModelAndView mav = new ModelAndView();



        mav.addObject("error","参数类型错误");



        mav.addObject("exception", e);



        mav.addObject("url", RequestUtils.getCompleteRequestUrl(req));



        mav.addObject("timestamp", new Date());



        mav.setViewName(DEFAULT_ERROR_VIEW);



        return mav;



    }}
```



通过@value注解来读取application.properties里面的配置

```java
# face++ key



face_api_key = R9Z3Vxc7ZcxfewgVrjOyrvu1d-qR****



face_api_secret =D9WUQGCYLvOCIdsbX35uTH********
 @Value("${face_api_key}")



    private String API_KEY;



 



    @Value("${face_api_secret}")



    private String API_SECRET;
```

注意使用這個注解的时候 使用@Value的类如果被其他类作为对象引用，必须要使用注入的方式，而不能new。这个很重要，我就是被这个坑了

所以一般常用的配置都是配置在application.properties文件的