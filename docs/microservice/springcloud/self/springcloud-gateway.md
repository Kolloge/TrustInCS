### 什么是Gateway

###自定义断言

假设我不想使用gateway自带的一些断言，想要自己实现断言，那么可以按照以下方式进行自定义

```java
//方法的名称命名十分重要，一定要保持着自定义断言名称 + RoutePredicateFactory结尾方式
//如我以UID进行判断，那么我就可以起如下名称，同样的这里类名怎么起，对应的在配置文件中配置断言的时候就写对应的自定义名称，原因放在yml配置后面说明
@Component
public class UidRoutePredicateFactory extends AbstractRoutePredicateFactory<UidRoutePredicateFactory.UidConfig> {


    public UidRoutePredicateFactory(){
        super(UidRoutePredicateFactory.UidConfig.class);
    }

    @Override
    public List<String> shortcutFieldOrder() {
        //按照yml中自己配置的顺序进行读取，放置到config中
        return Collections.singletonList("minId");
    }

    @Override
    public Predicate<ServerWebExchange> apply(UidRoutePredicateFactory.UidConfig config) {
        return serverWebExchange -> {
            //从请求里拿取出来我们要判断的参数
            String uidStr = serverWebExchange.getRequest().getQueryParams().getFirst("uid");
            if (StringUtils.isNotBlank(uidStr)){
                //进行uid判断
                return Integer.parseInt(uidStr) > config.getMinId();
            }
            return true;
        };
    }

    @Data
    static class UidConfig {
        //接受配置的参数信息
        private int minId;
    }
}
```

对应我在yml文件中的配置

```yaml
server:
  port: 8090

spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: consumer-feign
          order: 1
          uri: lb://service-consumer-feign
          predicates:
            - Path=/feignGateway/**
            - Uid=1252 #这里就一定要注意，需要是自己自定义断言类时自己定义的名称
          filters:
            - StripPrefix=1
eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka01.eomon.com:7001/eureka

```

yml中断言名称是怎么和我自定义的断言类关联上的？

```java
public class RouteDefinitionRouteLocator implements RouteLocator, BeanFactoryAware, ApplicationEventPublisherAware {

    /**
     * Default filters name.
     */
    public static final String DEFAULT_FILTERS = "defaultFilters";

    protected final Log logger = LogFactory.getLog(getClass());

    private final RouteDefinitionLocator routeDefinitionLocator;

    private final ConfigurationService configurationService;

    //核心的便是这个map，维护的是断言及断言名的关系，后续也是通过名称来进行查找使用
    private final Map<String, RoutePredicateFactory> predicates = new LinkedHashMap<>();

    private final Map<String, GatewayFilterFactory> gatewayFilterFactories = new HashMap<>();

    private final GatewayProperties gatewayProperties;


    public RouteDefinitionRouteLocator(RouteDefinitionLocator routeDefinitionLocator,
                                       List<RoutePredicateFactory> predicates,
                                       List<GatewayFilterFactory> gatewayFilterFactories,
                                       GatewayProperties gatewayProperties,
                                       ConfigurationService configurationService) {
        this.routeDefinitionLocator = routeDefinitionLocator;
        this.configurationService = configurationService;
        //对应的初始化方法
        initFactories(predicates);
        gatewayFilterFactories.forEach(
                factory -> this.gatewayFilterFactories.put(factory.name(), factory));
        this.gatewayProperties = gatewayProperties;
    }

    //初始化断言factory的方法
    private void initFactories(List<RoutePredicateFactory> predicates) {
        predicates.forEach(factory -> {
            //factory获取name，这个是RoutePredicateFactory接口默认实现的
            String key = factory.name();
            if (this.predicates.containsKey(key)) {
                this.logger.warn("A RoutePredicateFactory named " + key
                        + " already exists, class: " + this.predicates.get(key)
                        + ". It will be overwritten.");
            }
            this.predicates.put(key, factory);
            if (logger.isInfoEnabled()) {
                logger.info("Loaded RoutePredicateFactory [" + key + "]");
            }
        });
    }

    //这个其实是RoutePredicateFactory接口的方法
    default String name() {
        return NameUtils.normalizeRoutePredicateName(getClass());
    }

    //这个就是NameUtils里对应的方法
    public static String normalizeRoutePredicateName(
            Class<? extends RoutePredicateFactory> clazz) {
        //这个getSimpleName是Class的方法，其实就是全路径名称根据.来截取的类名，也就是类名。
        //那么很明显，我们自己定义的断言类的名称是：UidRoutePredicateFactory
        //RoutePredicateFactory.class.getSimpleName()的名称是：RoutePredicateFactory
        //直接replace掉不就是剩下了个Uid了
        //所以放入map时的key就是Uid，yml配置的名称就会传入去map根据名称找到对应的断言处理
        return removeGarbage(clazz.getSimpleName()
                .replace(RoutePredicateFactory.class.getSimpleName(), ""));
    }
}
```
