2025年07月14日 22:53  
自定义oneThread-SpringBoot-Starter基础组件，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



***内容摘要：本章节深入探讨了如何通过SpringBootStarter统一管理动态线程池，包括** 动态线程池标记* \*、\*\* 远程配置覆盖\*\*、\*\* Starter 开发**以及** 可插拔设计\*\*等功能。

课程目录如下所示：

*  
如何发现动态线程池？  
*  
Spring 后置处理器  
*  
开发 SpringBoot Starter  
*  
关于启用动态线程池标识  
*  
  文末总结

本章节将涉及到 `core`、`spring-base`、`starter/common-spring-boot-starter`、`nacos-cloud-example` 四个模块。

![iShot_2025-06-26_20.53.59.png](https://article-images.zsxq.com/FnGsvdmA6cTDjXJhsUxryzhs5XPB "iShot_2025-06-26_20.53.59.png")

> 这篇文章涉及多个模块之间的协作，**不像单点功能那样聚焦明确**，对于没接触过复杂多模块系统的同学，可能阅读起来会有一点"晕"。
>
> 不过没关系，文中我已经尽可能**梳理了关键流程和细节** ，帮助大家理解模块之间的关系。如果在阅读或实践过程中还有不清楚的地方，**也欢迎随时留言提问，我们一起讨论～**

如何发现动态线程池？
----------

上一章节我们已经了解了 SpringBoot Starter 的基本概念，本章节将具体介绍如何借助 SpringBoot Starter 将线程池注册到统一的线程池容器 `OneThreadRegistry` 中。

在开始之前，先提出一个问题：如何统一管理动态线程池？我想到的一个简单易行的方法是，将每个线程池定义为一个 Spring 的 Bean，并通过自定义的注解标记为动态线程池，如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池注解
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Target({ElementType.METHOD, ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface DynamicThreadPool {
    }
              
参考动态线程池创建的示例代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Bean
    @DynamicThreadPool
    public ThreadPoolExecutor onethreadProducer() {
        return ThreadPoolExecutorBuilder.builder()
                .threadPoolId("onethread-producer")
                .corePoolSize(2)
                .maximumPoolSize(4)
                .keepAliveTime(9999L)
                .awaitTerminationMillis(5000L)
                .workQueueType(BlockingQueueTypeEnum.SYNCHRONOUS_QUEUE)
                .threadFactory("onethread-producer_")
                .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                .dynamicPool()
                .build();
    }
              
到这里，可能有同学会疑惑，仅仅标记 `@Bean` 和 `@DynamicThreadPool` 就可以把动态线程池注册到统一的容器里吗？答案显然是否定的。

上述代码只是对动态线程池的标记，要想真正将它们加入统一管理的容器，还需要借助 Spring 提供的后置处理器 `BeanPostProcessor`。

Spring 后置处理器 {#spring}
----------------------

### 1. 逻辑概述 {#1}

后置处理器除了将动态线程池注册到统一容器 `OneThreadRegistry` 外，还承担另一个重要功能：从配置中心读取远程线程池配置并覆盖本地配置。

通俗地讲，就是尽管你本地定义了线程池的配置参数，但这些参数可能并不会被使用，而是在项目启动时，自动从远程配置中心（如 Nacos）拉取最新的线程池参数并生效。

配置示例如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      nacos:
        data-id: onethread-nacos-cloud-example-ding-ma.yaml
        group: DEFAULT_GROUP
      config-file-type: YAML
      web:
        core-pool-size: 17
        maximum-pool-size: 26
        keep-alive-time: 60
        notify:
          receives: xxx
      notify-platforms:
        platform: DING
        url: https://oapi.dingtalk.com/robot/send?access_token=xxx
      executors:
      - thread-pool-id: onethread-producer
        core-pool-size: 14
        maximum-pool-size: 22
        queue-capacity: 1999
        work-queue: ResizableCapacityLinkedBlockingQueue
        rejected-handler: DiscardOldestPolicy
        keep-alive-time: 160
        allow-core-thread-time-out: false
        notify:
          receives: xxx
          interval: 10
        alarm:
          enable: false
          queue-threshold: 90
          active-threshold: 90
      - thread-pool-id: onethread-consumer
        core-pool-size: 10
        maximum-pool-size: 20
        queue-capacity: 1024
        work-queue: LinkedBlockingQueue
        rejected-handler: AbortPolicy
        keep-alive-time: 9999
        allow-core-thread-time-out: true
        notify:
          receives: xxx
          interval: 5
        alarm:
          enable: false
          queue-threshold: 80
          active-threshold: 80
              
远端配置只覆盖配置中心中定义的参数，其他如线程工厂的定义则不会被覆盖。

### 2. 远端配置读取 {#2}

远程配置读取逻辑如下，以 Nacos 示例程序为例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    server:
      port: 18080  # 应用服务端口，启动后访问地址为 http://localhost:18080
    ​
    spring:
      application:
        name: nacos-cloud-example${unique-name:}  # Spring 应用名称，支持通过 unique-name 参数自定义服务名，方便多实例区分
      config:
        import: nacos:onethread-nacos-cloud-example${unique-name:}.yaml  # 从 Nacos 导入指定配置文件
      profiles:
        active: dev  # 激活的配置环境（开发环境）
    ​
      cloud:
        nacos:
          config:
            username: nacos  # 连接 Nacos 的用户名
            password: nacos  # 连接 Nacos 的密码
            file-extension: yaml  # 指定配置文件的后缀类型为 YAML
            extension-configs:
              - data-id: onethread-nacos-cloud-example${unique-name:}.yaml  # 指定扩展配置文件的 dataId
                group: DEFAULT_GROUP  # 配置文件所在的 Nacos 分组
                refresh: true  # 是否开启自动刷新，当 Nacos 配置变更时自动更新本地配置
          server-addr: 127.0.0.1:8848  # Nacos 服务器地址，默认端口为 8848
    ​
              
在这段配置中，有两个关键点需要特别关注：

*  
**`-data-id`** ：这是我们在 Nacos 中创建的配置文件的唯一标识，应用会根据它来读取对应的远程配置内容。一个项目可以配置多个 `data-id`，实现灵活的模块化配置。  
*  
  **`spring.config.import`** ：这里尤其值得注意。在 SpringBoot2 中，远程配置默认会自动合并到本地配置中；但从 Spring Boot 3 开始，**如果不通过`spring.config.import`明确指定远程配置文件的来源，SpringBoot将不会加载这些配置** 。因此，如果省略了该项，下面提到的"远程配置覆盖本地配置"的功能将无法生效。

此外，配置中心中存储的参数本质上是**字符串形式** 的键值对，直接使用时不够直观也不便于管理。在 Java 应用中，我们通常会将其**绑定为配置类的属性对象** ，这样更便于类型转换、代码提示和后续维护。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * oneThread 配置中心参数
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Data
    public class BootstrapConfigProperties {
    ​
        public static final String PREFIX = "onethread";
    ​
        /**
         * 是否开启动态线程池开关
         */
        private Boolean enable = Boolean.TRUE;
    ​
        /**
         * Nacos 配置文件
         */
        private NacosConfig nacos;
    ​
        /**
         * Apollo 配置文件
         */
        private ApolloConfig apollo;
    ​
        /**
         * Web 线程池配置
         */
        private WebThreadPoolExecutorConfig web;
    ​
        /**
         * Nacos 远程配置文件格式类型
         */
        private ConfigFileTypeEnum configFileType;
    ​
        /**
         * 通知配置
         */
        private NotifyPlatformsConfig notifyPlatforms;
    ​
        /**
         * 监控配置
         */
        private MonitorConfig monitorConfig = new MonitorConfig();
    ​
        /**
         * 线程池配置集合
         */
        private List<ThreadPoolExecutorProperties> executors;
    ​
        @Data
        public static class NotifyPlatformsConfig {
    ​
            /**
             * 通知类型，比如：DING
             */
            private String platform;
    ​
            /**
             * 完整 WebHook 地址
             */
            private String url;
        }
    ​
        @Data
        public static class MonitorConfig {
    ​
            /**
             * 默认开启监控配置
             */
            private Boolean enable = Boolean.TRUE;
    ​
            /**
             * 监控类型
             */
            private String collectType = "micrometer";
    ​
            /**
             * 采集间隔，默认 10 秒
             */
            private Long collectInterval = 10L;
        }
    ​
        @Data
        public static class NacosConfig {
    ​
            private String dataId;
    ​
            private String group;
        }
    ​
        @Data
        public static class ApolloConfig {
    ​
            private String namespace;
        }
    ​
        @Data
        public static class WebThreadPoolExecutorConfig {
    ​
            /**
             * 核心线程数
             */
            private Integer corePoolSize;
    ​
            /**
             * 最大线程数
             */
            private Integer maximumPoolSize;
    ​
            /**
             * 线程空闲存活时间（单位：秒）
             */
            private Long keepAliveTime;
    ​
            /**
             * 通知配置
             */
            private NotifyConfig notify;
        }
    ​
        @Data
        @NoArgsConstructor
        @AllArgsConstructor
        public static class NotifyConfig {
    ​
            /**
             * 接收人集合
             */
            private String receives;
        }
    ​
        private static BootstrapConfigProperties INSTANCE = new BootstrapConfigProperties();
    ​
        public static BootstrapConfigProperties getInstance() {
            return INSTANCE;
        }
    ​
        public static void setInstance(BootstrapConfigProperties properties) {
            INSTANCE = properties;
        }
    }
              
这里还涉及一层设计上的逻辑：**`core`包本身无法直接获取Spring容器中的Bean** ，但我们又希望在 `core` 包中的组件（如线程池监控、告警模块）能够使用远程配置中心下发的线程池参数。

为了解决这个矛盾，在装配 `BootstrapConfigProperties` Bean 时，我们做了一些处理，使用了一个 **"小技巧"** ：在 Bean 创建完成后，**将其实例手动赋值给类中的静态单例变量** ，从而实现全局共享。

这样，即使在非 Spring 环境下的模块中（比如 `core` 包），也可以通过 `BootstrapConfigProperties.getInstance()` 的方式获取到线程池的配置参数。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class CommonAutoConfiguration {
    ​
        @Bean
        public BootstrapConfigProperties bootstrapConfigProperties(Environment environment) {
            BootstrapConfigProperties bootstrapConfigProperties = Binder.get(environment)
                    .bind(BootstrapConfigProperties.PREFIX, Bindable.of(BootstrapConfigProperties.class))
                    .get();
            BootstrapConfigProperties.setInstance(bootstrapConfigProperties);
            return bootstrapConfigProperties;
        }
    }
              
通常情况下，我们只需在 `BootstrapConfigProperties` 类上添加 `@ConfigurationProperties(prefix = "onethread")` 注解，Spring Boot 就会自动完成属性的绑定，无需如此复杂的处理逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @ConfigurationProperties(prefix = "onethread")
    public class BootstrapConfigProperties {
      
        // ......
    }
    ​
    public class CommonAutoConfiguration {
    ​
        @Bean
        public BootstrapConfigProperties bootstrapConfigProperties() {
            return new BootstrapConfigProperties();
        }
    }
              
如果是一个**常规的SpringBootStarter项目** ，且不考虑兼容非 Spring 或早期 Spring 项目，使用 Spring Boot 提供的自动属性绑定机制（如 `@ConfigurationProperties`）就足够了，无需额外处理。

但考虑到我们希望框架具有更强的通用性和扩展性，因此采用了两个"小技巧"：

* 1.  
**手动绑定配置属性** ：不使用 SpringBoot 默认的自动绑定方式，而是通过 `Binder.bind(...)` 手动加载配置，显式控制绑定过程，并确保 `BootstrapConfigProperties` 实例在绑定完成后即为完整对象；  
* 2.  
  **维护内部单例** ：在 `BootstrapConfigProperties` 内部维护一个静态单例引用，Bean 创建并赋值后，即可通过静态方法全局访问该配置。

通过这种方式，即使在不依赖 Spring 容器的 `core` 包中，也能读取远程配置中心（如 Nacos）下发的线程池参数，实现配置的全局可用性与模块解耦。
> 这里需要补充一点说明：从架构设计角度来看，这种做法其实存在一定的**职责不清晰** 问题。按照理想的模块边界划分，`core` 包应当保持纯净，专注于非 Spring 依赖的通用逻辑，不应该直接感知或依赖 Spring 环境。不过，为了降低理解成本、提升使用便利性，我们在这里做了一定程度的**耦合处理** ，通过内部单例让 `core` 也能访问到配置中心下发的参数。
>
> 当然，这种耦合是可以避免的。例如，如果某些核心模块（如线程池告警）需要依赖配置项（如通知接收人、WebHook 地址），完全可以通过 **参数传递** 的方式将其注入进来。也就是说，由 Starter 作为入口，将相关配置作为方法参数传递给 `core` 层，既实现了功能，又保持了模块的独立性与解耦。

### 3. 远程参数替换 {#3}

这里我们先通过一张时序图，帮助大家快速建立整体流程的认知。有了全局视角之后，再去调试具体的代码逻辑，会更加清晰、事半功倍。

![iShot_2025-06-26_20.53.58.png](https://article-images.zsxq.com/FqPkMHh5S7O2G5wHepimLRHX0h2C "iShot_2025-06-26_20.53.58.png")

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池后置处理器，扫描 Bean 是否为动态线程池，如果是的话进行属性填充和注册
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Slf4j
    @RequiredArgsConstructor
    public class OneThreadBeanPostProcessor implements BeanPostProcessor {
    ​
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
            if (bean instanceof OneThreadExecutor) {
                DynamicThreadPool dynamicThreadPool;
                try {
                    // 通过 IOC 容器扫描 Bean 是否存在动态线程池注解
                    dynamicThreadPool = ApplicationContextHolder.findAnnotationOnBean(beanName, DynamicThreadPool.class);
                    if (Objects.isNull(dynamicThreadPool)) {
                        return bean;
                    }
                } catch (Exception ex) {
                    log.error("Failed to create dynamic thread pool in annotation mode.", ex);
                    return bean;
                }
    ​
                OneThreadExecutor oneThreadExecutor = (OneThreadExecutor) bean;
                // 从配置中心读取动态线程池配置并对线程池进行赋值
                ThreadPoolExecutorProperties executorProperties = properties.getExecutors()
                        .stream()
                        .filter(each -> Objects.equals(oneThreadExecutor.getThreadPoolId(), each.getThreadPoolId()))
                        .findFirst()
                        .orElseThrow(() -> new RuntimeException("The thread pool id does not exist in the configuration."));
    ​
                overrideLocalThreadPoolConfig(executorProperties, oneThreadExecutor);
    ​
                // 注册到动态线程池注册器，后续监控和报警从注册器获取线程池实例。同时，参数动态变更需要依赖 ThreadPoolExecutorProperties 比对是否有边跟
                OneThreadRegistry.putHolder(oneThreadExecutor.getThreadPoolId(), oneThreadExecutor, executorProperties);
            }
    ​
            return bean;
        }
    ​
        private void overrideLocalThreadPoolConfig(ThreadPoolExecutorProperties executorProperties, OneThreadExecutor oneThreadExecutor) {
            Integer remoteCorePoolSize = executorProperties.getCorePoolSize();
            Integer remoteMaximumPoolSize = executorProperties.getMaximumPoolSize();
            Assert.isTrue(remoteCorePoolSize <= remoteMaximumPoolSize, "remoteCorePoolSize must be smaller than remoteMaximumPoolSize.");
    ​
            // 如果不清楚为什么有这段逻辑，可以参考 Hippo4j Issue https://github.com/opengoofy/hippo4j/issues/1063
            int originalMaximumPoolSize = oneThreadExecutor.getMaximumPoolSize();
            if (remoteCorePoolSize > originalMaximumPoolSize) {
                oneThreadExecutor.setMaximumPoolSize(remoteMaximumPoolSize);
                oneThreadExecutor.setCorePoolSize(remoteCorePoolSize);
            } else {
                oneThreadExecutor.setCorePoolSize(remoteCorePoolSize);
                oneThreadExecutor.setMaximumPoolSize(remoteMaximumPoolSize);
            }
    ​
            // 阻塞队列没有常规 set 方法，所以使用反射赋值
            BlockingQueue workQueue = BlockingQueueTypeEnum.createBlockingQueue(executorProperties.getWorkQueue(), executorProperties.getQueueCapacity());
            // Java 9+ 的模块系统（JPMS）默认禁止通过反射访问 JDK 内部 API 的私有字段，所以需要配置开放反射权限
            // 在启动命令中增加以下参数，显式开放 java.util.concurrent 包
            // IDE 中通过在 VM options 中添加参数：--add-opens=java.base/java.util.concurrent=ALL-UNNAMED
            // 部署的时候，在启动脚本（如 java -jar 命令）中加入该参数：java -jar --add-opens=java.base/java.util.concurrent=ALL-UNNAMED your-app.jar
            ReflectUtil.setFieldValue(oneThreadExecutor, "workQueue", workQueue);
    ​
            // 赋值动态线程池其他核心参数
            oneThreadExecutor.setKeepAliveTime(executorProperties.getKeepAliveTime(), TimeUnit.SECONDS);
            oneThreadExecutor.allowCoreThreadTimeOut(executorProperties.getAllowCoreThreadTimeOut());
            oneThreadExecutor.setRejectedExecutionHandler(RejectedPolicyTypeEnum.createPolicy(executorProperties.getRejectedHandler()));
        }
    }
              
这里可能有同学会担心：**通过反射替换`workQueue`是否存在风险？比如队列中是否可能已经有未完成的任务？**

实际上这种情况是不存在的。因为此时线程池仍处于 **Bean创建阶段** ，尚未对外提供服务，也就不会有任何任务提交进来。因此，替换 `workQueue` 是安全且可控的。

开发 SpringBoot Starter {#spring-boot-starter}
--------------------------------------------

前面我们已经讲解了动态线程池的**标识方式** 、**远程配置的读取逻辑** 以及**核心参数的覆盖机制** ，也就是说实现层的代码基本已经完成。

但有一个关键问题大家需要思考：**代码写好了，如何才能让它在项目启动时自动生效？**

这就要回到我们在上一章节提到的 **SpringBootStarter** 机制，通过自动装配的方式，让相关逻辑在启动时被正确加载和执行。

我们可以把开发 Starter 比作"把大象装进冰箱"，只需要三步：

* 1.  
编写核心业务逻辑代码（比如远程配置读取、后置处理器等）；  
* 2.  
编写配置类，将这些逻辑注册为 Spring Bean；  
* 3.  
  通过自动装配机制，将配置类集成进应用启动流程中。

前两步我们已经介绍得差不多了，接下来我们重点演示**后两步** ：如何进行配置类注册与自动装配。

### 1. Spring 装配类 {#1-spring}

之所以将 `OneThreadBaseConfiguration` 放在 `spring-base` 模块，而将 `CommonAutoConfiguration` 放在 `common-spring-boot-starter` 模块，是因为我们在最初设计时就考虑到了要兼容 **普通Spring项目** （非 Spring Boot）。因此，基础配置类与自动装配类进行了合理拆分，分别放置于不同模块中，便于按需引入与复用。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池基础 Spring 配置类
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Configuration
    public class OneThreadBaseConfiguration {
    ​
        @Bean
        public ApplicationContextHolder applicationContextHolder() {
            return new ApplicationContextHolder();
        }
    ​
        @Bean
        @DependsOn("applicationContextHolder")
        public OneThreadBeanPostProcessor oneThreadBeanPostProcessor(BootstrapConfigProperties properties) {
            return new OneThreadBeanPostProcessor(properties);
        }
      
        // ......
    }
    ​
    /**
     * 基于配置中心的公共自动装配配置
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-28
     */
    @Import(OneThreadBaseConfiguration.class)
    @AutoConfigureAfter(OneThreadBaseConfiguration.class)
    public class CommonAutoConfiguration {
    ​
        @Bean
        public BootstrapConfigProperties bootstrapConfigProperties(Environment environment) {
            BootstrapConfigProperties bootstrapConfigProperties = Binder.get(environment)
                    .bind(BootstrapConfigProperties.PREFIX, Bindable.of(BootstrapConfigProperties.class))
                    .get();
            BootstrapConfigProperties.setInstance(bootstrapConfigProperties);
            return bootstrapConfigProperties;
        }
        // ......
    }
              
上述代码中包含三个关键细节，值得特别关注：

*  
**`@DependsOn`** ：由于 `OneThreadBeanPostProcessor` 依赖其他 Bean（如 `ApplicationContextHolder`），而 Spring 在初始化 Bean 时默认不保证顺序，因此通过 `@DependsOn` 显式声明依赖关系，以确保所需 Bean 已就绪，避免初始化异常。  
*  
**`@Import`** ：用于在一个自动配置类中引入另一个配置类，从而让其一并生效。这是模块化 Starter 中常用的装配手段。  
*  
  **`@AutoConfigureAfter`** ：指定当前自动配置类应在某个配置类之后加载。由于 `OneThreadBaseConfiguration` 属于基础配置，因此需要确保它优先于其他自动配置类被加载。

### 2. 自动装配 {#2}

通过上面的代码，我们已经明确了实际需要自动装配的配置类只有一个。接下来要做的，就是**让SpringBoot能够发现并加载它** 。

在 **SpringBoot3.x** 中，官方引入了全新的自动装配机制：需要在 `META-INF/spring/` 目录下，按类型维护对应的 **imports文件** 。对于自动配置类，需要创建如下文件：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    ​
    # 以下为文件具体内容
    com.nageoffer.onethread.config.common.starter.configuration.CommonAutoConfiguration
              
至此，我们的第一个 Spring Boot Starter ------ `common-spring-boot-starter` 已经完成。

有同学可能会疑问：**它本身还不包含动态线程池刷新的能力，那为什么还要单独定义这个Starter呢** ？

原因在于，我们后续将支持多种配置中心（如 Nacos、Apollo 等），而**动态配置刷新、通知告警等逻辑在各配置中心中是高度通用的** 。如果没有这一公共模块，相关逻辑就必须在每个配置中心的实现中重复编写，既增加了维护成本，也破坏了代码的可复用性。

通过抽象出 `common-spring-boot-starter`，我们将这部分通用能力统一封装，极大提升了整体的模块化与扩展性。

关于启用动态线程池标识
-----------

最早在研究 Zuul Starter 时，我接触到了"可插拔"这个概念。简单来说，可插拔指的是：**即使引入了某个StarterJar包，其功能是否生效仍由特定条件决定** 。

换句话说，只有当满足某些前置条件时，相关的自动配置类才会被加载；如果条件不满足，该模块就会被"晾在一边"。这种机制的本质就是**模块插件化** ，可以有效**降低耦合、提升灵活性** 。

### 1. 定义注解 {#1}

我们可以定义一个具有"中间件范式"的注解，通常以 `Enable` 开头，例如 `@EnableOneThread`，一眼就能看出它是用来开启某个特性或模块的开关。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态启用 oneThread 动态线程池开关注解
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import(MarkerConfiguration.class)
    public @interface EnableOneThread {
    }
              
当在项目中使用该注解时，实际上会触发内部的 `@Import` 机制，进而加载并执行 `MarkerConfiguration` 中的配置逻辑，从而启用动态线程池相关功能。

这类可插拔注解通常**加在应用的启动类上** ，用来显式开启某个模块功能：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @EnableOneThread
    @SpringBootApplication
    public class NacosCloudExampleApplication {
    ​
        public static void main(String[] args) {
            SpringApplication.run(NacosCloudExampleApplication.class, args);
        }
    }
              
### 2. 定义配置类 {#2}

可插拔机制的核心在于"按需加载"，而其实现方式是多种多样的，比如：通过配置文件中的开关（如指定前缀的 Key）、或自定义注解控制模块启用。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Configuration
    public class MarkerConfiguration {
    ​
        @Bean
        public Marker dynamicThreadPoolMarkerBean() {
            return new Marker();
        }
    ​
        /**
         * 标记类
         * 可用于条件装配（@ConditionalOnBean 等）中作为存在性的判断依据
         * <p>
         * 作者：马丁
         * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
         * 开发时间：2025-04-23
         */
        public class Marker {
    ​
        }
    }
              
具体来说，当项目中使用了 `@EnableOneThread` 注解，就会通过 `@Import` 注册一个标记类 `Marker`。 **有这个标记Bean，Starter中的动态线程池逻辑才会被加载；反之，则不会生效** ，实现真正意义上的"按需启用"。

### 3. 可插拔配置 {#3}

在我们的设计中，**这两种方式都支持** 。但无论是哪种方式，本质上都离不开 Spring Boot 提供的条件装配注解（如 `@ConditionalOnBean`、`@ConditionalOnProperty` 等）作为判断依据。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @ConditionalOnBean(MarkerConfiguration.Marker.class)
    @Import(OneThreadBaseConfiguration.class)
    @AutoConfigureAfter(OneThreadBaseConfiguration.class)
    public class CommonAutoConfiguration {
        // ......
    }
              
### 4. 基于 Property 实现可插拔 {#4-property}

除了使用可插拔注解的方式外，我们还实现了**基于配置文件属性的可插拔机制** 。大家可以注意到，在配置文件中我们提供了一个控制开关：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      enable: true  # 默认为 true，设置为 false 则不会加载动态线程池功能
              
为了实现这一功能，我们使用了 Spring Boot 提供的 `@ConditionalOnProperty` 注解，它的作用是：**根据配置文件中的某个属性值，动态判断是否加载指定的Bean或配置类** 。

自动装配类如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @ConditionalOnBean(MarkerConfiguration.Marker.class)
    @Import(OneThreadBaseConfiguration.class)
    @AutoConfigureAfter(OneThreadBaseConfiguration.class)
    @ConditionalOnProperty(prefix = BootstrapConfigProperties.PREFIX, value = "enable", matchIfMissing = true, havingValue = "true")
    public class CommonAutoConfiguration {
        // ......
    }
              
这种方式非常适合用于 Starter 模块，能够让使用者通过简单的配置，**显式地启用或关闭某些功能模块** ，从而增强了灵活性和可控性。  

|        参数        |            作用说明             |
|------------------|-----------------------------|
| `prefix`         | 配置前缀，比如 `spring.datasource` |
| `name` / `value` | 属性名，例如 `enabled`            |
| `havingValue`    | 属性值必须等于这个值时才生效              |
| `matchIfMissing` | 当配置项缺失时是否认为条件成立，默认 false    |

文末总结
----

至此，我们已经完整了解了 Starter 的构建流程和可插拔机制的实现原理。

在下一章节，我们将继续深入，带大家实现**线程池的动态刷新能力** ，让线程池在运行时也能灵活变更配置。

完结，撒花 🎉  
自定义oneThread-SpringBoot-Starter基础组件，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



***内容摘要：本章节深入探讨了如何通过SpringBootStarter统一管理动态线程池，包括** 动态线程池标记* \*、\*\* 远程配置覆盖\*\*、\*\* Starter 开发**以及** 可插拔设计\*\*等功能。

课程目录如下所示：

*  
如何发现动态线程池？  
*  
Spring 后置处理器  
*  
开发 SpringBoot Starter  
*  
关于启用动态线程池标识  
*  
  文末总结

本章节将涉及到 `core`、`spring-base`、`starter/common-spring-boot-starter`、`nacos-cloud-example` 四个模块。

![iShot_2025-06-26_20.53.59.png](https://article-images.zsxq.com/FnGsvdmA6cTDjXJhsUxryzhs5XPB "iShot_2025-06-26_20.53.59.png")

> 这篇文章涉及多个模块之间的协作，**不像单点功能那样聚焦明确**，对于没接触过复杂多模块系统的同学，可能阅读起来会有一点"晕"。
>
> 不过没关系，文中我已经尽可能**梳理了关键流程和细节** ，帮助大家理解模块之间的关系。如果在阅读或实践过程中还有不清楚的地方，**也欢迎随时留言提问，我们一起讨论～**

如何发现动态线程池？
----------

上一章节我们已经了解了 SpringBoot Starter 的基本概念，本章节将具体介绍如何借助 SpringBoot Starter 将线程池注册到统一的线程池容器 `OneThreadRegistry` 中。

在开始之前，先提出一个问题：如何统一管理动态线程池？我想到的一个简单易行的方法是，将每个线程池定义为一个 Spring 的 Bean，并通过自定义的注解标记为动态线程池，如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池注解
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Target({ElementType.METHOD, ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    public @interface DynamicThreadPool {
    }
              
参考动态线程池创建的示例代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Bean
    @DynamicThreadPool
    public ThreadPoolExecutor onethreadProducer() {
        return ThreadPoolExecutorBuilder.builder()
                .threadPoolId("onethread-producer")
                .corePoolSize(2)
                .maximumPoolSize(4)
                .keepAliveTime(9999L)
                .awaitTerminationMillis(5000L)
                .workQueueType(BlockingQueueTypeEnum.SYNCHRONOUS_QUEUE)
                .threadFactory("onethread-producer_")
                .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                .dynamicPool()
                .build();
    }
              
到这里，可能有同学会疑惑，仅仅标记 `@Bean` 和 `@DynamicThreadPool` 就可以把动态线程池注册到统一的容器里吗？答案显然是否定的。

上述代码只是对动态线程池的标记，要想真正将它们加入统一管理的容器，还需要借助 Spring 提供的后置处理器 `BeanPostProcessor`。

Spring 后置处理器 {#spring}
----------------------

### 1. 逻辑概述 {#1}

后置处理器除了将动态线程池注册到统一容器 `OneThreadRegistry` 外，还承担另一个重要功能：从配置中心读取远程线程池配置并覆盖本地配置。

通俗地讲，就是尽管你本地定义了线程池的配置参数，但这些参数可能并不会被使用，而是在项目启动时，自动从远程配置中心（如 Nacos）拉取最新的线程池参数并生效。

配置示例如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      nacos:
        data-id: onethread-nacos-cloud-example-ding-ma.yaml
        group: DEFAULT_GROUP
      config-file-type: YAML
      web:
        core-pool-size: 17
        maximum-pool-size: 26
        keep-alive-time: 60
        notify:
          receives: xxx
      notify-platforms:
        platform: DING
        url: https://oapi.dingtalk.com/robot/send?access_token=xxx
      executors:
      - thread-pool-id: onethread-producer
        core-pool-size: 14
        maximum-pool-size: 22
        queue-capacity: 1999
        work-queue: ResizableCapacityLinkedBlockingQueue
        rejected-handler: DiscardOldestPolicy
        keep-alive-time: 160
        allow-core-thread-time-out: false
        notify:
          receives: xxx
          interval: 10
        alarm:
          enable: false
          queue-threshold: 90
          active-threshold: 90
      - thread-pool-id: onethread-consumer
        core-pool-size: 10
        maximum-pool-size: 20
        queue-capacity: 1024
        work-queue: LinkedBlockingQueue
        rejected-handler: AbortPolicy
        keep-alive-time: 9999
        allow-core-thread-time-out: true
        notify:
          receives: xxx
          interval: 5
        alarm:
          enable: false
          queue-threshold: 80
          active-threshold: 80
              
远端配置只覆盖配置中心中定义的参数，其他如线程工厂的定义则不会被覆盖。

### 2. 远端配置读取 {#2}

远程配置读取逻辑如下，以 Nacos 示例程序为例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    server:
      port: 18080  # 应用服务端口，启动后访问地址为 http://localhost:18080
    ​
    spring:
      application:
        name: nacos-cloud-example${unique-name:}  # Spring 应用名称，支持通过 unique-name 参数自定义服务名，方便多实例区分
      config:
        import: nacos:onethread-nacos-cloud-example${unique-name:}.yaml  # 从 Nacos 导入指定配置文件
      profiles:
        active: dev  # 激活的配置环境（开发环境）
    ​
      cloud:
        nacos:
          config:
            username: nacos  # 连接 Nacos 的用户名
            password: nacos  # 连接 Nacos 的密码
            file-extension: yaml  # 指定配置文件的后缀类型为 YAML
            extension-configs:
              - data-id: onethread-nacos-cloud-example${unique-name:}.yaml  # 指定扩展配置文件的 dataId
                group: DEFAULT_GROUP  # 配置文件所在的 Nacos 分组
                refresh: true  # 是否开启自动刷新，当 Nacos 配置变更时自动更新本地配置
          server-addr: 127.0.0.1:8848  # Nacos 服务器地址，默认端口为 8848
    ​
              
在这段配置中，有两个关键点需要特别关注：

*  
**`-data-id`** ：这是我们在 Nacos 中创建的配置文件的唯一标识，应用会根据它来读取对应的远程配置内容。一个项目可以配置多个 `data-id`，实现灵活的模块化配置。  
*  
  **`spring.config.import`** ：这里尤其值得注意。在 SpringBoot2 中，远程配置默认会自动合并到本地配置中；但从 Spring Boot 3 开始，**如果不通过`spring.config.import`明确指定远程配置文件的来源，SpringBoot将不会加载这些配置** 。因此，如果省略了该项，下面提到的"远程配置覆盖本地配置"的功能将无法生效。

此外，配置中心中存储的参数本质上是**字符串形式** 的键值对，直接使用时不够直观也不便于管理。在 Java 应用中，我们通常会将其**绑定为配置类的属性对象** ，这样更便于类型转换、代码提示和后续维护。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * oneThread 配置中心参数
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Data
    public class BootstrapConfigProperties {
    ​
        public static final String PREFIX = "onethread";
    ​
        /**
         * 是否开启动态线程池开关
         */
        private Boolean enable = Boolean.TRUE;
    ​
        /**
         * Nacos 配置文件
         */
        private NacosConfig nacos;
    ​
        /**
         * Apollo 配置文件
         */
        private ApolloConfig apollo;
    ​
        /**
         * Web 线程池配置
         */
        private WebThreadPoolExecutorConfig web;
    ​
        /**
         * Nacos 远程配置文件格式类型
         */
        private ConfigFileTypeEnum configFileType;
    ​
        /**
         * 通知配置
         */
        private NotifyPlatformsConfig notifyPlatforms;
    ​
        /**
         * 监控配置
         */
        private MonitorConfig monitorConfig = new MonitorConfig();
    ​
        /**
         * 线程池配置集合
         */
        private List<ThreadPoolExecutorProperties> executors;
    ​
        @Data
        public static class NotifyPlatformsConfig {
    ​
            /**
             * 通知类型，比如：DING
             */
            private String platform;
    ​
            /**
             * 完整 WebHook 地址
             */
            private String url;
        }
    ​
        @Data
        public static class MonitorConfig {
    ​
            /**
             * 默认开启监控配置
             */
            private Boolean enable = Boolean.TRUE;
    ​
            /**
             * 监控类型
             */
            private String collectType = "micrometer";
    ​
            /**
             * 采集间隔，默认 10 秒
             */
            private Long collectInterval = 10L;
        }
    ​
        @Data
        public static class NacosConfig {
    ​
            private String dataId;
    ​
            private String group;
        }
    ​
        @Data
        public static class ApolloConfig {
    ​
            private String namespace;
        }
    ​
        @Data
        public static class WebThreadPoolExecutorConfig {
    ​
            /**
             * 核心线程数
             */
            private Integer corePoolSize;
    ​
            /**
             * 最大线程数
             */
            private Integer maximumPoolSize;
    ​
            /**
             * 线程空闲存活时间（单位：秒）
             */
            private Long keepAliveTime;
    ​
            /**
             * 通知配置
             */
            private NotifyConfig notify;
        }
    ​
        @Data
        @NoArgsConstructor
        @AllArgsConstructor
        public static class NotifyConfig {
    ​
            /**
             * 接收人集合
             */
            private String receives;
        }
    ​
        private static BootstrapConfigProperties INSTANCE = new BootstrapConfigProperties();
    ​
        public static BootstrapConfigProperties getInstance() {
            return INSTANCE;
        }
    ​
        public static void setInstance(BootstrapConfigProperties properties) {
            INSTANCE = properties;
        }
    }
              
这里还涉及一层设计上的逻辑：**`core`包本身无法直接获取Spring容器中的Bean** ，但我们又希望在 `core` 包中的组件（如线程池监控、告警模块）能够使用远程配置中心下发的线程池参数。

为了解决这个矛盾，在装配 `BootstrapConfigProperties` Bean 时，我们做了一些处理，使用了一个 **"小技巧"** ：在 Bean 创建完成后，**将其实例手动赋值给类中的静态单例变量** ，从而实现全局共享。

这样，即使在非 Spring 环境下的模块中（比如 `core` 包），也可以通过 `BootstrapConfigProperties.getInstance()` 的方式获取到线程池的配置参数。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class CommonAutoConfiguration {
    ​
        @Bean
        public BootstrapConfigProperties bootstrapConfigProperties(Environment environment) {
            BootstrapConfigProperties bootstrapConfigProperties = Binder.get(environment)
                    .bind(BootstrapConfigProperties.PREFIX, Bindable.of(BootstrapConfigProperties.class))
                    .get();
            BootstrapConfigProperties.setInstance(bootstrapConfigProperties);
            return bootstrapConfigProperties;
        }
    }
              
通常情况下，我们只需在 `BootstrapConfigProperties` 类上添加 `@ConfigurationProperties(prefix = "onethread")` 注解，Spring Boot 就会自动完成属性的绑定，无需如此复杂的处理逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @ConfigurationProperties(prefix = "onethread")
    public class BootstrapConfigProperties {
      
        // ......
    }
    ​
    public class CommonAutoConfiguration {
    ​
        @Bean
        public BootstrapConfigProperties bootstrapConfigProperties() {
            return new BootstrapConfigProperties();
        }
    }
              
如果是一个**常规的SpringBootStarter项目** ，且不考虑兼容非 Spring 或早期 Spring 项目，使用 Spring Boot 提供的自动属性绑定机制（如 `@ConfigurationProperties`）就足够了，无需额外处理。

但考虑到我们希望框架具有更强的通用性和扩展性，因此采用了两个"小技巧"：

* 1.  
**手动绑定配置属性** ：不使用 SpringBoot 默认的自动绑定方式，而是通过 `Binder.bind(...)` 手动加载配置，显式控制绑定过程，并确保 `BootstrapConfigProperties` 实例在绑定完成后即为完整对象；  
* 2.  
  **维护内部单例** ：在 `BootstrapConfigProperties` 内部维护一个静态单例引用，Bean 创建并赋值后，即可通过静态方法全局访问该配置。

通过这种方式，即使在不依赖 Spring 容器的 `core` 包中，也能读取远程配置中心（如 Nacos）下发的线程池参数，实现配置的全局可用性与模块解耦。
> 这里需要补充一点说明：从架构设计角度来看，这种做法其实存在一定的**职责不清晰** 问题。按照理想的模块边界划分，`core` 包应当保持纯净，专注于非 Spring 依赖的通用逻辑，不应该直接感知或依赖 Spring 环境。不过，为了降低理解成本、提升使用便利性，我们在这里做了一定程度的**耦合处理** ，通过内部单例让 `core` 也能访问到配置中心下发的参数。
>
> 当然，这种耦合是可以避免的。例如，如果某些核心模块（如线程池告警）需要依赖配置项（如通知接收人、WebHook 地址），完全可以通过 **参数传递** 的方式将其注入进来。也就是说，由 Starter 作为入口，将相关配置作为方法参数传递给 `core` 层，既实现了功能，又保持了模块的独立性与解耦。

### 3. 远程参数替换 {#3}

这里我们先通过一张时序图，帮助大家快速建立整体流程的认知。有了全局视角之后，再去调试具体的代码逻辑，会更加清晰、事半功倍。

![iShot_2025-06-26_20.53.58.png](https://article-images.zsxq.com/FqPkMHh5S7O2G5wHepimLRHX0h2C "iShot_2025-06-26_20.53.58.png")

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池后置处理器，扫描 Bean 是否为动态线程池，如果是的话进行属性填充和注册
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Slf4j
    @RequiredArgsConstructor
    public class OneThreadBeanPostProcessor implements BeanPostProcessor {
    ​
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
            if (bean instanceof OneThreadExecutor) {
                DynamicThreadPool dynamicThreadPool;
                try {
                    // 通过 IOC 容器扫描 Bean 是否存在动态线程池注解
                    dynamicThreadPool = ApplicationContextHolder.findAnnotationOnBean(beanName, DynamicThreadPool.class);
                    if (Objects.isNull(dynamicThreadPool)) {
                        return bean;
                    }
                } catch (Exception ex) {
                    log.error("Failed to create dynamic thread pool in annotation mode.", ex);
                    return bean;
                }
    ​
                OneThreadExecutor oneThreadExecutor = (OneThreadExecutor) bean;
                // 从配置中心读取动态线程池配置并对线程池进行赋值
                ThreadPoolExecutorProperties executorProperties = properties.getExecutors()
                        .stream()
                        .filter(each -> Objects.equals(oneThreadExecutor.getThreadPoolId(), each.getThreadPoolId()))
                        .findFirst()
                        .orElseThrow(() -> new RuntimeException("The thread pool id does not exist in the configuration."));
    ​
                overrideLocalThreadPoolConfig(executorProperties, oneThreadExecutor);
    ​
                // 注册到动态线程池注册器，后续监控和报警从注册器获取线程池实例。同时，参数动态变更需要依赖 ThreadPoolExecutorProperties 比对是否有边跟
                OneThreadRegistry.putHolder(oneThreadExecutor.getThreadPoolId(), oneThreadExecutor, executorProperties);
            }
    ​
            return bean;
        }
    ​
        private void overrideLocalThreadPoolConfig(ThreadPoolExecutorProperties executorProperties, OneThreadExecutor oneThreadExecutor) {
            Integer remoteCorePoolSize = executorProperties.getCorePoolSize();
            Integer remoteMaximumPoolSize = executorProperties.getMaximumPoolSize();
            Assert.isTrue(remoteCorePoolSize <= remoteMaximumPoolSize, "remoteCorePoolSize must be smaller than remoteMaximumPoolSize.");
    ​
            // 如果不清楚为什么有这段逻辑，可以参考 Hippo4j Issue https://github.com/opengoofy/hippo4j/issues/1063
            int originalMaximumPoolSize = oneThreadExecutor.getMaximumPoolSize();
            if (remoteCorePoolSize > originalMaximumPoolSize) {
                oneThreadExecutor.setMaximumPoolSize(remoteMaximumPoolSize);
                oneThreadExecutor.setCorePoolSize(remoteCorePoolSize);
            } else {
                oneThreadExecutor.setCorePoolSize(remoteCorePoolSize);
                oneThreadExecutor.setMaximumPoolSize(remoteMaximumPoolSize);
            }
    ​
            // 阻塞队列没有常规 set 方法，所以使用反射赋值
            BlockingQueue workQueue = BlockingQueueTypeEnum.createBlockingQueue(executorProperties.getWorkQueue(), executorProperties.getQueueCapacity());
            // Java 9+ 的模块系统（JPMS）默认禁止通过反射访问 JDK 内部 API 的私有字段，所以需要配置开放反射权限
            // 在启动命令中增加以下参数，显式开放 java.util.concurrent 包
            // IDE 中通过在 VM options 中添加参数：--add-opens=java.base/java.util.concurrent=ALL-UNNAMED
            // 部署的时候，在启动脚本（如 java -jar 命令）中加入该参数：java -jar --add-opens=java.base/java.util.concurrent=ALL-UNNAMED your-app.jar
            ReflectUtil.setFieldValue(oneThreadExecutor, "workQueue", workQueue);
    ​
            // 赋值动态线程池其他核心参数
            oneThreadExecutor.setKeepAliveTime(executorProperties.getKeepAliveTime(), TimeUnit.SECONDS);
            oneThreadExecutor.allowCoreThreadTimeOut(executorProperties.getAllowCoreThreadTimeOut());
            oneThreadExecutor.setRejectedExecutionHandler(RejectedPolicyTypeEnum.createPolicy(executorProperties.getRejectedHandler()));
        }
    }
              
这里可能有同学会担心：**通过反射替换`workQueue`是否存在风险？比如队列中是否可能已经有未完成的任务？**

实际上这种情况是不存在的。因为此时线程池仍处于 **Bean创建阶段** ，尚未对外提供服务，也就不会有任何任务提交进来。因此，替换 `workQueue` 是安全且可控的。

开发 SpringBoot Starter {#spring-boot-starter}
--------------------------------------------

前面我们已经讲解了动态线程池的**标识方式** 、**远程配置的读取逻辑** 以及**核心参数的覆盖机制** ，也就是说实现层的代码基本已经完成。

但有一个关键问题大家需要思考：**代码写好了，如何才能让它在项目启动时自动生效？**

这就要回到我们在上一章节提到的 **SpringBootStarter** 机制，通过自动装配的方式，让相关逻辑在启动时被正确加载和执行。

我们可以把开发 Starter 比作"把大象装进冰箱"，只需要三步：

* 1.  
编写核心业务逻辑代码（比如远程配置读取、后置处理器等）；  
* 2.  
编写配置类，将这些逻辑注册为 Spring Bean；  
* 3.  
  通过自动装配机制，将配置类集成进应用启动流程中。

前两步我们已经介绍得差不多了，接下来我们重点演示**后两步** ：如何进行配置类注册与自动装配。

### 1. Spring 装配类 {#1-spring}

之所以将 `OneThreadBaseConfiguration` 放在 `spring-base` 模块，而将 `CommonAutoConfiguration` 放在 `common-spring-boot-starter` 模块，是因为我们在最初设计时就考虑到了要兼容 **普通Spring项目** （非 Spring Boot）。因此，基础配置类与自动装配类进行了合理拆分，分别放置于不同模块中，便于按需引入与复用。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池基础 Spring 配置类
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Configuration
    public class OneThreadBaseConfiguration {
    ​
        @Bean
        public ApplicationContextHolder applicationContextHolder() {
            return new ApplicationContextHolder();
        }
    ​
        @Bean
        @DependsOn("applicationContextHolder")
        public OneThreadBeanPostProcessor oneThreadBeanPostProcessor(BootstrapConfigProperties properties) {
            return new OneThreadBeanPostProcessor(properties);
        }
      
        // ......
    }
    ​
    /**
     * 基于配置中心的公共自动装配配置
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-28
     */
    @Import(OneThreadBaseConfiguration.class)
    @AutoConfigureAfter(OneThreadBaseConfiguration.class)
    public class CommonAutoConfiguration {
    ​
        @Bean
        public BootstrapConfigProperties bootstrapConfigProperties(Environment environment) {
            BootstrapConfigProperties bootstrapConfigProperties = Binder.get(environment)
                    .bind(BootstrapConfigProperties.PREFIX, Bindable.of(BootstrapConfigProperties.class))
                    .get();
            BootstrapConfigProperties.setInstance(bootstrapConfigProperties);
            return bootstrapConfigProperties;
        }
        // ......
    }
              
上述代码中包含三个关键细节，值得特别关注：

*  
**`@DependsOn`** ：由于 `OneThreadBeanPostProcessor` 依赖其他 Bean（如 `ApplicationContextHolder`），而 Spring 在初始化 Bean 时默认不保证顺序，因此通过 `@DependsOn` 显式声明依赖关系，以确保所需 Bean 已就绪，避免初始化异常。  
*  
**`@Import`** ：用于在一个自动配置类中引入另一个配置类，从而让其一并生效。这是模块化 Starter 中常用的装配手段。  
*  
  **`@AutoConfigureAfter`** ：指定当前自动配置类应在某个配置类之后加载。由于 `OneThreadBaseConfiguration` 属于基础配置，因此需要确保它优先于其他自动配置类被加载。

### 2. 自动装配 {#2}

通过上面的代码，我们已经明确了实际需要自动装配的配置类只有一个。接下来要做的，就是**让SpringBoot能够发现并加载它** 。

在 **SpringBoot3.x** 中，官方引入了全新的自动装配机制：需要在 `META-INF/spring/` 目录下，按类型维护对应的 **imports文件** 。对于自动配置类，需要创建如下文件：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
    ​
    # 以下为文件具体内容
    com.nageoffer.onethread.config.common.starter.configuration.CommonAutoConfiguration
              
至此，我们的第一个 Spring Boot Starter ------ `common-spring-boot-starter` 已经完成。

有同学可能会疑问：**它本身还不包含动态线程池刷新的能力，那为什么还要单独定义这个Starter呢** ？

原因在于，我们后续将支持多种配置中心（如 Nacos、Apollo 等），而**动态配置刷新、通知告警等逻辑在各配置中心中是高度通用的** 。如果没有这一公共模块，相关逻辑就必须在每个配置中心的实现中重复编写，既增加了维护成本，也破坏了代码的可复用性。

通过抽象出 `common-spring-boot-starter`，我们将这部分通用能力统一封装，极大提升了整体的模块化与扩展性。

关于启用动态线程池标识
-----------

最早在研究 Zuul Starter 时，我接触到了"可插拔"这个概念。简单来说，可插拔指的是：**即使引入了某个StarterJar包，其功能是否生效仍由特定条件决定** 。

换句话说，只有当满足某些前置条件时，相关的自动配置类才会被加载；如果条件不满足，该模块就会被"晾在一边"。这种机制的本质就是**模块插件化** ，可以有效**降低耦合、提升灵活性** 。

### 1. 定义注解 {#1}

我们可以定义一个具有"中间件范式"的注解，通常以 `Enable` 开头，例如 `@EnableOneThread`，一眼就能看出它是用来开启某个特性或模块的开关。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态启用 oneThread 动态线程池开关注解
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-23
     */
    @Target(ElementType.TYPE)
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import(MarkerConfiguration.class)
    public @interface EnableOneThread {
    }
              
当在项目中使用该注解时，实际上会触发内部的 `@Import` 机制，进而加载并执行 `MarkerConfiguration` 中的配置逻辑，从而启用动态线程池相关功能。

这类可插拔注解通常**加在应用的启动类上** ，用来显式开启某个模块功能：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @EnableOneThread
    @SpringBootApplication
    public class NacosCloudExampleApplication {
    ​
        public static void main(String[] args) {
            SpringApplication.run(NacosCloudExampleApplication.class, args);
        }
    }
              
### 2. 定义配置类 {#2}

可插拔机制的核心在于"按需加载"，而其实现方式是多种多样的，比如：通过配置文件中的开关（如指定前缀的 Key）、或自定义注解控制模块启用。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Configuration
    public class MarkerConfiguration {
    ​
        @Bean
        public Marker dynamicThreadPoolMarkerBean() {
            return new Marker();
        }
    ​
        /**
         * 标记类
         * 可用于条件装配（@ConditionalOnBean 等）中作为存在性的判断依据
         * <p>
         * 作者：马丁
         * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
         * 开发时间：2025-04-23
         */
        public class Marker {
    ​
        }
    }
              
具体来说，当项目中使用了 `@EnableOneThread` 注解，就会通过 `@Import` 注册一个标记类 `Marker`。 **有这个标记Bean，Starter中的动态线程池逻辑才会被加载；反之，则不会生效** ，实现真正意义上的"按需启用"。

### 3. 可插拔配置 {#3}

在我们的设计中，**这两种方式都支持** 。但无论是哪种方式，本质上都离不开 Spring Boot 提供的条件装配注解（如 `@ConditionalOnBean`、`@ConditionalOnProperty` 等）作为判断依据。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @ConditionalOnBean(MarkerConfiguration.Marker.class)
    @Import(OneThreadBaseConfiguration.class)
    @AutoConfigureAfter(OneThreadBaseConfiguration.class)
    public class CommonAutoConfiguration {
        // ......
    }
              
### 4. 基于 Property 实现可插拔 {#4-property}

除了使用可插拔注解的方式外，我们还实现了**基于配置文件属性的可插拔机制** 。大家可以注意到，在配置文件中我们提供了一个控制开关：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      enable: true  # 默认为 true，设置为 false 则不会加载动态线程池功能
              
为了实现这一功能，我们使用了 Spring Boot 提供的 `@ConditionalOnProperty` 注解，它的作用是：**根据配置文件中的某个属性值，动态判断是否加载指定的Bean或配置类** 。

自动装配类如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @ConditionalOnBean(MarkerConfiguration.Marker.class)
    @Import(OneThreadBaseConfiguration.class)
    @AutoConfigureAfter(OneThreadBaseConfiguration.class)
    @ConditionalOnProperty(prefix = BootstrapConfigProperties.PREFIX, value = "enable", matchIfMissing = true, havingValue = "true")
    public class CommonAutoConfiguration {
        // ......
    }
              
这种方式非常适合用于 Starter 模块，能够让使用者通过简单的配置，**显式地启用或关闭某些功能模块** ，从而增强了灵活性和可控性。  

|        参数        |            作用说明             |
|------------------|-----------------------------|
| `prefix`         | 配置前缀，比如 `spring.datasource` |
| `name` / `value` | 属性名，例如 `enabled`            |
| `havingValue`    | 属性值必须等于这个值时才生效              |
| `matchIfMissing` | 当配置项缺失时是否认为条件成立，默认 false    |

文末总结
----

至此，我们已经完整了解了 Starter 的构建流程和可插拔机制的实现原理。

在下一章节，我们将继续深入，带大家实现**线程池的动态刷新能力** ，让线程池在运行时也能灵活变更配置。

完结，撒花 🎉

