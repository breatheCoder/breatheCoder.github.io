2025年08月07日 22:45  
使用观察者模式重构配置动态刷新，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本文深入剖析了 oneThread 动态线程池框架中跨模块通信的设计挑战，通过对配置中心 Starter 与 Web Starter 解耦需求的分析，引出观察者模式在框架架构中的核心价值。文章从实际业务场景出发，详细解析了观察者模式的设计理念、实现机制以及在动态线程池配置刷新中的具体应用。

课程目录如下所示：

*  
前言  
*  
跨模块通信的设计挑战  
*  
为什么选择观察者模式？  
*  
什么是观察者模式？  
*  
oneThread 中的观察者模式实现  
*  
扩展性分析  
*  
Guava vs Spring 观察者模式  
*  
  文末总结

前言
---

在构建 oneThread 动态线程池框架的过程中，我们面临着一个经典的架构设计问题：如何在保持模块独立性的前提下，实现跨模块的高效通信？

特别是在配置中心 Starter 需要通知 Web Starter 进行线程池参数更新时，传统的直接依赖方式会带来模块耦合、包体积膨胀等问题。为了解决这一挑战，我们引入了**观察者模式**，构建了一套松耦合、高扩展的事件驱动架构。

本文将带你深入了解这一设计决策的来龙去脉，以及观察者模式在 oneThread 框架中的精妙应用。

跨模块通信的设计挑战
----------

### 1. 模块架构概览 {#1}

oneThread 框架采用分层模块化设计，主要包含以下核心模块：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    └── starter # 动态线程池配置中心组件包，实现线程池结合Spring框架和配置中心动态刷新
        ├── adapter # 动态线程池适配层，比如对接 Web 容器 Tomcat 线程池等
        │   └── web-spring-boot-starter # Web 容器线程池组件库
        ├── apollo-spring-boot-starter # Apollo 配置中心动态监控线程池组件库
        ├── common-spring-boot-starter # 配置中心公共监听等逻辑抽象组件库
        ├── dashboard-dev-spring-boot-starter # 控制台 API 组件库
        └── nacos-cloud-spring-boot-starter # Nacos 配置中心动态监控线程池组件库
              
和本文章相关的在 starter 模块下，子模块职责如下：

*  
**common-spring-boot-starter**：提供配置解析、事件定义等公共能力。  
*  
**nacos/apollo-spring-boot-starter**：监听配置中心变更，触发刷新逻辑。  
*  
  **web-spring-boot-starter**：管理 Web 容器线程池，响应配置变更。

如果采用直接依赖的方式解决跨模块通信，会面临以下问题：

*  
配置中心相关的 Starter 依赖 web-starter，如果用户仅需要动态线程池依赖，无法按需依赖。  
*  
  扩展性差，每增加一种线程池适配器（如 Dubbo、Hystrix 等），都需要修改配置中心 Starter 代码，增加新的依赖关系等。

代码耦合如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 配置中心 Starter 需要了解所有可能的线程池类型
    public class AbstractDynamicThreadPoolRefresher {
        
        @Autowired(required = false)
        private WebThreadPoolService webService;
        
        @Autowired(required = false) 
        private RocketMQThreadPoolService rocketMQService;
        
        @Autowired(required = false)
        private RabbitMQThreadPoolService rabbitMQService;
        
        public void refreshConfig(String config) {
            if (webService != null) {
                webService.update(config);
            }
            if (rocketMQService != null) {
                rocketMQService.update(config);
            }
            // ... 更多if判断
        }
    }
              
### 2. 理想的解决方案 {#2}

我们需要一种机制，能够完成以下需求：

*  
**解耦模块依赖**：配置中心 Starter 无需直接依赖具体的线程池管理模块。  
*  
**支持动态扩展**：新增线程池适配器时，无需修改现有代码。  
*  
**保持高内聚**：每个模块专注自己的核心职责。  
*  
  **简化测试**：模块间松耦合，便于单元测试和集成测试。

为什么选择观察者模式？
-----------

### 1. 业务场景分析 {#1}

在 oneThread 框架中，配置变更的处理流程如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    配置中心变更 → 配置解析 → 参数对比 → 线程池更新 → 变更通知
              
这个流程中存在典型的"一对多"通知场景：

*  
**一个事件源**：配置中心的配置变更。  
*  
  **多个观察者**：动态线程池管理器、Web 线程池管理器、RocketMQ 线程池管理器、自定义线程池管理器等。甚至加点想象力，是不是连接池动态变更也能做。

### 2. 观察者模式的优势 {#2}

#### 2.1 松耦合设计 {#2-1}

![iShot_2025-08-01_11.34.12.png](https://article-images.zsxq.com/FmoI2Ryd_GA2O_ltrZiR2MrBTkYE)

#### 2.2 动态扩展能力 {#2-2}

新增线程池适配器时，只需：

* 1.  
开发监听者，实现 `ApplicationListener<ThreadPoolConfigUpdateEvent>` 接口。  
* 2.  
  将监听者注册为 Spring Bean，无需修改任何现有代码。

这种方案设计下，职责分离足够清晰：

*  
**配置中心模块**：专注配置监听和解析。  
*  
**线程池管理模块**：不同组件专注自己职责内的线程池参数更新。  
*  
  **事件机制**：负责消息传递和路由。

什么是观察者模式？
---------

### 1. 观察者模式定义 {#1}

**观察者模式**（Observer Pattern）是一种行为设计模式，它定义了对象间的一对多依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知并自动更新。

### 2. 核心角色 {#2}

*  
**Subject（主题/被观察者）**：维护观察者列表，提供注册、移除观察者的方法。  
*  
**Observer（观察者）**：定义更新接口，当收到主题通知时执行相应操作。  
*  
**ConcreteSubject（具体主题）**：实现主题接口，状态改变时通知所有观察者。  
*  
  **ConcreteObserver（具体观察者）**：实现观察者接口，定义具体的更新逻辑。

### 3. 经典实现示例 {#3}

以新闻订阅为例，展示观察者模式的基本实现：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 观察者接口
    public interface NewsObserver {
        void update(String news);
    }
    ​
    // 主题接口  
    public interface NewsSubject {
        void addObserver(NewsObserver observer);
        void removeObserver(NewsObserver observer);
        void notifyObservers(String news);
    }
    ​
    // 具体主题：新闻发布者
    public class NewsPublisher implements NewsSubject {
        private List<NewsObserver> observers = new ArrayList<>();
        
        @Override
        public void addObserver(NewsObserver observer) {
            observers.add(observer);
        }
        
        @Override
        public void removeObserver(NewsObserver observer) {
            observers.remove(observer);
        }
        
        @Override
        public void notifyObservers(String news) {
            for (NewsObserver observer : observers) {
                observer.update(news);
            }
        }
        
        public void publishNews(String news) {
            System.out.println("发布新闻: " + news);
            notifyObservers(news);
        }
    }
    ​
    // 具体观察者：邮件订阅者
    public class EmailSubscriber implements NewsObserver {
        private String email;
        
        public EmailSubscriber(String email) {
            this.email = email;
        }
        
        @Override
        public void update(String news) {
            System.out.println("邮件通知 " + email + ": " + news);
        }
    }
    ​
    // 具体观察者：短信订阅者
    public class SMSSubscriber implements NewsObserver {
        private String phone;
        
        public SMSSubscriber(String phone) {
            this.phone = phone;
        }
        
        @Override
        public void update(String news) {
            System.out.println("短信通知 " + phone + ": " + news);
        }
    }
              
观察者模式时序图如下所示：

![image-20250807144451601.png](https://article-images.zsxq.com/Frgyaj-mQ-dr7XiYJPFPp8yoHYCh)

oneThread 中的观察者模式实现 {#one-thread}
---------------------------------

### 1. Spring 事件机制 {#1-spring}

oneThread 框架基于 Spring 的事件发布机制实现观察者模式，这种方式具有以下优势：

*  
**框架集成**：与 Spring 容器深度集成，无需手动管理观察者列表。  
*  
**异步支持**：支持同步和异步事件处理。  
*  
  **类型安全**：基于泛型的强类型事件定义。

![image-20250807140723647.png](https://article-images.zsxq.com/FpCwABaNwasah2CRBNqeKxm_yb1N)

### 2. 事件定义 {#2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池配置更新事件
     * 当配置中心检测到线程池参数变更时，发布此事件通知所有相关组件
     */
    public class ThreadPoolConfigUpdateEvent extends ApplicationEvent {
    ​
        @Getter
        @Setter
        private BootstrapConfigProperties bootstrapConfigProperties;
    ​
        public ThreadPoolConfigUpdateEvent(Object source, BootstrapConfigProperties bootstrapConfigProperties) {
            super(source);
            this.bootstrapConfigProperties = bootstrapConfigProperties;
        }
    }
              
事件类设计要点：

*  
继承 `ApplicationEvent`，符合 Spring 事件规范。  
*  
  携带配置信息 `BootstrapConfigProperties`，为观察者提供必要数据。

### 3. 事件发布者（被观察者） {#3}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public abstract class AbstractDynamicThreadPoolRefresher implements ApplicationRunner {
    ​
        protected final BootstrapConfigProperties properties;
    ​
        /**
         * 配置刷新的核心方法
         * 解析配置信息并发布更新事件
         */
        @SneakyThrows
        public void refreshThreadPoolProperties(String configInfo) {
            // 1. 解析配置信息
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance()
                .parseConfig(configInfo, properties.getConfigFileType());
            
            // 2. 绑定到配置对象
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
            BootstrapConfigProperties refresherProperties = binder
                .bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties))
                .get();
    ​
            // 3. 发布线程池配置变更事件（观察者模式的核心）
            ApplicationContextHolder.publishEvent(
                new ThreadPoolConfigUpdateEvent(this, refresherProperties)
            );
        }
    }
              
### 4. 事件观察者实现 {#4}

#### 4.1 通用线程池配置监听器 {#4-1}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池刷新监听器
     * 处理通用线程池（非Web容器）的配置更新
     */
    @Slf4j
    @RequiredArgsConstructor
    public class DynamicThreadPoolRefreshListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        private final NotifierDispatcher notifierDispatcher;
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            BootstrapConfigProperties refresherProperties = event.getBootstrapConfigProperties();
    ​
            // 检查远程配置文件是否包含线程池配置
            if (CollUtil.isEmpty(refresherProperties.getExecutors())) {
                return;
            }
    ​
            // 刷新动态线程池对象核心参数
            for (ThreadPoolExecutorProperties remoteProperties : refresherProperties.getExecutors()) {
                // .....
            });
        }
    }
              
#### 4.2 Web容器线程池监听器 {#4-2-web}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * Web容器线程池刷新监听器
     * 专门处理Web容器（Tomcat/Jetty/Undertow）线程池的配置更新
     */
    @RequiredArgsConstructor
    public class WebThreadPoolRefreshListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        private final WebThreadPoolService webThreadPoolService;
        private final NotifierDispatcher notifierDispatcher;
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            BootstrapConfigProperties.WebThreadPoolExecutorConfig webExecutorConfig = event.getBootstrapConfigProperties().getWeb();
            if (Objects.isNull(webExecutorConfig)) {
                return;
            }
    ​
            WebThreadPoolBaseMetrics basicMetrics = webThreadPoolService.getBasicMetrics();
            if (!Objects.equals(basicMetrics.getCorePoolSize(), webExecutorConfig.getCorePoolSize())
                    || !Objects.equals(basicMetrics.getMaximumPoolSize(), webExecutorConfig.getMaximumPoolSize())
                    || !Objects.equals(basicMetrics.getKeepAliveTime(), webExecutorConfig.getKeepAliveTime())) {
                // 变更 Web 线程池配置
                webThreadPoolService.updateThreadPool(BeanUtil.toBean(webExecutorConfig, WebThreadPoolConfig.class));
    ​
                // 发送 Web 线程池配置变更通知
                sendWebThreadPoolConfigChangeMessage(basicMetrics, webExecutorConfig);
            }
        }
    ​
        // ......
    }
              
扩展性分析
-----

### 1. 新增线程池适配器 {#1}

假设我们需要支持 RocketMQ 的线程池动态调整，只需要以下步骤：

#### 1.1 创建监听器 {#1-1}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * RocketMQ线程池刷新监听器
     */
    @Component
    @RequiredArgsConstructor
    public class RocketMQThreadPoolRefreshListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        private final RocketMQThreadPoolService rocketMQService;
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            BootstrapConfigProperties.RocketMQConfig rocketMQConfig = 
                event.getBootstrapConfigProperties().getRocketMQ();
    ​
            if (Objects.isNull(rocketMQConfig)) {
                return;
            }
    ​
            // 更新RocketMQ线程池配置
            rocketMQService.updateThreadPool(rocketMQConfig);
        }
    }
              
#### 1.2 配置类扩展 {#1-2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 在BootstrapConfigProperties中添加RocketMQ配置
    @Data
    @ConfigurationProperties(prefix = BootstrapConfigProperties.PREFIX)
    public class BootstrapConfigProperties {
    ​
        // 现有配置...
    ​
        /**
         * RocketMQ线程池配置
         */
        private RocketMQConfig rocketMQ;
    ​
        @Data
        public static class RocketMQConfig {
            private Integer consumerCorePoolSize = 20;
            private Integer consumerMaximumPoolSize = 200;
            private Integer producerCorePoolSize = 4;
            private Integer producerMaximumPoolSize = 8;
        }
    }
              
### 2. 自定义事件处理 {#2}

开发者还可以监听配置更新事件，实现自定义逻辑：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 自定义配置变更监听器
     * 可以实现配置变更日志记录、指标上报等功能
     */
    @Component
    @Slf4j
    public class CustomConfigChangeListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            // 记录配置变更日志
            logConfigChange(event);
    ​
            // 上报监控指标
            reportMetrics(event);
    ​
            // 发送告警通知
            sendAlert(event);
        }
    ​
        private void logConfigChange(ThreadPoolConfigUpdateEvent event) {
            log.info("线程池配置发生变更: {}", 
                JSON.toJSONString(event.getBootstrapConfigProperties()));
        }
    }
              
Guava vs Spring 观察者模式 {#guava-vs-spring}
----------------------------------------

Guava 的观察者模式网上资料很多了，而且咱们也没有用到，这里我就不赘述了。这里重点说明下两者的区别：  

|   维度    |           **Spring ApplicationEvent**            |     **Guava EventBus**      |
|---------|--------------------------------------------------|-----------------------------|
| 所属框架    | Spring 框架                                        | Guava 工具库                   |
| 事件订阅方式  | 实现 `ApplicationListener` 或 `@EventListener` 注解监听 | 使用 `@Subscribe` 注解的方法监听     |
| 注册监听器   | 自动注册（通过 Spring 容器扫描 Bean）                        | 需手动注册到 EventBus             |
| 同步/异步支持 | 默认同步，支持 `@Async` 异步                              | 默认同步，也支持 `AsyncEventBus` 异步 |
| 解耦程度    | 高，基于事件名称和类型，天然解耦                                 | 中，需显式注册对象                   |

整体来看，Spring 事件机制与 Guava EventBus 在实现观察者模式上并无本质差异，都适用于常规的业务解耦场景。不过，考虑到 Spring 提供了原生支持，且生态完善、集成便捷，**更推荐优先使用 Spring 的事件监听机制，无需额外引入 Guava 依赖**。

同时，如果在代码中直接 `ApplicationContext` 发布事件监听，还能通过 IDEA 自带图标看到对应的观察者集合，Guava EventBus 是没有这个展示的。

![image-20250807165443521.png](https://article-images.zsxq.com/Fu23vEYZ1jiUQkCEt5YNTSNfxWwG)

文末总结
----

通过引入观察者模式，oneThread 动态线程池框架成功解决了跨模块通信的设计挑战，实现了配置中心 Starter 与各种线程池管理模块之间的松耦合通信。

通过观察者模式和 Spring 事件结合，有以下优势：

* 1.  
**模块解耦**：配置中心模块无需直接依赖具体的线程池管理模块，避免了循环依赖和模块耦合问题。  
* 2.  
**扩展性强**：新增线程池适配器时，只需实现监听器接口并注册为Spring Bean，无需修改现有代码。  
* 3.  
**职责清晰**：每个模块专注自己的核心职责，配置监听、事件路由、参数更新各司其职。  
* 4.  
  **易于测试**：松耦合的设计使得单元测试和集成测试更加简单。

完结，撒花 🎉  
使用观察者模式重构配置动态刷新，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本文深入剖析了 oneThread 动态线程池框架中跨模块通信的设计挑战，通过对配置中心 Starter 与 Web Starter 解耦需求的分析，引出观察者模式在框架架构中的核心价值。文章从实际业务场景出发，详细解析了观察者模式的设计理念、实现机制以及在动态线程池配置刷新中的具体应用。

课程目录如下所示：

*  
前言  
*  
跨模块通信的设计挑战  
*  
为什么选择观察者模式？  
*  
什么是观察者模式？  
*  
oneThread 中的观察者模式实现  
*  
扩展性分析  
*  
Guava vs Spring 观察者模式  
*  
  文末总结

前言
---

在构建 oneThread 动态线程池框架的过程中，我们面临着一个经典的架构设计问题：如何在保持模块独立性的前提下，实现跨模块的高效通信？

特别是在配置中心 Starter 需要通知 Web Starter 进行线程池参数更新时，传统的直接依赖方式会带来模块耦合、包体积膨胀等问题。为了解决这一挑战，我们引入了**观察者模式**，构建了一套松耦合、高扩展的事件驱动架构。

本文将带你深入了解这一设计决策的来龙去脉，以及观察者模式在 oneThread 框架中的精妙应用。

跨模块通信的设计挑战
----------

### 1. 模块架构概览 {#1}

oneThread 框架采用分层模块化设计，主要包含以下核心模块：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    └── starter # 动态线程池配置中心组件包，实现线程池结合Spring框架和配置中心动态刷新
        ├── adapter # 动态线程池适配层，比如对接 Web 容器 Tomcat 线程池等
        │   └── web-spring-boot-starter # Web 容器线程池组件库
        ├── apollo-spring-boot-starter # Apollo 配置中心动态监控线程池组件库
        ├── common-spring-boot-starter # 配置中心公共监听等逻辑抽象组件库
        ├── dashboard-dev-spring-boot-starter # 控制台 API 组件库
        └── nacos-cloud-spring-boot-starter # Nacos 配置中心动态监控线程池组件库
              
和本文章相关的在 starter 模块下，子模块职责如下：

*  
**common-spring-boot-starter**：提供配置解析、事件定义等公共能力。  
*  
**nacos/apollo-spring-boot-starter**：监听配置中心变更，触发刷新逻辑。  
*  
  **web-spring-boot-starter**：管理 Web 容器线程池，响应配置变更。

如果采用直接依赖的方式解决跨模块通信，会面临以下问题：

*  
配置中心相关的 Starter 依赖 web-starter，如果用户仅需要动态线程池依赖，无法按需依赖。  
*  
  扩展性差，每增加一种线程池适配器（如 Dubbo、Hystrix 等），都需要修改配置中心 Starter 代码，增加新的依赖关系等。

代码耦合如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 配置中心 Starter 需要了解所有可能的线程池类型
    public class AbstractDynamicThreadPoolRefresher {
        
        @Autowired(required = false)
        private WebThreadPoolService webService;
        
        @Autowired(required = false) 
        private RocketMQThreadPoolService rocketMQService;
        
        @Autowired(required = false)
        private RabbitMQThreadPoolService rabbitMQService;
        
        public void refreshConfig(String config) {
            if (webService != null) {
                webService.update(config);
            }
            if (rocketMQService != null) {
                rocketMQService.update(config);
            }
            // ... 更多if判断
        }
    }
              
### 2. 理想的解决方案 {#2}

我们需要一种机制，能够完成以下需求：

*  
**解耦模块依赖**：配置中心 Starter 无需直接依赖具体的线程池管理模块。  
*  
**支持动态扩展**：新增线程池适配器时，无需修改现有代码。  
*  
**保持高内聚**：每个模块专注自己的核心职责。  
*  
  **简化测试**：模块间松耦合，便于单元测试和集成测试。

为什么选择观察者模式？
-----------

### 1. 业务场景分析 {#1}

在 oneThread 框架中，配置变更的处理流程如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    配置中心变更 → 配置解析 → 参数对比 → 线程池更新 → 变更通知
              
这个流程中存在典型的"一对多"通知场景：

*  
**一个事件源**：配置中心的配置变更。  
*  
  **多个观察者**：动态线程池管理器、Web 线程池管理器、RocketMQ 线程池管理器、自定义线程池管理器等。甚至加点想象力，是不是连接池动态变更也能做。

### 2. 观察者模式的优势 {#2}

#### 2.1 松耦合设计 {#2-1}

![iShot_2025-08-01_11.34.12.png](https://article-images.zsxq.com/FmoI2Ryd_GA2O_ltrZiR2MrBTkYE)

#### 2.2 动态扩展能力 {#2-2}

新增线程池适配器时，只需：

* 1.  
开发监听者，实现 `ApplicationListener<ThreadPoolConfigUpdateEvent>` 接口。  
* 2.  
  将监听者注册为 Spring Bean，无需修改任何现有代码。

这种方案设计下，职责分离足够清晰：

*  
**配置中心模块**：专注配置监听和解析。  
*  
**线程池管理模块**：不同组件专注自己职责内的线程池参数更新。  
*  
  **事件机制**：负责消息传递和路由。

什么是观察者模式？
---------

### 1. 观察者模式定义 {#1}

**观察者模式**（Observer Pattern）是一种行为设计模式，它定义了对象间的一对多依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知并自动更新。

### 2. 核心角色 {#2}

*  
**Subject（主题/被观察者）**：维护观察者列表，提供注册、移除观察者的方法。  
*  
**Observer（观察者）**：定义更新接口，当收到主题通知时执行相应操作。  
*  
**ConcreteSubject（具体主题）**：实现主题接口，状态改变时通知所有观察者。  
*  
  **ConcreteObserver（具体观察者）**：实现观察者接口，定义具体的更新逻辑。

### 3. 经典实现示例 {#3}

以新闻订阅为例，展示观察者模式的基本实现：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 观察者接口
    public interface NewsObserver {
        void update(String news);
    }
    ​
    // 主题接口  
    public interface NewsSubject {
        void addObserver(NewsObserver observer);
        void removeObserver(NewsObserver observer);
        void notifyObservers(String news);
    }
    ​
    // 具体主题：新闻发布者
    public class NewsPublisher implements NewsSubject {
        private List<NewsObserver> observers = new ArrayList<>();
        
        @Override
        public void addObserver(NewsObserver observer) {
            observers.add(observer);
        }
        
        @Override
        public void removeObserver(NewsObserver observer) {
            observers.remove(observer);
        }
        
        @Override
        public void notifyObservers(String news) {
            for (NewsObserver observer : observers) {
                observer.update(news);
            }
        }
        
        public void publishNews(String news) {
            System.out.println("发布新闻: " + news);
            notifyObservers(news);
        }
    }
    ​
    // 具体观察者：邮件订阅者
    public class EmailSubscriber implements NewsObserver {
        private String email;
        
        public EmailSubscriber(String email) {
            this.email = email;
        }
        
        @Override
        public void update(String news) {
            System.out.println("邮件通知 " + email + ": " + news);
        }
    }
    ​
    // 具体观察者：短信订阅者
    public class SMSSubscriber implements NewsObserver {
        private String phone;
        
        public SMSSubscriber(String phone) {
            this.phone = phone;
        }
        
        @Override
        public void update(String news) {
            System.out.println("短信通知 " + phone + ": " + news);
        }
    }
              
观察者模式时序图如下所示：

![image-20250807144451601.png](https://article-images.zsxq.com/Frgyaj-mQ-dr7XiYJPFPp8yoHYCh)

oneThread 中的观察者模式实现 {#one-thread}
---------------------------------

### 1. Spring 事件机制 {#1-spring}

oneThread 框架基于 Spring 的事件发布机制实现观察者模式，这种方式具有以下优势：

*  
**框架集成**：与 Spring 容器深度集成，无需手动管理观察者列表。  
*  
**异步支持**：支持同步和异步事件处理。  
*  
  **类型安全**：基于泛型的强类型事件定义。

![image-20250807140723647.png](https://article-images.zsxq.com/FpCwABaNwasah2CRBNqeKxm_yb1N)

### 2. 事件定义 {#2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池配置更新事件
     * 当配置中心检测到线程池参数变更时，发布此事件通知所有相关组件
     */
    public class ThreadPoolConfigUpdateEvent extends ApplicationEvent {
    ​
        @Getter
        @Setter
        private BootstrapConfigProperties bootstrapConfigProperties;
    ​
        public ThreadPoolConfigUpdateEvent(Object source, BootstrapConfigProperties bootstrapConfigProperties) {
            super(source);
            this.bootstrapConfigProperties = bootstrapConfigProperties;
        }
    }
              
事件类设计要点：

*  
继承 `ApplicationEvent`，符合 Spring 事件规范。  
*  
  携带配置信息 `BootstrapConfigProperties`，为观察者提供必要数据。

### 3. 事件发布者（被观察者） {#3}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public abstract class AbstractDynamicThreadPoolRefresher implements ApplicationRunner {
    ​
        protected final BootstrapConfigProperties properties;
    ​
        /**
         * 配置刷新的核心方法
         * 解析配置信息并发布更新事件
         */
        @SneakyThrows
        public void refreshThreadPoolProperties(String configInfo) {
            // 1. 解析配置信息
            Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance()
                .parseConfig(configInfo, properties.getConfigFileType());
            
            // 2. 绑定到配置对象
            ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
            Binder binder = new Binder(sources);
            BootstrapConfigProperties refresherProperties = binder
                .bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties))
                .get();
    ​
            // 3. 发布线程池配置变更事件（观察者模式的核心）
            ApplicationContextHolder.publishEvent(
                new ThreadPoolConfigUpdateEvent(this, refresherProperties)
            );
        }
    }
              
### 4. 事件观察者实现 {#4}

#### 4.1 通用线程池配置监听器 {#4-1}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池刷新监听器
     * 处理通用线程池（非Web容器）的配置更新
     */
    @Slf4j
    @RequiredArgsConstructor
    public class DynamicThreadPoolRefreshListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        private final NotifierDispatcher notifierDispatcher;
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            BootstrapConfigProperties refresherProperties = event.getBootstrapConfigProperties();
    ​
            // 检查远程配置文件是否包含线程池配置
            if (CollUtil.isEmpty(refresherProperties.getExecutors())) {
                return;
            }
    ​
            // 刷新动态线程池对象核心参数
            for (ThreadPoolExecutorProperties remoteProperties : refresherProperties.getExecutors()) {
                // .....
            });
        }
    }
              
#### 4.2 Web容器线程池监听器 {#4-2-web}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * Web容器线程池刷新监听器
     * 专门处理Web容器（Tomcat/Jetty/Undertow）线程池的配置更新
     */
    @RequiredArgsConstructor
    public class WebThreadPoolRefreshListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        private final WebThreadPoolService webThreadPoolService;
        private final NotifierDispatcher notifierDispatcher;
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            BootstrapConfigProperties.WebThreadPoolExecutorConfig webExecutorConfig = event.getBootstrapConfigProperties().getWeb();
            if (Objects.isNull(webExecutorConfig)) {
                return;
            }
    ​
            WebThreadPoolBaseMetrics basicMetrics = webThreadPoolService.getBasicMetrics();
            if (!Objects.equals(basicMetrics.getCorePoolSize(), webExecutorConfig.getCorePoolSize())
                    || !Objects.equals(basicMetrics.getMaximumPoolSize(), webExecutorConfig.getMaximumPoolSize())
                    || !Objects.equals(basicMetrics.getKeepAliveTime(), webExecutorConfig.getKeepAliveTime())) {
                // 变更 Web 线程池配置
                webThreadPoolService.updateThreadPool(BeanUtil.toBean(webExecutorConfig, WebThreadPoolConfig.class));
    ​
                // 发送 Web 线程池配置变更通知
                sendWebThreadPoolConfigChangeMessage(basicMetrics, webExecutorConfig);
            }
        }
    ​
        // ......
    }
              
扩展性分析
-----

### 1. 新增线程池适配器 {#1}

假设我们需要支持 RocketMQ 的线程池动态调整，只需要以下步骤：

#### 1.1 创建监听器 {#1-1}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * RocketMQ线程池刷新监听器
     */
    @Component
    @RequiredArgsConstructor
    public class RocketMQThreadPoolRefreshListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        private final RocketMQThreadPoolService rocketMQService;
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            BootstrapConfigProperties.RocketMQConfig rocketMQConfig = 
                event.getBootstrapConfigProperties().getRocketMQ();
    ​
            if (Objects.isNull(rocketMQConfig)) {
                return;
            }
    ​
            // 更新RocketMQ线程池配置
            rocketMQService.updateThreadPool(rocketMQConfig);
        }
    }
              
#### 1.2 配置类扩展 {#1-2}

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 在BootstrapConfigProperties中添加RocketMQ配置
    @Data
    @ConfigurationProperties(prefix = BootstrapConfigProperties.PREFIX)
    public class BootstrapConfigProperties {
    ​
        // 现有配置...
    ​
        /**
         * RocketMQ线程池配置
         */
        private RocketMQConfig rocketMQ;
    ​
        @Data
        public static class RocketMQConfig {
            private Integer consumerCorePoolSize = 20;
            private Integer consumerMaximumPoolSize = 200;
            private Integer producerCorePoolSize = 4;
            private Integer producerMaximumPoolSize = 8;
        }
    }
              
### 2. 自定义事件处理 {#2}

开发者还可以监听配置更新事件，实现自定义逻辑：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 自定义配置变更监听器
     * 可以实现配置变更日志记录、指标上报等功能
     */
    @Component
    @Slf4j
    public class CustomConfigChangeListener implements ApplicationListener<ThreadPoolConfigUpdateEvent> {
    ​
        @Override
        public void onApplicationEvent(ThreadPoolConfigUpdateEvent event) {
            // 记录配置变更日志
            logConfigChange(event);
    ​
            // 上报监控指标
            reportMetrics(event);
    ​
            // 发送告警通知
            sendAlert(event);
        }
    ​
        private void logConfigChange(ThreadPoolConfigUpdateEvent event) {
            log.info("线程池配置发生变更: {}", 
                JSON.toJSONString(event.getBootstrapConfigProperties()));
        }
    }
              
Guava vs Spring 观察者模式 {#guava-vs-spring}
----------------------------------------

Guava 的观察者模式网上资料很多了，而且咱们也没有用到，这里我就不赘述了。这里重点说明下两者的区别：  

|   维度    |           **Spring ApplicationEvent**            |     **Guava EventBus**      |
|---------|--------------------------------------------------|-----------------------------|
| 所属框架    | Spring 框架                                        | Guava 工具库                   |
| 事件订阅方式  | 实现 `ApplicationListener` 或 `@EventListener` 注解监听 | 使用 `@Subscribe` 注解的方法监听     |
| 注册监听器   | 自动注册（通过 Spring 容器扫描 Bean）                        | 需手动注册到 EventBus             |
| 同步/异步支持 | 默认同步，支持 `@Async` 异步                              | 默认同步，也支持 `AsyncEventBus` 异步 |
| 解耦程度    | 高，基于事件名称和类型，天然解耦                                 | 中，需显式注册对象                   |

整体来看，Spring 事件机制与 Guava EventBus 在实现观察者模式上并无本质差异，都适用于常规的业务解耦场景。不过，考虑到 Spring 提供了原生支持，且生态完善、集成便捷，**更推荐优先使用 Spring 的事件监听机制，无需额外引入 Guava 依赖**。

同时，如果在代码中直接 `ApplicationContext` 发布事件监听，还能通过 IDEA 自带图标看到对应的观察者集合，Guava EventBus 是没有这个展示的。

![image-20250807165443521.png](https://article-images.zsxq.com/Fu23vEYZ1jiUQkCEt5YNTSNfxWwG)

文末总结
----

通过引入观察者模式，oneThread 动态线程池框架成功解决了跨模块通信的设计挑战，实现了配置中心 Starter 与各种线程池管理模块之间的松耦合通信。

通过观察者模式和 Spring 事件结合，有以下优势：

* 1.  
**模块解耦**：配置中心模块无需直接依赖具体的线程池管理模块，避免了循环依赖和模块耦合问题。  
* 2.  
**扩展性强**：新增线程池适配器时，只需实现监听器接口并注册为Spring Bean，无需修改现有代码。  
* 3.  
**职责清晰**：每个模块专注自己的核心职责，配置监听、事件路由、参数更新各司其职。  
* 4.  
  **易于测试**：松耦合的设计使得单元测试和集成测试更加简单。

完结，撒花 🎉

