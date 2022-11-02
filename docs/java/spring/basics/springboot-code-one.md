## SpringBoot的启动类
对于一个普通的SpringBoot项目而言，都会在项目包目录下存在一个包含main方法的启动类，作为入口，我们的分析也是从这里开始。首先注解@SpringBootApplication就有很多需要讲的，这个放在后面进行。

```java
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        //就从这里开始
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

run方法其实返回的就是我们的上下文容器ConfigurableApplicationContext，这里有个最简单获取上下文容器的方式，就是新建一个静态成员变量指向结果。
```java
@SpringBootApplication
public class DemoApplication {

    public static ConfigurableApplicationContext applicationContext;


    public static void main(String[] args) {
        //最简单的获取上下文容器的方式
        applicationContext = SpringApplication.run(DemoApplication.class, args);
    }

}
```

然后我们顺藤摸瓜，点着run方法就是一顿看
```java
public class SpringApplication {
    
    public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
        //秀还是spring秀
        return run(new Class<?>[] { primarySource }, args);
    }

    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        //先new一个SpringApplication实例出来
        return new SpringApplication(primarySources).run(args);
    }

    public SpringApplication(Class<?>... primarySources) {
        //调用构造方法
        this(null, primarySources);
    }

    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        //loader为空
        this.resourceLoader = resourceLoader;
        //判断一下primarySources是不是null，是的话抛出IllegalArgumentException异常
        Assert.notNull(primarySources, "PrimarySources must not be null");
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        //主要就是判断web应用是servlet还是reactive或者就是一个普通的web应用，内部是通过查询对应的类来进行判断的，放在后面说
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        //直接获取启动辅助类
        this.bootstrappers = new ArrayList<>(getSpringFactoriesInstances(Bootstrapper.class));
        //初始化
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //监听者
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        //内部就是在stackTrace里找含方法名叫做"main"的类名，然后通过Class.forName拿到对应类的Class信息
        //很明显，这个有main方法的就是我们写的入口类DemoApplication
        this.mainApplicationClass = deduceMainApplicationClass();
    }

    //经过上面我们获取了一个SpringApplication实例，接着就是启动Spring应用，并且创建和刷新上下文容器ApplicationContext
    public ConfigurableApplicationContext run(String... args) {
        //这玩意两个作用，第一就是记录时间，然后就是防止重复启动，这点主要就是看对应的start方法
        StopWatch stopWatch = new StopWatch();
        //判断了当前任务名称是否为空，为空抛出IllegalStateException异常，否则记录当前任务名称（这里就是记得空串），以及开始时间
        stopWatch.start();
        //创建一个默认启动引导容器，内部都是空的，没有值
        DefaultBootstrapContext bootstrapContext = createBootstrapContext();
        //容器对象不过是null
        ConfigurableApplicationContext context = null;
        configureHeadlessProperty();
        //获取监听器，获取到事件发布监听器，EventPublishingRunListener
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting(bootstrapContext, this.mainApplicationClass);
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //ConfigurableEnvironment是个很关键的类，内部包含一个propertySourceList里面就是各种我们的配置信息，yml配置内容等就是加载在environment里
            //如果想要自己实现配置中心并且和Spring结合，那么就需要认真研究一下
            ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            //这里是设置是否忽略beanInfo的搜索，在没配置的情况下默认置为True，以后再补充这里相关信息
            configureIgnoreBeanInfo(environment);
            //打印BANNER，内部会判断bannerMode处于什么模式，分别由三种，OFF不打印，CONSOLE控制台，LOG日志
            Banner printedBanner = printBanner(environment);
            //内部是按照类型创建context，这个类型也就是上面webApplicationType变量
            //如果是SERVLET：那么就创建的是AnnotationConfigServletWebServerApplicationContext
            //如果是REACTIVE：那么就创建的是AnnotationConfigReactiveWebServerApplicationContext
            //默认，也就是识别的是普通web应用那么久创建的是AnnotationConfigApplicationContext
            context = createApplicationContext();
            //为容器设置上下文工厂，使用的是DefaultApplicationStartup，用来打tag标记步骤
            //这个其实就是记录当前容器刷新到什么步骤了
            //如容器刷新完成的时候就会记录，见下方listeners.started(context);
            context.setApplicationStartup(this.applicationStartup);
            //准备容器
            prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            //容器刷新，经典核心内容，也就是常说的spring 容器刷新过程
            refreshContext(context);
            //刷新后处理
            afterRefresh(context, applicationArguments);
            //记一记时间，清除下任务名称，和start呼应起来
            stopWatch.stop();
            if (this.logStartupInfo) {
                //打印之
                new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
            }
            //doWithListeners("spring.boot.application.started", (listener) -> listener.started(context));
            listeners.started(context);
            callRunners(context, applicationArguments);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            listeners.running(context);
        }
        catch (Throwable ex) {
            handleRunFailure(context, ex, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }
    
}
```