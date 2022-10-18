### 什么是OpenFeign

OpenFeign是Spring在netflix宣布后续不再更新Feign之后自己基于Feign功能升级的微服务调用组件，在2020版之前内置了netflix的Ribbon，但是我们知道Ribbon其实也是停止了更新，所以2020及之后版本也是移除了Ribbon。文章以下内容将2020之前版本称之为旧版本，之后版本称之为新版本，OpenFeign就直接叫Feign。同样的其实netflix的Hystrix也是一样的结果。

### OpenFeign的用法

pom添加对应的依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

首先我们需要的是在消费者的启动类上声明EnableFeignClients注解

```java
@SpringCloudApplication
@EnableFeignClients
public class ServiceConsumerFeignApplication {

    public static void main(String[] args) {
        SpringApplication.run(ServiceConsumerFeignApplication.class, args);
    }
}
```

调用服务端接口时只需要一个一样的接口类添加FeignClient注解

```java
//value内容填写的时对应的提供者注册到注册中心时对应的服务名称
@FeignClient("service-provider")
public interface ConsumerFeignService {
    /**
     * 获取提供者提供的信息
     * @param request r
     * @return 数据
     */
    //这里写的和服务端请求地址等一样
    @PostMapping("/provider/userInfo.do")
    NormalResultDTO getProviderInfo(NormalRequestDTO request);
}
```

### 如何配置超时时间等条件

这个问题初看很简单，实际上还是有点来头的，因为我们单纯的没有使用feign的时候，使用Ribbon结合RestTemplate可以使用Ribbon进行超时配置，那么引入的是Feign的话，在旧版本里由于内部集成了Ribbon，Ribbon有个超时配置，Feign其实也有自己的超时配置，那我到底怎么配置？

首先优先级说一下：OpenFeign自己的配置优先级高于Ribbon的配置
```java
public class FeignLoadBalancer {

    private final RibbonProperties ribbon;
    protected int connectTimeout;
    protected int readTimeout;
    protected IClientConfig clientConfig;
    protected ServerIntrospector serverIntrospector;

    public FeignLoadBalancer(ILoadBalancer lb, IClientConfig clientConfig, ServerIntrospector serverIntrospector) {
        super(lb, clientConfig);
        this.setRetryHandler(RetryHandler.DEFAULT);
        this.clientConfig = clientConfig;
        this.ribbon = RibbonProperties.from(clientConfig);
        RibbonProperties ribbon = this.ribbon;
        //来自ribbon的连接超时时间
        this.connectTimeout = ribbon.getConnectTimeout();
        //来自ribbon的读取超时时间
        this.readTimeout = ribbon.getReadTimeout();
        this.serverIntrospector = serverIntrospector;
    }
    
    public FeignLoadBalancer.RibbonResponse execute(FeignLoadBalancer.RibbonRequest request, IClientConfig configOverride) throws IOException {
        Options options;
        //有自己的配置的时候
        if (configOverride != null) {
            //通过自己配置的clientConfig生成新的RibbonProperties
            RibbonProperties override = RibbonProperties.from(configOverride);
            //传递自己的时间配置，如果时间没有的情况下默认依旧走的是ribbon的时间配置
            options = new Options(override.connectTimeout(this.connectTimeout), override.readTimeout(this.readTimeout));
        } else {
            //否则的话走按照构造器内的ribbon的时间配置
            options = new Options(this.connectTimeout, this.readTimeout);
        }

        Response response = request.client().execute(request.toRequest(), options);
        return new FeignLoadBalancer.RibbonResponse(request.getUri(), response);
    }
}


public final class Request {
    public static class Options {
        private final int connectTimeoutMillis;
        private final int readTimeoutMillis;
        private final boolean followRedirects;

        public Options(int connectTimeoutMillis, int readTimeoutMillis, boolean followRedirects) {
            this.connectTimeoutMillis = connectTimeoutMillis;
            this.readTimeoutMillis = readTimeoutMillis;
            this.followRedirects = followRedirects;
        }

        public Options(int connectTimeoutMillis, int readTimeoutMillis) {
            this(connectTimeoutMillis, readTimeoutMillis, true);
        }

        public Options() {
            //本身feign的默认连接超时时间是10s和读取超时时间是60s
            this(10000, 60000);
        }

        public int connectTimeoutMillis() {
            return this.connectTimeoutMillis;
        }

        public int readTimeoutMillis() {
            return this.readTimeoutMillis;
        }

        public boolean isFollowRedirects() {
            return this.followRedirects;
        }
    }
}
```
首先是Feign的不配，Ribbon的不配，此时走的是Ribbon的默认超时时间，连接和读取都是1s
```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties
@Import({ HttpClientConfiguration.class, OkHttpRibbonConfiguration.class,
        RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class })
public class RibbonClientConfiguration {

    /**
     * Ribbon client default connect timeout. 默认的连接超时时间
     */
    public static final int DEFAULT_CONNECT_TIMEOUT = 1000;

    /**
     * Ribbon client default read timeout. 默认的读取超时时间
     */
    public static final int DEFAULT_READ_TIMEOUT = 1000;

    /**
     * Ribbon client default Gzip Payload flag.
     */
    public static final boolean DEFAULT_GZIP_PAYLOAD = true;

    @RibbonClientName
    private String name = "client";

    // TODO: maybe re-instate autowired load balancers: identified by name they could be
    // associated with ribbon clients

    @Autowired
    private PropertiesFactory propertiesFactory;

    @Bean
    @ConditionalOnMissingBean
    public IClientConfig ribbonClientConfig() {
        DefaultClientConfigImpl config = new DefaultClientConfigImpl();
        config.loadProperties(this.name);
        //没有我们配置的ribbon的连接超时时间就走的是上面的默认连接超时时间配置：1s
        config.set(CommonClientConfigKey.ConnectTimeout, DEFAULT_CONNECT_TIMEOUT);
        //同样的也是，优先走配置的读取超时时间，不然就走上面默认的读取超时时间：1s
        config.set(CommonClientConfigKey.ReadTimeout, DEFAULT_READ_TIMEOUT);
        config.set(CommonClientConfigKey.GZipPayload, DEFAULT_GZIP_PAYLOAD);
        return config;
    }
    
    //这里顺带的写一下，Ribbon读取的超时时间的配置key
    public abstract class CommonClientConfigKey<T> implements IClientConfigKey<T> {
        //连接超时的配置key
        public static final IClientConfigKey<Integer> ConnectTimeout = new CommonClientConfigKey<Integer>("ConnectTimeout") {
        };
        //读取超时的配置key
        public static final IClientConfigKey<Integer> ReadTimeout = new CommonClientConfigKey<Integer>("ReadTimeout") {
        };
    }
}
```



那么在我们自己不配置Feign的超时时间的时候就会走Ribbon的超时时间，我们自己配置Ribbon的超时时间，idea在yml这里不会进行智能提示，但是我们根据上面知道对应的配置key是什么，放心的填写就完事了/
```yml
ribbon:
  ReadTimeout: 5000
  ConnectTimeout: 5000
```

当然肯定是推荐使用Feign的本身的配置，新旧版本都会生效，如果你自身项目的Feign版本在2020版本之后，你还是看的过时资料使用Ribbon的配置去设置超时时间，其实是无效的就会走Feign默认的超时时间配置，无法做到控制连接时间。
```yml
feign:
  client:
    config:
      default: #默认所有对应的服务都会走这个配置，如果你要不同的服务调用超时时间不一致，那么就将服务提供者的名称在这里单独配置，优先单独配置，没有单独配置走默认配置
        connectTimeout: 5000
        readTimeout: 5000
      service-provider: #给单独服务指定单独的超时时间
        connectTimeout: 2000
        readTimeout: 6000
```