2025年07月15日 22:13  
通过 Nacos 实现线程池参数配置，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本节主要讲解了当 Nacos 配置发生变更时，监听器接收到的是完整的配置文件字符串内容（如 YAML 格式）。由于这类字符串无法直接使用，我们需要对其进行解析，转换成可操作的 Java 对象（如 `BootstrapConfigProperties`），以便后续动态刷新线程池配置。

课程目录如下所示：

*  
什么是 Nacos？  
*  
Nacos 如何完成配置监听？  
*  
  文末总结

什么是 Nacos？ {#nacos}
-------------------

> 该章节取自 [Nacos 官网](https://nacos.io/docs/v2.5/overview) Version 2.5，本章节重点使用的是 Nacos 动态配置服务，如已了解可跳过该小节。

Nacos `/nɑ:kəʊs/` 是 Dynamic Naming and Configuration Service 的首字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以\*\*"服务"\*\* 为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

### 1. 产品功能 {#1}

#### 1.1 服务发现和服务健康监测 {#1-1}

Nacos 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用原生SDK、OpenAPI、或一个独立的 Agent 注册 Service 后，服务消费者可以使用 HTTP\&API 查找和发现服务。

Nacos 提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求。Nacos 支持传输层 (PING 或 TCP)和应用层 (如 HTTP、MySQL、用户自定义）的健康检查。 对于复杂的云环境和网络拓扑环境中（如 VPC、边缘网络等）服务的健康检查，Nacos 提供了 agent 上报模式和服务端主动检测2种健康检查模式。Nacos 还提供了统一的健康检查仪表盘，帮助您根据健康状态管理服务的可用性及流量。

#### 1.2 动态配置服务 {#1-2}

动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。

动态配置消除了配置变更时重新部署应用和服务的需要，让配置管理变得更加高效和敏捷。

配置中心化管理让实现无状态服务变得更简单，让服务按需弹性扩展变得更容易。

Nacos 提供了一个简洁易用的UI ([控制台样例 Demo](http://console.nacos.io/index.html?spm=5238cd80.72a042d5.0.0.5bc0cd36IfxBwB)) 帮助您管理所有的服务和应用的配置。Nacos 还提供包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性，帮助您更安全地在生产环境中管理配置变更和降低配置变更带来的风险。

#### 1.3 动态 DNS 服务 {#1-3-dns}

动态 DNS 服务支持权重路由，让您更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。动态DNS服务还能让您更容易地实现以 DNS 协议为基础的服务发现，以帮助您消除耦合到厂商私有服务发现 API 上的风险。

Nacos 提供了一些简单的 DNS APIs 帮助您管理服务的关联域名和可用的 IP 列表。

#### 1.4 服务及其元数据管理 {#1-4}

Nacos 能让您从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。

### 2. 产品优势 {#2}

*  
**易于使用** ：Nacos 经历几万人使用反馈优化，提供统一的服务发现和配置管理功能，通过直观的 Web 界面和简洁的 API，为开发和运维人员在云原生环境中带来了便捷的服务注册、发现和配置更新操作。  
*  
**特性丰富** ：Nacos 提供了包括服务发现、配置管理、动态 DNS 服务、服务元数据管理、流量管理、服务监控、服务治理等在内的一系列特性，帮助您在云原生时代，更轻松的构建、交付和管理微服务。  
*  
**极致性能** ：Nacos 经过阿里双十一超快伸缩场景的锤炼，提供高性能的服务注册和发现能力，以及低延迟的配置更新响应，确保在大规模分布式系统中的高效率和稳定运行。  
*  
**超大容量** ：Nacos 诞生自阿里的百万实例规模，造就支持海量服务和配置的管理，能够满足大型分布式系统对高并发和高可用性的需求。  
*  
**稳定可用** ：Nacos 通过自研的同步协议，配合生态中应用广泛的 Raft 协议，确保了服务的高可用性和数据的稳定性，保证阿里双十一系统的高可用稳定运行。  
*  
  **开放生态** ：Nacos 拥有活跃的开源社区、广泛的生态整合和持续的创新发展，不仅大量兼容了 Spring Cloud、Dubbo 等大受欢迎的开源框架、还提供了丰富的插件化能力，帮助用户在云原生时代，提供可定制满足自身特殊需求的独有云原生微服务系统。

### 3. 设计理念 {#3}

> 我们相信一切都是服务，每个服务节点被构想为一个星球，每个服务都是一个星系。Nacos 致力于帮助建立这些服务之间的**连接** ，助力每个面向星辰的梦想能够透过云层，飞在云上，更好的链接整片星空。

Nacos希望帮助用户在云原生时代，在私有云、混合云或者公有云等所有云环境中，更好的构建、交付、管理自己的微服务平台，更快的复用和组合业务服务，更快的交付商业创新的价值，从而为用户赢得市场。正是基于这一愿景，Nacos的设计理念被定位为`易于使用`、`面向标准`、`高可用`和`方便扩展`。

![image.png](https://article-images.zsxq.com/FhHLKQNLcrcqSVCCoJ3GgZvlm0Ei "image.png")

#### 3.1 易于使用 {#3-1}

易于使用是 Nacos 的一个核心理念，它通过提供用户友好的 Web 界面和简洁的 API 来简化服务的注册、发现和配置管理过程。开发者可以轻松集成 Nacos 到他们的应用中，无需投入大量时间在复杂的设置和学习上。

#### 3.2 面向标准 {#3-2}

Nacos 采用面向标准的设计理念，遵循云原生应用开发的最佳实践和标准协议，以确保其服务发现和配置管理功能与广泛的技术栈和平台无缝对接。

#### 3.3 高可用 {#3-3}

为了满足企业级应用对高可用的需求，Nacos 实现了集群模式，确保在节点发生故障时，服务的发现和配置管理功能不会受影响。集群模式也意味着 Nacos 可以通过增加节点来水平扩展，提升系统的整体性能和承载能力。

![image.png](https://article-images.zsxq.com/Ful0-ca1czSHxE6seFD10ti0wuFz "image.png")

#### 3.4 方便扩展 {#3-4}

Nacos 还注重易于扩展，它采用了模块化的设计使得各个组件都可以独立地进行扩展或替换。这也为社区贡献者提供了方便，使他们能够针对特定的需求开发新的功能或者改善现有功能，进一步推动 Nacos 的生态发展。

![image.png](https://article-images.zsxq.com/FuYbQ9uL7c0Jx25R0YxEhN1wqAQX "image.png")

通过上述设计理念的实现，Nacos 为用户提供了一个强大而灵活的平台，以支持不断变化的业务需求，加速业务创新和数字转型，最终帮助用户在竞争激烈的市场中占据有利地位。

Nacos 如何完成配置监听？ {#nacos}
------------------------

由于 Nacos 的完整配置变更监听机制涉及到模板方法、观察者模式等进阶设计模式，整体流程相对复杂。为了让大家更容易理解和循序渐进地掌握相关原理，Breathe将拆分为多个章节讲解。期间如果你看到文档某些类名后缀带有 V1、V2 等字样，代表这是演进过程中的阶段性版本代码，方便大家跟进理解。
> 本章节的重点内容集中在两个模块：`common-spring-boot-starter` 和 `nacos-cloud-spring-boot-starter`。

从整体上看，动态线程池参数变更的流程其实并不复杂，可以类比为"把大象装进冰箱"的三步走：

* 1.  
**监听配置变更** ：SpringBoot 注册了监听器，一旦检测到 Nacos 配置文件发生变更，立即获取最新的配置信息；  
* 2.  
**参数绑定** ：将接收到的最新配置内容，绑定到 `BootstrapConfigProperties` 对象中，完成字符串 → Java 对象的转换；  
* 3.  
  **更新线程池** ：通过 `BootstrapConfigProperties` 与当前线程池参数进行比对，发现变更则执行更新操作，无变更则跳过，避免无意义刷新。

为了帮助大家快速理清流程，这里先放上一张**时序图** ，帮助大家对整体执行过程有个初步的理解：

![image-20250715175156406.png](https://article-images.zsxq.com/FubBL66Iesi7afmCmfqnmbK_N-ok "image-20250715175156406.png")

如果大家想要使用 Nacos，需要在 pom.xml 文件中添加对应的依赖，Nacos 在 SpringCloud 领域对应的依赖为：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
              
因为咱们在根节点进行了版本管理，所以在 `nacos-cloud-spring-boot-starter` 组件中只需要写依赖即可。

### 1. 注册 Nacos Listener {#1-nacos-listener}

从 Nacos 提供的配置中心抽象接口中，我们可以看到其核心方法之一是 `addListener`，用于注册配置变更监听器。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface ConfigService {
        // ......
        /**
         * Add a listener to the configuration, after the server modified the configuration, the client will use the
         * incoming listener callback. Recommended asynchronous processing, the application can implement the getExecutor
         * method in the ManagerListener, provide a thread pool of execution. If not provided, use the main thread callback, May
         * block other configurations or be blocked by other configurations.
         *
         * @param dataId   dataId
         * @param group    group
         * @param listener listener
         * @throws NacosException NacosException
         */
        void addListener(String dataId, String group, Listener listener) throws NacosException;
        // ......
    }
              
下面是 `addListener` 方法的注释翻译，方便大家更好理解它的作用：
> 向配置添加监听器，当服务端修改配置后，客户端会通过传入的监听器进行回调。推荐使用异步处理，应用可以在 `ManagerListener` 中实现 `getExecutor` 方法，提供一个用于执行的线程池。如果没有提供，将使用主线程进行回调，这可能会阻塞其他配置，或被其他配置阻塞。

接下来就是在什么时机进行添加监听呢？毕竟你写个添加监听方法总要被调用才能生效。这里使用 SpringBoot 扩展接口 `ApplicationRunner` 实现，一个启动时自动回调的"钩子"接口。

*  
**触发时机** ：在 Spring 容器完成初始化、所有 Bean 都加载好、并且应用准备就绪后自动调用。  
*  
  **典型用途** ：做项目启动后的预加载、注册监听器、初始化缓存、启动任务、检查配置等一切需要在启动后马上做的操作。

有同学可能会好奇："Breathe，为什么你不用 `CommandLineRunner` 呢？"其实无所谓，**在咱们这个场景下，`ApplicationRunner`和`CommandLineRunner`用哪个都行，主要看心情，效果上没有本质区别** 。

好，咱们继续往下看 ------ **来看下注册配置监听的方法是怎么实现的：**  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class NacosCloudRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final ConfigService configService;
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.NacosConfig nacosConfig = properties.getNacos();
            configService.addListener(
                    nacosConfig.getDataId(),
                    nacosConfig.getGroup(),
                    new Listener() {
    ​
                        @Override
                        public Executor getExecutor() {
                            return ThreadPoolExecutorBuilder.builder()
                                    .corePoolSize(1)
                                    .maximumPoolSize(1)
                                    .keepAliveTime(9999L)
                                    .workQueueType(BlockingQueueTypeEnum.SYNCHRONOUS_QUEUE)
                                    .threadFactory("clod-nacos-refresher-thread_")
                                    .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                                    .build();
                        }
    ​
                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            // 如果 Nacos 配置文件变更，会触发该方法进行回调
                        }
                    });
    ​
            log.info("Dynamic thread pool refresher, add nacos cloud listener success. data-id: {}, group: {}", nacosConfig.getDataId(), nacosConfig.getGroup());
        }
    }
              
方法的具体逻辑可以分为以下几个步骤：

* 1.  
**获取Nacos配置参数** ：首先，通过 `properties.getNacos()` 获取配置中心的关键参数，比如 `dataId` 和 `group`，这两个字段共同确定了唯一的配置文件位置。  
* 2.  
**注册配置变更监听器** ：接着调用 `configService.addListener(...)` 方法，向指定的 `dataId` 和 `group` 注册监听器。一旦 Nacos 端对应的配置发生变更，监听器就会被自动触发。  
* 3.  
**自定义Listener的执行线程池** ：在匿名 `Listener` 实现中，重写了 `getExecutor()` 方法，自定义了一个**单线程池** 来异步执行回调逻辑，避免阻塞 Nacos 客户端的主线程，同时也规避了并发带来的副作用。  
* 4.  
**配置变更回调** ：当配置发生变更时，`receiveConfigInfo(...)` 方法会被回调。我们通常会在这里调用 `refreshThreadPoolProperties(...)`，将最新的配置信息解析出来并动态刷新线程池参数。  
* 5.  
  **日志输出** ：为了方便后续运维排查，注册成功后会打印一条 `info` 级别的日志，明确表示监听器已经生效。

配置刷新讲得再多，不如直接看看"长什么样"。下图就是配置文件变更后的实际内容截图，配合后面的讲解一起理解会更清晰。

![image-20250715184829862.png](https://article-images.zsxq.com/FuI91qQOQK9sEZ2zbx_R7Ayp6SDM "image-20250715184829862.png")

### 2. 配置属性动态绑定 {#2}

通过前面的展示，大家应该注意到了：Nacos 在配置变更时传递给监听器的，是整个配置文件内容的字符串。由于我们配置的是 **YAML格式** ，所以这里接收到的也是一整段 YAML 字符串（如果是 `properties` 格式的配置，同理返回的就是 properties 字符串）。

但这段字符串是原始内容，没法直接用在业务逻辑里。那我们该怎么办？很简单：**将它解析为Java对象** 。在我们的项目中，就是将它转换为 `BootstrapConfigProperties` 实例，后续线程池的动态刷新才可以进行。

下面是具体的解析代码逻辑：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void refreshThreadPoolProperties(String configInfo) throws IOException {
        Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
        ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
        Binder binder = new Binder(sources);
    ​
        BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
        log.info("Latest updated configuration: \n{}", configInfo);
        log.info("Java configuration object binding: \n{}", JSON.toJSONString(refresherProperties));
    ​
        // ...... 检测线程池参数是否变更，如果已变更则进行更新
    }
              
#### 2.1 configInfo → Map：配置字符串解析 {#2-1-config-info-map}

`parseConfig` 方法将来自 Nacos 的配置内容（字符串）解析为一个扁平化的 `Map<Object, Object>`，用于后续绑定。

运行时截图如下所示：

![image-20250715190627335.png](https://article-images.zsxq.com/FpnmOiPAmPl1IrqEWfJAhJdtRMIe "image-20250715190627335.png")

配置解析器 ConfigParserHandler 方法职责大家已经清楚了，代码并不多也不难，简单用到了单例和简单工厂模式，大家可以点 Debug 看下，这里就不再赘述。

#### 2.2 Map → PropertySource：适配为 Spring 配置源 {#2-2-map-property-source-spring}

将 Map 封装为 SpringBoot 的 `ConfigurationPropertySource`，让其具备"配置绑定"的能力。该能力为 SpringBoot 原生提供的能力。

同时，创建一个配置绑定器，用于将 `PropertySource` 中的配置项绑定到 Java 对象上。`Binder` 是 SpringBoot 的底层绑定引擎，能够将配置源的数据绑定到 Java 对象中。
> 它支持默认值、嵌套对象、集合、泛型、类型转换等各种复杂配置的自动绑定逻辑，真的是强到离谱。讲个真事，最早我还不知道 SpringBoot 有 `Binder` 这个东西的时候，Hippo4j 解析字符串逻辑是自己一行行写解析逻辑，遍历配置、手动类型转换、判空......写完那段代码，我自己都觉得像是一坨。。。

写这篇文章的时候，考古了一下 Hippo4j 的早期提交记录，果然惨不忍睹。要不是 SpringBoot 给力，大家现在还真得翻我那段"遗作"调 Bug......  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                      /**
       * 此处采用硬编码适配低版本 SpringBoot 1.5.x, 如果有更好的方法进行逻辑转换的话, 欢迎 PR.
       *
       * @param configInfo
       * @return
       */
      private static BootstrapCoreProperties adapt(Map<Object, Object> configInfo) {
          BootstrapCoreProperties bindableCoreProperties;
          try {
              // filter
              Map<Object, Object> targetMap = Maps.newHashMap();
              configInfo.forEach((key, val) -> {
                  if (key != null && StringUtil.isNotBlank((String) key) && ((String) key).indexOf(PREFIX + ".executors") != -1) {
                      String targetKey = key.toString().replace(PREFIX + ".", "");
                      targetMap.put(targetKey, val);
                  }
              });
    ​
              // convert
              List<ExecutorProperties> executorPropertiesList = Lists.newArrayList();
              for (int i = 0; i < Integer.MAX_VALUE; i++) {
                  Map<String, Object> tarterSingleMap = Maps.newHashMap();
    ​
                  for (Map.Entry entry : targetMap.entrySet()) {
                      String key = entry.getKey().toString();
                      if (key.indexOf(i + "") != -1) {
                          key = key.replace("executors[" + i + "].", "");
    ​
                          String[] keySplit = key.split("-");
                          if (keySplit != null && keySplit.length > 0) {
                              key = key.replace("-", "_");
                          }
    ​
                          tarterSingleMap.put(key, entry.getValue());
                      }
                  }
    ​
                  if (CollectionUtil.isEmpty(tarterSingleMap)) {
                      break;
                  }
    ​
                  ExecutorProperties executorProperties = BeanUtil.mapToBean(tarterSingleMap, ExecutorProperties.class, true, CopyOptions.create());
                  if (executorProperties != null) {
                      executorPropertiesList.add(executorProperties);
                  }
              }
    ​
              bindableCoreProperties = new BootstrapCoreProperties();
              bindableCoreProperties.setExecutors(executorPropertiesList);
          } catch (Exception ex) {
              throw ex;
          }
    ​
          return bindableCoreProperties;
      }
              
所以，这里必须得总结一条血的教训：当你准备手撸一些复杂底层逻辑时，别急着闭门造车。因为你遇到的问题，大概率已经被无数大佬踩过坑了------关键是他们不仅踩过，还把轮子造得又稳又帅，我们直接拿来用，省时省力还更专业。

#### 2.3 bind → Java对象：绑定为配置类实例 {#2-3-bind-java}

通过 `Binder` 将配置源的内容绑定到已有的 `properties` 对象上，生成一个最新的配置实例 `refresherProperties`。其中的 `"onethread"` 是绑定的前缀，表示只会注入这个前缀下对应的配置字段。

底层原理是：Spring Boot 会使用反射机制，根据配置项自动调用 JavaObject 的 `setXxx()` 方法完成属性赋值，非常智能。

最后的 `.get()` 表示取出绑定结果，**注意** ：如果没有成功匹配配置项，这里会直接抛出异常，因此通常在生产场景中建议配合 `Optional` 或 `orElse` 进行兜底处理。

### 3. Starter 自动装配 {#3-starter}

以上就是实现 **动态线程池参数监听与刷新** 的核心逻辑。和我们之前构建 `common-spring-boot-starter` 的过程一样，这里依然遵循 **SpringBootStarter的自动装配三部曲** ：

* 1.  
编写核心业务逻辑（监听配置变化、刷新线程池参数）；  
* 2.  
编写配置类，将核心逻辑注册为 Spring Bean；  
* 3.  
  通过自动装配机制让 SpringBoot 感知并加载这些 Bean。

遵循这个套路，我们就能顺利完成基于 Nacos 的动态线程池配置监听功能。

文末总结
----

动态线程池的核心在于"动态"二字，而这个动态的起点，就是监听配置中心的变更事件。我们通过监听器拿到最新配置，解析成配置对象后，再交由后续逻辑完成参数刷新。在这个过程中，无论是 Nacos 提供的回调机制，还是我们封装的配置解析与刷新逻辑，都体现了 SpringBoot 与中间件深度集成的优势。

下一小节，我们将继续深入探索参数变更后，线程池是如何完成更新的，敬请期待。

完结，撒花 🎉  
通过 Nacos 实现线程池参数配置，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本节主要讲解了当 Nacos 配置发生变更时，监听器接收到的是完整的配置文件字符串内容（如 YAML 格式）。由于这类字符串无法直接使用，我们需要对其进行解析，转换成可操作的 Java 对象（如 `BootstrapConfigProperties`），以便后续动态刷新线程池配置。

课程目录如下所示：

*  
什么是 Nacos？  
*  
Nacos 如何完成配置监听？  
*  
  文末总结

什么是 Nacos？ {#nacos}
-------------------

> 该章节取自 [Nacos 官网](https://nacos.io/docs/v2.5/overview) Version 2.5，本章节重点使用的是 Nacos 动态配置服务，如已了解可跳过该小节。

Nacos `/nɑ:kəʊs/` 是 Dynamic Naming and Configuration Service 的首字母简称，一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台。

Nacos 致力于帮助您发现、配置和管理微服务。Nacos 提供了一组简单易用的特性集，帮助您快速实现动态服务发现、服务配置、服务元数据及流量管理。

Nacos 帮助您更敏捷和容易地构建、交付和管理微服务平台。 Nacos 是构建以\*\*"服务"\*\* 为中心的现代应用架构 (例如微服务范式、云原生范式) 的服务基础设施。

### 1. 产品功能 {#1}

#### 1.1 服务发现和服务健康监测 {#1-1}

Nacos 支持基于 DNS 和基于 RPC 的服务发现。服务提供者使用原生SDK、OpenAPI、或一个独立的 Agent 注册 Service 后，服务消费者可以使用 HTTP\&API 查找和发现服务。

Nacos 提供对服务的实时的健康检查，阻止向不健康的主机或服务实例发送请求。Nacos 支持传输层 (PING 或 TCP)和应用层 (如 HTTP、MySQL、用户自定义）的健康检查。 对于复杂的云环境和网络拓扑环境中（如 VPC、边缘网络等）服务的健康检查，Nacos 提供了 agent 上报模式和服务端主动检测2种健康检查模式。Nacos 还提供了统一的健康检查仪表盘，帮助您根据健康状态管理服务的可用性及流量。

#### 1.2 动态配置服务 {#1-2}

动态配置服务可以让您以中心化、外部化和动态化的方式管理所有环境的应用配置和服务配置。

动态配置消除了配置变更时重新部署应用和服务的需要，让配置管理变得更加高效和敏捷。

配置中心化管理让实现无状态服务变得更简单，让服务按需弹性扩展变得更容易。

Nacos 提供了一个简洁易用的UI ([控制台样例 Demo](http://console.nacos.io/index.html?spm=5238cd80.72a042d5.0.0.5bc0cd36IfxBwB)) 帮助您管理所有的服务和应用的配置。Nacos 还提供包括配置版本跟踪、金丝雀发布、一键回滚配置以及客户端配置更新状态跟踪在内的一系列开箱即用的配置管理特性，帮助您更安全地在生产环境中管理配置变更和降低配置变更带来的风险。

#### 1.3 动态 DNS 服务 {#1-3-dns}

动态 DNS 服务支持权重路由，让您更容易地实现中间层负载均衡、更灵活的路由策略、流量控制以及数据中心内网的简单DNS解析服务。动态DNS服务还能让您更容易地实现以 DNS 协议为基础的服务发现，以帮助您消除耦合到厂商私有服务发现 API 上的风险。

Nacos 提供了一些简单的 DNS APIs 帮助您管理服务的关联域名和可用的 IP 列表。

#### 1.4 服务及其元数据管理 {#1-4}

Nacos 能让您从微服务平台建设的视角管理数据中心的所有服务及元数据，包括管理服务的描述、生命周期、服务的静态依赖分析、服务的健康状态、服务的流量管理、路由及安全策略、服务的 SLA 以及最首要的 metrics 统计数据。

### 2. 产品优势 {#2}

*  
**易于使用** ：Nacos 经历几万人使用反馈优化，提供统一的服务发现和配置管理功能，通过直观的 Web 界面和简洁的 API，为开发和运维人员在云原生环境中带来了便捷的服务注册、发现和配置更新操作。  
*  
**特性丰富** ：Nacos 提供了包括服务发现、配置管理、动态 DNS 服务、服务元数据管理、流量管理、服务监控、服务治理等在内的一系列特性，帮助您在云原生时代，更轻松的构建、交付和管理微服务。  
*  
**极致性能** ：Nacos 经过阿里双十一超快伸缩场景的锤炼，提供高性能的服务注册和发现能力，以及低延迟的配置更新响应，确保在大规模分布式系统中的高效率和稳定运行。  
*  
**超大容量** ：Nacos 诞生自阿里的百万实例规模，造就支持海量服务和配置的管理，能够满足大型分布式系统对高并发和高可用性的需求。  
*  
**稳定可用** ：Nacos 通过自研的同步协议，配合生态中应用广泛的 Raft 协议，确保了服务的高可用性和数据的稳定性，保证阿里双十一系统的高可用稳定运行。  
*  
  **开放生态** ：Nacos 拥有活跃的开源社区、广泛的生态整合和持续的创新发展，不仅大量兼容了 Spring Cloud、Dubbo 等大受欢迎的开源框架、还提供了丰富的插件化能力，帮助用户在云原生时代，提供可定制满足自身特殊需求的独有云原生微服务系统。

### 3. 设计理念 {#3}

> 我们相信一切都是服务，每个服务节点被构想为一个星球，每个服务都是一个星系。Nacos 致力于帮助建立这些服务之间的**连接** ，助力每个面向星辰的梦想能够透过云层，飞在云上，更好的链接整片星空。

Nacos希望帮助用户在云原生时代，在私有云、混合云或者公有云等所有云环境中，更好的构建、交付、管理自己的微服务平台，更快的复用和组合业务服务，更快的交付商业创新的价值，从而为用户赢得市场。正是基于这一愿景，Nacos的设计理念被定位为`易于使用`、`面向标准`、`高可用`和`方便扩展`。

![image.png](https://article-images.zsxq.com/FhHLKQNLcrcqSVCCoJ3GgZvlm0Ei "image.png")

#### 3.1 易于使用 {#3-1}

易于使用是 Nacos 的一个核心理念，它通过提供用户友好的 Web 界面和简洁的 API 来简化服务的注册、发现和配置管理过程。开发者可以轻松集成 Nacos 到他们的应用中，无需投入大量时间在复杂的设置和学习上。

#### 3.2 面向标准 {#3-2}

Nacos 采用面向标准的设计理念，遵循云原生应用开发的最佳实践和标准协议，以确保其服务发现和配置管理功能与广泛的技术栈和平台无缝对接。

#### 3.3 高可用 {#3-3}

为了满足企业级应用对高可用的需求，Nacos 实现了集群模式，确保在节点发生故障时，服务的发现和配置管理功能不会受影响。集群模式也意味着 Nacos 可以通过增加节点来水平扩展，提升系统的整体性能和承载能力。

![image.png](https://article-images.zsxq.com/Ful0-ca1czSHxE6seFD10ti0wuFz "image.png")

#### 3.4 方便扩展 {#3-4}

Nacos 还注重易于扩展，它采用了模块化的设计使得各个组件都可以独立地进行扩展或替换。这也为社区贡献者提供了方便，使他们能够针对特定的需求开发新的功能或者改善现有功能，进一步推动 Nacos 的生态发展。

![image.png](https://article-images.zsxq.com/FuYbQ9uL7c0Jx25R0YxEhN1wqAQX "image.png")

通过上述设计理念的实现，Nacos 为用户提供了一个强大而灵活的平台，以支持不断变化的业务需求，加速业务创新和数字转型，最终帮助用户在竞争激烈的市场中占据有利地位。

Nacos 如何完成配置监听？ {#nacos}
------------------------

由于 Nacos 的完整配置变更监听机制涉及到模板方法、观察者模式等进阶设计模式，整体流程相对复杂。为了让大家更容易理解和循序渐进地掌握相关原理，Breathe将拆分为多个章节讲解。期间如果你看到文档某些类名后缀带有 V1、V2 等字样，代表这是演进过程中的阶段性版本代码，方便大家跟进理解。
> 本章节的重点内容集中在两个模块：`common-spring-boot-starter` 和 `nacos-cloud-spring-boot-starter`。

从整体上看，动态线程池参数变更的流程其实并不复杂，可以类比为"把大象装进冰箱"的三步走：

* 1.  
**监听配置变更** ：SpringBoot 注册了监听器，一旦检测到 Nacos 配置文件发生变更，立即获取最新的配置信息；  
* 2.  
**参数绑定** ：将接收到的最新配置内容，绑定到 `BootstrapConfigProperties` 对象中，完成字符串 → Java 对象的转换；  
* 3.  
  **更新线程池** ：通过 `BootstrapConfigProperties` 与当前线程池参数进行比对，发现变更则执行更新操作，无变更则跳过，避免无意义刷新。

为了帮助大家快速理清流程，这里先放上一张**时序图** ，帮助大家对整体执行过程有个初步的理解：

![image-20250715175156406.png](https://article-images.zsxq.com/FubBL66Iesi7afmCmfqnmbK_N-ok "image-20250715175156406.png")

如果大家想要使用 Nacos，需要在 pom.xml 文件中添加对应的依赖，Nacos 在 SpringCloud 领域对应的依赖为：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>
              
因为咱们在根节点进行了版本管理，所以在 `nacos-cloud-spring-boot-starter` 组件中只需要写依赖即可。

### 1. 注册 Nacos Listener {#1-nacos-listener}

从 Nacos 提供的配置中心抽象接口中，我们可以看到其核心方法之一是 `addListener`，用于注册配置变更监听器。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface ConfigService {
        // ......
        /**
         * Add a listener to the configuration, after the server modified the configuration, the client will use the
         * incoming listener callback. Recommended asynchronous processing, the application can implement the getExecutor
         * method in the ManagerListener, provide a thread pool of execution. If not provided, use the main thread callback, May
         * block other configurations or be blocked by other configurations.
         *
         * @param dataId   dataId
         * @param group    group
         * @param listener listener
         * @throws NacosException NacosException
         */
        void addListener(String dataId, String group, Listener listener) throws NacosException;
        // ......
    }
              
下面是 `addListener` 方法的注释翻译，方便大家更好理解它的作用：
> 向配置添加监听器，当服务端修改配置后，客户端会通过传入的监听器进行回调。推荐使用异步处理，应用可以在 `ManagerListener` 中实现 `getExecutor` 方法，提供一个用于执行的线程池。如果没有提供，将使用主线程进行回调，这可能会阻塞其他配置，或被其他配置阻塞。

接下来就是在什么时机进行添加监听呢？毕竟你写个添加监听方法总要被调用才能生效。这里使用 SpringBoot 扩展接口 `ApplicationRunner` 实现，一个启动时自动回调的"钩子"接口。

*  
**触发时机** ：在 Spring 容器完成初始化、所有 Bean 都加载好、并且应用准备就绪后自动调用。  
*  
  **典型用途** ：做项目启动后的预加载、注册监听器、初始化缓存、启动任务、检查配置等一切需要在启动后马上做的操作。

有同学可能会好奇："Breathe，为什么你不用 `CommandLineRunner` 呢？"其实无所谓，**在咱们这个场景下，`ApplicationRunner`和`CommandLineRunner`用哪个都行，主要看心情，效果上没有本质区别** 。

好，咱们继续往下看 ------ **来看下注册配置监听的方法是怎么实现的：**  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    @RequiredArgsConstructor
    public class NacosCloudRefresherHandlerV1 implements ApplicationRunner {
    ​
        private final ConfigService configService;
        private final BootstrapConfigProperties properties;
    ​
        @Override
        public void run(ApplicationArguments args) throws Exception {
            BootstrapConfigProperties.NacosConfig nacosConfig = properties.getNacos();
            configService.addListener(
                    nacosConfig.getDataId(),
                    nacosConfig.getGroup(),
                    new Listener() {
    ​
                        @Override
                        public Executor getExecutor() {
                            return ThreadPoolExecutorBuilder.builder()
                                    .corePoolSize(1)
                                    .maximumPoolSize(1)
                                    .keepAliveTime(9999L)
                                    .workQueueType(BlockingQueueTypeEnum.SYNCHRONOUS_QUEUE)
                                    .threadFactory("clod-nacos-refresher-thread_")
                                    .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                                    .build();
                        }
    ​
                        @Override
                        public void receiveConfigInfo(String configInfo) {
                            // 如果 Nacos 配置文件变更，会触发该方法进行回调
                        }
                    });
    ​
            log.info("Dynamic thread pool refresher, add nacos cloud listener success. data-id: {}, group: {}", nacosConfig.getDataId(), nacosConfig.getGroup());
        }
    }
              
方法的具体逻辑可以分为以下几个步骤：

* 1.  
**获取Nacos配置参数** ：首先，通过 `properties.getNacos()` 获取配置中心的关键参数，比如 `dataId` 和 `group`，这两个字段共同确定了唯一的配置文件位置。  
* 2.  
**注册配置变更监听器** ：接着调用 `configService.addListener(...)` 方法，向指定的 `dataId` 和 `group` 注册监听器。一旦 Nacos 端对应的配置发生变更，监听器就会被自动触发。  
* 3.  
**自定义Listener的执行线程池** ：在匿名 `Listener` 实现中，重写了 `getExecutor()` 方法，自定义了一个**单线程池** 来异步执行回调逻辑，避免阻塞 Nacos 客户端的主线程，同时也规避了并发带来的副作用。  
* 4.  
**配置变更回调** ：当配置发生变更时，`receiveConfigInfo(...)` 方法会被回调。我们通常会在这里调用 `refreshThreadPoolProperties(...)`，将最新的配置信息解析出来并动态刷新线程池参数。  
* 5.  
  **日志输出** ：为了方便后续运维排查，注册成功后会打印一条 `info` 级别的日志，明确表示监听器已经生效。

配置刷新讲得再多，不如直接看看"长什么样"。下图就是配置文件变更后的实际内容截图，配合后面的讲解一起理解会更清晰。

![image-20250715184829862.png](https://article-images.zsxq.com/FuI91qQOQK9sEZ2zbx_R7Ayp6SDM "image-20250715184829862.png")

### 2. 配置属性动态绑定 {#2}

通过前面的展示，大家应该注意到了：Nacos 在配置变更时传递给监听器的，是整个配置文件内容的字符串。由于我们配置的是 **YAML格式** ，所以这里接收到的也是一整段 YAML 字符串（如果是 `properties` 格式的配置，同理返回的就是 properties 字符串）。

但这段字符串是原始内容，没法直接用在业务逻辑里。那我们该怎么办？很简单：**将它解析为Java对象** 。在我们的项目中，就是将它转换为 `BootstrapConfigProperties` 实例，后续线程池的动态刷新才可以进行。

下面是具体的解析代码逻辑：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void refreshThreadPoolProperties(String configInfo) throws IOException {
        Map<Object, Object> configInfoMap = ConfigParserHandler.getInstance().parseConfig(configInfo, properties.getConfigFileType());
        ConfigurationPropertySource sources = new MapConfigurationPropertySource(configInfoMap);
        Binder binder = new Binder(sources);
    ​
        BootstrapConfigProperties refresherProperties = binder.bind(BootstrapConfigProperties.PREFIX, Bindable.ofInstance(properties)).get();
        log.info("Latest updated configuration: \n{}", configInfo);
        log.info("Java configuration object binding: \n{}", JSON.toJSONString(refresherProperties));
    ​
        // ...... 检测线程池参数是否变更，如果已变更则进行更新
    }
              
#### 2.1 configInfo → Map：配置字符串解析 {#2-1-config-info-map}

`parseConfig` 方法将来自 Nacos 的配置内容（字符串）解析为一个扁平化的 `Map<Object, Object>`，用于后续绑定。

运行时截图如下所示：

![image-20250715190627335.png](https://article-images.zsxq.com/FpnmOiPAmPl1IrqEWfJAhJdtRMIe "image-20250715190627335.png")

配置解析器 ConfigParserHandler 方法职责大家已经清楚了，代码并不多也不难，简单用到了单例和简单工厂模式，大家可以点 Debug 看下，这里就不再赘述。

#### 2.2 Map → PropertySource：适配为 Spring 配置源 {#2-2-map-property-source-spring}

将 Map 封装为 SpringBoot 的 `ConfigurationPropertySource`，让其具备"配置绑定"的能力。该能力为 SpringBoot 原生提供的能力。

同时，创建一个配置绑定器，用于将 `PropertySource` 中的配置项绑定到 Java 对象上。`Binder` 是 SpringBoot 的底层绑定引擎，能够将配置源的数据绑定到 Java 对象中。
> 它支持默认值、嵌套对象、集合、泛型、类型转换等各种复杂配置的自动绑定逻辑，真的是强到离谱。讲个真事，最早我还不知道 SpringBoot 有 `Binder` 这个东西的时候，Hippo4j 解析字符串逻辑是自己一行行写解析逻辑，遍历配置、手动类型转换、判空......写完那段代码，我自己都觉得像是一坨。。。

写这篇文章的时候，考古了一下 Hippo4j 的早期提交记录，果然惨不忍睹。要不是 SpringBoot 给力，大家现在还真得翻我那段"遗作"调 Bug......  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                      /**
       * 此处采用硬编码适配低版本 SpringBoot 1.5.x, 如果有更好的方法进行逻辑转换的话, 欢迎 PR.
       *
       * @param configInfo
       * @return
       */
      private static BootstrapCoreProperties adapt(Map<Object, Object> configInfo) {
          BootstrapCoreProperties bindableCoreProperties;
          try {
              // filter
              Map<Object, Object> targetMap = Maps.newHashMap();
              configInfo.forEach((key, val) -> {
                  if (key != null && StringUtil.isNotBlank((String) key) && ((String) key).indexOf(PREFIX + ".executors") != -1) {
                      String targetKey = key.toString().replace(PREFIX + ".", "");
                      targetMap.put(targetKey, val);
                  }
              });
    ​
              // convert
              List<ExecutorProperties> executorPropertiesList = Lists.newArrayList();
              for (int i = 0; i < Integer.MAX_VALUE; i++) {
                  Map<String, Object> tarterSingleMap = Maps.newHashMap();
    ​
                  for (Map.Entry entry : targetMap.entrySet()) {
                      String key = entry.getKey().toString();
                      if (key.indexOf(i + "") != -1) {
                          key = key.replace("executors[" + i + "].", "");
    ​
                          String[] keySplit = key.split("-");
                          if (keySplit != null && keySplit.length > 0) {
                              key = key.replace("-", "_");
                          }
    ​
                          tarterSingleMap.put(key, entry.getValue());
                      }
                  }
    ​
                  if (CollectionUtil.isEmpty(tarterSingleMap)) {
                      break;
                  }
    ​
                  ExecutorProperties executorProperties = BeanUtil.mapToBean(tarterSingleMap, ExecutorProperties.class, true, CopyOptions.create());
                  if (executorProperties != null) {
                      executorPropertiesList.add(executorProperties);
                  }
              }
    ​
              bindableCoreProperties = new BootstrapCoreProperties();
              bindableCoreProperties.setExecutors(executorPropertiesList);
          } catch (Exception ex) {
              throw ex;
          }
    ​
          return bindableCoreProperties;
      }
              
所以，这里必须得总结一条血的教训：当你准备手撸一些复杂底层逻辑时，别急着闭门造车。因为你遇到的问题，大概率已经被无数大佬踩过坑了------关键是他们不仅踩过，还把轮子造得又稳又帅，我们直接拿来用，省时省力还更专业。

#### 2.3 bind → Java对象：绑定为配置类实例 {#2-3-bind-java}

通过 `Binder` 将配置源的内容绑定到已有的 `properties` 对象上，生成一个最新的配置实例 `refresherProperties`。其中的 `"onethread"` 是绑定的前缀，表示只会注入这个前缀下对应的配置字段。

底层原理是：Spring Boot 会使用反射机制，根据配置项自动调用 JavaObject 的 `setXxx()` 方法完成属性赋值，非常智能。

最后的 `.get()` 表示取出绑定结果，**注意** ：如果没有成功匹配配置项，这里会直接抛出异常，因此通常在生产场景中建议配合 `Optional` 或 `orElse` 进行兜底处理。

### 3. Starter 自动装配 {#3-starter}

以上就是实现 **动态线程池参数监听与刷新** 的核心逻辑。和我们之前构建 `common-spring-boot-starter` 的过程一样，这里依然遵循 **SpringBootStarter的自动装配三部曲** ：

* 1.  
编写核心业务逻辑（监听配置变化、刷新线程池参数）；  
* 2.  
编写配置类，将核心逻辑注册为 Spring Bean；  
* 3.  
  通过自动装配机制让 SpringBoot 感知并加载这些 Bean。

遵循这个套路，我们就能顺利完成基于 Nacos 的动态线程池配置监听功能。

文末总结
----

动态线程池的核心在于"动态"二字，而这个动态的起点，就是监听配置中心的变更事件。我们通过监听器拿到最新配置，解析成配置对象后，再交由后续逻辑完成参数刷新。在这个过程中，无论是 Nacos 提供的回调机制，还是我们封装的配置解析与刷新逻辑，都体现了 SpringBoot 与中间件深度集成的优势。

下一小节，我们将继续深入探索参数变更后，线程池是如何完成更新的，敬请期待。

完结，撒花 🎉

