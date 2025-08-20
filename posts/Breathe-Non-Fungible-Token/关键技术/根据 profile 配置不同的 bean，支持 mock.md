我们的项目中，为了方便测试，有很多环节都是需要证书和money的，所以提供掉了mock，可以mock掉这些服务。



实现方式其实不复杂，主要就是通过profile 来区分环境，然后做条件化的 Bean 配置



比如在SmsConfiguration中，我们定义了一个两个 bean，一个是smsService一个是mockSmsService：



```java
@Configuration
@EnableConfigurationProperties(SmsProperties.class)
public class SmsConfiguration {

    @Autowired
    private SmsProperties properties;

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = SmsProperties.PREFIX, value = "enabled", havingValue = "true")
    @Profile({"default","prod"})
    public SmsService smsService() {
        SmsServiceImpl smsService = new SmsServiceImpl();
        smsService.setHost(properties.getHost());
        smsService.setPath(properties.getPath());
        smsService.setAppcode(properties.getAppcode());
        smsService.setSmsSignId(properties.getSmsSignId());
        smsService.setTemplateId(properties.getTemplateId());
        return smsService;
    }

    @Bean
    @ConditionalOnMissingBean
    @Profile("dev")
    public SmsService mockSmsService() {
        MockSmsService smsService = new MockSmsService();
        return smsService;
    }

}
```



smsService被定义出来的前提是`@Profile({"default","prod"})`，而mockSmsService的被定义的前提是 profile 为 dev。



这样，我们只需要通过在配置文件中调整 `spring.profiles.active`的值，或者通过 JVM 的启动参数指定`spring.profiles.active`即可做一键开启 mock！

