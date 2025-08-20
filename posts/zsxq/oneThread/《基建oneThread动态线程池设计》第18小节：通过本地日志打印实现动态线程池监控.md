2025年07月27日 21:55  
通过本地日志打印实现动态线程池监控，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文详细介绍 oneThread 动态线程池框架的**监控功能设计与实现** ，重点阐述**本地日志监控** 方式的架构设计。通过**定时任务调度** 、**运行时信息采集** 和**多维度指标监控**，实现了线程池运行状态的全面可观测性。

课程目录如下所示：

*  
前言  
*  
监控架构设计  
*  
本地日志监控实现  
*  
运行时信息采集优化  
*  
  文末总结

前言
---

在日常开发中，虽然 JDK 自带的线程池够用，但要说配置灵活性和监控能力，确实有点拉垮，很多时候我们连它是"健康"还是"爆了"都不清楚。

举个例子：
> 某天晚上 10 点，支付系统突然收到告警，说下游消息堆积严重，消费速度明显下降。但你打开日志一看，线程池既没报错、也没异常栈，表面上一切正常。你开始排查网络、排查 MQ、甚至重启服务，最后才发现------是线程池的处理能力（比如队列堆积）到了瓶颈，导致任务消费变慢。

虽然现在我们可以通过**拒绝策略告警、线程活跃度告警、队列使用率告警** 等机制及时发现部分异常，但**如果缺乏对线程池运行状态的持续观测与数据分析能力** ，很多问题依然难以深入理解。

比如：

*  
为什么这个线程池在某些时段会频繁打满？是核心线程数设置不合理？还是任务突发增长？  
*  
某个接口偶发超时，是否跟线程池的排队时间过长有关？  
*  
同一个线程池服务多个任务，是否存在"某类任务占满资源、其他任务被饿死"的情况？  
*  
  系统扩容之后线程池是否真的起效？性能瓶颈有没有缓解？

这些都需要依赖**细粒度的指标监控和趋势分析** ，仅靠"事发时的告警"远远不够。没有监控，开发人员对线程池的理解就是一片黑盒，排查问题只能靠猜，做性能调优也没底。

所以我们强调，线程池监控不是只为了报警，它更重要的价值在于：

*  
**辅助定位问题** ：出故障时，能看到是线程数打满了，还是队列堆积了；  
*  
**支持容量规划** ：通过长期趋势判断线程池配置是否合理；  
*  
  **洞察系统瓶颈** ：比如是否存在某些任务执行时间异常拉长，影响整体调度效率。

这些能力，才是一个成熟线程池监控体系真正应该具备的。本文就带大家看看**日志监控** 和 **Micrometer** 两套机制的设计思路。

监控架构设计
------

oneThread 的监控功能采用了**分层架构设计** ，主要包括以下几个核心组件：

![iShot_2025-07-19_13.07.34.png](https://article-images.zsxq.com/FkV2JMFEvFQtBPDSGeEUD15Qmqoh "iShot_2025-07-19_13.07.34.png")

在设计监控这块的时候，我们主要坚持了几个原则：

*  
**职责清晰** ：谁负责啥一目了然，每个模块只干自己的事。  
*  
**易扩展** ：以后想加新的监控方式，直接扩展就行，不用改老代码。  
*  
**性能优先** ：监控再重要，也不能拖慢业务，监控所需要执行的代码的都尽量轻量。  
*  
  **配置控制** ：所有监控行为都靠配置说了算，开关灵活，调整方便。

本地日志监控实现
--------

### 1. 定时任务调度机制 {#1}

本地日志监控的核心是**定时任务调度机制** ，通过`ScheduledExecutorService`实现周期性监控：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    public class ThreadPoolMonitor {

        private ScheduledExecutorService scheduler;
        private Map<String, ThreadPoolRuntimeInfo> micrometerMonitorCache;

        /**
         * 启动定时检查任务
         */
        public void start() {
            BootstrapConfigProperties.MonitorConfig monitorConfig = 
                BootstrapConfigProperties.getInstance().getMonitorConfig();
            
            if (!monitorConfig.getEnable()) {
                return;
            }

            // 初始化监控相关资源
            micrometerMonitorCache = new ConcurrentHashMap<>();
            scheduler = Executors.newScheduledThreadPool(
                    1,
                    ThreadFactoryBuilder.builder()
                            .namePrefix("scheduler_thread-pool_monitor")
                            .build()
            );

            // 每指定时间检查一次，初始延迟0秒
            scheduler.scheduleWithFixedDelay(() -> {
                Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
                for (ThreadPoolExecutorHolder holder : holders) {
                    ThreadPoolRuntimeInfo runtimeInfo = buildThreadPoolRuntimeInfo(holder);

                    // 根据采集类型判断
                    if (Objects.equals(monitorConfig.getCollectType(), "log")) {
                        logMonitor(runtimeInfo);
                    } else if (Objects.equals(monitorConfig.getCollectType(), "micrometer")) {
                        micrometerMonitor(runtimeInfo);
                    }
                }
            }, 0, monitorConfig.getCollectInterval(), TimeUnit.SECONDS);
        }
    }
              
### 2. 本地日志输出实现 {#2}

本地日志监控的实现相对简单，但需要确保日志格式的规范性和可读性：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void logMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
        log.info("[ThreadPool Monitor] {} | Content: {}", 
                 runtimeInfo.getThreadPoolId(), 
                 JSON.toJSON(runtimeInfo));
    }
              
通过JSON格式输出，确保了监控信息的结构化，便于日志分析和处理。
> 在没有接入标准监控系统的情况下，把线程池的运行状态定期打印到日志里，其实也是一种挺实用的排查手段。虽然不够高级，但关键时候能帮上忙。当然，下一篇文章我们会讲讲更专业、也更推荐的监控接入方式。

### 3. 设计模式应用 {#3}

对于监控采集类型的判断逻辑，其实还有优化空间。比如我们现在的两种存储方式，其实可以拆分成两个独立的实现类。这样一来，如果后续框架有计划支持更多存储策略（比如 ElasticSearch 等），就可以引入**策略模式** ，实现按需切换，结构更清晰；但如果当前只打算用一种或两种固定方式，那用一个**简单工厂模式** 来创建也更轻量，足够应对，扩展成本也低。

当前实现方案中，通过配置参数`collectType`来决定使用哪种监控策略：

*  
**log策略** ：将监控信息输出到本地日志  
*  
  **micrometer策略** ：将监控指标发送到Micrometer监控系统

这种设计使得监控方式的选择变得灵活，未来可以轻松扩展其他监控策略。

运行时信息采集优化
---------

### 1. 信息采集策略 {#1}

运行时信息采集是监控功能的核心，需要平衡**信息完整性** 和**性能影响** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @SneakyThrows
    private ThreadPoolRuntimeInfo buildThreadPoolRuntimeInfo(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        BlockingQueue<?> queue = executor.getQueue();
    ​
        long rejectCount = -1L;
        if (executor instanceof OneThreadExecutor) {
            rejectCount = ((OneThreadExecutor) executor).getRejectCount().get();
        }
    ​
        int workQueueSize = queue.size();
        int remainingCapacity = queue.remainingCapacity();
        
        return ThreadPoolRuntimeInfo.builder()
                .threadPoolId(holder.getThreadPoolId())
                .corePoolSize(executor.getCorePoolSize())
                .maximumPoolSize(executor.getMaximumPoolSize())
                .activePoolSize(executor.getActiveCount())  // API 有锁，避免高频率调用
                .currentPoolSize(executor.getPoolSize())  // API 有锁，避免高频率调用
                .completedTaskCount(executor.getCompletedTaskCount())  // API 有锁，避免高频率调用
                .largestPoolSize(executor.getLargestPoolSize())  // API 有锁，避免高频率调用
                .workQueueName(queue.getClass().getSimpleName())
                .workQueueSize(workQueueSize)
                .workQueueRemainingCapacity(remainingCapacity)
                .workQueueCapacity(workQueueSize + remainingCapacity)
                .rejectedHandlerName(executor.getRejectedExecutionHandler().toString())
                .rejectCount(rejectCount)
                .build();
    }
              
### 2. 性能优化策略 {#2}

在代码注释中，我们明确标注了哪些API调用是有锁的，这提醒开发者在设计监控频率时要考虑性能影响：

*  
**有锁API** ：`getActiveCount()`、`getPoolSize()`、`getCompletedTaskCount()`等。  
*  
  **无锁API** ：`getCorePoolSize()`、`getMaximumPoolSize()`等。

### 3. 运行时信息模型 {#3}

`ThreadPoolRuntimeInfo` 类定义了完整的线程池运行时信息模型：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ThreadPoolRuntimeInfo {
        private String threadPoolId;           // 线程池唯一标识
        private Integer corePoolSize;          // 核心线程数
        private Integer maximumPoolSize;       // 最大线程数
        private Integer currentPoolSize;       // 当前线程数
        private Integer activePoolSize;        // 活跃线程数
        private Integer largestPoolSize;       // 历史最大线程数
        private Long completedTaskCount;       // 已完成任务数
        private String workQueueName;          // 队列类型
        private Integer workQueueCapacity;     // 队列容量
        private Integer workQueueSize;         // 队列当前大小
        private Integer workQueueRemainingCapacity; // 队列剩余容量
        private String rejectedHandlerName;    // 拒绝策略名称
        private Long rejectCount;              // 拒绝次数
    }
              
这个模型涵盖了线程池运行状态的所有关键维度，为监控分析提供了完整的数据基础。

### 4. 定时任务优化 {#4}

在定时任务设计中，我们采用了以下优化策略：

*  
**单线程调度器** ：使用 `newScheduledThreadPool(1)` 确保监控任务串行执行，避免并发问题。  
*  
**固定延迟调度** ：使用 `scheduleWithFixedDelay` 而不是 `scheduleAtFixedRate`，避免任务堆积。  
*  
  **可配置间隔** ：通过配置文件控制监控频率，平衡监控精度和性能影响。

### 5. 监控开关控制 {#5}

通过配置开关控制监控功能的启用，避免在生产环境中意外启用监控：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (!monitorConfig.getEnable()) {
        return;
    }
              
文末总结
----

这篇文章主要聊了 oneThread 动态线程池里监控功能是怎么做的，包含整体架构、实现思路、以及一些性能上的优化细节。

有了这套监控机制，线程池不再是"黑盒"了。我们不仅能实时看到它的运行状态，还能更早发现问题、辅助排查、做容量评估。说白了，就是让线程池这块更透明、更可控，也让系统跑得更稳、更安心。

完结，撒花 🎉  
通过本地日志打印实现动态线程池监控，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文详细介绍 oneThread 动态线程池框架的**监控功能设计与实现** ，重点阐述**本地日志监控** 方式的架构设计。通过**定时任务调度** 、**运行时信息采集** 和**多维度指标监控**，实现了线程池运行状态的全面可观测性。

课程目录如下所示：

*  
前言  
*  
监控架构设计  
*  
本地日志监控实现  
*  
运行时信息采集优化  
*  
  文末总结

前言
---

在日常开发中，虽然 JDK 自带的线程池够用，但要说配置灵活性和监控能力，确实有点拉垮，很多时候我们连它是"健康"还是"爆了"都不清楚。

举个例子：
> 某天晚上 10 点，支付系统突然收到告警，说下游消息堆积严重，消费速度明显下降。但你打开日志一看，线程池既没报错、也没异常栈，表面上一切正常。你开始排查网络、排查 MQ、甚至重启服务，最后才发现------是线程池的处理能力（比如队列堆积）到了瓶颈，导致任务消费变慢。

虽然现在我们可以通过**拒绝策略告警、线程活跃度告警、队列使用率告警** 等机制及时发现部分异常，但**如果缺乏对线程池运行状态的持续观测与数据分析能力** ，很多问题依然难以深入理解。

比如：

*  
为什么这个线程池在某些时段会频繁打满？是核心线程数设置不合理？还是任务突发增长？  
*  
某个接口偶发超时，是否跟线程池的排队时间过长有关？  
*  
同一个线程池服务多个任务，是否存在"某类任务占满资源、其他任务被饿死"的情况？  
*  
  系统扩容之后线程池是否真的起效？性能瓶颈有没有缓解？

这些都需要依赖**细粒度的指标监控和趋势分析** ，仅靠"事发时的告警"远远不够。没有监控，开发人员对线程池的理解就是一片黑盒，排查问题只能靠猜，做性能调优也没底。

所以我们强调，线程池监控不是只为了报警，它更重要的价值在于：

*  
**辅助定位问题** ：出故障时，能看到是线程数打满了，还是队列堆积了；  
*  
**支持容量规划** ：通过长期趋势判断线程池配置是否合理；  
*  
  **洞察系统瓶颈** ：比如是否存在某些任务执行时间异常拉长，影响整体调度效率。

这些能力，才是一个成熟线程池监控体系真正应该具备的。本文就带大家看看**日志监控** 和 **Micrometer** 两套机制的设计思路。

监控架构设计
------

oneThread 的监控功能采用了**分层架构设计** ，主要包括以下几个核心组件：

![iShot_2025-07-19_13.07.34.png](https://article-images.zsxq.com/FkV2JMFEvFQtBPDSGeEUD15Qmqoh "iShot_2025-07-19_13.07.34.png")

在设计监控这块的时候，我们主要坚持了几个原则：

*  
**职责清晰** ：谁负责啥一目了然，每个模块只干自己的事。  
*  
**易扩展** ：以后想加新的监控方式，直接扩展就行，不用改老代码。  
*  
**性能优先** ：监控再重要，也不能拖慢业务，监控所需要执行的代码的都尽量轻量。  
*  
  **配置控制** ：所有监控行为都靠配置说了算，开关灵活，调整方便。

本地日志监控实现
--------

### 1. 定时任务调度机制 {#1}

本地日志监控的核心是**定时任务调度机制** ，通过`ScheduledExecutorService`实现周期性监控：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    public class ThreadPoolMonitor {

        private ScheduledExecutorService scheduler;
        private Map<String, ThreadPoolRuntimeInfo> micrometerMonitorCache;

        /**
         * 启动定时检查任务
         */
        public void start() {
            BootstrapConfigProperties.MonitorConfig monitorConfig = 
                BootstrapConfigProperties.getInstance().getMonitorConfig();
            
            if (!monitorConfig.getEnable()) {
                return;
            }

            // 初始化监控相关资源
            micrometerMonitorCache = new ConcurrentHashMap<>();
            scheduler = Executors.newScheduledThreadPool(
                    1,
                    ThreadFactoryBuilder.builder()
                            .namePrefix("scheduler_thread-pool_monitor")
                            .build()
            );

            // 每指定时间检查一次，初始延迟0秒
            scheduler.scheduleWithFixedDelay(() -> {
                Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
                for (ThreadPoolExecutorHolder holder : holders) {
                    ThreadPoolRuntimeInfo runtimeInfo = buildThreadPoolRuntimeInfo(holder);

                    // 根据采集类型判断
                    if (Objects.equals(monitorConfig.getCollectType(), "log")) {
                        logMonitor(runtimeInfo);
                    } else if (Objects.equals(monitorConfig.getCollectType(), "micrometer")) {
                        micrometerMonitor(runtimeInfo);
                    }
                }
            }, 0, monitorConfig.getCollectInterval(), TimeUnit.SECONDS);
        }
    }
              
### 2. 本地日志输出实现 {#2}

本地日志监控的实现相对简单，但需要确保日志格式的规范性和可读性：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void logMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
        log.info("[ThreadPool Monitor] {} | Content: {}", 
                 runtimeInfo.getThreadPoolId(), 
                 JSON.toJSON(runtimeInfo));
    }
              
通过JSON格式输出，确保了监控信息的结构化，便于日志分析和处理。
> 在没有接入标准监控系统的情况下，把线程池的运行状态定期打印到日志里，其实也是一种挺实用的排查手段。虽然不够高级，但关键时候能帮上忙。当然，下一篇文章我们会讲讲更专业、也更推荐的监控接入方式。

### 3. 设计模式应用 {#3}

对于监控采集类型的判断逻辑，其实还有优化空间。比如我们现在的两种存储方式，其实可以拆分成两个独立的实现类。这样一来，如果后续框架有计划支持更多存储策略（比如 ElasticSearch 等），就可以引入**策略模式** ，实现按需切换，结构更清晰；但如果当前只打算用一种或两种固定方式，那用一个**简单工厂模式** 来创建也更轻量，足够应对，扩展成本也低。

当前实现方案中，通过配置参数`collectType`来决定使用哪种监控策略：

*  
**log策略** ：将监控信息输出到本地日志  
*  
  **micrometer策略** ：将监控指标发送到Micrometer监控系统

这种设计使得监控方式的选择变得灵活，未来可以轻松扩展其他监控策略。

运行时信息采集优化
---------

### 1. 信息采集策略 {#1}

运行时信息采集是监控功能的核心，需要平衡**信息完整性** 和**性能影响** ：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @SneakyThrows
    private ThreadPoolRuntimeInfo buildThreadPoolRuntimeInfo(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        BlockingQueue<?> queue = executor.getQueue();
    ​
        long rejectCount = -1L;
        if (executor instanceof OneThreadExecutor) {
            rejectCount = ((OneThreadExecutor) executor).getRejectCount().get();
        }
    ​
        int workQueueSize = queue.size();
        int remainingCapacity = queue.remainingCapacity();
        
        return ThreadPoolRuntimeInfo.builder()
                .threadPoolId(holder.getThreadPoolId())
                .corePoolSize(executor.getCorePoolSize())
                .maximumPoolSize(executor.getMaximumPoolSize())
                .activePoolSize(executor.getActiveCount())  // API 有锁，避免高频率调用
                .currentPoolSize(executor.getPoolSize())  // API 有锁，避免高频率调用
                .completedTaskCount(executor.getCompletedTaskCount())  // API 有锁，避免高频率调用
                .largestPoolSize(executor.getLargestPoolSize())  // API 有锁，避免高频率调用
                .workQueueName(queue.getClass().getSimpleName())
                .workQueueSize(workQueueSize)
                .workQueueRemainingCapacity(remainingCapacity)
                .workQueueCapacity(workQueueSize + remainingCapacity)
                .rejectedHandlerName(executor.getRejectedExecutionHandler().toString())
                .rejectCount(rejectCount)
                .build();
    }
              
### 2. 性能优化策略 {#2}

在代码注释中，我们明确标注了哪些API调用是有锁的，这提醒开发者在设计监控频率时要考虑性能影响：

*  
**有锁API** ：`getActiveCount()`、`getPoolSize()`、`getCompletedTaskCount()`等。  
*  
  **无锁API** ：`getCorePoolSize()`、`getMaximumPoolSize()`等。

### 3. 运行时信息模型 {#3}

`ThreadPoolRuntimeInfo` 类定义了完整的线程池运行时信息模型：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ThreadPoolRuntimeInfo {
        private String threadPoolId;           // 线程池唯一标识
        private Integer corePoolSize;          // 核心线程数
        private Integer maximumPoolSize;       // 最大线程数
        private Integer currentPoolSize;       // 当前线程数
        private Integer activePoolSize;        // 活跃线程数
        private Integer largestPoolSize;       // 历史最大线程数
        private Long completedTaskCount;       // 已完成任务数
        private String workQueueName;          // 队列类型
        private Integer workQueueCapacity;     // 队列容量
        private Integer workQueueSize;         // 队列当前大小
        private Integer workQueueRemainingCapacity; // 队列剩余容量
        private String rejectedHandlerName;    // 拒绝策略名称
        private Long rejectCount;              // 拒绝次数
    }
              
这个模型涵盖了线程池运行状态的所有关键维度，为监控分析提供了完整的数据基础。

### 4. 定时任务优化 {#4}

在定时任务设计中，我们采用了以下优化策略：

*  
**单线程调度器** ：使用 `newScheduledThreadPool(1)` 确保监控任务串行执行，避免并发问题。  
*  
**固定延迟调度** ：使用 `scheduleWithFixedDelay` 而不是 `scheduleAtFixedRate`，避免任务堆积。  
*  
  **可配置间隔** ：通过配置文件控制监控频率，平衡监控精度和性能影响。

### 5. 监控开关控制 {#5}

通过配置开关控制监控功能的启用，避免在生产环境中意外启用监控：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (!monitorConfig.getEnable()) {
        return;
    }
              
文末总结
----

这篇文章主要聊了 oneThread 动态线程池里监控功能是怎么做的，包含整体架构、实现思路、以及一些性能上的优化细节。

有了这套监控机制，线程池不再是"黑盒"了。我们不仅能实时看到它的运行状态，还能更早发现问题、辅助排查、做容量评估。说白了，就是让线程池这块更透明、更可控，也让系统跑得更稳、更安心。

完结，撒花 🎉

