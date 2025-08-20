Spring-Boot的starter可以帮我们简化很多配置，非常的方便，定义起来其实也不复杂， 我们的项目中定义了很多starter，比如job的starter，以他为例，介绍如何定义starter。



## 添加依赖
添加Spring Boot的依赖：

```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>

</dependencies>

```



## 实现自动配置
在starter项目中，创建自动配置类。这个类要使用`@Configuration`注解，并根据条件使用`@ConditionalOn...`注解来条件化地配置beans。



首先是看有没有自定义的配置，如果有的话，添加@ConfigurationProperties注解，并且值和application.yml中相同



```java
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = XxlJobProperties.PREFIX)
public class XxlJobProperties {

    public static final String PREFIX = "spring.xxl.job";

    private boolean enabled;

    private String adminAddresses;

    private String accessToken;

    private String appName;

    private String ip;

    private int port;

    private String logPath;

    private int logRetentionDays = 30;

    //getter setter
}
```



接下来定义Configuration，并在里面创建需要的bean，如果有properites的话，就还要加@EnableConfigurationProperties注解

```java
import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
@EnableConfigurationProperties(XxlJobProperties.class)
public class XxlJobConfiguration {

    private static final Logger logger = LoggerFactory.getLogger(XxlJobConfiguration.class);

    @Autowired
    private XxlJobProperties properties;

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = XxlJobProperties.PREFIX, value = "enabled", havingValue = "true")
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(properties.getAdminAddresses());
        xxlJobSpringExecutor.setAppname(properties.getAppName());
        xxlJobSpringExecutor.setIp(properties.getIp());
        xxlJobSpringExecutor.setPort(properties.getPort());
        xxlJobSpringExecutor.setAccessToken(properties.getAccessToken());
        xxlJobSpringExecutor.setLogPath(properties.getLogPath());
        xxlJobSpringExecutor.setLogRetentionDays(properties.getLogRetentionDays());
        return xxlJobSpringExecutor;
    }
}
```



这里面用@Bean 注解声明了一个bean，并且使用`@ConditionalOnMissingBean`类指定这个bean的创建条件，即在缺失的时候创建。



`@ConditionalOnProperty(prefix = XxlJobProperties.PREFIX, value = "enabled", havingValue = "true")`约定了当我们配置了`spring.xxl.job.enable=true`的时候才会生效。



但是这样的话这个Cofniguration因为目录不同，所以导入这个依赖的话，程序也还是找不到configuration，



所以要下一步

## 创建配置类入口文件
在spring2.7以前，写的是spring.factories文件，2.7以后，为了云原生（native）考虑，改成了org.springframework.boot.autoconfigure.AutoConfiguration.imports文件，内容如下就是你的全类名。



以上就定义好了一个starter，只需要在需要的地方引入，并且配置上相应的配置项就行了，配置项内容就是我们定义在XxlJobProperties中的。

