mp中有很多好用的插件，通过这些插件，可以帮我们实现很多复杂的功能，在我们的项目中，用到了乐观锁插件，防止全表更新和分页插件。



插件的配置很简单，只需要依赖了mp以后，定义一个mybatisPlusInterceptor，然后在其中把想要的插件注册进去就可以了。



```java
@Configuration
public class DatasourceConfiguration {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        //乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        //防全表更新与删除插件
        interceptor.addInnerInterceptor(new BlockAttackInnerInterceptor());
        //分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

