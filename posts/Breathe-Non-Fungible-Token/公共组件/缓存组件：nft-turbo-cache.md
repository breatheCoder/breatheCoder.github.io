
![[Pasted image 20250819155239.png]]

这是我们项目中的缓存组件，他主要集成了 Redis、Redisson、Caffeine 以及 JetCache 等多个缓存组件。

同时，我们定义了一个 cache.yml 文件，在其中，把一些通用的配置放到里面配置好了，方面各个微服务在引用的时候可以简化一些通用化的配置：

```yaml
spring:  
  data:  
    redis:  
      host: ${nft.turbo.redis.url}  
      port: ${nft.turbo.redis.port}  
      password: ${nft.turbo.redis.password}  
      ssl:  
        enabled: true  
  redis:  
    redisson:  
      config: |  
        singleServerConfig:          idleConnectionTimeout: 10000          connectTimeout: 10000          timeout: 3000          retryAttempts: 3          retryInterval: 1500          password: ${nft.turbo.redis.password}          subscriptionsPerConnection: 5          clientName: null          address: "redis://${nft.turbo.redis.url}:${nft.turbo.redis.port}"          subscriptionConnectionMinimumIdleSize: 1          subscriptionConnectionPoolSize: 50          connectionMinimumIdleSize: 24          connectionPoolSize: 64          database: 0          dnsMonitoringInterval: 5000        threads: 16        nettyThreads: 32        codec: !<org.redisson.client.codec.StringCodec> {}        transportMode: "NIO"jetcache:  
  statIntervalMinutes: 1  
  areaInCacheName: false  
  local:  
    default:  
      type: caffeine  
      keyConvertor: fastjson2  
  remote:  
    default:  
      type: redisson  
      keyConvertor: fastjson2  
      broadcastChannel: ${spring.application.name}  
      keyPrefix: ${spring.application.name}  
      valueEncoder: java  
      valueDecoder: java  
      defaultExpireInMillis: 5000
```

