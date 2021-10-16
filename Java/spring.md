# 微服务
微服务是一种开发软件的架构和组织方法，其中软件由通过明确定义的 API 进行通信的小型独立服务组成。这些服务由各个小型独立团队负责。

微服务架构使应用程序更易于扩展和更快地开发，从而加速创新并缩短新功能的上市时间。
## 整体式架构与微服务架构
通过整体式架构，所有进程紧密耦合，并可作为单项服务运行。这意味着，如果应用程序的一个进程遇到需求峰值，则必须扩展整个架构。随着代码库的增长，添加或改进整体式应用程序的功能变得更加复杂。这种复杂性限制了试验的可行性，并使实施新概念变得困难。整体式架构增加了应用程序可用性的风险，因为许多依赖且紧密耦合的进程会扩大单个进程故障的影响。

使用微服务架构，将应用程序构建为独立的组件，并将每个应用程序进程作为一项服务运行。这些服务使用轻量级 API 通过明确定义的接口进行通信。这些服务是围绕业务功能构建的，每项服务执行一项功能。由于它们是独立运行的，因此可以针对各项服务进行更新、部署和扩展，以满足对应用程序特定功能的需求。

# IOC
## 何为控制，控制的是什么
是 bean 的创建、管理的权利，控制 bean 的整个生命周期。
## 何为反转，反转了什么
把这个权利交给了 Spring 容器，而不是自己去控制，就是反转。由之前的自己主动创建对象，变成现在被动接收别人给我们的对象的过程，这就是反转。

## 具体
IoC（Inverse of Control:控制反转）是一种设计思想，就是 将原本在程序中手动创建对象的控制权，交由Spring框架来管理。 IoC 在其他语言中也有应用，并非 Spirng 特有。 IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个Map（key，value）,Map 中存放的是各种对象。

将对象之间的相互依赖关系交给 IoC 容器来管理，并由 IoC 容器完成对象的注入。这样可以很大程度上简化应用的开发，把应用从复杂的依赖关系中解放出来。 IoC 容器就像是一个工厂一样，当我们需要创建一个对象的时候，只需要配置好配置文件/注解即可，完全不用考虑对象是如何被创建出来的。 在实际项目中一个 Service 类可能有几百甚至上千个类作为它的底层，假如我们需要实例化这个 Service，你可能要每次都要搞清这个 Service 所有底层类的构造函数，这可能会把人逼疯。如果利用 IoC 的话，你只需要配置好，然后在需要的地方引用就行了，这大大增加了项目的可维护性且降低了开发难度。

Spring 时代我们一般通过 XML 文件来配置 Bean，后来开发人员觉得 XML 文件来配置不太好，于是 SpringBoot 注解配置就慢慢开始流行起来。

![img](../img/ioc.png)

## 好处
解耦

它把对象之间的依赖关系转成用配置文件来管理，由 Spring IoC Container 来管理。

在项目中，底层的实现都是由很多个对象组成的，对象之间彼此合作实现项目的业务逻辑。但是，很多很多对象紧密结合在一起，一旦有一方出问题了，必然会对其他对象有所影响，所以才有了解藕的这种设计思想

在平时的JavaWeb应用程序开发中，我们要完成某一个模块功能时，可能会需要两个或以上的对象来一起协作完成，在以前没有使用Spring框架的时候，每个对象在需要别的对象协助时，都需要自己通过new的方式将协作的对象实例化出来，创建协作对象的权限在自己手上，自己需要哪个，就主动去创建就可以了，而这样做呢，会使得对象之间的耦合变高，A对象需要B对象时，A对象来实例化B对象，这样A对象就对B对象产生了依赖，也就是A对象和B对象之间存在一种紧密耦合关系，而使用了Spring框架之后就不一样了，创建B对象的工作交由Spring框架来帮我们完成，Spring创建好这个对象以后，会存储到一个容器里面，当A对象需要时，Spring就会从容器中注入进来，至于Spring是如何创建这个对象的，A对象其实并不需要关心，A对象需要B对象时，Spring来创建B对象，然后给A对象就可以，从而达到松耦合的目的。

# AOP
定义注解
```js
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface GlobalErrorCatch {

}
```
然后再指定注解形式的 pointcuts 及 around advice


```js
@Aspect
@Component
public class TestAdvice {
   // 1. 定义所有带有 GlobalErrorCatch 的注解的方法为 Pointcut
   @Pointcut("@annotation(com.example.demo.annotation.GlobalErrorCatch)")
   private void globalCatch(){}
   // 2. 将 around advice 作用于 globalCatch(){} 此 PointCut 
   @Around("globalCatch()")
   public Object handlerGlobalResult(ProceedingJoinPoint point) throws Throwable {
       try {
           return point.proceed();
       } catch (Exception e) {
           System.out.println("执行错误" + e);
           return ServiceResultTO.buildFailed("系统错误");
       }
   }

}
```

## 原理
代理
![img](../img/aop.png)
Client 是直接和 Proxy 打交道的，Proxy 是 Client 要真正调用的 RealSubject 的代理，它确实执行了 RealSubject 的 request 方法，不过在这个执行前后 Proxy 也加上了额外的 PreRequest()，afterRequest() 方法，注意 Proxy 和 RealSubject 都实现了 Subject 这个接口，这样在 Client 看起来调用谁是没有什么分别的（面向接口编程，对调用方无感，因为实现的接口方法是一样的），Proxy 通过其属性持有真正要代理的目标对象（RealSubject）以达到既能调用目标对象的方法也能在方法前后注入其它逻辑的目的


首先 Java 源代码经过编译生成字节码，然后再由 JVM 经过类加载，连接，初始化成 Java 类型，可以看到字节码是关键，静态和动态的区别就在于字节码生成的时机
## 静态代理
由程序员创建代理类或特定工具自动生成源代码再对其编译。在编译时已经将接口，被代理类（委托类），代理类等确定下来，在程序运行前代理类的.class文件就已经存在了.
```js
public interface Subject {
   public void request();
}

public class RealSubject implements Subject {
   @Override
   public void request() {
       // 卖房
       System.out.println("卖房");
   }
}

public class Proxy implements Subject {

   private RealSubject realSubject;

   public Proxy(RealSubject subject) {
       this.realSubject = subject;
   }


   @Override
   public void request() {
    // 执行代理逻辑
       System.out.println("卖房前");

       // 执行目标对象方法
       realSubject.request();

       // 执行代理逻辑
       System.out.println("卖房后");
   }

   public static void main(String[] args) {
       // 被代理对象
       RealSubject subject = new RealSubject();

       // 代理
       Proxy proxy = new Proxy(subject);

       // 代理请求
       proxy.request();
   }
}
```

### 劣势
1. 代理类只代理一个委托类（其实可以代理多个，但不符合单一职责原则），也就意味着如果要代理多个委托类，就要写多个代理（别忘了静态代理在编译前必须确定）
2. 第一点还不是致命的，再考虑这样一种场景：如果每个委托类的每个方法都要被织入同样的逻辑，比如说我要计算前文提到的每个委托类每个方法的耗时，就要在方法开始前，开始后分别织入计算时间的代码，那就算用代理类，它的方法也有无数这种重复的计算时间的代码

## 动态代理
在程序运行后通过反射创建生成字节码再由 JVM 加载而成。
### JDK动态代理
```js
// 委托类
public class RealSubject implements Subject {
   @Override
   public void request() {
       // 卖房
       System.out.println("卖房");
   }
}


import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;

public class ProxyFactory {

   private Object target;// 维护一个目标对象

   public ProxyFactory(Object target) {
       this.target = target;
   }

   // 为目标对象生成代理对象
   public Object getProxyInstance() {
       return Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(),
               new InvocationHandler() {

                   @Override
                   public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                       System.out.println("计算开始时间");
                       // 执行目标对象方法
                       method.invoke(target, args);
                       System.out.println("计算结束时间");
                       return null;
                   }
               });
   }

   public static void main(String[] args) {
       RealSubject realSubject = new RealSubject();
       System.out.println(realSubject.getClass());
       Subject subject = (Subject) new ProxyFactory(realSubject).getProxyInstance();
       System.out.println(subject.getClass());
       subject.request();
   }
}
打印结果如下:
shell
原始类:class com.example.demo.proxy.staticproxy.RealSubject
代理类:class com.sun.proxy.$Proxy0
计算开始时间
卖房
计算结束时间
```
#### 好处
由于动态代理是程序运行后才生成的，哪个委托类需要被代理到，只要生成动态代理即可，避免了静态代理那样的硬编码，另外所有委托类实现接口的方法都会在 Proxy 的 InvocationHandler.invoke() 中执行，这样如果要统计所有方法执行时间这样相同的逻辑，可以统一在 InvocationHandler 里写， 也就避免了静态代理那样需要在所有的方法中插入同样代码的问题，代码的可维护性极大的提高了。

### Spring AOP 的实现为啥却不用JDK动态代理
JDK 动态代理虽好，但也有弱点，我们注意到 newProxyInstance 的方法签名
```js
public static Object newProxyInstance(ClassLoader loader,
                                         Class<?>[] interfaces,
                                         InvocationHandler h);
```
注意第二个参数 Interfaces 是委托类的接口，是必传的， JDK 动态代理是通过与委托类实现同样的接口，然后在实现的接口方法里进行增强来实现的，<font color="green">这就意味着如果要用 JDK 代理，委托类必须实现接口，</font>这样的实现方式看起来有点蠢，更好的方式是什么呢，<font color="green">直接继承自委托类</font>不就行了，这样委托类的逻辑不需要做任何改动，CGlib 就是这么做的。


### CGLib动态代理
AOP 就是用的 CGLib 的形式来生成的，JDK 动态代理使用 Proxy 来创建代理类，增强逻辑写在 InvocationHandler.invoke() 里，CGlib 动态代理也提供了类似的  Enhance 类，增强逻辑写在 MethodInterceptor.intercept() 中，也就是说所有委托类的非 final 方法都会被方法拦截器拦截，在说它的原理之前首先来看看它怎么用的
```js
public class MyMethodInterceptor implements MethodInterceptor {
   @Override
   public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
       System.out.println("目标类增强前！！！");
       //注意这里的方法调用，不是用反射哦！！！
       Object object = proxy.invokeSuper(obj, args);
       System.out.println("目标类增强后！！！");
       return object;
   }
}

public class CGlibProxy {
   public static void main(String[] args) {
       //创建Enhancer对象，类似于JDK动态代理的Proxy类，下一步就是设置几个参数
       Enhancer enhancer = new Enhancer();
       //设置目标类的字节码文件
       enhancer.setSuperclass(RealSubject.class);
       //设置回调函数
       enhancer.setCallback(new MyMethodInterceptor());

       //这里的creat方法就是正式创建代理类
       RealSubject proxyDog = (RealSubject) enhancer.create();
       //调用代理类的eat方法
       proxyDog.request();
   }
}
```
打印如下
```
代理类:class com.example.demo.proxy.staticproxy.RealSubject$$EnhancerByCGLIB$$889898c5
目标类增强前！！！
卖房
目标类增强后！！！
```
可以看到主要就是利用 Enhancer 这个类来设置委托类与方法拦截器，这样委托类的所有非 final 方法就能被方法拦截器拦截，从而在拦截器里实现增强

#### CGlib 动态代理使用上的限制
第一点之前已经已经说了，只能代理委托类中任意的非 final 的方法，另外它是通过继承自委托类来生成代理的，所以如果委托类是 final 的，就无法被代理了（final 类不能被继承）
#### JDK 动态代理的拦截对象是通过反射的机制来调用被拦截方法的，CGlib 呢，它通过什么机制来提升了方法的调用效率。
由于反射的效率比较低，所以 CGlib 采用了FastClass 的机制来实现对被拦截方法的调用。FastClass 机制就是对一个类的方法建立索引，通过索引来直接调用相应的方法，建议参考下https://www.cnblogs.com/cruze/p/3865180.html这个链接好好学学

# Spring启动流程 
[详细](https://mp.weixin.qq.com/s/ut3mRwhfqXNjrBtTmI0oWg)

# springboot启动流程
SpringBoot相对于spring，内置了tomcat，通过启动类启动，配置也集中在一个application.yml/properties中。
## 启动类
```js
@SpringBootApplication
public class SpringmvcApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringmvcApplication.class, args);
    }
}
```
把启动类分解一下，实际上就是两部分:
- @SpringBootApplication注解
- 一个main()方法，里面调用SpringApplication.run()方法。
  
## @SpringBootApplication

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

}
```
很明显，@SpringBootApplication注解由三个注解组合而成，分别是：

- @ComponentScan
- @EnableAutoConfiguration
- @SpringBootConfiguration

###  @ComponentScan
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Repeatable(ComponentScans.class)
public @interface ComponentScan {
    
}
```
这个注解的作用是告诉Spring扫描哪个包下面类，加载符合条件的组件(比如贴有@Component和@Repository等的类)或者bean的定义。
所以有一个basePackages的属性，如果默认不写，则从声明@ComponentScan所在类的package进行扫描。
所以启动类最好定义在Root package下，因为一般我们在使用@SpringBootApplication时，都不指定basePackages的。
### @EnableAutoConfiguration
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    
}
```
这是一个复合注解，看起来很多注解，实际上关键在@Import注解，它会加载AutoConfigurationImportSelector类，然后就会触发这个类的selectImports()方法。根据返回的String数组(配置类的Class的名称)加载配置类。

```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
    //返回的String[]数组，是配置类Class的类名
    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        if (!isEnabled(annotationMetadata)) {
            return NO_IMPORTS;
        }
        AutoConfigurationEntry autoConfigurationEntry = getAutoConfigurationEntry(annotationMetadata);
        //返回配置类的类名
        return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
    }
}
```
我们一直点下去，就可以找到最后的幕后英雄，就是SpringFactoriesLoader类，通过loadSpringFactories()方法加载META-INF/spring.factories中的配置类。
![img](../img/springboot1.png)
![img](../img/springboot2.png)
这里使用了spring.factories文件的方式加载配置类，提供了很好的扩展性。

所以@EnableAutoConfiguration注解的作用其实就是开启自动配置，自动配置主要则依靠这种加载方式来实现。

### @SpringBootConfiguration
继承自@Configuration，二者功能也一致，标注当前类是配置类， 并会将当前类内声明的一个或多个以@Bean注解标记的方法的实例纳入到spring容器中，并且实例名就是方法名。

## 小结
![img](../img/sp3.png)

## SpringApplication类
接下来讲main方法里执行的这句代码，这是SpringApplication类的静态方法run()。
```java
//启动类的main方法
public static void main(String[] args) {
    SpringApplication.run(SpringmvcApplication.class, args);
}

//启动类调的run方法
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    //调的是下面的，参数是数组的run方法
    return run(new Class<?>[] { primarySource }, args);
}

//和上面的方法区别在于第一个参数是一个数组
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    //实际上new一个SpringApplication实例，调的是一个实例方法run()
    return new SpringApplication(primarySources).run(args);
}
```
通过上面的源码，发现实际上最后调的并不是静态方法，而是实例方法，需要new一个SpringApplication实例，这个构造器还带有一个primarySources的参数。所以我们直接定位到构造器。

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;
    //断言primarySources不能为null，如果为null，抛出异常提示
    Assert.notNull(primarySources, "PrimarySources must not be null");
    //启动类传入的Class
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //判断当前项目类型，有三种：NONE、SERVLET、REACTIVE
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //设置ApplicationContextInitializer
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //设置监听器
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //判断主类，初始化入口类
    this.mainApplicationClass = deduceMainApplicationClass();
}

//判断主类
private Class<?> deduceMainApplicationClass() {
    try {
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}
```
![img](../img/sp4.png)

创建了SpringApplication实例之后，就完成了SpringApplication类的初始化工作，这个实例里包括监听器、初始化器，项目应用类型，启动类集合，类加载器。如图所示。

![img](../img/sp5.png)

得到SpringApplication实例后，接下来就调用实例方法run()。继续看。

```java
public ConfigurableApplicationContext run(String... args) {
    //创建计时器
    StopWatch stopWatch = new StopWatch();
    //开始计时
    stopWatch.start();
    //定义上下文对象
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    //Headless模式设置
    configureHeadlessProperty();
    //加载SpringApplicationRunListeners监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    //发送ApplicationStartingEvent事件
    listeners.starting();
    try {
        //封装ApplicationArguments对象
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        //配置环境模块
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        //根据环境信息配置要忽略的bean信息
        configureIgnoreBeanInfo(environment);
        //打印Banner标志
        Banner printedBanner = printBanner(environment);
        //创建ApplicationContext应用上下文
        context = createApplicationContext();
        //加载SpringBootExceptionReporter
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                                                         new Class[] { ConfigurableApplicationContext.class }, context);
        //ApplicationContext基本属性配置
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        //刷新上下文
        refreshContext(context);
        //刷新后的操作，由子类去扩展
        afterRefresh(context, applicationArguments);
        //计时结束
        stopWatch.stop();
        //打印日志
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        //发送ApplicationStartedEvent事件，标志spring容器已经刷新，此时所有的bean实例都已经加载完毕
        listeners.started(context);
        //查找容器中注册有CommandLineRunner或者ApplicationRunner的bean，遍历并执行run方法
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        //发送ApplicationFailedEvent事件，标志SpringBoot启动失败
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        //发送ApplicationReadyEvent事件，标志SpringApplication已经正在运行，即已经成功启动，可以接收服务请求。
        listeners.running(context);
    }
    catch (Throwable ex) {
        //报告异常，但是不发送任何事件
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```
结合注释和源码，其实很清晰了，为了加深印象，画张图看一下整个流程。

![img](../img/sp6.png)

# SpringMVC理解

MVC 是一种设计模式,Spring MVC 是一款很优秀的 MVC 框架。Spring MVC 可以帮助我们进行更简洁的Web层的开发，并且它天生与 Spring 框架集成。Spring MVC 下我们一般把后端项目分为 Service层（处理业务）、Dao层（数据库操作）、Entity层（实体类）、Controller层(控制层，返回数据给前台页面)。

![img](../img/springmvc.png)


# Spring 框架中用到了哪些设计模式
- 工厂设计模式 : Spring使用工厂模式通过 BeanFactory、ApplicationContext 创建 bean 对象。

- 代理设计模式 : Spring AOP 功能的实现。

- 单例设计模式 : Spring 中的 Bean 默认都是单例的。

- 模板方法模式 : Spring 中 jdbcTemplate、hibernateTemplate 等以 Template 结尾的对数据库操作的类，它们就使用到了模板模式。

- 包装器设计模式 : 我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。这种模式让我们可以根据客户的需求能够动态切换不同的数据源。

- 观察者模式: Spring 事件驱动模型就是观察者模式很经典的一个应用。

- 适配器模式 :Spring AOP 的增强或通知(Advice)使用到了适配器模式、spring MVC 中也是用到了适配器模式适配Controller。

# Mybatis

MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架。

MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及对结果集的检索封装。

MyBatis 可以对配置和原生 Map 使用简单的 XML 或注解，将接口和Java的POJOs（Plain Old Java Objects（普通的 Java 对象））映射成数据库中的记录。

MyBatis 主要思想是将程序中的大量SQL语句抽取出来，配置在配置文件中，以实现SQL的灵活配置。

MyBatis 并不完全是一种ORM框架，它的设计思想和ORM相似，只是它允许直接编写SQL语句，使得数据库访问更加灵活。

MyBatis 提供了一种“半自动化”的ORM实现，是一种“SQL Mapping”框架。

# @Component 和 @Bean 的区别是什么

1. 作用对象不同: @Component 注解作用于类，而@Bean注解作用于方法。

2. @Component通常是通过类路径扫描来自动侦测以及自动装配到Spring容器中（我们可以使用 @ComponentScan 注解定义要扫描的路径从中找出标识了需要装配的类自动装配到 Spring 的 bean 容器中）。@Bean 注解通常是我们在标有该注解的方法中定义产生这个 bean,@Bean告诉了Spring这是某个类的示例，当我需要用它的时候还给我。

3. @Bean 注解比 Component 注解的自定义性更强，而且很多地方我们只能通过 @Bean 注解来注册bean。比如当我们引用第三方库中的类需要装配到 Spring容器时，则只能通过 @Bean来实现。

@Bean注解使用示例：

```java
@Configuration
public class AppConfig {
    @Bean
    public TransferService transferService() {
        return new TransferServiceImpl();
    }

}
```

下面这个例子是通过 @Component 无法实现的。

```java
@Bean
public OneService getService(status) {
    case (status)  {
        when 1:
                return new serviceImpl1();
        when 2:
                return new serviceImpl2();
        when 3:
                return new serviceImpl3();
    }
}
```

# 将一个类声明为Spring的 bean 的注解有哪些?

我们一般使用 @Autowired 注解自动装配 bean，要想把类标识成可用于 @Autowired 注解自动装配的 bean 的类,采用以下注解可实现：

- @Component ：通用的注解，可标注任意类为 Spring 组件。如果一个Bean不知道属于哪个层，可以使用@Component 注解标注。

- @Repository : 对应持久层即 Dao 层，主要用于数据库相关操作。

- @Service : 对应服务层，主要涉及一些复杂的逻辑，需要用到 Dao层。

- @Controller : 对应 Spring MVC 控制层，主要用户接受用户请求并调用 Service 层返回数据给前端页面。



## ORM 基本映射关系：
数据表映射类。
数据表的行映射对象（实例）。
数据表的列（字段）映射对象的属性。

![img](../img/mybatis.png)

## MyBatis功能结构
1. API接口层：
提供给外部使用的接口API，开发人员通过这些本地API来操纵数据库。接口层接收到调用请求就会调用数据处理层来完成具体的数据处理。
2. 数据处理层：
   负责具体的SQL查找、SQL解析、SQL执行和执行结果映射处理等。它主要的目的是根据调用的请求完成一次数据库操作。
3. 基础支撑层：负责最基础的功能支撑，包括连接管理、事务管理、配置加载和缓存处理，这些都是共用的东西，将他们抽取出来作为最基础的组件，为上层的数据处理层提供最基础的支撑。

![img](../img/mybatis1.png)

## MyBatis 框架结构
1. 加载配置：
MyBatis 应用程序根据XML配置文件加载运行环境，创建SqlSessionFactory, SqlSession，将SQL的配置信息加载成为一个个MappedStatement对象（包括了传入参数映射配置、执行的SQL语句、结果映射配置），存储在内存中。（配置来源于两个地方，一处是配置文件，一处是 Java 代码的注解）
2. SQL解析：
当API接口层接收到调用请求时，会接收到传入SQL的ID和传入对象（可以是Map、JavaBean或者基本数据类型），MyBatis 会根据SQL的ID找到对应的MappedStatement，然后根据传入参数对象对MappedStatment进行解析，解析后可以得到最终要执行的SQL语句和参数。
3. SQL的执行：
SqlSession将最终得到的SQL和参数拿到数据库进行执行，得到操作数据库的结果。
4. 结果映射：
将操作数据库的结果按照映射的配置进行转换，可以转换成HashMap、JavaBean或者基本数据类型，并将最终结果返回，用完之后关闭SqlSession。

SqlSessionFactory：
>每个基于MyBatis的应用都是以一个SqlSessionFactory的实例为核心的。
SqlSessionFactory是单个数据库映射关系经过编译后的内存映像。
SqlSessionFactory的实例可以通过SqlSessionFactoryBuilder获得。
而SqlSessionFactoryBuilder则可以从XML配置文件或一个预先定制的Configuration的实例构建出SqlSessionFactory的实例。
SqlSessioFactory是创建SqlSession的工厂。

SqlSession:
>SqlSession是执行持久化操作的对象，它完全包含了面向数据库执行SQL命令所需的所有方法。
可以通过SqlSession实例来直接执行已映射的SQL语句，在使用完SqlSession后我们应该使用finally块来确保关闭它。

## MyBatis、JDBC、Hibernate的区别：
- MyBatis也是基于JDBC的，Java与数据库操作仅能通过JDBC完成。MyBatis也要通过JDBC完成数据查询、更新这些动作。
- MyBatis仅仅是在JDBC基础上做了 OO化、封装事务管理接口这些东西。
- MyBatis和Hibernate都屏蔽JDBC API的底层访问细节，使我们不用跟JDBC API打交道就可以访问数据库。
- 但是，Hibernate是全自动的ORM映射工具，可以自动生成SQL语句。
- MyBatis需要在xml配置文件中写SQL语句。
- 因为Hibernate是自动生成SQL语句的，在写复杂查询时，Hibernate实现比MyBatis复杂的多。

## 通常一个 Xml 映射文件，都会写一个 Dao 接口与之对应，请 问，这个 Dao 接口的工作原理是什么?Dao 接口里的方法，参数不同时，方法能重载吗?

Dao 接口，就是人们常说的 Mapper 接口，接口的全限名，就是映射文件中的 namespace 的值， 接口的方法名，就是映射文件中MappedStatement的 id 值，接口方法内的参数，就是传递给 sql 的参数。Mapper接口是没有实现类的，当调用接口方法时，接口全限名+方法名拼接字符串作为 key 值，可唯一定位一个 MappedStatement。

举例:com.mybatis3.mappers.StudentDao.findStudentById，可以唯一找到 namespace 为com.mybatis3.mappers.StudentDao下面id = findStudentById的MappedStatement。 在 Mybatis 中，每一个 ```<select>```、```<insert>```、```<update>```、```<delete>```标签，都会被解析为一个 MappedStatement 对象。

Dao 接口里的方法，是不能重载的，因为是全限名+方法名的保存和寻找策略。
Dao 接口的工作原理是 JDK 动态代理，Mybatis 运行时会使用 JDK 动态代理为 Dao 接口生成代理 proxy 对象，代理对象 proxy 会拦截接口方法，转而执行MappedStatement所代表的 sql，然后将 sql 执行结果返回。

## 为什么说 Mybatis 是半自动 ORM 映射工具?它与全自动的区别在哪里?
Hibernate 属于全自动 ORM 映射工具，使用 Hibernate 查询关联对象或者关联集合对象时，可以 根据对象关系模型直接获取，所以它是全自动的。而 Mybatis 在查询关联对象或关联集合对象时，需 要手动编写 sql 来完成，所以，称之为半自动 ORM 映射工具。