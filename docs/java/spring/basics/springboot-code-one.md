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
        //设置初始化器，一下就给整了整整八个，后面在处理容器的时候就会调用这些初始化器来对容器进行初始化操作
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
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


别的以后再说，直接看看准备容器做了什么事情
```java
public class SpringApplication {

    private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
                                SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
        //没啥问题，之前初步处理的environment赋给容器，这里要情调一点啊，就是如apollo等配置内容还没有加载到environment中，要到spring的refresh容器刷新的时候才添加到environment中
        context.setEnvironment(environment);
        //没干啥正经事儿，里面前两个if其实都没有处理，也就放了个ConversionService到BeanFactory里去
        postProcessApplicationContext(context);
        //对容器应用所有的初始化器，一共八个，以后再说
        applyInitializers(context);
        //发布ApplicationContext已经准备好的时间，这时候还没有加载任何bean definition，只是调用的了ApplicationContextInitializers
        //这里面的listener前面也讲过就一个EventPublishingRunListener
        listeners.contextPrepared(context);
        //打打日志
        if (this.logStartupInfo) {
            logStartupInfo(context.getParent() == null);
            logStartupProfileInfo(context);
        }
        // 上面容器已经准备好了，这里先上往容器里添加几个boot专用的bean
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        if (printedBanner != null) {
            beanFactory.registerSingleton("springBootBanner", printedBanner);
        }
        if (beanFactory instanceof DefaultListableBeanFactory) {
            ((DefaultListableBeanFactory) beanFactory)
                    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
        if (this.lazyInitialization) {
            context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
        }
        //获取当前SpringApplication的primarySource
        //也就一个class org.springframework.cloud.bootstrap.BootstrapImportSelectorConfiguration
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        //内部是创建了一个BeanDefinitionLoader，定义了annotatedReader、xmlReader以及scanner
        //并且替换了annotatedReader、xmlReader以及scanner自身的environment为当前SpringApplication的environment
        //因为source BootstrapImportSelectorConfiguration 上加持了@Configuration注解，也就是有@Component，所以最后使用的就是annotatedReader来注册BootstrapImportSelectorConfiguration
        load(context, sources.toArray(new Object[0]));
        listeners.contextLoaded(context);
    }
    
}
```

refreshContext，核心就是spring容器的刷新过程，内部也是调用传说中大名鼎鼎的AbstractApplicationContext的refresh方法
```java
public abstract class AbstractApplicationContext  {
    
    @Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // 准备开始刷新了
            prepareRefresh();

            // Tell the subclass to refresh the internal bean factory.
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                //很重要的一点，首先我们知道后置处理器有两种，一种是BeanFactory的后置处理器，一种是Bean的后置处理器
                //BeanFactory后置处理器作为Spring初始化BeanFactory时的扩展点，我们可以自己定义后置处理器在容器实例化任何bean之前读取BeanDefinition，能够读取的同时也可以修改bean的定义
                //在这个方法中，会实例化调用所有的BeanFactoryPostProcessor以及其子类BeanDefinitionRegistryPostProcessor
                //两者优先级上是BeanDefinitionRegistryPostProcessor先执行，BeanFactoryPostProcessor后执行
                //而apollo就实现了一个SpringValueDefinitionProcessor的BeanDefinitionRegistryPostProcessor类型后置处理器，然后遍历了容器中所有的BeanDefinition，将属性中含有占位符${key}中的key找出来(不包括@Value注解的占位符)，key和field等做好关联，封装成SpringValueDefinition对象，放入集合中
                //同时这一步apollo也将远程的配置对应的properties放到了environment里的propertySources.propertySourceList的第一位，这里其实也涉及到占位符替换时，我们取对应的配置变量时是从environment的propertySources.propertySourceList从前往后遍历，取到key对应的value后就停止遍历，如果想要本地优先，那么就放到最后就能实现优先级变动。
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            } catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            } finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
}
```