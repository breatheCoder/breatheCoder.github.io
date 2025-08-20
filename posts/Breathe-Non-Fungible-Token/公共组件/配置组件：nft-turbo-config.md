
![[Pasted image 20250819155355.png]]

pom依赖中需要增加 Nacos 相关的配置信息：

我们把一些基础已经通用或者基础设施的配置放在config这一层，目前把nacos相关的配置放在config里面

```yaml
spring:  
  cloud:  
    nacos:  
      discovery:  
        server-addr: ${nft.turbo.nacos.server.url}  
      config:  
        server-addr: ${nft.turbo.nacos.server.url}  
        file-extension: properties  
        name: ${spring.application.name}
```

### 配置项详细解释

- spring.cloud.nacos.discovery.server-addr:
	- 这是Nacos服务发现的服务器地址。
- spring.cloud.nacos.config.server-addr:
	- 这是Nacos配置管理的服务器地址。
- spring.cloud.nacos.config.file-extension:
	- 这是配置文件的扩展名。
	- 这里设置为properties，表示配置文件是.properties格式。
- spring.cloud.nacos.config.name:
	- 配置文件的名称。
- ${spring.application.name}是一个占位符，它会被Spring应用的名称所替换。

