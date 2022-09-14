### Spring 基础知识汇总

#### Spring有哪些模块？
- aop
  - 面向切面编程
- aspects
  - 提供对aspect的支持
- beans
- context
- core
- expression
  - 表达式语言
- instrument
  - java装配支持
- jcl
  - 日志框架
- jdbc
  - 对jdbc的整合
- jms
  - 消息服务
- messaging
  - 消息统一实现标准
- orm
- oxm
  - XML编列
- test
  - 测试
- tx
  - 事务抽象
- web
- webflux
- webmvc
- websocket

#### 什么是Spring IoC？

- inversion of control(控制反转)，又可以称为好莱坞原则，不要给我们打电话，我们会给你打电话。
- IoC也就是由Spring来控制对象的生命周期以及关系，而非我们自己控制。

#### 依赖查找和依赖注入的区别？
- 依赖查找
  - 主动去IoC容器获取
  - 实现比较繁琐
  - 由于需要主动去获取，会侵入业务逻辑
  - 依赖于容器提供的API
  - 可读性较好
- 依赖注入
  - 由IoC容器提供数据，处于被动
  - 实现起来十分便利
  - 侵入性低
  - 不依赖于容器的API
  - 可读性较低

#### 依赖查找的安全性
依赖查找是否安全主要指的是底层是否有容错机制，如没有对应的bean时是否抛出异常。
- 单一类型的查找
  - BeanFactory#getBean 不安全
  - ObjectFactory#getBean 不安全
  - ObjectProvider#getIfAvailable 安全
- 集合类型查找
  - ListableBeanFactory#getBeansOfType 安全
  - ObjectProvider#Stream 安全

#### 依赖查找时经典的异常
Spring中定义了一个经典的BeansException，其对应的子类型均为常见的Bean相关的错误异常。
- NoSuchBeanDefinitionException
  - 若是查找的Bean在IoC容器中不存在时会触发，如之前说明的BeanFactory#getBean，ObjectFactory#getBean
- NoUniqueBeanDefinitionException
  - 按照类型进行查找时，IoC中存在多个实例，常见的就是多个实现都注册到了IoC
- BeanInstantiationException
  - 当Bean所对应的类型非具体的类时，如注册一个接口进IoC时
- BeanCreationException
  - 在Bean初始化过程中，也就是进行初始化方法的时候异常，如实现InitializingBean在afterPropertiesSet重写中异常，以及@PostConstrut注解声明的初始化方法
- BeanDefinitionStoreException
  - 当BeanDefinition配置元信息非法时，如XML资源无法打开

#### 依赖注入的模式
- 手动模式 配置或编程的方式，提前安排注入规则
  - XML资源配置元信息
  - Java注解配置元信息
  - API配置元信息
- 自动模式 实现方提供依赖自动关联的方式，按照内建的注入规则
  - Autowiring自动绑定

#### 依赖注入的类型
- Setter注入
  - 通过调用set方法进行注入，可以实现再次注入
- 构造器注入
  - 通过构造器参数进行注入
- 字段
  - 比如在成员变量上直接使用Autowired注解进行注入，不被推荐
- 接口回调
  - 如内建Aware接口

#### bean的生命周期

- bean的核心周期为：实例化->依赖注入->初始化->使用->销毁
  - 实例化的方式有哪些：
    - 使用构造器进行实例化
    - 使用静态工厂方法进行实例化
    - 使用Bean工厂方法
    - 使用FactoryBean
    - 通过serviceLoaderFactoryBean的方式：此种方式使用的是java的spi机制，通过使用serviceloader进行实现类的加载。
  - 初始化的方式有哪些：
    - @PostConstruct，容器刷新过程中会在最后初始化非lazy bean的时候进行回调对应bean中被该注解标记的方法。
    - 实现了InitializingBean接口，需要重写afterPropertiesSet，同样的容器刷新时最后进行初始化的时候会调用afterPropertiesSet方法，执行方法中自定义的初始化方式。
    - @Bean注解的时候指定对应的初始化方法
  - 销毁的方式有哪些：
    - 使用@preDestroy注解
    - 实现DisposableBean接口的destroy方法
    - 同样的是使用@Bean的时候指定对应的destroy的方法

#### Spring 后置处理器是做什么的？

- 在Spring中后置处理器分为：BeanFactory后置处理器和Bean后置处理器。其实后置处理器这种机制主要是为了在实例化对象之后进行一系列的操作。
- BeanFactory后置处理器：实例化BeanFactory之后进行的扫描操作由后置处理器进行。
- Bean后置处理器：实例化一个bean之后，使用后置处理器对对象进行加工，如以来注入，aop等操作。