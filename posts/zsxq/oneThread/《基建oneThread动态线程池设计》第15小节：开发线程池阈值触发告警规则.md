2025年07月22日 21:47  
开发线程池阈值触发告警规则，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文介绍 oneThread 中线程池告警机制的设计与实现，涵盖三大告警维度、定时检查器的源码解析，以及告警机制在实际项目中的常见问题与优化建议，帮助系统快速构建稳定可控的线程池监控体系。

课程目录如下所示：

*  
前言  
*  
告警规则  
*  
实现告警检查器  
*  
常见问题  
*  
  文末总结

前言
---

在日常服务运行过程中，线程池作为业务异步处理的重要基础组件，其健康状态直接影响系统的吞吐能力与稳定性。然而，大多数线程池在使用过程中缺乏运行时监控和告警机制，导致问题往往在"任务堆积严重"、"线程打满"、"请求被拒"后才被动发现，极易引发雪崩效应。

为了解决这一痛点，**oneThread 在框架层内置了线程池运行状态的告警机制**，并通过统一的定时检查器配合线程池注册表，提供了轻量、通用、可配置的线程池健康巡检能力。

告警规则
----

如果让你来设计线程池告警机制，你会关注哪些维度？是**线程池任务堆积太多** ，还是**线程都被打满** ，又或者是**任务被拒绝的次数陡增**？

在 oneThread 中，我们结合大量项目实践与告警命中率，最终提炼出了三条"高命中"的告警策略，并给出默认的触发阈值与判定逻辑，覆盖了**最常见的线程池异常使用场景**。

告警策略如下所示：  

|    维度    |                      触发条件                      |            检测含义             |
|----------|------------------------------------------------|-----------------------------|
| **活跃度**  | `activeCount / maximumPoolSize` 连续高于阈值（默认 80%） | 线程资源已逼近瓶颈，需扩容或对入口流量做限流      |
| **队列负载** | `queueSize / queueCapacity` 超过阈值               | 排队任务激增，处理能力被入口流量压制，易引发大面积超时 |
| **拒绝异常** | 监控到新的 `RejectedExecutionException`             | 线程池已无法接收新任务，属于阻断场景，应立刻介入    |

活跃度和队列负载的监控规则较为简单，通过定时任务扫描即可实现。不过需要注意的是，定时任务的执行间隔需合理设置：过短会因监控 API 加锁导致与线程池其他操作竞争锁资源，过长则可能错过重要的告警时机。oneThread 在充分权衡后，默认将扫描间隔设置为 **5 秒**。
> 因为拒绝策略告警设计到动态代理相关知识，为了文章内容垂直，会放到下一章节讲解。

实现告警检查器
-------

### 1. 告警定时检查 {#1}

由于线程池状态相关的检查 API（如 `getActiveCount`等）会竞争 `mainLock`，若在高频场景下调用，可能对业务线程产生性能干扰。因此，线程池状态监控通常采用定时任务方式进行，**以延迟换取业务稳定性** 。此类定时检查无需引入额外框架，JDK 提供的 `ScheduledExecutorService` 已能满足稳定的调度需求。

`ThreadPoolAlarmChecker` 利用一个单线程的调度器，**定期扫描系统中所有已注册线程池的运行状态**，并针对启用了告警的线程池执行各类运行指标检测，及时触发相关告警处理。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池运行状态报警检查器
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-05-04
     */
    @Slf4j
    @RequiredArgsConstructor
    public class ThreadPoolAlarmChecker {
    ​
        // ......
    ​
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(
                1,
                ThreadFactoryBuilder.builder()
                        .namePrefix("scheduler_thread-pool_alarm_checker")
                        .build()
        );
    ​
        /**
         * 启动定时检查任务
         */
        public void start() {
            // 每10秒检查一次，初始延迟0秒
            scheduler.scheduleWithFixedDelay(this::checkAlarm, 0, 5, TimeUnit.SECONDS);
        }
    ​
        /**
         * 停止报警检查
         */
        public void stop() {
            if (!scheduler.isShutdown()) {
                scheduler.shutdown();
            }
        }
    ​
        /**
         * 报警检查核心逻辑
         */
        private void checkAlarm() {
            Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
            for (ThreadPoolExecutorHolder holder : holders) {
                if (holder.getExecutorProperties().getAlarm().getEnable()) {
                    checkQueueUsage(holder);
                    checkActiveRate(holder);
                    // ......
                }
            }
        }
        // ......
    }
              
### 2. 活跃度告警 {#2}

`checkActiveRate` 会监控线程池中活跃线程数的使用比例，**当活跃度高于配置阈值时，触发"Activity"类型的告警**，帮助开发者及时发现线程池可能存在"线程资源耗尽"或"处理能力过载"的风险。  
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
              
代码执行流程如下所示：

* 1.  
从 `ThreadPoolExecutorHolder` 中获取实际的线程池实例（`ThreadPoolExecutor`）和对应的配置属性；  
* 2.  
获取当前线程池中"正在执行任务"的线程数。这是一个有同步锁的调用，**频繁获取会有性能开销**，所以建议定时调度而非高频轮询。  
* 3.  
获取线程池的最大线程数；**防止除以 0 的情况**，这里直接提前 return 掉。  
* 4.  
计算活跃线程的使用率（百分比），如当前活跃线程为 8，最大线程数为 10，则 `activeRate = 80`。  
* 5.  
获取配置中设定的"活跃度告警阈值"（比如 80 或 90）。  
* 6.  
  如果活跃线程使用率超过（或等于）设定阈值，就调用 `sendAlarmMessage(...)` 方法，触发 **线程活跃度过高** 的报警。

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
              
### 3. 容量告警 {#3}

`checkQueueUsage` 用于实时监控线程池任务队列的使用率，**当排队任务接近或达到容量上限时，触发告警**，以便及时发现任务堆积或线程处理能力不足的问题。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 检查队列使用率
     */
    private void checkQueueUsage(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties properties = holder.getExecutorProperties();
    ​
        BlockingQueue<?> queue = executor.getQueue();
        int queueSize = queue.size();
        int capacity = queueSize + queue.remainingCapacity();
    ​
        if (capacity == 0) {
            return;
        }
    ​
        int usageRate = (int) Math.round((queueSize * 100.0) / capacity);
        int threshold = properties.getAlarm().getQueueThreshold();
    ​
        if (usageRate >= threshold) {
            sendAlarmMessage("Capacity", holder);
        }
    }
              
代码执行流程如下所示：

* 1.  
获取当前线程池实例及其配置项；  
* 2.  
获取线程池的任务队列；计算当前队列中已排队的任务数量（`queueSize`）；  
* 3.  
使用 `queue.remainingCapacity()` 获取**剩余可接收的任务容量** ；通过两者相加，得出队列的**理论总容量**。  
* 4.  
安全防御代码：**避免除以 0** 的异常情况（如极端情况下队列容量为 0）。  
* 5.  
计算当前队列使用率，结果为百分比（如 8 / 10 = 80%）。  
* 6.  
从配置中获取队列使用率的告警阈值，比如 `80` 表示队列使用率超过 80% 时报警。  
* 7.  
  若当前队列使用率超出设定阈值，触发一条 `"Capacity"` 类型的报警消息。

相较于线程活跃度检查，阻塞队列的使用率统计依赖的 API 较为轻量，对业务线程性能影响可忽略。

### 4. 告警参数组装 {#4}

下面是我们实际运行中的一张告警截图，可以看到其中包含了较多的运行时信息。

![image-20250525171313388 (3).png](https://article-images.zsxq.com/FrbsER6iTn8Nfi5HcaYzE63a9XCx)

这些字段在不同类型的告警中都是必需的，因此我们对公共的参数封装逻辑进行了抽象，提取成一个统一的方法，**以减少重复代码**。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @SneakyThrows
    private void sendAlarmMessage(String alarmType, ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties properties = holder.getExecutorProperties();
        BlockingQueue<?> queue = executor.getQueue();
    ​
        long rejectCount = -1L;
        if (executor instanceof OneThreadExecutor) {
            rejectCount = ((OneThreadExecutor) executor).getRejectCount().get();
        }
    ​
        int workQueueSize = queue.size(); // API 有锁，避免高频率调用
        int remainingCapacity = queue.remainingCapacity(); // API 有锁，避免高频率调用
        ThreadPoolAlarmNotifyDTO alarm = ThreadPoolAlarmNotifyDTO.builder()
                .applicationName(ApplicationProperties.getApplicationName())
                .activeProfile(ApplicationProperties.getActiveProfile())
                .identify(InetAddress.getLocalHost().getHostAddress())
                .alarmType(alarmType)
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
                .receives(properties.getNotify().getReceives())
                .currentTime(DateUtil.now())
                .interval(properties.getNotify().getInterval())
                .build();
        notifierDispatcher.sendAlarmMessage(alarm);
    }
              
常见问题
----

在引入线程池运行状态的告警机制之后，考虑到大家往往会关心其运行方式、资源消耗以及对业务的潜在影响。下面将围绕几个实际使用中常见的问题，逐一进行解答。

### 1. 谁来调用 start() 方法启动检查？ {#1-start}

`ThreadPoolAlarmChecker` 并不会在创建时自动启动告警逻辑。你需要在项目初始化阶段，**显式调用 `start()` 方法**，以启动定时检查任务。

常见做法包括在 SpringBoot 项目的启动回调（如 `ApplicationRunner`、`InitializingBean`）中调用；这样可以确保告警逻辑启动时，系统中已经存在可检查的线程池实例。

实际上，我们也是采用了相同的方式来管理线程池告警检查的生命周期。在 `spring-base` 模块中，通过 `OneThreadBaseConfiguration` 配置类完成了告警检查器的自动装配与调度控制：  
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
        // ......
    ​
        @Bean(initMethod = "start", destroyMethod = "stop")
        public ThreadPoolAlarmChecker threadPoolAlarmChecker(NotifierDispatcher notifierDispatcher) {
            return new ThreadPoolAlarmChecker(notifierDispatcher);
        }
    ​
        // ......
    }
              
告警检查器的装载流程：

*  
通过 `@Bean` 注解定义了 `ThreadPoolAlarmChecker` 的 Spring 管理对象；  
*  
  利用 `initMethod = "start"` 和 `destroyMethod = "stop"`，分别在 Bean 初始化与销毁阶段启动和停止定时检查任务。

### 2. 如果上一次检查没有结束，下一次检查又来了怎么办？ {#2}

`ThreadPoolAlarmChecker` 使用的是 JDK 自带的 `ScheduledExecutorService`，具体调度方式如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    scheduler.scheduleWithFixedDelay(this::checkAlarm, 0, 5, TimeUnit.SECONDS);
              
这里采用了 `scheduleWithFixedDelay`，意味着：

*  
**每次检查结束后再等待 5 秒，才会触发下一次执行**；  
*  
  若某次检查耗时较长，不会并发触发下一次，而是顺延执行，避免了重叠和堆积。

这种设计非常适合对线程池状态进行"低频、稳态"的监控，有效避免了因监控本身导致的资源竞争或死锁风险。

### 3. 如果项目中动态线程池过多，是否会有影响？ {#3}

当线程池数量较多时（例如数十个甚至上百个），确实可能对告警检查性能提出一定挑战。但从实现层面看，该告警检查器具有如下几个特性：

*  
**使用单线程定时调度器执行所有告警逻辑**，避免在高并发场景中对业务线程产生干扰。  
*  
**指标采集过程尽量使用轻量级 API** ，如 `getActiveCount()` ，虽然部分接口持有 `mainLock`，但整体频率较低。  
*  
  **每次检查本质上是一次遍历 + 统计计算**，即使线程池较多，也不会产生明显的 CPU 或内存压力。

综上所述，**只要合理配置检查频率和告警启用策略，即便系统中存在大量线程池实例，告警机制也不会对业务系统造成明显性能影响**。对于性能和准确性要求更高的场景，还可以进一步引入优化手段，例如：

*  
将告警处理逻辑异步化，避免阻塞检查线程；  
*  
引入专用线程池，对需要检查的线程池的状态并发检查与告警发送；  
*  
  配合监控系统进行异步指标上报和阈值告警判断。

通过上述手段，可以在保证系统稳定性的同时，进一步提升告警机制在复杂场景下的扩展能力与运行效率。

### 4. 运行过程中定时检查抛异常了怎么办？ {#4}

在定时执行 `checkAlarm()` 的过程中，如果某个线程池实例状态异常、配置错误，或内部检查逻辑抛出未捕获异常，**很可能会导致本次检查任务中断甚至整个定时调度器崩溃退出** 。一旦调度器停止，线程池告警机制将失效，后续运行状态异常将无法被及时感知和上报，**形成监控盲区**。

为避免这种情况，`ThreadPoolAlarmChecker` 内部可以采用将所有检查逻辑包裹在统一的异常保护块中，确保单次任务失败不会影响调度器的存活性：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void checkAlarm() {
        try {
            Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
            for (ThreadPoolExecutorHolder holder : holders) {
                if (holder.getExecutorProperties().getAlarm().getEnable()) {
                    checkQueueUsage(holder);
                    checkActiveRate(holder);
                    // ...
                }
            }
        } catch (Throwable t) {
            log.error("[oneThread] 线程池告警检查异常", t);
        }
    }
    ​
              
使用 `try-catch` 捕获 `Throwable`，可以防止包括运行时异常和错误在内的所有异常中断调度线程。

在极端情况下，即使某个线程池本身已被销毁或状态不正常，也不应该让整个监控任务崩盘，而应通过日志记录和告警提示来暴露问题。必要时也可以对调度器自身做健康检查，确保它始终在线运行。
> 这里示例中使用的是整体的 try-catch，其实也可以在具体方法内部加 try，粒度更细。咱们这边没有在外层额外加 try，是因为从整体逻辑来看，只有告警发送那部分可能抛异常，而底层已经做了 try-catch 处理，不会影响上层逻辑。
>
> 但如果你调用的是其他人的接口，建议还是加上必要的异常保护，避免影响整个调度任务的正常运行。

文末总结
----

本文围绕 oneThread 动态线程池的运行状态告警机制，从告警维度设计出发，详细介绍了"活跃度"、"队列使用率"两类核心告警策略，并结合源码讲解了 `ThreadPoolAlarmChecker` 的实现方式。

通过定时任务进行统一调度检查，结合各线程池是否启用告警配置，可以在不干扰业务的前提下，及时识别线程资源不足、任务堆积严重等风险场景。

最后咱们也解答了实践中常见的几个关键问题，包括告警器如何初始化、是否存在并发风险、在大规模线程池环境下是否稳定等。

总的来说，这套告警机制具有以下优点：

*  
**入侵性小**：不影响业务线程，采用单线程异步调度。  
*  
**适配性强**：支持对任意线程池启用或关闭告警。  
*  
**开箱即用**：在基础配置模块中已自动装配。  
*  
  **可拓展性高**：未来可支持更多维度与异步通知通道。

在下一篇中，我们将深入探讨 **任务拒绝告警机制** 的实现 ------ 包括如何借助代理拦截线程池执行过程中的 `RejectedExecutionException`，并配合上下文信息进行精准告警，敬请期待。

完结，撒花 🎉  
开发线程池阈值触发告警规则，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文介绍 oneThread 中线程池告警机制的设计与实现，涵盖三大告警维度、定时检查器的源码解析，以及告警机制在实际项目中的常见问题与优化建议，帮助系统快速构建稳定可控的线程池监控体系。

课程目录如下所示：

*  
前言  
*  
告警规则  
*  
实现告警检查器  
*  
常见问题  
*  
  文末总结

前言
---

在日常服务运行过程中，线程池作为业务异步处理的重要基础组件，其健康状态直接影响系统的吞吐能力与稳定性。然而，大多数线程池在使用过程中缺乏运行时监控和告警机制，导致问题往往在"任务堆积严重"、"线程打满"、"请求被拒"后才被动发现，极易引发雪崩效应。

为了解决这一痛点，**oneThread 在框架层内置了线程池运行状态的告警机制**，并通过统一的定时检查器配合线程池注册表，提供了轻量、通用、可配置的线程池健康巡检能力。

告警规则
----

如果让你来设计线程池告警机制，你会关注哪些维度？是**线程池任务堆积太多** ，还是**线程都被打满** ，又或者是**任务被拒绝的次数陡增**？

在 oneThread 中，我们结合大量项目实践与告警命中率，最终提炼出了三条"高命中"的告警策略，并给出默认的触发阈值与判定逻辑，覆盖了**最常见的线程池异常使用场景**。

告警策略如下所示：  

|    维度    |                      触发条件                      |            检测含义             |
|----------|------------------------------------------------|-----------------------------|
| **活跃度**  | `activeCount / maximumPoolSize` 连续高于阈值（默认 80%） | 线程资源已逼近瓶颈，需扩容或对入口流量做限流      |
| **队列负载** | `queueSize / queueCapacity` 超过阈值               | 排队任务激增，处理能力被入口流量压制，易引发大面积超时 |
| **拒绝异常** | 监控到新的 `RejectedExecutionException`             | 线程池已无法接收新任务，属于阻断场景，应立刻介入    |

活跃度和队列负载的监控规则较为简单，通过定时任务扫描即可实现。不过需要注意的是，定时任务的执行间隔需合理设置：过短会因监控 API 加锁导致与线程池其他操作竞争锁资源，过长则可能错过重要的告警时机。oneThread 在充分权衡后，默认将扫描间隔设置为 **5 秒**。
> 因为拒绝策略告警设计到动态代理相关知识，为了文章内容垂直，会放到下一章节讲解。

实现告警检查器
-------

### 1. 告警定时检查 {#1}

由于线程池状态相关的检查 API（如 `getActiveCount`等）会竞争 `mainLock`，若在高频场景下调用，可能对业务线程产生性能干扰。因此，线程池状态监控通常采用定时任务方式进行，**以延迟换取业务稳定性** 。此类定时检查无需引入额外框架，JDK 提供的 `ScheduledExecutorService` 已能满足稳定的调度需求。

`ThreadPoolAlarmChecker` 利用一个单线程的调度器，**定期扫描系统中所有已注册线程池的运行状态**，并针对启用了告警的线程池执行各类运行指标检测，及时触发相关告警处理。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池运行状态报警检查器
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-05-04
     */
    @Slf4j
    @RequiredArgsConstructor
    public class ThreadPoolAlarmChecker {
    ​
        // ......
    ​
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(
                1,
                ThreadFactoryBuilder.builder()
                        .namePrefix("scheduler_thread-pool_alarm_checker")
                        .build()
        );
    ​
        /**
         * 启动定时检查任务
         */
        public void start() {
            // 每10秒检查一次，初始延迟0秒
            scheduler.scheduleWithFixedDelay(this::checkAlarm, 0, 5, TimeUnit.SECONDS);
        }
    ​
        /**
         * 停止报警检查
         */
        public void stop() {
            if (!scheduler.isShutdown()) {
                scheduler.shutdown();
            }
        }
    ​
        /**
         * 报警检查核心逻辑
         */
        private void checkAlarm() {
            Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
            for (ThreadPoolExecutorHolder holder : holders) {
                if (holder.getExecutorProperties().getAlarm().getEnable()) {
                    checkQueueUsage(holder);
                    checkActiveRate(holder);
                    // ......
                }
            }
        }
        // ......
    }
              
### 2. 活跃度告警 {#2}

`checkActiveRate` 会监控线程池中活跃线程数的使用比例，**当活跃度高于配置阈值时，触发"Activity"类型的告警**，帮助开发者及时发现线程池可能存在"线程资源耗尽"或"处理能力过载"的风险。  
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
              
代码执行流程如下所示：

* 1.  
从 `ThreadPoolExecutorHolder` 中获取实际的线程池实例（`ThreadPoolExecutor`）和对应的配置属性；  
* 2.  
获取当前线程池中"正在执行任务"的线程数。这是一个有同步锁的调用，**频繁获取会有性能开销**，所以建议定时调度而非高频轮询。  
* 3.  
获取线程池的最大线程数；**防止除以 0 的情况**，这里直接提前 return 掉。  
* 4.  
计算活跃线程的使用率（百分比），如当前活跃线程为 8，最大线程数为 10，则 `activeRate = 80`。  
* 5.  
获取配置中设定的"活跃度告警阈值"（比如 80 或 90）。  
* 6.  
  如果活跃线程使用率超过（或等于）设定阈值，就调用 `sendAlarmMessage(...)` 方法，触发 **线程活跃度过高** 的报警。

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
              
### 3. 容量告警 {#3}

`checkQueueUsage` 用于实时监控线程池任务队列的使用率，**当排队任务接近或达到容量上限时，触发告警**，以便及时发现任务堆积或线程处理能力不足的问题。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 检查队列使用率
     */
    private void checkQueueUsage(ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties properties = holder.getExecutorProperties();
    ​
        BlockingQueue<?> queue = executor.getQueue();
        int queueSize = queue.size();
        int capacity = queueSize + queue.remainingCapacity();
    ​
        if (capacity == 0) {
            return;
        }
    ​
        int usageRate = (int) Math.round((queueSize * 100.0) / capacity);
        int threshold = properties.getAlarm().getQueueThreshold();
    ​
        if (usageRate >= threshold) {
            sendAlarmMessage("Capacity", holder);
        }
    }
              
代码执行流程如下所示：

* 1.  
获取当前线程池实例及其配置项；  
* 2.  
获取线程池的任务队列；计算当前队列中已排队的任务数量（`queueSize`）；  
* 3.  
使用 `queue.remainingCapacity()` 获取**剩余可接收的任务容量** ；通过两者相加，得出队列的**理论总容量**。  
* 4.  
安全防御代码：**避免除以 0** 的异常情况（如极端情况下队列容量为 0）。  
* 5.  
计算当前队列使用率，结果为百分比（如 8 / 10 = 80%）。  
* 6.  
从配置中获取队列使用率的告警阈值，比如 `80` 表示队列使用率超过 80% 时报警。  
* 7.  
  若当前队列使用率超出设定阈值，触发一条 `"Capacity"` 类型的报警消息。

相较于线程活跃度检查，阻塞队列的使用率统计依赖的 API 较为轻量，对业务线程性能影响可忽略。

### 4. 告警参数组装 {#4}

下面是我们实际运行中的一张告警截图，可以看到其中包含了较多的运行时信息。

![image-20250525171313388 (3).png](https://article-images.zsxq.com/FrbsER6iTn8Nfi5HcaYzE63a9XCx)

这些字段在不同类型的告警中都是必需的，因此我们对公共的参数封装逻辑进行了抽象，提取成一个统一的方法，**以减少重复代码**。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @SneakyThrows
    private void sendAlarmMessage(String alarmType, ThreadPoolExecutorHolder holder) {
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties properties = holder.getExecutorProperties();
        BlockingQueue<?> queue = executor.getQueue();
    ​
        long rejectCount = -1L;
        if (executor instanceof OneThreadExecutor) {
            rejectCount = ((OneThreadExecutor) executor).getRejectCount().get();
        }
    ​
        int workQueueSize = queue.size(); // API 有锁，避免高频率调用
        int remainingCapacity = queue.remainingCapacity(); // API 有锁，避免高频率调用
        ThreadPoolAlarmNotifyDTO alarm = ThreadPoolAlarmNotifyDTO.builder()
                .applicationName(ApplicationProperties.getApplicationName())
                .activeProfile(ApplicationProperties.getActiveProfile())
                .identify(InetAddress.getLocalHost().getHostAddress())
                .alarmType(alarmType)
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
                .receives(properties.getNotify().getReceives())
                .currentTime(DateUtil.now())
                .interval(properties.getNotify().getInterval())
                .build();
        notifierDispatcher.sendAlarmMessage(alarm);
    }
              
常见问题
----

在引入线程池运行状态的告警机制之后，考虑到大家往往会关心其运行方式、资源消耗以及对业务的潜在影响。下面将围绕几个实际使用中常见的问题，逐一进行解答。

### 1. 谁来调用 start() 方法启动检查？ {#1-start}

`ThreadPoolAlarmChecker` 并不会在创建时自动启动告警逻辑。你需要在项目初始化阶段，**显式调用 `start()` 方法**，以启动定时检查任务。

常见做法包括在 SpringBoot 项目的启动回调（如 `ApplicationRunner`、`InitializingBean`）中调用；这样可以确保告警逻辑启动时，系统中已经存在可检查的线程池实例。

实际上，我们也是采用了相同的方式来管理线程池告警检查的生命周期。在 `spring-base` 模块中，通过 `OneThreadBaseConfiguration` 配置类完成了告警检查器的自动装配与调度控制：  
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
        // ......
    ​
        @Bean(initMethod = "start", destroyMethod = "stop")
        public ThreadPoolAlarmChecker threadPoolAlarmChecker(NotifierDispatcher notifierDispatcher) {
            return new ThreadPoolAlarmChecker(notifierDispatcher);
        }
    ​
        // ......
    }
              
告警检查器的装载流程：

*  
通过 `@Bean` 注解定义了 `ThreadPoolAlarmChecker` 的 Spring 管理对象；  
*  
  利用 `initMethod = "start"` 和 `destroyMethod = "stop"`，分别在 Bean 初始化与销毁阶段启动和停止定时检查任务。

### 2. 如果上一次检查没有结束，下一次检查又来了怎么办？ {#2}

`ThreadPoolAlarmChecker` 使用的是 JDK 自带的 `ScheduledExecutorService`，具体调度方式如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    scheduler.scheduleWithFixedDelay(this::checkAlarm, 0, 5, TimeUnit.SECONDS);
              
这里采用了 `scheduleWithFixedDelay`，意味着：

*  
**每次检查结束后再等待 5 秒，才会触发下一次执行**；  
*  
  若某次检查耗时较长，不会并发触发下一次，而是顺延执行，避免了重叠和堆积。

这种设计非常适合对线程池状态进行"低频、稳态"的监控，有效避免了因监控本身导致的资源竞争或死锁风险。

### 3. 如果项目中动态线程池过多，是否会有影响？ {#3}

当线程池数量较多时（例如数十个甚至上百个），确实可能对告警检查性能提出一定挑战。但从实现层面看，该告警检查器具有如下几个特性：

*  
**使用单线程定时调度器执行所有告警逻辑**，避免在高并发场景中对业务线程产生干扰。  
*  
**指标采集过程尽量使用轻量级 API** ，如 `getActiveCount()` ，虽然部分接口持有 `mainLock`，但整体频率较低。  
*  
  **每次检查本质上是一次遍历 + 统计计算**，即使线程池较多，也不会产生明显的 CPU 或内存压力。

综上所述，**只要合理配置检查频率和告警启用策略，即便系统中存在大量线程池实例，告警机制也不会对业务系统造成明显性能影响**。对于性能和准确性要求更高的场景，还可以进一步引入优化手段，例如：

*  
将告警处理逻辑异步化，避免阻塞检查线程；  
*  
引入专用线程池，对需要检查的线程池的状态并发检查与告警发送；  
*  
  配合监控系统进行异步指标上报和阈值告警判断。

通过上述手段，可以在保证系统稳定性的同时，进一步提升告警机制在复杂场景下的扩展能力与运行效率。

### 4. 运行过程中定时检查抛异常了怎么办？ {#4}

在定时执行 `checkAlarm()` 的过程中，如果某个线程池实例状态异常、配置错误，或内部检查逻辑抛出未捕获异常，**很可能会导致本次检查任务中断甚至整个定时调度器崩溃退出** 。一旦调度器停止，线程池告警机制将失效，后续运行状态异常将无法被及时感知和上报，**形成监控盲区**。

为避免这种情况，`ThreadPoolAlarmChecker` 内部可以采用将所有检查逻辑包裹在统一的异常保护块中，确保单次任务失败不会影响调度器的存活性：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void checkAlarm() {
        try {
            Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
            for (ThreadPoolExecutorHolder holder : holders) {
                if (holder.getExecutorProperties().getAlarm().getEnable()) {
                    checkQueueUsage(holder);
                    checkActiveRate(holder);
                    // ...
                }
            }
        } catch (Throwable t) {
            log.error("[oneThread] 线程池告警检查异常", t);
        }
    }
    ​
              
使用 `try-catch` 捕获 `Throwable`，可以防止包括运行时异常和错误在内的所有异常中断调度线程。

在极端情况下，即使某个线程池本身已被销毁或状态不正常，也不应该让整个监控任务崩盘，而应通过日志记录和告警提示来暴露问题。必要时也可以对调度器自身做健康检查，确保它始终在线运行。
> 这里示例中使用的是整体的 try-catch，其实也可以在具体方法内部加 try，粒度更细。咱们这边没有在外层额外加 try，是因为从整体逻辑来看，只有告警发送那部分可能抛异常，而底层已经做了 try-catch 处理，不会影响上层逻辑。
>
> 但如果你调用的是其他人的接口，建议还是加上必要的异常保护，避免影响整个调度任务的正常运行。

文末总结
----

本文围绕 oneThread 动态线程池的运行状态告警机制，从告警维度设计出发，详细介绍了"活跃度"、"队列使用率"两类核心告警策略，并结合源码讲解了 `ThreadPoolAlarmChecker` 的实现方式。

通过定时任务进行统一调度检查，结合各线程池是否启用告警配置，可以在不干扰业务的前提下，及时识别线程资源不足、任务堆积严重等风险场景。

最后咱们也解答了实践中常见的几个关键问题，包括告警器如何初始化、是否存在并发风险、在大规模线程池环境下是否稳定等。

总的来说，这套告警机制具有以下优点：

*  
**入侵性小**：不影响业务线程，采用单线程异步调度。  
*  
**适配性强**：支持对任意线程池启用或关闭告警。  
*  
**开箱即用**：在基础配置模块中已自动装配。  
*  
  **可拓展性高**：未来可支持更多维度与异步通知通道。

在下一篇中，我们将深入探讨 **任务拒绝告警机制** 的实现 ------ 包括如何借助代理拦截线程池执行过程中的 `RejectedExecutionException`，并配合上下文信息进行精准告警，敬请期待。

完结，撒花 🎉  

