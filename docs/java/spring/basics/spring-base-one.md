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
    - @bean注解的时候指定对应的初始化方法
  - 销毁的方式有哪些：
    - 使用@preDestroy注解
    - 实现DisposableBean接口的destroy方法
    - 同样的是使用@Bean的时候指定对应的destroy的方法

#### Spring 后置处理器是做什么的？

- 在Spring中后置处理器分为：BeanFactory后置处理器和Bean后置处理器。其实后置处理器这种机制主要是为了在实例化对象之后进行一系列的操作。
- BeanFactory后置处理器：实例化BeanFactory之后进行的扫描操作由后置处理器进行。
- Bean后置处理器：实例化一个bean之后，使用后置处理器对对象进行加工，如以来注入，aop等操作。