2025年07月07日 21:46  
如何设计oneThread动态线程池？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本文系统解析了线程池管理的五大核心痛点，通过 oneThread 框架的创新设计，提出资源统一注册、动态参数热更新、三重告警维度、Prometheus+Grafana 可视化监控及优雅关闭机制等解决方案，全面提升系统稳定性和运维效率。另外，深度剖析模块化开发经验，揭示 Spring Boot 轻量化集成的关键实践。

课程目录如下所示：

*  
背景概要  
*  
聚焦线程池问题  
*  
线程池问题解决思路  
*  
基础组件库开发经验  
*  
  文末总结

背景概要
----

很多人阅读开源框架时，容易陷入"一行行扒源码、一个类一个类追调用栈"的细节漩涡，结果既忘了入口，也说不清框架到底「解决了什么、靠什么机制解决」。因此，Breathe在 oneThread 的开篇选择先讲"如何设计动态线程池"，而不是直接剖析源码：先把要解决的问题、核心思路和整体架构讲清楚，再带着全局视角去看代码，才能读得快、记得牢。

**阅读源码的正确姿势应是：**

* 1.  
**先聚焦问题** → **梳理整体设计** → **划定模块边界** → **弄清关键机制**  
* 2.  
  在此基础上，再深入到具体实现，逐步拆解数据结构、并发细节和异常分支。

> 推荐参考三友大佬的文章[《如何去阅读源码，我总结了18条心法》](https://mp.weixin.qq.com/s/kYmZYyaKG_4EJ8ya_0qbIw)，我觉得挺中肯的，推荐大家学习。

聚焦线程池问题
-------

我们再来看一遍 JDK 线程池的问题，这些问题与业务规模无关------**只要你用线程池，就势必会遇到这几道问题**。  

|       痛点        |    本质归因     |              oneThread 设计指针              |
|-----------------|-------------|------------------------------------------|
| 线程池随意 new，资源失控  | 缺乏统一注册表     | **线程池注册中心** + 统一管控                       |
| 参数难估算、只能重发      | 静态配置 \& 无监控 | **运行中热刷新** + **实时指标采集**                  |
| 队列堵塞 / 拒绝策略"黑盒" | 无告警 \& 无追踪  | **三重告警触发器** （活跃度 / 队列 / 拒绝）              |
| 下线时任务丢失         | 线程池生命周期缺口   | **优雅关闭 Hook** (`awaitTerminationMillis`) |

这里给大家放一张之前画的架构图，是一种宏观角度上的解决方案，大家简单看看，下面会具体说明。

![image-20250626140245388.png](https://article-images.zsxq.com/FhFm8y2CCuum97fGvPbVCcvEF3l4)

线程池问题解决思路
---------

### 1. 线程池资源管理 {#1}

随着业务迭代，开发人员各自「就地 new」线程池，许多功能重复或语义相近的线程池不断堆积，结果就是**资源被悄悄蚕食**。

我们的做法是把线程池的声明全部**收敛到配置中心**：先在配置中心登记，再由应用按需装配。这样不仅能一眼看到每个线程池的参数与使用场景，还能快速判断是否可复用，从源头上杜绝盲目新建、浪费服务器资源。

以下内容是 Nacos 等配置中心的线程池配置清单。**项目统一约定：如需新增线程池，必须先在配置中心完成登记**，再由应用自动装配，确保规范一致、可追溯。  
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
              
### 2. 线程池参数动态变更 {#2}

大家都知道，如果要修改运行中应用线程池参数，需要停止线上应用，调整成功后再发布，而这个过程异常的繁琐，如果能在运行中动态调整线程池的参数多好。

众所周知，Nacos 既是配置中心也是注册中心。**只要把线程池参数集中存放在 Nacos** ，Spring Boot 客户端即可持续监听。当检测到线程池配置有更新时，立即拉取最新参数并触发**动态线程池刷新**流程，做到配置一改、线上秒生效。

![iShot_2024-08-16_18.10.555.png](https://article-images.zsxq.com/FlVY5exEmd3RWhG13aroJkoUXf8t)

看起来只是"监听-刷新"，真正落地却有两座大山：

* 1.  
**YAML → Java 映射**：Spring Boot 监听器拿到的仅是一段纯 YAML 字符串------如何优雅地反序列化成线程池配置对象？  
* 2.  
  **多配置中心的代码复用**：作为通用组件，我们势必要同时适配 N 个配置中心。怎样抽象公共逻辑，既避免 if-else 轰炸，又能随时 plug-in 新配置中心？

先把这两个疑问放在脑海里，下面的方案章节将逐一拆解。

### 3. 运行时通知告警 {#3}

如果让大家来设计线程池告警，会关注哪些维度？oneThread 目前提炼出三条"高命中"告警策略，并给出默认阈值与触发逻辑：  

|    维度    |                      触发条件                      |            检测含义             |
|----------|------------------------------------------------|-----------------------------|
| **活跃度**  | `activeCount / maximumPoolSize` 连续高于阈值（默认 80%） | 线程资源已逼近瓶颈，需扩容或对入口流量做限流      |
| **队列负载** | `queueSize / queueCapacity` 超过阈值               | 排队任务激增，处理能力被入口流量压制，易引发大面积超时 |
| **拒绝异常** | 监控到新的 `RejectedExecutionException`             | 线程池已无法接收新任务，属于阻断场景，应立刻介入    |

活跃度和队列负载的监控规则较为简单，通过定时任务扫描即可实现。不过需要注意的是，定时任务的执行间隔需合理设置：过短会因监控 API 加锁导致与线程池其他操作竞争锁资源，过长则可能错过重要的告警时机。oneThread 在充分权衡后，默认将扫描间隔设置为 **5 秒**。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 检查线程活跃度（活跃线程数 / 最大线程数）
     */
    private void checkActiveRate(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties properties = holder.getExecutorProperties();
    ​
        int activeCount = executor.getActiveCount(); // API 有锁，避免高频率调用
        int maximumPoolSize = executor.getMaximumPoolSize();
    ​
        if (maximumPoolSize == 0) {
            return;
        }
    ​
        int activeRate = (int) Math.round((activeCount * 100.0) / maximumPoolSize);
        int threshold = properties.getAlarm().getActiveThreshold();
    ​
        if (activeRate >= threshold) {
            sendAlarmMessage("Activity", holder);
        }
    }
              
线程池获取活跃线程方法源代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public int getActiveCount() {
        // 获取了线程池核心主锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            int n = 0;
            for (Worker w : workers)
                if (w.isLocked())
                    ++n;
            return n;
        } finally {
            mainLock.unlock();
        }
    }
              
拒绝策略的告警机制是通过动态代理实现的：每当线程池触发一次拒绝策略，对应的计数器就自增一次。在定时任务扫描时，会比较当前的拒绝次数与上次扫描记录的次数，如果出现新增，则立即触发告警。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final Map<String, Long> lastRejectCountMap = new ConcurrentHashMap<>();
    ​
    /**
     * 检查拒绝策略执行次数
     */
    private void checkRejectCount(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        String threadPoolId = holder.getThreadPoolId();
    ​
        // 只处理自定义线程池类型
        if (!(executor instanceof OneThreadExecutor)) {
            return;
        }
    ​
        OneThreadExecutor oneThreadExecutor = (OneThreadExecutor) executor;
        long currentRejectCount = oneThreadExecutor.getRejectCount().get();
        long lastRejectCount = lastRejectCountMap.getOrDefault(threadPoolId, 0L);
    ​
        // 首次初始化或拒绝次数增加时触发
        if (currentRejectCount > lastRejectCount) {
            sendAlarmMessage("Reject", holder);
            // 更新最后记录值
            lastRejectCountMap.put(threadPoolId, currentRejectCount);
        }
    }
              
### 4. 线程池运行监控 {#4}

线程池监控并非锦上添花，而是高并发应用的核心基础设施之一。日常开发中，线程池虽然提升了系统性能，但隐藏的风险却不容忽视：

*  
线程池任务排队过长、拒绝任务时，业务可能出现大面积超时甚至瘫痪，但如果缺乏监控机制，开发人员很难在故障初期发现问题根源。  
*  
监控能及时识别异常，帮助技术人员第一时间介入处理，降低业务受损程度。  
*  
  未经监控的线程池往往存在资源浪费（线程数过多）或性能瓶颈（线程数过少）。

明确线程池监控的核心需求，一般分为：  

|     指标     |      需求点      |   需要程度   |
|------------|---------------|----------|
| **活跃线程数**  | 判断线程池是否逼近性能瓶颈 | ⭐️⭐️     |
| **队列负载**   | 识别任务堆积风险      | ⭐️⭐️⭐️   |
| **拒绝策略触发** | 及时发现线程池溢出异常   | ⭐️⭐️⭐️⭐️ |

> 通常『拒绝策略』相对更复杂，因为需要额外设计机制（如动态代理等）进行统计。

在当前主流的监控中间件中，**Prometheus + Grafana** 是一套成熟、易用且稳定的组合方案，广泛应用于各类系统的指标监控场景。选择它们作为线程池监控的基础，主要基于以下考虑：

*  
**Prometheus** 采用主动拉取（Pull）模型，能够定时抓取线程池的核心运行指标，如活跃线程数、队列长度、拒绝次数等。同时内置时序数据库，部署简单、性能可靠，已在大量场景中验证其稳定性。  
*  
  **Grafana** 与 Prometheus 深度集成，配置数据源即可使用，支持丰富的图表类型和灵活的仪表盘定制，能够清晰展示线程池的运行状态与历史趋势，便于问题定位与运维决策。

通过 Prometheus 采集与存储指标，配合 Grafana 的可视化能力，构建完整的线程池监控体系，既适用于开发调试，也适用于生产环境的持续观测。

![image-20250707195417499.png](https://article-images.zsxq.com/FvlElNmBHQ3wy-yOGaeoNfvKXhAI)

### 5. 优雅关闭防任务丢失 {#5}

如果仅按示例代码直接创建并使用线程池，而 **没有补充任何优雅停机逻辑**，一旦应用关闭或重启，线程池中尚未处理完的任务必然丢失。原因在于：应用停机流程里缺少"兜底"步骤------先检测线程池是否仍有待执行任务，再在设定的宽限期内等待其完成。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Service
    public class ThreadPoolHookTest {
    ​
        private ThreadPoolExecutor executor = ThreadPoolExecutorBuilder.builder()
                .corePoolSize(2)
                .maximumPoolSize(4)
                .keepAliveTime(9999L)
                .awaitTerminationMillis(5000L)
                .workQueueType(BlockingQueueTypeEnum.LINKED_BLOCKING_QUEUE)
                .threadFactory("onethread-producer_")
                .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                .build();
        
        public void sendMessage() {
            // xxx
            executor.execute(() -> System.out.println("Test..."));
        }
    }
              
这种兜底通常依赖 **Shutdown Hook**：当进程收到停止信号时，系统会调用预先注册的 Hook，在真正退出前执行自定义清理代码。若未注册 Hook，线程池会被立即终止，残余任务也随之丢弃。

Hook 在当前应用层面可以分为三类：  

|          层次          |                   触发方式                   |                典型用法                |
|----------------------|------------------------------------------|------------------------------------|
| **操作系统信号**           | `SIGTERM` / `SIGINT` / `SIGKILL`         | 容器编排、CI/CD 滚动发布会发 `SIGTERM` 通知应用退出 |
| **JVM ShutdownHook** | `Runtime.getRuntime().addShutdownHook()` | 最后时刻执行 Java 线程，处理自定义清理逻辑           |
| **框架 / 容器 Hook**     | Spring、Tomcat、Netty 等框架生命周期回调            | 在 **Bean** 级别或 **组件** 级别做收尾工作      |

这三层 Hook **自上而下串联** ：`SIGTERM` → JVM 捕获并触发 ShutdownHook → Spring 发布 `ContextClosedEvent` → Bean 销毁回调。

在 **oneThread** 中，我们利用 **Spring Bean 级 Hook** 完成线程池的**任务检测与等待** ：收到停机信号后，先检查线程池中是否仍有未完成任务，再在设定的宽限期内调用 `awaitTermination`，确保任务尽量执行完毕后再退出。

![image-20250707210417370.png](https://article-images.zsxq.com/Fm-INtBKEd5KysITCNoV7suKzApC)

基础组件开发经验
--------

很多同学都会问我：Breathe，为什么 oneThread 要拆成这么多 Module，干脆做成一个"大而全"的 Starter 不就省事了吗？  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    ├── core # 动态线程池核心模块包，实现动态线程池相关基础类定义
    ├── dashboard-dev # 应广大同学要求，以后每个项目尽量有前端页面方便查看和调试
    ├── example # 动态线程池示例包，演示线程池动态参数变更、监控和告警等功能
    │   ├── apollo-example
    │   └── nacos-cloud-example
    ├── spring-base # 动态线程池基础模块包，包含Spring扫描动态线程池、是否启用以及Banner打印等
    └── starter # 动态线程池配置中心组件包，实现线程池结合Spring框架和配置中心动态刷新
        ├── adapter # 动态线程池适配层，比如对接 Web 容器 Tomcat 线程池等
        │   └── web-spring-boot-starter # Web 容器线程池组件库
        ├── apollo-spring-boot-starter # Apollo 配置中心动态监控线程池组件库
        ├── common-spring-boot-starter # 配置中心公共监听等逻辑抽象组件库
        ├── dashboard-dev-spring-boot-starter # 控制台 API 组件库
        └── nacos-cloud-spring-boot-starter # Nacos 配置中心动态监控线程池组件库
              
Starter 的确能让 Spring Boot 项目"一键启用"，但它只服务于 Boot。oneThread 想覆盖的不止是 Boot 应用，还包括普通 Spring 项目，甚至纯 Java 程序。如果把所有功能都塞进 Starter，非 Boot 场景根本没法无缝接入。

oneThread 的做法是把核心能力放进 `onethread-core`，Spring 扩展放进 `onethread-spring`，Boot 自动装配再用一个很薄的 `onethread-starter`。这样纯 Java 项目只引 core，Spring 项目加 spring，Spring Boot 项目再多一个 starter，就都能跑起来。  

|         项目类型          |            应该引入的模块             |
|-----------------------|--------------------------------|
| **纯 Java / 非 Spring** | `onethread-core`               |
| **Spring (非 Boot)**   | `onethread-spring`             |
| **Spring Boot**       | `onethread-starter`（很薄，仅做自动装配） |

如果你在非 Boot 环境里硬拉一个 Starter，然后自己写启动逻辑去"模拟"自动装配，会踩两大坑：其一，Boot 独有的条件注解和配置绑定在普通 Spring 根本解析不了；其二，Starter 写得越重，耦合就越深，想自定义启动点几乎没有空间。

最好的例子就是 XXL-Job，它只给了一个 core 包，任何项目只要 new 一下 `XxlJobSpringExecutor`，注册成 Bean 就能用。如果 XXL-Job 将复杂初始化逻辑（假设有）当初全都写在 Starter 里，非 Boot 项目就得二次开发才能接入。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(applicationName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(StrUtil.isNotEmpty(accessToken) ? accessToken : null);
        xxlJobSpringExecutor.setLogPath(StrUtil.isNotEmpty(logPath) ? logPath : Paths.get("").toAbsolutePath().getParent() + "/tmp");
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
              
还有版本升级的隐患。我经历过 Boot 1.5 → 2.x → 3.x，每次 API 都大改。如果核心代码强依赖 Boot，就不得不维护 N 套实现。oneThread 把 Starter 做得极薄，就是为了在 Boot 下一次大升级时，只动最外面那层皮，核心逻辑几乎不用碰。

所以，模块拆分不是折腾，而是为了兼容更多项目形态，降低升级成本，并保持架构的简洁度。真正把所有东西塞进一个胖 Starter，短期看似方便，长期一定埋坑。
> 在 Hippo4j 的实践中，我深刻体会到"深度绑定 Spring Boot"带来的痛点：版本升级一动就牵一发而动全身，依赖矩阵异常复杂。因此，后续做开源框架时，我更倾向**只依赖 Spring（或者不强依赖）** 而非 Spring Boot------第三方依赖越少，架构就越纯净、可维护性也越高。

对于 **面向开源社区的基础组件库** ，代码必须足够通用：既要兼容不同 Spring/SpringBoot 版本，又要尽量减少三方依赖才能让别人无缝集成。因此，**核心逻辑与框架集成必须彻底解耦**，Starter 只做薄薄一层自动装配。

而 **公司内部的基础组件** 场景要简单得多：运行环境、Spring Boot 版本乃至依赖清单都高度一致。如果不考虑对外复用，做成一个"大而全"的内部 Starter 也未尝不可。

不过，一旦企业内部开始出现 **多条技术栈并行**（例如新旧项目 Boot 版本不一致），还是建议把核心能力抽离出来，Starter 只负责适配各版本 ------ 这样既能降低升级成本，也为未来的开源或跨团队复用留下余地。

文末总结
----

线程池用不好真挺闹心：一不小心线程开多了服务器扛不住，开少了任务又堵着干不完；出问题了不知道咋回事，关了服务还可能丢任务。

**oneThread 就是来搞定这些的：**

* 1.  
**统一管**：大家别乱搞线程池了，先去配置中心登记一下，省得搞出好多重复的浪费资源。  
* 2.  
**灵活调**：参数不用担心设置不好了！线上跑着也能随时调整核心线程数、队列大小这些，灵活得很。  
* 3.  
**及时报**：快撑不住（线程太忙/队列快满/任务被拒）了，马上发消息告诉你，及时处理不误事。  
* 4.  
**看得清**：线程池里面啥情况（有多少线程在忙、排了多少队、被拒绝多少活），整得明明白白展示出来。  
* 5.  
  **安心关**：关服务也不怕丢任务，设定一个等待时间，让还在跑的活尽量搞定走人。

**为啥拆好几个包（core,spring,starter）？** 就为了让不管是纯 Java 项目、老 Spring 项目、还是 SpringBoot 项目，都能方便地用上，哪部分合适就装哪部分，不添堵。这种设计升级也方便点，亲测靠谱。

完结，撒花 🎉  
如何设计oneThread动态线程池？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本文系统解析了线程池管理的五大核心痛点，通过 oneThread 框架的创新设计，提出资源统一注册、动态参数热更新、三重告警维度、Prometheus+Grafana 可视化监控及优雅关闭机制等解决方案，全面提升系统稳定性和运维效率。另外，深度剖析模块化开发经验，揭示 Spring Boot 轻量化集成的关键实践。

课程目录如下所示：

*  
背景概要  
*  
聚焦线程池问题  
*  
线程池问题解决思路  
*  
基础组件库开发经验  
*  
  文末总结

背景概要
----

很多人阅读开源框架时，容易陷入"一行行扒源码、一个类一个类追调用栈"的细节漩涡，结果既忘了入口，也说不清框架到底「解决了什么、靠什么机制解决」。因此，Breathe在 oneThread 的开篇选择先讲"如何设计动态线程池"，而不是直接剖析源码：先把要解决的问题、核心思路和整体架构讲清楚，再带着全局视角去看代码，才能读得快、记得牢。

**阅读源码的正确姿势应是：**

* 1.  
**先聚焦问题** → **梳理整体设计** → **划定模块边界** → **弄清关键机制**  
* 2.  
  在此基础上，再深入到具体实现，逐步拆解数据结构、并发细节和异常分支。

> 推荐参考三友大佬的文章[《如何去阅读源码，我总结了18条心法》](https://mp.weixin.qq.com/s/kYmZYyaKG_4EJ8ya_0qbIw)，我觉得挺中肯的，推荐大家学习。

聚焦线程池问题
-------

我们再来看一遍 JDK 线程池的问题，这些问题与业务规模无关------**只要你用线程池，就势必会遇到这几道问题**。  

|       痛点        |    本质归因     |              oneThread 设计指针              |
|-----------------|-------------|------------------------------------------|
| 线程池随意 new，资源失控  | 缺乏统一注册表     | **线程池注册中心** + 统一管控                       |
| 参数难估算、只能重发      | 静态配置 \& 无监控 | **运行中热刷新** + **实时指标采集**                  |
| 队列堵塞 / 拒绝策略"黑盒" | 无告警 \& 无追踪  | **三重告警触发器** （活跃度 / 队列 / 拒绝）              |
| 下线时任务丢失         | 线程池生命周期缺口   | **优雅关闭 Hook** (`awaitTerminationMillis`) |

这里给大家放一张之前画的架构图，是一种宏观角度上的解决方案，大家简单看看，下面会具体说明。

![image-20250626140245388.png](https://article-images.zsxq.com/FhFm8y2CCuum97fGvPbVCcvEF3l4)

线程池问题解决思路
---------

### 1. 线程池资源管理 {#1}

随着业务迭代，开发人员各自「就地 new」线程池，许多功能重复或语义相近的线程池不断堆积，结果就是**资源被悄悄蚕食**。

我们的做法是把线程池的声明全部**收敛到配置中心**：先在配置中心登记，再由应用按需装配。这样不仅能一眼看到每个线程池的参数与使用场景，还能快速判断是否可复用，从源头上杜绝盲目新建、浪费服务器资源。

以下内容是 Nacos 等配置中心的线程池配置清单。**项目统一约定：如需新增线程池，必须先在配置中心完成登记**，再由应用自动装配，确保规范一致、可追溯。  
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
              
### 2. 线程池参数动态变更 {#2}

大家都知道，如果要修改运行中应用线程池参数，需要停止线上应用，调整成功后再发布，而这个过程异常的繁琐，如果能在运行中动态调整线程池的参数多好。

众所周知，Nacos 既是配置中心也是注册中心。**只要把线程池参数集中存放在 Nacos** ，Spring Boot 客户端即可持续监听。当检测到线程池配置有更新时，立即拉取最新参数并触发**动态线程池刷新**流程，做到配置一改、线上秒生效。

![iShot_2024-08-16_18.10.555.png](https://article-images.zsxq.com/FlVY5exEmd3RWhG13aroJkoUXf8t)

看起来只是"监听-刷新"，真正落地却有两座大山：

* 1.  
**YAML → Java 映射**：Spring Boot 监听器拿到的仅是一段纯 YAML 字符串------如何优雅地反序列化成线程池配置对象？  
* 2.  
  **多配置中心的代码复用**：作为通用组件，我们势必要同时适配 N 个配置中心。怎样抽象公共逻辑，既避免 if-else 轰炸，又能随时 plug-in 新配置中心？

先把这两个疑问放在脑海里，下面的方案章节将逐一拆解。

### 3. 运行时通知告警 {#3}

如果让大家来设计线程池告警，会关注哪些维度？oneThread 目前提炼出三条"高命中"告警策略，并给出默认阈值与触发逻辑：  

|    维度    |                      触发条件                      |            检测含义             |
|----------|------------------------------------------------|-----------------------------|
| **活跃度**  | `activeCount / maximumPoolSize` 连续高于阈值（默认 80%） | 线程资源已逼近瓶颈，需扩容或对入口流量做限流      |
| **队列负载** | `queueSize / queueCapacity` 超过阈值               | 排队任务激增，处理能力被入口流量压制，易引发大面积超时 |
| **拒绝异常** | 监控到新的 `RejectedExecutionException`             | 线程池已无法接收新任务，属于阻断场景，应立刻介入    |

活跃度和队列负载的监控规则较为简单，通过定时任务扫描即可实现。不过需要注意的是，定时任务的执行间隔需合理设置：过短会因监控 API 加锁导致与线程池其他操作竞争锁资源，过长则可能错过重要的告警时机。oneThread 在充分权衡后，默认将扫描间隔设置为 **5 秒**。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 检查线程活跃度（活跃线程数 / 最大线程数）
     */
    private void checkActiveRate(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties properties = holder.getExecutorProperties();
    ​
        int activeCount = executor.getActiveCount(); // API 有锁，避免高频率调用
        int maximumPoolSize = executor.getMaximumPoolSize();
    ​
        if (maximumPoolSize == 0) {
            return;
        }
    ​
        int activeRate = (int) Math.round((activeCount * 100.0) / maximumPoolSize);
        int threshold = properties.getAlarm().getActiveThreshold();
    ​
        if (activeRate >= threshold) {
            sendAlarmMessage("Activity", holder);
        }
    }
              
线程池获取活跃线程方法源代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public int getActiveCount() {
        // 获取了线程池核心主锁
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            int n = 0;
            for (Worker w : workers)
                if (w.isLocked())
                    ++n;
            return n;
        } finally {
            mainLock.unlock();
        }
    }
              
拒绝策略的告警机制是通过动态代理实现的：每当线程池触发一次拒绝策略，对应的计数器就自增一次。在定时任务扫描时，会比较当前的拒绝次数与上次扫描记录的次数，如果出现新增，则立即触发告警。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final Map<String, Long> lastRejectCountMap = new ConcurrentHashMap<>();
    ​
    /**
     * 检查拒绝策略执行次数
     */
    private void checkRejectCount(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        String threadPoolId = holder.getThreadPoolId();
    ​
        // 只处理自定义线程池类型
        if (!(executor instanceof OneThreadExecutor)) {
            return;
        }
    ​
        OneThreadExecutor oneThreadExecutor = (OneThreadExecutor) executor;
        long currentRejectCount = oneThreadExecutor.getRejectCount().get();
        long lastRejectCount = lastRejectCountMap.getOrDefault(threadPoolId, 0L);
    ​
        // 首次初始化或拒绝次数增加时触发
        if (currentRejectCount > lastRejectCount) {
            sendAlarmMessage("Reject", holder);
            // 更新最后记录值
            lastRejectCountMap.put(threadPoolId, currentRejectCount);
        }
    }
              
### 4. 线程池运行监控 {#4}

线程池监控并非锦上添花，而是高并发应用的核心基础设施之一。日常开发中，线程池虽然提升了系统性能，但隐藏的风险却不容忽视：

*  
线程池任务排队过长、拒绝任务时，业务可能出现大面积超时甚至瘫痪，但如果缺乏监控机制，开发人员很难在故障初期发现问题根源。  
*  
监控能及时识别异常，帮助技术人员第一时间介入处理，降低业务受损程度。  
*  
  未经监控的线程池往往存在资源浪费（线程数过多）或性能瓶颈（线程数过少）。

明确线程池监控的核心需求，一般分为：  

|     指标     |      需求点      |   需要程度   |
|------------|---------------|----------|
| **活跃线程数**  | 判断线程池是否逼近性能瓶颈 | ⭐️⭐️     |
| **队列负载**   | 识别任务堆积风险      | ⭐️⭐️⭐️   |
| **拒绝策略触发** | 及时发现线程池溢出异常   | ⭐️⭐️⭐️⭐️ |

> 通常『拒绝策略』相对更复杂，因为需要额外设计机制（如动态代理等）进行统计。

在当前主流的监控中间件中，**Prometheus + Grafana** 是一套成熟、易用且稳定的组合方案，广泛应用于各类系统的指标监控场景。选择它们作为线程池监控的基础，主要基于以下考虑：

*  
**Prometheus** 采用主动拉取（Pull）模型，能够定时抓取线程池的核心运行指标，如活跃线程数、队列长度、拒绝次数等。同时内置时序数据库，部署简单、性能可靠，已在大量场景中验证其稳定性。  
*  
  **Grafana** 与 Prometheus 深度集成，配置数据源即可使用，支持丰富的图表类型和灵活的仪表盘定制，能够清晰展示线程池的运行状态与历史趋势，便于问题定位与运维决策。

通过 Prometheus 采集与存储指标，配合 Grafana 的可视化能力，构建完整的线程池监控体系，既适用于开发调试，也适用于生产环境的持续观测。

![image-20250707195417499.png](https://article-images.zsxq.com/FvlElNmBHQ3wy-yOGaeoNfvKXhAI)

### 5. 优雅关闭防任务丢失 {#5}

如果仅按示例代码直接创建并使用线程池，而 **没有补充任何优雅停机逻辑**，一旦应用关闭或重启，线程池中尚未处理完的任务必然丢失。原因在于：应用停机流程里缺少"兜底"步骤------先检测线程池是否仍有待执行任务，再在设定的宽限期内等待其完成。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Service
    public class ThreadPoolHookTest {
    ​
        private ThreadPoolExecutor executor = ThreadPoolExecutorBuilder.builder()
                .corePoolSize(2)
                .maximumPoolSize(4)
                .keepAliveTime(9999L)
                .awaitTerminationMillis(5000L)
                .workQueueType(BlockingQueueTypeEnum.LINKED_BLOCKING_QUEUE)
                .threadFactory("onethread-producer_")
                .rejectedHandler(new ThreadPoolExecutor.CallerRunsPolicy())
                .build();
        
        public void sendMessage() {
            // xxx
            executor.execute(() -> System.out.println("Test..."));
        }
    }
              
这种兜底通常依赖 **Shutdown Hook**：当进程收到停止信号时，系统会调用预先注册的 Hook，在真正退出前执行自定义清理代码。若未注册 Hook，线程池会被立即终止，残余任务也随之丢弃。

Hook 在当前应用层面可以分为三类：  

|          层次          |                   触发方式                   |                典型用法                |
|----------------------|------------------------------------------|------------------------------------|
| **操作系统信号**           | `SIGTERM` / `SIGINT` / `SIGKILL`         | 容器编排、CI/CD 滚动发布会发 `SIGTERM` 通知应用退出 |
| **JVM ShutdownHook** | `Runtime.getRuntime().addShutdownHook()` | 最后时刻执行 Java 线程，处理自定义清理逻辑           |
| **框架 / 容器 Hook**     | Spring、Tomcat、Netty 等框架生命周期回调            | 在 **Bean** 级别或 **组件** 级别做收尾工作      |

这三层 Hook **自上而下串联** ：`SIGTERM` → JVM 捕获并触发 ShutdownHook → Spring 发布 `ContextClosedEvent` → Bean 销毁回调。

在 **oneThread** 中，我们利用 **Spring Bean 级 Hook** 完成线程池的**任务检测与等待** ：收到停机信号后，先检查线程池中是否仍有未完成任务，再在设定的宽限期内调用 `awaitTermination`，确保任务尽量执行完毕后再退出。

![image-20250707210417370.png](https://article-images.zsxq.com/Fm-INtBKEd5KysITCNoV7suKzApC)

基础组件开发经验
--------

很多同学都会问我：Breathe，为什么 oneThread 要拆成这么多 Module，干脆做成一个"大而全"的 Starter 不就省事了吗？  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    ├── core # 动态线程池核心模块包，实现动态线程池相关基础类定义
    ├── dashboard-dev # 应广大同学要求，以后每个项目尽量有前端页面方便查看和调试
    ├── example # 动态线程池示例包，演示线程池动态参数变更、监控和告警等功能
    │   ├── apollo-example
    │   └── nacos-cloud-example
    ├── spring-base # 动态线程池基础模块包，包含Spring扫描动态线程池、是否启用以及Banner打印等
    └── starter # 动态线程池配置中心组件包，实现线程池结合Spring框架和配置中心动态刷新
        ├── adapter # 动态线程池适配层，比如对接 Web 容器 Tomcat 线程池等
        │   └── web-spring-boot-starter # Web 容器线程池组件库
        ├── apollo-spring-boot-starter # Apollo 配置中心动态监控线程池组件库
        ├── common-spring-boot-starter # 配置中心公共监听等逻辑抽象组件库
        ├── dashboard-dev-spring-boot-starter # 控制台 API 组件库
        └── nacos-cloud-spring-boot-starter # Nacos 配置中心动态监控线程池组件库
              
Starter 的确能让 Spring Boot 项目"一键启用"，但它只服务于 Boot。oneThread 想覆盖的不止是 Boot 应用，还包括普通 Spring 项目，甚至纯 Java 程序。如果把所有功能都塞进 Starter，非 Boot 场景根本没法无缝接入。

oneThread 的做法是把核心能力放进 `onethread-core`，Spring 扩展放进 `onethread-spring`，Boot 自动装配再用一个很薄的 `onethread-starter`。这样纯 Java 项目只引 core，Spring 项目加 spring，Spring Boot 项目再多一个 starter，就都能跑起来。  

|         项目类型          |            应该引入的模块             |
|-----------------------|--------------------------------|
| **纯 Java / 非 Spring** | `onethread-core`               |
| **Spring (非 Boot)**   | `onethread-spring`             |
| **Spring Boot**       | `onethread-starter`（很薄，仅做自动装配） |

如果你在非 Boot 环境里硬拉一个 Starter，然后自己写启动逻辑去"模拟"自动装配，会踩两大坑：其一，Boot 独有的条件注解和配置绑定在普通 Spring 根本解析不了；其二，Starter 写得越重，耦合就越深，想自定义启动点几乎没有空间。

最好的例子就是 XXL-Job，它只给了一个 core 包，任何项目只要 new 一下 `XxlJobSpringExecutor`，注册成 Bean 就能用。如果 XXL-Job 将复杂初始化逻辑（假设有）当初全都写在 Starter 里，非 Boot 项目就得二次开发才能接入。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(applicationName);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(StrUtil.isNotEmpty(accessToken) ? accessToken : null);
        xxlJobSpringExecutor.setLogPath(StrUtil.isNotEmpty(logPath) ? logPath : Paths.get("").toAbsolutePath().getParent() + "/tmp");
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
              
还有版本升级的隐患。我经历过 Boot 1.5 → 2.x → 3.x，每次 API 都大改。如果核心代码强依赖 Boot，就不得不维护 N 套实现。oneThread 把 Starter 做得极薄，就是为了在 Boot 下一次大升级时，只动最外面那层皮，核心逻辑几乎不用碰。

所以，模块拆分不是折腾，而是为了兼容更多项目形态，降低升级成本，并保持架构的简洁度。真正把所有东西塞进一个胖 Starter，短期看似方便，长期一定埋坑。
> 在 Hippo4j 的实践中，我深刻体会到"深度绑定 Spring Boot"带来的痛点：版本升级一动就牵一发而动全身，依赖矩阵异常复杂。因此，后续做开源框架时，我更倾向**只依赖 Spring（或者不强依赖）** 而非 Spring Boot------第三方依赖越少，架构就越纯净、可维护性也越高。

对于 **面向开源社区的基础组件库** ，代码必须足够通用：既要兼容不同 Spring/SpringBoot 版本，又要尽量减少三方依赖才能让别人无缝集成。因此，**核心逻辑与框架集成必须彻底解耦**，Starter 只做薄薄一层自动装配。

而 **公司内部的基础组件** 场景要简单得多：运行环境、Spring Boot 版本乃至依赖清单都高度一致。如果不考虑对外复用，做成一个"大而全"的内部 Starter 也未尝不可。

不过，一旦企业内部开始出现 **多条技术栈并行**（例如新旧项目 Boot 版本不一致），还是建议把核心能力抽离出来，Starter 只负责适配各版本 ------ 这样既能降低升级成本，也为未来的开源或跨团队复用留下余地。

文末总结
----

线程池用不好真挺闹心：一不小心线程开多了服务器扛不住，开少了任务又堵着干不完；出问题了不知道咋回事，关了服务还可能丢任务。

**oneThread 就是来搞定这些的：**

* 1.  
**统一管**：大家别乱搞线程池了，先去配置中心登记一下，省得搞出好多重复的浪费资源。  
* 2.  
**灵活调**：参数不用担心设置不好了！线上跑着也能随时调整核心线程数、队列大小这些，灵活得很。  
* 3.  
**及时报**：快撑不住（线程太忙/队列快满/任务被拒）了，马上发消息告诉你，及时处理不误事。  
* 4.  
**看得清**：线程池里面啥情况（有多少线程在忙、排了多少队、被拒绝多少活），整得明明白白展示出来。  
* 5.  
  **安心关**：关服务也不怕丢任务，设定一个等待时间，让还在跑的活尽量搞定走人。

**为啥拆好几个包（core,spring,starter）？** 就为了让不管是纯 Java 项目、老 Spring 项目、还是 SpringBoot 项目，都能方便地用上，哪部分合适就装哪部分，不添堵。这种设计升级也方便点，亲测靠谱。

完结，撒花 🎉  

