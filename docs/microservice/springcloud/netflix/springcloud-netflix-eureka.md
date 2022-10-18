### 什么是Eureka
eureka是由netflix公司推出的SpringCloud中服务注册和发现的组件，目前已经停止维护。

### Eureka的组件
- EurekaServer
  - 各个微服务节点通过配置启动后，会在EurekaServer中进行注册，这样EurekaServer中的服务注册表中将会存储所有可用服务节点的信息，服务节点的信息可以在界面中直观的看到。
- EurekaClient
  - EurekaClient是一个Java客户端，用于简化EurekaServer的交互，客户端同时也具备一个内置的，使用轮询负载算法的负载均衡器。在启动应用之后，将会向EurekaServer发送心跳（默认周期是30s）。如果EurekaServer在多个心跳周期内没有收到某个节点的心跳，EurekaServer将会从服务注册表中把这个服务节点移除。（默认90s）

### Eureka的自我保护模式
- 什么是自我保护模式
  - 保护模式主要用于一组客户端和EurekaServer之间存在网络分区场景下的为了避免因为网络问题导致错误剔除微服务实例而产生的一种机制。一旦进入保护模式，EurekaServer将会尝试保护其服务区注册表中的信息，不再删除服务注册表中的数据，也就是不会注销任何微服务。
- 触发的条件
  - 首先EurekaServer需要开启保护机制
    - eureka:server:enable-self-preservation: true 默认是开启的
  - 近十五分钟内所有的Eureka实例正常心跳占比，如果正常的低于85%就会触发
- 与非保护模式区别
  - EurekaServer每60s（默认）扫描一次，若是在90s（默认）内没有收到某个实例的心跳，那么就会将其删除，而在保护模式下不会删除。
    - 超时剔除时间配置
      - lease-expiration-duration-in-seconds: 90
      - 这个值由各个实例在配置文件自己指定，所以不同的实例可能存在不一样的过期时间
    - 扫描间隔配置
      - eviction-interval-timer-in-ms: 60
      - 这个值由server配置
  