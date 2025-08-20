2025年07月21日 20:00  
通过 Apollo 实现线程池参数配置，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节讲解了 Apollo 配置中心的基本原理、核心特性、适用场景，并结合实际项目分析了 Apollo 与 Nacos 在配置管理中的优劣对比。最后，通过一个示例展示了如何基于 `apollo-spring-boot-starter` 快速实现配置变更监听与动态刷新线程池配置的落地实现。

课程目录如下所示：

*  
前言  
*  
什么是 Apollo？  
*  
Apollo vs Nacos  
*  
Apollo 如何完成配置监听？  
*  
  文末总结

前言
---

如果大家只关注 `oneThread` 动态线程池的核心功能，而使用的是 **Nacos配置中心** ，则本章节内容可作为参考阅读，不强依赖。两者使用方式虽然不同，但监听配置变更的核心思想完全一致。
> Breathe，你在前面章节已经用 Nacos 实现了动态线程池配置刷新，为什么还要讲 Apollo？

这是个很好的问题。答案很简单：**我们做的是基础组件开发，不是仅应用单个项目的** 。

在真实的业务系统中，用户的基础设施环境往往是多样化的 ------ 有的公司用 Nacos，有的用 Apollo，还有的可能使用 Consul、Zookeeper、Etcd 甚至自研配置中心。这就要求我们构建的组件具备**良好的可插拔能力** ，可以适配不同的配置中心，按需选择、灵活集成。

因此，Apollo 的支持并不是"重复造轮子"，而是对动态线程池组件能力边界的延伸，**让用户可以无感集成到任意配置体系下的项目中** 。

什么是 Apollo？ {#apollo}
---------------------

### 1. 基础介绍 {#1}

Apollo（阿波罗）是一款可靠的分布式配置管理中心，诞生于携程框架研发部，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

服务端基于 SpringBoot 和 SpringCloud 开发，打包后可以直接运行，不需要额外安装 Tomcat 等应用容器。

Java客户端不依赖任何框架，能够运行于所有 Java 运行时环境，同时对 Spring/SpringBoot 环境也有较好的支持。

正是基于配置的特殊性，所以 Apollo 从设计之初就立志于成为一个有治理能力的配置发布平台，目前提供了以下的特性：

*  
  **统一管理不同环境、不同集群的配置** ：
  *  
  Apollo 提供了一个统一界面集中式管理不同环境（environment）、不同集群（cluster）、不同命名空间（namespace）的配置。  
  *  
  同一份代码部署在不同的集群，可以有不同的配置，比如 zookeeper 的地址等。  
  *  
通过命名空间（namespace）可以很方便地支持多个不同应用共享同一份配置，同时还允许应用对共享的配置进行覆盖。  
*  
**配置修改实时生效（热发布）** ：用户在 Apollo 修改完配置并发布后，客户端能实时（1秒）接收到最新的配置，并通知到应用程序。  
*  
**版本发布管理** ：所有的配置发布都有版本概念，从而可以方便地支持配置的回滚。  
*  
**灰度发布** ：支持配置的灰度发布，比如点了发布后，只对部分应用实例生效，等观察一段时间没问题后再推给所有应用实例。  
*  
  **配置项的全局视角搜索** ：
  *  
  通过对配置项的 key 与 value 进行的模糊检索，找到拥有对应值的配置项在哪个应用、环境、集群、命名空间中被使用。  
  *  
通过高亮显示、分页与跳转配置等操作，便于让管理员以及 SRE 角色快速、便捷地找到与更改资源的配置值。  
*  
  **权限管理、发布审核、操作审计**
  *  
  应用和配置的管理都有完善的权限管理机制，对配置的管理还分为了编辑和发布两个环节，从而减少人为的错误。  
  *  
所有的操作都有审计日志，可以方便地追踪问题。  
*  
**客户端配置信息监控** ：可以在界面上方便地看到配置在被哪些实例使用。  
*  
  **提供Java和.Net原生客户端** ：
  *  
  提供了 Java 和 .Net 的原生客户端，方便应用集成。  
  *  
  支持 Spring Placeholder, Annotation 和 Spring Boot 的 ConfigurationProperties，方便应用使用（需要Spring 3.1.1+）。  
  *  
同时提供了 Http 接口，非 Java 和 .Net 应用也可以方便地使用。  
*  
  **提供开放平台API** ：
  *  
  Apollo 自身提供了比较完善的统一配置管理界面，支持多环境、多数据中心配置管理、权限、流程治理等特性。不过 Apollo 出于通用性考虑，不会对配置的修改做过多限制，只要符合基本的格式就能保存，不会针对不同的配置值进行针对性的校验，如数据库用户名、密码，Redis 服务地址等。  
  *  
对于这类应用配置，Apollo 支持应用方通过开放平台 API 在 Apollo 进行配置的修改和发布，并且具备完善的授权和权限控制。  
*  
  **部署简单** ：
  *  
  配置中心作为基础服务，可用性要求非常高，这就要求 Apollo 对外部依赖尽可能地少。  
  *  
  目前唯一的外部依赖是 MySQL，所以部署非常简单，只要安装好 Java 和 MySQL 就可以让 Apollo 跑起来。  
  *  
    Apollo 还提供了打包脚本，一键就可以生成所有需要的安装包，并且支持自定义运行时参数。

### 2. Apollo at a glance {#2-apollo-at-a-glance}

#### 2.1 基础模型 {#2-1}

如下即是 Apollo 的基础模型：

* 1.  
用户在配置中心对配置进行修改并发布；  
* 2.  
配置中心通知 Apollo 客户端有配置更新；  
* 3.  
  Apollo 客户端从配置中心拉取最新的配置、更新本地配置并通知到应用。

![basic-architecture.png](https://article-images.zsxq.com/FrS7SKti6CZHPzZZlOvMNSaiic9q "basic-architecture.png")

#### 2.2 界面概览 {#2-2}

下图是 Apollo 配置中心中一个项目的配置首页：

*  
在页面左上方的环境列表模块展示了所有的环境和集群，用户可以随时切换。  
*  
页面中央展示了两个 namespace（application 和 FX.apollo）的配置信息，默认按照表格模式展示、编辑。用户也可以切换到文本模式，以文件形式查看、编辑。  
*  
  页面上可以方便地进行发布、回滚、灰度、授权、查看更改历史和发布历史等操作。

![apollo-home-screenshot.jpg](https://article-images.zsxq.com/FqzqAfUb1bp6UYmaZhz7S4Bd58zv "apollo-home-screenshot.jpg")

### 3. 架构设计 {#3}

#### 3.1 总体设计 {#3-1}

![overall-architecture.png](https://article-images.zsxq.com/FnkSs9Sy6brFvCyqFIJKtK9uVsxJ "overall-architecture.png")

上图简要描述了 Apollo 的总体设计，我们可以从下往上看：

*  
Config Service 提供配置的读取、推送等功能，服务对象是 Apollo 客户端。  
*  
Admin Service 提供配置的修改、发布等功能，服务对象是 Apollo Portal（管理界面）。  
*  
Config Service 和 Admin Service 都是多实例、无状态部署，所以需要将自己注册到 Eureka 中并保持心跳。  
*  
在 Eureka 之上我们架了一层 Meta Server 用于封装 Eureka 的服务发现接口。  
*  
Client 通过域名访问 Meta Server 获取 Config Service 服务列表（IP+Port），而后直接通过 IP+Port 访问服务，同时在 Client 侧会做 load balance、错误重试。  
*  
Portal 通过域名访问 Meta Server 获取 Admin Service 服务列表（IP+Port），而后直接通过 IP+Port 访问服务，同时在 Portal 侧会做 load balance、错误重试。  
*  
  为了简化部署，我们实际上会把 Config Service、Eureka 和 Meta Server 三个逻辑角色部署在同一个 JVM 进程中。

#### 3.2 客户端设计 {#3-2}

![client-architecture.png](https://article-images.zsxq.com/FnG-jOncdDf9GAZT2Y38gWC6u41k "client-architecture.png")

上图简要描述了 Apollo 客户端的实现原理：

* 1.  
客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。  
* 2.  
  客户端还会定时从 Apollo 配置中心服务端拉取应用的最新配置。
  *  
  这是一个 fallback 机制，为了防止推送机制失效导致配置不更新；  
  *  
  客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回 304 - Not Modified；  
  *  
定时频率默认为每 5 分钟拉取一次，客户端也可以通过在运行时指定 System Property: `apollo.refreshInterval` 来覆盖，单位为分钟。  
* 3.  
客户端从 Apollo 配置中心服务端获取到应用的最新配置后，会保存在内存中。  
* 4.  
客户端会把从服务端获取到的配置在本地文件系统缓存一份。在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置。  
* 5.  
  应用程序从 Apollo 客户端获取最新的配置、订阅配置更新通知。

看了上面的描述，相信大家对 Apollo 有了简单认识，如果想学习更多知识，参考 [Apollo 官方文档](https://www.apolloconfig.com/#/zh/README)。

Apollo vs Nacos {#apollo-vs-nacos}
----------------------------------

### 1. 基础定位 {#1}

|  特性  |             Apollo             |        Nacos        |
|------|--------------------------------|---------------------|
| 核心定位 | 配置中心（Configuration Management） | 配置中心 + 服务注册与发现（双核心） |
| 主导方  | 携程                             | 阿里巴巴                |
| 开源时间 | 2017                           | 2018                |

### 2. 功能对比 {#2}

|     功能特性      |        Apollo         |        Nacos        |
|---------------|-----------------------|---------------------|
| 配置管理          | ✅ 支持多环境、多集群、灰度、注释等功能  | ✅ 支持，配置轻量，支持动态刷新    |
| 服务注册与发现       | ❌ 不支持                 | ✅ 支持 DNS+RPC 注册发现   |
| 权限与审计         | ✅ 支持细粒度权限控制、操作日志      | ⚠️ 支持较弱，社区版权限较简单    |
| 多租户/Namespace | ✅ 支持，较为完善             | ✅ 支持，但隔离粒度和隔离机制略弱   |
| 灰度发布          | ✅ 支持灰度配置、灰度规则等        | ❌ 默认不支持，需要额外接入      |
| 控制台体验         | ✅ 完善，功能细化、可编辑历史配置、权限等 | ⚠️ 简洁但偏轻量，适合基础使用    |
| 历史版本 \& 回滚    | ✅ 支持查看、回滚历史版本         | ✅ 支持，但功能较简单         |
| 持久化存储         | ✅ 默认基于 MySQL          | ✅ 默认基于 MySQL，也支持嵌入式 |

### 3. 使用场景 {#3}

Apollo 适合的场景：

*  
企业级配置中心需求，要求权限、审计、版本回滚完善。  
*  
配置维度多、业务线多、环境划分复杂。  
*  
  对灰度发布、配置审计等有强需求。

Nacos 适合的场景：

*  
需要配置管理 + 服务注册一体化解决方案。  
*  
微服务治理场景中，希望轻量、快速接入（如 Dubbo/Spring Cloud）。  
*  
  注册中心与配置中心统一部署场景。

总结对比：  

|  对比维度  |    Apollo    |          Nacos          |
|--------|--------------|-------------------------|
| 配置管理能力 | ⭐⭐⭐⭐⭐（功能丰富）  | ⭐⭐⭐（轻量实用）               |
| 服务注册发现 | ❌            | ⭐⭐⭐⭐（功能齐全）              |
| 控制台体验  | ⭐⭐⭐⭐（企业级 UI） | ⭐⭐（基础可用）                |
| 运维复杂度  | ⭐⭐（需部署多个组件）  | ⭐⭐⭐⭐（轻量级部署）             |
| 灰度与权限  | ⭐⭐⭐⭐（支持细粒度）  | ⭐⭐（简单支持）                |
| 使用门槛   | ⭐⭐（需配套 SDK）  | ⭐⭐⭐⭐（Spring Cloud 快速整合） |
| 社区与生态  | ⭐⭐⭐⭐         | ⭐⭐⭐⭐                    |

### 4. Breathe主观比较 {#4}

这一章节，Breathe不打算从官方文档或功能清单去评比 Apollo 和 Nacos，而是结合自己在项目中接触和落地的实际体验，做一波**纯主观、偏实战的横向对比** 。如果你有不同观点，欢迎评论区一起交流。

我主要从以下几个维度做个简要分析：

*  
**架构层面** ：两者的底层架构其实都已经非常成熟，经历了大规模生产验证。无论选哪一个，稳定性和可用性都无需担心，因此架构本身不构成选型门槛。  
*  
**功能完备性** ：就"配置中心"本身而言，Apollo 的功能更偏企业级，**在灰度发布、权限控制、版本回滚等方面优势明显** ，这方面 Nacos 是无法比的，属于被 Apollo 碾压的存在。  
*  
**集成便利性** ：但 Nacos 胜在"全能型选手"------**服务注册+配置中心一体化** ，对中小项目尤其友好，减少了中间件部署、维护和学习成本，我个人在业务场景中更偏好 Nacos。  
*  
**代码观感** ：Apollo 的早期源码风格有较明显的 `.NET` 影子，比如 `m_` 命名前缀等，对于 Java 开发者不太友好；相比之下，Nacos 的代码结构更贴合现代 Java 开发习惯，上手更自然。  
*  
  **社区活跃度** ：目前来看，Apollo 的发布频率有所下降（23年两个版本，24年一个，25年上半年一个），但我更倾向于理解为"功能趋于稳定"；而 Nacos 因兼具注册中心职能，社区仍较活跃，PMC 主导下 Bug 修复和发版也更持续。

对大多数团队而言，**Nacos已足够满足日常配置需求** ；但如果你对权限、审计、灰度发布等有更高要求，**Apollo是更合适的选择** 。两者并非替代关系，而是可按需选型的不同侧重。

Apollo 如何完成配置监听？ {#apollo}
--------------------------

如果大家看过 Nacos 的配置监听和参数变更，其实 Apollo 流程是一样的，只是监听逻辑是不同的。
> 本章节的重点内容集中在模块： `apollo-spring-boot-starter`。

如果大家想要在项目中使用 Apollo 配置管理，需要在 pom.xml 文件中添加对应的依赖：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>com.ctrip.framework.apollo</groupId>
        <artifactId>apollo-client-config-data</artifactId>
    </dependency>
              
从 Apollo 提供的配置中心抽象接口中，我们可以看到其核心方法之一是 `addChangeListener`，用于注册配置变更监听器。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface Config {
        // ......
        /**
         * Add change listener to this config instance, will be notified when any key is changed in this namespace.
         *
         * @param listener the config change listener
         */
        void addChangeListener(ConfigChangeListener listener);
        // ......
    }
              
下方 Yaml 配置为 Apollo 需要设置参数内容：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    app:
      id: onethread-apollo-example${unique-name:} # Apollo 配置中心的 应用唯一标识（AppId）
    env: dev # 指定 Apollo 的环境（Environment），如 dev、fat、uat、pro 等
    ​
    apollo:
      meta: http://127.0.0.1:8080 # 指定 Apollo 的 Meta Server 地址。客户端通过 Meta Server 获取对应的配置服务（Config Service）地址
      autoUpdateInjectedSpringProperties: true # 是否支持 Spring 注入属性的自动刷新
      bootstrap:
        enabled: true # 启用 Apollo 的 Bootstrap 加载机制，默认 SpringBoot 的配置加载时机晚于 Apollo
        namespaces: application # 指定从哪些 Namespace 加载配置（多个以逗号分隔）
        eagerLoad:
          enabled: true # 强制在 Spring 容器加载前，立即加载 Apollo 配置
      configService: http://127.0.0.1:8080 # 手动指定 Config Service 地址（可选）
              
下面我们先通过一张时序图，从宏观角度快速了解 Apollo 配置监听注册的大致流程，帮助大家建立整体认知：

![image-20250721110846775.png](https://article-images.zsxq.com/FlVvfusM1n8nNUAMeL9oqNFh2PwC "image-20250721110846775.png")

好，咱们继续往下看 ------ **来看下注册配置监听的方法是怎么实现的：**  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class ApolloRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.ApolloConfig apolloConfig = properties.getApollo();
            String[] apolloNamespaces = apolloConfig.getNamespace().split(",");
    ​
            String namespace = apolloNamespaces[0];
            String configFileType = properties.getConfigFileType().getValue();
            Config config = ConfigService.getConfig(String.format("%s.%s", namespace, properties.getConfigFileType().getValue()));
    ​
            ConfigChangeListener configChangeListener = createConfigChangeListener(namespace, configFileType);
            config.addChangeListener(configChangeListener);
    ​
            log.info("Dynamic thread pool refresher, add apollo listener success. namespace: {}", namespace);
        }
    ​
        private ConfigChangeListener createConfigChangeListener(String namespace, String configFileType) {
            return configChangeEvent -> {
                // 如果 Apollo 配置文件变更，会触发该方法进行回调
                String namespaceItem = namespace.replace("." + configFileType, "");
                ConfigFileFormat configFileFormat = ConfigFileFormat.fromString(configFileType);
                ConfigFile configFile = ConfigService.getConfigFile(namespaceItem, configFileFormat);
                String content = configFile.getContent(); // 变更后的最新内容
                refreshThreadPoolProperties(content); // 刷新动态线程池配置
            };
        }
    }
              
方法的具体逻辑可以分为以下几个步骤：

* 1.  
**获取Apollo配置** ：首先，通过 `properties.getApollo()` 获取配置中心的关键参数，读取项目中的 Apollo 命名空间配置，支持多个，以逗号分隔。  
* 2.  
**注册配置变更监听器** ：接着调用 `config.addChangeListener(...)` 方法，向指定的 `nameserver` 注册监听器。一旦 Apollo 端对应的配置发生变更，监听器就会被自动触发。  
* 3.  
**配置变更回调** ：当配置发生变更时，`configChangeListener` 方法会被回调。我们通常会在这里调用 `refreshThreadPoolProperties(...)`，将最新的配置信息解析出来并动态刷新线程池参数。  
* 4.  
  **日志输出** ：为了方便后续运维排查，注册成功后会打印一条 `info` 级别的日志，明确表示监听器已经生效。

Apollo 默认使用 `properties` 格式作为配置载体，从返回结果中我们也可以清晰看出。

![image-20250721105752254.png](https://article-images.zsxq.com/Fsv06HfBpYsD2e-BlDf_U5b8gSux "image-20250721105752254.png")

整体监听机制的实现逻辑与 Nacos 十分相似，因此本章节不再赘述重复内容，重点聚焦 Apollo 的差异化实现。
> 需要注意的是，Apollo 的启动流程相对复杂，我当时也尝试了多种方式才最终通过官网提供的 `docker-compose` 成功启动。如果你只是想聚焦 `oneThread` 的核心能力，对 Apollo 本身并不感兴趣，可以跳过部署部分，直接进入后续内容。

文末总结
----

Apollo 并非 oneThread 的强依赖，但支持它，是为了让组件具备更强的**通用性与适配性** ，真正做到在不同配置中心之间**无缝接入、按需集成** 。

此外，Breathe也特意考古了一下 Apollo 作者宋顺（花名：齐天）的早期博客，内容质量很高，对于理解 Apollo 的设计初衷和落地思路非常有帮助，推荐大家作为延伸阅读深入思考。
> 宋顺（花名：齐天），蚂蚁集团资深技术专家，Apollo Config PMC。
>
> 在微服务架构、分布式计算等领域有着丰富的经验，2019 年加入蚂蚁集团，目前专注于云原生和微服务方向，如 Service Mesh、Serverless、Application Runtime 等。
>
> 毕业于复旦大学软件工程系，曾就职于大众点评、携程，负责后台系统、中间件等研发工作。
>
> *  
> [Apollo配置中心介绍](https://nobodyiam.com/2016/07/09/introduction-to-apollo/)  
> *  
>   [配置中心，让微服务更『智能』](https://nobodyiam.com/2018/07/29/configuration-center-makes-microservices-smart/)

完结，撒花 🎉  
通过 Apollo 实现线程池参数配置，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本章节讲解了 Apollo 配置中心的基本原理、核心特性、适用场景，并结合实际项目分析了 Apollo 与 Nacos 在配置管理中的优劣对比。最后，通过一个示例展示了如何基于 `apollo-spring-boot-starter` 快速实现配置变更监听与动态刷新线程池配置的落地实现。

课程目录如下所示：

*  
前言  
*  
什么是 Apollo？  
*  
Apollo vs Nacos  
*  
Apollo 如何完成配置监听？  
*  
  文末总结

前言
---

如果大家只关注 `oneThread` 动态线程池的核心功能，而使用的是 **Nacos配置中心** ，则本章节内容可作为参考阅读，不强依赖。两者使用方式虽然不同，但监听配置变更的核心思想完全一致。
> Breathe，你在前面章节已经用 Nacos 实现了动态线程池配置刷新，为什么还要讲 Apollo？

这是个很好的问题。答案很简单：**我们做的是基础组件开发，不是仅应用单个项目的** 。

在真实的业务系统中，用户的基础设施环境往往是多样化的 ------ 有的公司用 Nacos，有的用 Apollo，还有的可能使用 Consul、Zookeeper、Etcd 甚至自研配置中心。这就要求我们构建的组件具备**良好的可插拔能力** ，可以适配不同的配置中心，按需选择、灵活集成。

因此，Apollo 的支持并不是"重复造轮子"，而是对动态线程池组件能力边界的延伸，**让用户可以无感集成到任意配置体系下的项目中** 。

什么是 Apollo？ {#apollo}
---------------------

### 1. 基础介绍 {#1}

Apollo（阿波罗）是一款可靠的分布式配置管理中心，诞生于携程框架研发部，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

服务端基于 SpringBoot 和 SpringCloud 开发，打包后可以直接运行，不需要额外安装 Tomcat 等应用容器。

Java客户端不依赖任何框架，能够运行于所有 Java 运行时环境，同时对 Spring/SpringBoot 环境也有较好的支持。

正是基于配置的特殊性，所以 Apollo 从设计之初就立志于成为一个有治理能力的配置发布平台，目前提供了以下的特性：

*  
  **统一管理不同环境、不同集群的配置** ：
  *  
  Apollo 提供了一个统一界面集中式管理不同环境（environment）、不同集群（cluster）、不同命名空间（namespace）的配置。  
  *  
  同一份代码部署在不同的集群，可以有不同的配置，比如 zookeeper 的地址等。  
  *  
通过命名空间（namespace）可以很方便地支持多个不同应用共享同一份配置，同时还允许应用对共享的配置进行覆盖。  
*  
**配置修改实时生效（热发布）** ：用户在 Apollo 修改完配置并发布后，客户端能实时（1秒）接收到最新的配置，并通知到应用程序。  
*  
**版本发布管理** ：所有的配置发布都有版本概念，从而可以方便地支持配置的回滚。  
*  
**灰度发布** ：支持配置的灰度发布，比如点了发布后，只对部分应用实例生效，等观察一段时间没问题后再推给所有应用实例。  
*  
  **配置项的全局视角搜索** ：
  *  
  通过对配置项的 key 与 value 进行的模糊检索，找到拥有对应值的配置项在哪个应用、环境、集群、命名空间中被使用。  
  *  
通过高亮显示、分页与跳转配置等操作，便于让管理员以及 SRE 角色快速、便捷地找到与更改资源的配置值。  
*  
  **权限管理、发布审核、操作审计**
  *  
  应用和配置的管理都有完善的权限管理机制，对配置的管理还分为了编辑和发布两个环节，从而减少人为的错误。  
  *  
所有的操作都有审计日志，可以方便地追踪问题。  
*  
**客户端配置信息监控** ：可以在界面上方便地看到配置在被哪些实例使用。  
*  
  **提供Java和.Net原生客户端** ：
  *  
  提供了 Java 和 .Net 的原生客户端，方便应用集成。  
  *  
  支持 Spring Placeholder, Annotation 和 Spring Boot 的 ConfigurationProperties，方便应用使用（需要Spring 3.1.1+）。  
  *  
同时提供了 Http 接口，非 Java 和 .Net 应用也可以方便地使用。  
*  
  **提供开放平台API** ：
  *  
  Apollo 自身提供了比较完善的统一配置管理界面，支持多环境、多数据中心配置管理、权限、流程治理等特性。不过 Apollo 出于通用性考虑，不会对配置的修改做过多限制，只要符合基本的格式就能保存，不会针对不同的配置值进行针对性的校验，如数据库用户名、密码，Redis 服务地址等。  
  *  
对于这类应用配置，Apollo 支持应用方通过开放平台 API 在 Apollo 进行配置的修改和发布，并且具备完善的授权和权限控制。  
*  
  **部署简单** ：
  *  
  配置中心作为基础服务，可用性要求非常高，这就要求 Apollo 对外部依赖尽可能地少。  
  *  
  目前唯一的外部依赖是 MySQL，所以部署非常简单，只要安装好 Java 和 MySQL 就可以让 Apollo 跑起来。  
  *  
    Apollo 还提供了打包脚本，一键就可以生成所有需要的安装包，并且支持自定义运行时参数。

### 2. Apollo at a glance {#2-apollo-at-a-glance}

#### 2.1 基础模型 {#2-1}

如下即是 Apollo 的基础模型：

* 1.  
用户在配置中心对配置进行修改并发布；  
* 2.  
配置中心通知 Apollo 客户端有配置更新；  
* 3.  
  Apollo 客户端从配置中心拉取最新的配置、更新本地配置并通知到应用。

![basic-architecture.png](https://article-images.zsxq.com/FrS7SKti6CZHPzZZlOvMNSaiic9q "basic-architecture.png")

#### 2.2 界面概览 {#2-2}

下图是 Apollo 配置中心中一个项目的配置首页：

*  
在页面左上方的环境列表模块展示了所有的环境和集群，用户可以随时切换。  
*  
页面中央展示了两个 namespace（application 和 FX.apollo）的配置信息，默认按照表格模式展示、编辑。用户也可以切换到文本模式，以文件形式查看、编辑。  
*  
  页面上可以方便地进行发布、回滚、灰度、授权、查看更改历史和发布历史等操作。

![apollo-home-screenshot.jpg](https://article-images.zsxq.com/FqzqAfUb1bp6UYmaZhz7S4Bd58zv "apollo-home-screenshot.jpg")

### 3. 架构设计 {#3}

#### 3.1 总体设计 {#3-1}

![overall-architecture.png](https://article-images.zsxq.com/FnkSs9Sy6brFvCyqFIJKtK9uVsxJ "overall-architecture.png")

上图简要描述了 Apollo 的总体设计，我们可以从下往上看：

*  
Config Service 提供配置的读取、推送等功能，服务对象是 Apollo 客户端。  
*  
Admin Service 提供配置的修改、发布等功能，服务对象是 Apollo Portal（管理界面）。  
*  
Config Service 和 Admin Service 都是多实例、无状态部署，所以需要将自己注册到 Eureka 中并保持心跳。  
*  
在 Eureka 之上我们架了一层 Meta Server 用于封装 Eureka 的服务发现接口。  
*  
Client 通过域名访问 Meta Server 获取 Config Service 服务列表（IP+Port），而后直接通过 IP+Port 访问服务，同时在 Client 侧会做 load balance、错误重试。  
*  
Portal 通过域名访问 Meta Server 获取 Admin Service 服务列表（IP+Port），而后直接通过 IP+Port 访问服务，同时在 Portal 侧会做 load balance、错误重试。  
*  
  为了简化部署，我们实际上会把 Config Service、Eureka 和 Meta Server 三个逻辑角色部署在同一个 JVM 进程中。

#### 3.2 客户端设计 {#3-2}

![client-architecture.png](https://article-images.zsxq.com/FnG-jOncdDf9GAZT2Y38gWC6u41k "client-architecture.png")

上图简要描述了 Apollo 客户端的实现原理：

* 1.  
客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。  
* 2.  
  客户端还会定时从 Apollo 配置中心服务端拉取应用的最新配置。
  *  
  这是一个 fallback 机制，为了防止推送机制失效导致配置不更新；  
  *  
  客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回 304 - Not Modified；  
  *  
定时频率默认为每 5 分钟拉取一次，客户端也可以通过在运行时指定 System Property: `apollo.refreshInterval` 来覆盖，单位为分钟。  
* 3.  
客户端从 Apollo 配置中心服务端获取到应用的最新配置后，会保存在内存中。  
* 4.  
客户端会把从服务端获取到的配置在本地文件系统缓存一份。在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置。  
* 5.  
  应用程序从 Apollo 客户端获取最新的配置、订阅配置更新通知。

看了上面的描述，相信大家对 Apollo 有了简单认识，如果想学习更多知识，参考 [Apollo 官方文档](https://www.apolloconfig.com/#/zh/README)。

Apollo vs Nacos {#apollo-vs-nacos}
----------------------------------

### 1. 基础定位 {#1}

|  特性  |             Apollo             |        Nacos        |
|------|--------------------------------|---------------------|
| 核心定位 | 配置中心（Configuration Management） | 配置中心 + 服务注册与发现（双核心） |
| 主导方  | 携程                             | 阿里巴巴                |
| 开源时间 | 2017                           | 2018                |

### 2. 功能对比 {#2}

|     功能特性      |        Apollo         |        Nacos        |
|---------------|-----------------------|---------------------|
| 配置管理          | ✅ 支持多环境、多集群、灰度、注释等功能  | ✅ 支持，配置轻量，支持动态刷新    |
| 服务注册与发现       | ❌ 不支持                 | ✅ 支持 DNS+RPC 注册发现   |
| 权限与审计         | ✅ 支持细粒度权限控制、操作日志      | ⚠️ 支持较弱，社区版权限较简单    |
| 多租户/Namespace | ✅ 支持，较为完善             | ✅ 支持，但隔离粒度和隔离机制略弱   |
| 灰度发布          | ✅ 支持灰度配置、灰度规则等        | ❌ 默认不支持，需要额外接入      |
| 控制台体验         | ✅ 完善，功能细化、可编辑历史配置、权限等 | ⚠️ 简洁但偏轻量，适合基础使用    |
| 历史版本 \& 回滚    | ✅ 支持查看、回滚历史版本         | ✅ 支持，但功能较简单         |
| 持久化存储         | ✅ 默认基于 MySQL          | ✅ 默认基于 MySQL，也支持嵌入式 |

### 3. 使用场景 {#3}

Apollo 适合的场景：

*  
企业级配置中心需求，要求权限、审计、版本回滚完善。  
*  
配置维度多、业务线多、环境划分复杂。  
*  
  对灰度发布、配置审计等有强需求。

Nacos 适合的场景：

*  
需要配置管理 + 服务注册一体化解决方案。  
*  
微服务治理场景中，希望轻量、快速接入（如 Dubbo/Spring Cloud）。  
*  
  注册中心与配置中心统一部署场景。

总结对比：  

|  对比维度  |    Apollo    |          Nacos          |
|--------|--------------|-------------------------|
| 配置管理能力 | ⭐⭐⭐⭐⭐（功能丰富）  | ⭐⭐⭐（轻量实用）               |
| 服务注册发现 | ❌            | ⭐⭐⭐⭐（功能齐全）              |
| 控制台体验  | ⭐⭐⭐⭐（企业级 UI） | ⭐⭐（基础可用）                |
| 运维复杂度  | ⭐⭐（需部署多个组件）  | ⭐⭐⭐⭐（轻量级部署）             |
| 灰度与权限  | ⭐⭐⭐⭐（支持细粒度）  | ⭐⭐（简单支持）                |
| 使用门槛   | ⭐⭐（需配套 SDK）  | ⭐⭐⭐⭐（Spring Cloud 快速整合） |
| 社区与生态  | ⭐⭐⭐⭐         | ⭐⭐⭐⭐                    |

### 4. Breathe主观比较 {#4}

这一章节，Breathe不打算从官方文档或功能清单去评比 Apollo 和 Nacos，而是结合自己在项目中接触和落地的实际体验，做一波**纯主观、偏实战的横向对比** 。如果你有不同观点，欢迎评论区一起交流。

我主要从以下几个维度做个简要分析：

*  
**架构层面** ：两者的底层架构其实都已经非常成熟，经历了大规模生产验证。无论选哪一个，稳定性和可用性都无需担心，因此架构本身不构成选型门槛。  
*  
**功能完备性** ：就"配置中心"本身而言，Apollo 的功能更偏企业级，**在灰度发布、权限控制、版本回滚等方面优势明显** ，这方面 Nacos 是无法比的，属于被 Apollo 碾压的存在。  
*  
**集成便利性** ：但 Nacos 胜在"全能型选手"------**服务注册+配置中心一体化** ，对中小项目尤其友好，减少了中间件部署、维护和学习成本，我个人在业务场景中更偏好 Nacos。  
*  
**代码观感** ：Apollo 的早期源码风格有较明显的 `.NET` 影子，比如 `m_` 命名前缀等，对于 Java 开发者不太友好；相比之下，Nacos 的代码结构更贴合现代 Java 开发习惯，上手更自然。  
*  
  **社区活跃度** ：目前来看，Apollo 的发布频率有所下降（23年两个版本，24年一个，25年上半年一个），但我更倾向于理解为"功能趋于稳定"；而 Nacos 因兼具注册中心职能，社区仍较活跃，PMC 主导下 Bug 修复和发版也更持续。

对大多数团队而言，**Nacos已足够满足日常配置需求** ；但如果你对权限、审计、灰度发布等有更高要求，**Apollo是更合适的选择** 。两者并非替代关系，而是可按需选型的不同侧重。

Apollo 如何完成配置监听？ {#apollo}
--------------------------

如果大家看过 Nacos 的配置监听和参数变更，其实 Apollo 流程是一样的，只是监听逻辑是不同的。
> 本章节的重点内容集中在模块： `apollo-spring-boot-starter`。

如果大家想要在项目中使用 Apollo 配置管理，需要在 pom.xml 文件中添加对应的依赖：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>com.ctrip.framework.apollo</groupId>
        <artifactId>apollo-client-config-data</artifactId>
    </dependency>
              
从 Apollo 提供的配置中心抽象接口中，我们可以看到其核心方法之一是 `addChangeListener`，用于注册配置变更监听器。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface Config {
        // ......
        /**
         * Add change listener to this config instance, will be notified when any key is changed in this namespace.
         *
         * @param listener the config change listener
         */
        void addChangeListener(ConfigChangeListener listener);
        // ......
    }
              
下方 Yaml 配置为 Apollo 需要设置参数内容：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    app:
      id: onethread-apollo-example${unique-name:} # Apollo 配置中心的 应用唯一标识（AppId）
    env: dev # 指定 Apollo 的环境（Environment），如 dev、fat、uat、pro 等
    ​
    apollo:
      meta: http://127.0.0.1:8080 # 指定 Apollo 的 Meta Server 地址。客户端通过 Meta Server 获取对应的配置服务（Config Service）地址
      autoUpdateInjectedSpringProperties: true # 是否支持 Spring 注入属性的自动刷新
      bootstrap:
        enabled: true # 启用 Apollo 的 Bootstrap 加载机制，默认 SpringBoot 的配置加载时机晚于 Apollo
        namespaces: application # 指定从哪些 Namespace 加载配置（多个以逗号分隔）
        eagerLoad:
          enabled: true # 强制在 Spring 容器加载前，立即加载 Apollo 配置
      configService: http://127.0.0.1:8080 # 手动指定 Config Service 地址（可选）
              
下面我们先通过一张时序图，从宏观角度快速了解 Apollo 配置监听注册的大致流程，帮助大家建立整体认知：

![image-20250721110846775.png](https://article-images.zsxq.com/FlVvfusM1n8nNUAMeL9oqNFh2PwC "image-20250721110846775.png")

好，咱们继续往下看 ------ **来看下注册配置监听的方法是怎么实现的：**  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class ApolloRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.ApolloConfig apolloConfig = properties.getApollo();
            String[] apolloNamespaces = apolloConfig.getNamespace().split(",");
    ​
            String namespace = apolloNamespaces[0];
            String configFileType = properties.getConfigFileType().getValue();
            Config config = ConfigService.getConfig(String.format("%s.%s", namespace, properties.getConfigFileType().getValue()));
    ​
            ConfigChangeListener configChangeListener = createConfigChangeListener(namespace, configFileType);
            config.addChangeListener(configChangeListener);
    ​
            log.info("Dynamic thread pool refresher, add apollo listener success. namespace: {}", namespace);
        }
    ​
        private ConfigChangeListener createConfigChangeListener(String namespace, String configFileType) {
            return configChangeEvent -> {
                // 如果 Apollo 配置文件变更，会触发该方法进行回调
                String namespaceItem = namespace.replace("." + configFileType, "");
                ConfigFileFormat configFileFormat = ConfigFileFormat.fromString(configFileType);
                ConfigFile configFile = ConfigService.getConfigFile(namespaceItem, configFileFormat);
                String content = configFile.getContent(); // 变更后的最新内容
                refreshThreadPoolProperties(content); // 刷新动态线程池配置
            };
        }
    }
              
方法的具体逻辑可以分为以下几个步骤：

* 1.  
**获取Apollo配置** ：首先，通过 `properties.getApollo()` 获取配置中心的关键参数，读取项目中的 Apollo 命名空间配置，支持多个，以逗号分隔。  
* 2.  
**注册配置变更监听器** ：接着调用 `config.addChangeListener(...)` 方法，向指定的 `nameserver` 注册监听器。一旦 Apollo 端对应的配置发生变更，监听器就会被自动触发。  
* 3.  
**配置变更回调** ：当配置发生变更时，`configChangeListener` 方法会被回调。我们通常会在这里调用 `refreshThreadPoolProperties(...)`，将最新的配置信息解析出来并动态刷新线程池参数。  
* 4.  
  **日志输出** ：为了方便后续运维排查，注册成功后会打印一条 `info` 级别的日志，明确表示监听器已经生效。

Apollo 默认使用 `properties` 格式作为配置载体，从返回结果中我们也可以清晰看出。

![image-20250721105752254.png](https://article-images.zsxq.com/Fsv06HfBpYsD2e-BlDf_U5b8gSux "image-20250721105752254.png")

整体监听机制的实现逻辑与 Nacos 十分相似，因此本章节不再赘述重复内容，重点聚焦 Apollo 的差异化实现。
> 需要注意的是，Apollo 的启动流程相对复杂，我当时也尝试了多种方式才最终通过官网提供的 `docker-compose` 成功启动。如果你只是想聚焦 `oneThread` 的核心能力，对 Apollo 本身并不感兴趣，可以跳过部署部分，直接进入后续内容。

文末总结
----

Apollo 并非 oneThread 的强依赖，但支持它，是为了让组件具备更强的**通用性与适配性** ，真正做到在不同配置中心之间**无缝接入、按需集成** 。

此外，Breathe也特意考古了一下 Apollo 作者宋顺（花名：齐天）的早期博客，内容质量很高，对于理解 Apollo 的设计初衷和落地思路非常有帮助，推荐大家作为延伸阅读深入思考。
> 宋顺（花名：齐天），蚂蚁集团资深技术专家，Apollo Config PMC。
>
> 在微服务架构、分布式计算等领域有着丰富的经验，2019 年加入蚂蚁集团，目前专注于云原生和微服务方向，如 Service Mesh、Serverless、Application Runtime 等。
>
> 毕业于复旦大学软件工程系，曾就职于大众点评、携程，负责后台系统、中间件等研发工作。
>
> *  
> [Apollo配置中心介绍](https://nobodyiam.com/2016/07/09/introduction-to-apollo/)  
> *  
>   [配置中心，让微服务更『智能』](https://nobodyiam.com/2018/07/29/configuration-center-makes-microservices-smart/)

完结，撒花 🎉

