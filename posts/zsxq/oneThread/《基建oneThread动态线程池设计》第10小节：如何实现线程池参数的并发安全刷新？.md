2025年07月18日 22:10  
如何实现线程池参数的并发安全刷新？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文聚焦 oneThread 动态线程池的参数刷新机制，详细拆解"检测差异 → 热更新参数 → 更新元数据 → 消息通知与审计日志"全链路实现。通过逐行解析关键代码，帮助读者理解如何在运行期安全、无感知地将远程配置同步到线程池。

课程目录如下所示：

*  
业务时序  
*  
配置前置验证  
*  
线程池配置变更  
*  
如何保障线程池配置刷新的并发安全？  
*  
  文末总结

在上一章节，咱们已经实现了从配置中心获取最新的变更内容，并将对应的字符串序列化为 Java 对象供后续流程使用。接下来会从最新配置和内存中的线程池配置进行比较，如果有变化则更新，没有变化则跳过。

业务时序
----

动态线程池的核心能力之一，就是运行时可以自动感知配置变化并热更新，无需重启服务。为实现这一能力，我们需要：

* 1.  
获取远程最新线程池配置；  
* 2.  
对比当前内存中已有的线程池配置；  
* 3.  
如果检测到配置发生变更，则执行更新；  
* 4.  
存储最新配置，方便下次配置更新时比对；  
* 5.  
  通知各方配置已变更，并打印变更日志。

![iShot_2025-07-17_19.45.60.png](https://article-images.zsxq.com/Fk88zLZ1HELu4EnKh0in2pgvGGB8 "iShot_2025-07-17_19.45.60.png")

配置前置验证
------

### 1. 判断是否有线程池配置 {#1}

代码首先使用检查远程配置是否包含线程池设置。如果为空，则直接返回，跳过后续处理。这确保了只有当有有效的线程池配置时才会继续执行。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (CollUtil.isEmpty(refresherProperties.getExecutors())) {
        return;
    }
              
### 2. 判断线程池配置是否发生变化 {#2}

我们逐个遍历线程池 ID，并通过 `hasThreadPoolConfigChanged()` 进行对比。该方法会检查核心线程数、最大线程数、拒绝策略、队列容量等多个关键参数是否发生了变化。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    for (ThreadPoolExecutorProperties remoteProperties : refresherProperties.getExecutors()) {
        boolean changed = hasThreadPoolConfigChanged(remoteProperties);
        if (!changed) {
            continue;
        }
        ...
    }
              
根据线程池 ID 获取注册中心中的 `ThreadPoolExecutorHolder`。如果未找到线程池实例，记录警告日志并返回 `false`。若找到实例，调用 `hasDifference` 对比新旧配置。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private boolean hasThreadPoolConfigChanged(ThreadPoolExecutorProperties remoteProperties) {
        String threadPoolId = remoteProperties.getThreadPoolId();
        ThreadPoolExecutorHolder holder = OneThreadRegistry.getHolder(threadPoolId);
        if (holder == null) {
            log.warn("No thread pool found for thread pool id: {}", threadPoolId);
            return false;
        }
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties originalProperties = holder.getExecutorProperties();
    ​
        return hasDifference(originalProperties, remoteProperties, executor);
    }
              
该方法逐字段比较关键属性是否发生变化，判断逻辑为：

*  
只要有一个字段不相等，则认为配置已变更。  
*  
  字段包括：核心线程数、最大线程数、是否允许超时、存活时间、拒绝策略、队列容量等。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private boolean hasDifference(ThreadPoolExecutorProperties original,
                                  ThreadPoolExecutorProperties remote,
                                  ThreadPoolExecutor executor) {
        return isChanged(original.getCorePoolSize(), remote.getCorePoolSize())
            || isChanged(original.getMaximumPoolSize(), remote.getMaximumPoolSize())
            || isChanged(original.getAllowCoreThreadTimeOut(), remote.getAllowCoreThreadTimeOut())
            || isChanged(original.getKeepAliveTime(), remote.getKeepAliveTime())
            || isChanged(original.getRejectedHandler(), remote.getRejectedHandler())
            || isQueueCapacityChanged(original, remote, executor);
    }
              
`isChanged` 是一个辅助方法，用于比较两个值是否不同：

*  
忽略空值，仅对非 null 字段进行判断；  
*  
  使用 `Objects.equals()` 保证泛型类型安全比较。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private <T> boolean isChanged(T before, T after) {
        return after != null && !Objects.equals(before, after);
    }
              
> 网上也有一些专业做 Java 对象比对是否一致的框架：[dadiyang/equator](https://github.com/dadiyang/equator)，因为我懒得去看里面的 API，所以就自己进行的判断。如果有复杂对象且字段多的，推荐使用现成工具类。

对于队列容量，有一个特殊的检查 `isQueueCapacityChanged`，因为不是所有队列都支持动态调整容量。队列容量仅支持我们自定义的 `ResizableCapacityLinkedBlockingQueue`，否则不可动态调整。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private boolean isQueueCapacityChanged(...){
        Integer remoteCapacity = remoteProperties.getQueueCapacity();
        Integer originalCapacity = originalProperties.getQueueCapacity();
        BlockingQueue<?> queue = executor.getQueue();
    ​
        return remoteCapacity != null
            && !Objects.equals(remoteCapacity, originalCapacity)
            && Objects.equals("ResizableCapacityLinkedBlockingQueue", queue.getClass().getSimpleName());
    }
    ​
              
只有远程配置中的队列容量不为空，并且与当前内存中的配置不一致，同时当前线程池使用的是支持动态扩容的队列（如`ResizableCapacityLinkedBlockingQueue`），才认为容量确实发生了变化。
> 默认的 ArrayBlockingQueue 和 LinkedBlockingQueue 为什么不能改变容量，下一章节会详细说明。

线程池配置变更
-------

如果检测到变化，调用线程池刷新方法应用新的配置：

### 1. 核心 / 最大线程数 {#1}

线程数更新的原则是：**先最大后核心** 。如果新核心线程数大于当前最大线程数，必须先调大 `maximumPoolSize`，否则 JDK 会抛 `IllegalArgumentException`。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Integer remoteCorePoolSize = remoteProperties.getCorePoolSize();
    Integer remoteMaximumPoolSize = remoteProperties.getMaximumPoolSize();
    if (remoteCorePoolSize != null && remoteMaximumPoolSize != null) {
        int originalMaximumPoolSize = executor.getMaximumPoolSize();
        if (remoteCorePoolSize > originalMaximumPoolSize) {
            executor.setMaximumPoolSize(remoteMaximumPoolSize);
            executor.setCorePoolSize(remoteCorePoolSize);
        } else {
            executor.setCorePoolSize(remoteCorePoolSize);
            executor.setMaximumPoolSize(remoteMaximumPoolSize);
        }
    } else {
        if (remoteMaximumPoolSize != null) {
            executor.setMaximumPoolSize(remoteMaximumPoolSize);
        }
        if (remoteCorePoolSize != null) {
            executor.setCorePoolSize(remoteCorePoolSize);
        }
    }
              
之所以会抛出异常，是因为 JDK17 **线程池底层在设置核心线程数时做了参数限制校验** 。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0 || maximumPoolSize < corePoolSize)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // We don't really know how many new threads are "needed".
            // As a heuristic, prestart enough new workers (up to new
            // core size) to handle the current number of tasks in
            // queue, but stop if queue becomes empty while doing so.
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }
              
有些同学可能打开自己项目一看，发现代码里并没有相关校验逻辑，这是因为 **JDK8并未引入这段线程数的校验机制** 。我在开发 Hippo4j 的过程中也曾踩过这个坑，感同身受。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // We don't really know how many new threads are "needed".
            // As a heuristic, prestart enough new workers (up to new
            // core size) to handle the current number of tasks in
            // queue, but stop if queue becomes empty while doing so.
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }
              
具体可以参考这个 Issue 👉 [Hippo4j Issue#1063](https://github.com/opengoofy/hippo4j/issues/1063)，里面有详细的分析过程和复现场景。

### 2. 拒绝策略 {#2}

借助枚举工厂方法把字符串策略名转换为真正的 `RejectedExecutionHandler` 实例，保证可插拔。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (remoteProperties.getRejectedHandler() != null &&
            !Objects.equals(remoteProperties.getRejectedHandler(), originalProperties.getRejectedHandler())) {
        RejectedExecutionHandler handler = RejectedPolicyTypeEnum.createPolicy(remoteProperties.getRejectedHandler());
        executor.setRejectedExecutionHandler(handler);
    }
              
### 3. 队列容量动态扩容 {#3}

只有当队列实例实现了可扩容接口时才可以修改容量，避免 `LinkedBlockingQueue` 等 JDK 原生队列不支持容量变化导致的风险。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (isQueueCapacityChanged(originalProperties, remoteProperties, executor)) {
        BlockingQueue<Runnable> queue = executor.getQueue();
        ResizableCapacityLinkedBlockingQueue<?> resizableQueue = (ResizableCapacityLinkedBlockingQueue<?>) queue;
        resizableQueue.setCapacity(remoteProperties.getQueueCapacity());
    }
              
### 4. 刷新元数据、发送通知、打印审计日志 {#4}

线程池运行时参数变更后，还有一些后置逻辑需要处理：

* 1.  
把最新配置写回注册表，保证后续再读取时就是新的值；  
* 2.  
然后会把"旧值 → 新值"的映射封装成 DTO，通过钉钉、企业微信、邮件等渠道推送给开发 / 运维，做到即时可见。  
* 3.  
  为实现日志留痕，会通过 `log.info` 统一打印所有关键字段的 **"旧值-\>新值"** 。

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 线程池参数变更后进行日志打印
    String threadPoolId = remoteProperties.getThreadPoolId();
    ThreadPoolExecutorHolder holder = OneThreadRegistry.getHolder(threadPoolId);
    ThreadPoolExecutorProperties originalProperties = holder.getExecutorProperties();
    holder.setExecutorProperties(remoteProperties);
    ​
    // 发送线程池配置变更消息通知
    sendThreadPoolConfigChangeMessage(originalProperties, remoteProperties);
    ​
    // 打印线程池配置变更日志
    log.info(CHANGE_THREAD_POOL_TEXT,
            threadPoolId,
            String.format(CHANGE_DELIMITER, originalProperties.getCorePoolSize(), remoteProperties.getCorePoolSize()),
            String.format(CHANGE_DELIMITER, originalProperties.getMaximumPoolSize(), remoteProperties.getMaximumPoolSize()),
            String.format(CHANGE_DELIMITER, originalProperties.getQueueCapacity(), remoteProperties.getQueueCapacity()),
            String.format(CHANGE_DELIMITER, originalProperties.getKeepAliveTime(), remoteProperties.getKeepAliveTime()),
            String.format(CHANGE_DELIMITER, originalProperties.getRejectedHandler(), remoteProperties.getRejectedHandler()),
            String.format(CHANGE_DELIMITER, originalProperties.getAllowCoreThreadTimeOut(), remoteProperties.getAllowCoreThreadTimeOut()));
              
如何保障线程池配置刷新的并发安全？
-----------------

在动态线程池的配置刷新过程中，我们需要支持多个线程同时触发配置变更（比如配置中心推送受网络影响重复推送、多用户手动调用重复、定时校验等），但必须**保证同一个线程池的参数刷新是串行、原子、安全的** 。

否则就可能导致：

*  
参数错乱：两个线程同时修改 corePoolSize 和 maximumPoolSize，最终值不可预期；  
*  
日志混乱：原始值和新值打印错位；  
*  
队列扩容失败：并发修改 `ResizableCapacityLinkedBlockingQueue` 会抛异常；  
*  
  不一致性：通知发送和实际线程池状态不一致。

在处理并发安全问题时，我考虑了两种方案：

* 1.  
**对整个方法加锁** ：实现最简单，能快速解决线程安全问题；  
* 2.  
  **仅对指定的线程池ID加锁** ：粒度更细，性能更优。

虽然第一种方式实现起来最省事，但**独属于程序员的"强迫症"让我不太能接受这种处理方式** 。因此，最终我选择了第二种方案，对线程池 ID 维度加锁。

### 1. 使用 synchronized (id) {#1-synchronized-id}

代码示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    synchronized (threadPoolId) {
        // do refresh
    }
              
存在的问题：

*  
如果 `threadPoolId` 是从对象字段获取的（例如 `.getThreadPoolId()`），多个对象即使返回相同内容，也可能是**不同的String实例** 。  
*  
  JVM 会对不同的引用分配不同的锁 → **锁不生效** ，并发冲突依然会发生。

### 2. 使用 ConcurrentHashMap\<String, ReentrantLock\> {#2-concurrent-hash-map-string-reentrant-lock}

代码示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private static final ConcurrentMap<String, ReentrantLock> lockMap = new ConcurrentHashMap<>();
    ​
    ReentrantLock lock = lockMap.computeIfAbsent(threadPoolId, k -> new ReentrantLock());
    lock.lock();
    try {
        // do refresh
    } finally {
        lock.unlock();
    }
              
存在的问题：

*  
需要维护锁生命周期，比如什么时候释放锁内存？  
*  
  对于中小项目或轻量组件来说有点"重"。

### 3. 使用 .intern() 基于内容值构建锁 {#3-intern}

代码示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    String threadPoolId = remoteProperties.getThreadPoolId();
    synchronized (threadPoolId.intern()) {
        // do refresh
    }
              
优势：

*  
任何内容相同的字符串，调用 `.intern()` 后都会返回**同一个对象引用** ；  
*  
不依赖外部锁表，**零依赖、线程安全** ；  
*  
  锁粒度以线程池 ID 为单位，天然支持并发刷新多个线程池。

#### 3.1 .intern() 原理 {#3-1-intern}

`.intern()` 是 Java 提供的**字符串常量池机制** 的一部分。

*  
它会将当前字符串加入 JVM 的字符串池（String Pool）中；  
*  
如果字符串池中已有相同内容的字符串，则直接返回那一份对象引用；  
*  
  这就确保了**同内容字符串→同引用对象** ，从而让 `synchronized` 生效。

举个例子：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    String s1 = new String("abc");
    String s2 = new String("abc");
    ​
    System.out.println(s1 == s2);                // false，不同引用
    System.out.println(s1.intern() == s2.intern());  // true，intern 后相同引用
              
所以，在做字符串内容级别的同步控制时，**只有`.intern()`能够在不同对象间复用同一锁对象** 。

![image-20250718154039085.png](https://article-images.zsxq.com/FtPYd1R2h_PFdJxxfdAusRRqGAUp "image-20250718154039085.png")

#### 3.2 刷新代码实战 {#3-2}

实际代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 刷新动态线程池对象核心参数
    for (ThreadPoolExecutorProperties remoteProperties : refresherProperties.getExecutors()) {
        String threadPoolId = remoteProperties.getThreadPoolId();
        // 以线程池 ID 为粒度加锁，避免多个线程同时刷新同一个线程池
        synchronized (threadPoolId.intern()) {
            // 检查线程池配置是否发生变化（与当前内存中的配置对比）
            boolean changed = hasThreadPoolConfigChanged(remoteProperties);
            if (!changed) {
                continue;
            }
            // ......
        }
    }
              
#### 3.3 并发单元测试 {#3-3}

大家感兴趣可以试试这这个单元测试，分别是加 `.intern()` 和不加的方案。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;

    public class InternFromObjectPropertyTest {

        public static void main(String[] args) {
            ExecutorService executor = Executors.newFixedThreadPool(5);

            for (int i = 0; i < 5; i++) {
                executor.submit(() -> {
                    // 每个线程构造一个独立对象，但 threadPoolId 内容相同
                    ThreadPoolExecutorProperties props = new ThreadPoolExecutorProperties("core-biz-pool");

                    // ❌ 不加 intern：锁失效
                    // Object lock = props.getThreadPoolId();

                    // ✅ 加 intern：同内容，同锁对象
                    Object lock = props.getThreadPoolId().intern();

                    synchronized (lock) {
                        String threadName = Thread.currentThread().getName();
                        System.out.printf("[%d] %s 正在刷新线程池 %s%n",
                                System.currentTimeMillis(), threadName, props.getThreadPoolId());

                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }

                        System.out.printf("[%d] %s 刷新完成%n",
                                System.currentTimeMillis(), threadName);
                    }
                });
            }

            executor.shutdown();
        }
    }

    @Data
    public class ThreadPoolExecutorProperties {

        private String threadPoolId;

        /**
         * 如果大家对手动 new String（强制创建新对象）有疑惑
         * 可以把这行代码加到动态线程池配置刷新的流程里，查看每一次相同的 threadPoolId HashCode 值是否相同
         * System.out.println(System.identityHashCode(threadPoolId));
         * <p>
         * 因为每次配置中心的字符串都是重新创建的，所以这里为了贴合实际场景，所以是直接 new String
         */
        public ThreadPoolExecutorProperties(String threadPoolId) {
            // 模拟内容相同，但引用不同
            this.threadPoolId = new String(threadPoolId);
        }
    }
              
文末总结
----

动态线程池组件的实现看似简单，实则暗藏诸多细节与挑战。从远程配置感知、差异化比对、线程池参数安全刷新，到注册中心的元数据更新与可观测日志输出，每一步都需要严谨设计。而在实际开发中，还必须考虑多种因素：不同版本的 JDK 对线程池行为的差异、阻塞队列是否支持扩容、拒绝策略的可插拔性、配置变更的幂等性、并发场景下的线程安全等。

写一个"能用"的组件很容易，但要写一个"健壮、通用、可维护"的组件，就必须在设计层面下足功夫。希望本文的拆解，能为大家学习线程池治理体系提供一些经验。

完结，撒花 🎉  
如何实现线程池参数的并发安全刷新？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文聚焦 oneThread 动态线程池的参数刷新机制，详细拆解"检测差异 → 热更新参数 → 更新元数据 → 消息通知与审计日志"全链路实现。通过逐行解析关键代码，帮助读者理解如何在运行期安全、无感知地将远程配置同步到线程池。

课程目录如下所示：

*  
业务时序  
*  
配置前置验证  
*  
线程池配置变更  
*  
如何保障线程池配置刷新的并发安全？  
*  
  文末总结

在上一章节，咱们已经实现了从配置中心获取最新的变更内容，并将对应的字符串序列化为 Java 对象供后续流程使用。接下来会从最新配置和内存中的线程池配置进行比较，如果有变化则更新，没有变化则跳过。

业务时序
----

动态线程池的核心能力之一，就是运行时可以自动感知配置变化并热更新，无需重启服务。为实现这一能力，我们需要：

* 1.  
获取远程最新线程池配置；  
* 2.  
对比当前内存中已有的线程池配置；  
* 3.  
如果检测到配置发生变更，则执行更新；  
* 4.  
存储最新配置，方便下次配置更新时比对；  
* 5.  
  通知各方配置已变更，并打印变更日志。

![iShot_2025-07-17_19.45.60.png](https://article-images.zsxq.com/Fk88zLZ1HELu4EnKh0in2pgvGGB8 "iShot_2025-07-17_19.45.60.png")

配置前置验证
------

### 1. 判断是否有线程池配置 {#1}

代码首先使用检查远程配置是否包含线程池设置。如果为空，则直接返回，跳过后续处理。这确保了只有当有有效的线程池配置时才会继续执行。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (CollUtil.isEmpty(refresherProperties.getExecutors())) {
        return;
    }
              
### 2. 判断线程池配置是否发生变化 {#2}

我们逐个遍历线程池 ID，并通过 `hasThreadPoolConfigChanged()` 进行对比。该方法会检查核心线程数、最大线程数、拒绝策略、队列容量等多个关键参数是否发生了变化。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    for (ThreadPoolExecutorProperties remoteProperties : refresherProperties.getExecutors()) {
        boolean changed = hasThreadPoolConfigChanged(remoteProperties);
        if (!changed) {
            continue;
        }
        ...
    }
              
根据线程池 ID 获取注册中心中的 `ThreadPoolExecutorHolder`。如果未找到线程池实例，记录警告日志并返回 `false`。若找到实例，调用 `hasDifference` 对比新旧配置。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private boolean hasThreadPoolConfigChanged(ThreadPoolExecutorProperties remoteProperties) {
        String threadPoolId = remoteProperties.getThreadPoolId();
        ThreadPoolExecutorHolder holder = OneThreadRegistry.getHolder(threadPoolId);
        if (holder == null) {
            log.warn("No thread pool found for thread pool id: {}", threadPoolId);
            return false;
        }
        ThreadPoolExecutor executor = holder.getExecutor();
        ThreadPoolExecutorProperties originalProperties = holder.getExecutorProperties();
    ​
        return hasDifference(originalProperties, remoteProperties, executor);
    }
              
该方法逐字段比较关键属性是否发生变化，判断逻辑为：

*  
只要有一个字段不相等，则认为配置已变更。  
*  
  字段包括：核心线程数、最大线程数、是否允许超时、存活时间、拒绝策略、队列容量等。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private boolean hasDifference(ThreadPoolExecutorProperties original,
                                  ThreadPoolExecutorProperties remote,
                                  ThreadPoolExecutor executor) {
        return isChanged(original.getCorePoolSize(), remote.getCorePoolSize())
            || isChanged(original.getMaximumPoolSize(), remote.getMaximumPoolSize())
            || isChanged(original.getAllowCoreThreadTimeOut(), remote.getAllowCoreThreadTimeOut())
            || isChanged(original.getKeepAliveTime(), remote.getKeepAliveTime())
            || isChanged(original.getRejectedHandler(), remote.getRejectedHandler())
            || isQueueCapacityChanged(original, remote, executor);
    }
              
`isChanged` 是一个辅助方法，用于比较两个值是否不同：

*  
忽略空值，仅对非 null 字段进行判断；  
*  
  使用 `Objects.equals()` 保证泛型类型安全比较。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private <T> boolean isChanged(T before, T after) {
        return after != null && !Objects.equals(before, after);
    }
              
> 网上也有一些专业做 Java 对象比对是否一致的框架：[dadiyang/equator](https://github.com/dadiyang/equator)，因为我懒得去看里面的 API，所以就自己进行的判断。如果有复杂对象且字段多的，推荐使用现成工具类。

对于队列容量，有一个特殊的检查 `isQueueCapacityChanged`，因为不是所有队列都支持动态调整容量。队列容量仅支持我们自定义的 `ResizableCapacityLinkedBlockingQueue`，否则不可动态调整。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private boolean isQueueCapacityChanged(...){
        Integer remoteCapacity = remoteProperties.getQueueCapacity();
        Integer originalCapacity = originalProperties.getQueueCapacity();
        BlockingQueue<?> queue = executor.getQueue();
    ​
        return remoteCapacity != null
            && !Objects.equals(remoteCapacity, originalCapacity)
            && Objects.equals("ResizableCapacityLinkedBlockingQueue", queue.getClass().getSimpleName());
    }
    ​
              
只有远程配置中的队列容量不为空，并且与当前内存中的配置不一致，同时当前线程池使用的是支持动态扩容的队列（如`ResizableCapacityLinkedBlockingQueue`），才认为容量确实发生了变化。
> 默认的 ArrayBlockingQueue 和 LinkedBlockingQueue 为什么不能改变容量，下一章节会详细说明。

线程池配置变更
-------

如果检测到变化，调用线程池刷新方法应用新的配置：

### 1. 核心 / 最大线程数 {#1}

线程数更新的原则是：**先最大后核心** 。如果新核心线程数大于当前最大线程数，必须先调大 `maximumPoolSize`，否则 JDK 会抛 `IllegalArgumentException`。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Integer remoteCorePoolSize = remoteProperties.getCorePoolSize();
    Integer remoteMaximumPoolSize = remoteProperties.getMaximumPoolSize();
    if (remoteCorePoolSize != null && remoteMaximumPoolSize != null) {
        int originalMaximumPoolSize = executor.getMaximumPoolSize();
        if (remoteCorePoolSize > originalMaximumPoolSize) {
            executor.setMaximumPoolSize(remoteMaximumPoolSize);
            executor.setCorePoolSize(remoteCorePoolSize);
        } else {
            executor.setCorePoolSize(remoteCorePoolSize);
            executor.setMaximumPoolSize(remoteMaximumPoolSize);
        }
    } else {
        if (remoteMaximumPoolSize != null) {
            executor.setMaximumPoolSize(remoteMaximumPoolSize);
        }
        if (remoteCorePoolSize != null) {
            executor.setCorePoolSize(remoteCorePoolSize);
        }
    }
              
之所以会抛出异常，是因为 JDK17 **线程池底层在设置核心线程数时做了参数限制校验** 。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0 || maximumPoolSize < corePoolSize)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // We don't really know how many new threads are "needed".
            // As a heuristic, prestart enough new workers (up to new
            // core size) to handle the current number of tasks in
            // queue, but stop if queue becomes empty while doing so.
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }
              
有些同学可能打开自己项目一看，发现代码里并没有相关校验逻辑，这是因为 **JDK8并未引入这段线程数的校验机制** 。我在开发 Hippo4j 的过程中也曾踩过这个坑，感同身受。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // We don't really know how many new threads are "needed".
            // As a heuristic, prestart enough new workers (up to new
            // core size) to handle the current number of tasks in
            // queue, but stop if queue becomes empty while doing so.
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }
              
具体可以参考这个 Issue 👉 [Hippo4j Issue#1063](https://github.com/opengoofy/hippo4j/issues/1063)，里面有详细的分析过程和复现场景。

### 2. 拒绝策略 {#2}

借助枚举工厂方法把字符串策略名转换为真正的 `RejectedExecutionHandler` 实例，保证可插拔。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (remoteProperties.getRejectedHandler() != null &&
            !Objects.equals(remoteProperties.getRejectedHandler(), originalProperties.getRejectedHandler())) {
        RejectedExecutionHandler handler = RejectedPolicyTypeEnum.createPolicy(remoteProperties.getRejectedHandler());
        executor.setRejectedExecutionHandler(handler);
    }
              
### 3. 队列容量动态扩容 {#3}

只有当队列实例实现了可扩容接口时才可以修改容量，避免 `LinkedBlockingQueue` 等 JDK 原生队列不支持容量变化导致的风险。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    if (isQueueCapacityChanged(originalProperties, remoteProperties, executor)) {
        BlockingQueue<Runnable> queue = executor.getQueue();
        ResizableCapacityLinkedBlockingQueue<?> resizableQueue = (ResizableCapacityLinkedBlockingQueue<?>) queue;
        resizableQueue.setCapacity(remoteProperties.getQueueCapacity());
    }
              
### 4. 刷新元数据、发送通知、打印审计日志 {#4}

线程池运行时参数变更后，还有一些后置逻辑需要处理：

* 1.  
把最新配置写回注册表，保证后续再读取时就是新的值；  
* 2.  
然后会把"旧值 → 新值"的映射封装成 DTO，通过钉钉、企业微信、邮件等渠道推送给开发 / 运维，做到即时可见。  
* 3.  
  为实现日志留痕，会通过 `log.info` 统一打印所有关键字段的 **"旧值-\>新值"** 。

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 线程池参数变更后进行日志打印
    String threadPoolId = remoteProperties.getThreadPoolId();
    ThreadPoolExecutorHolder holder = OneThreadRegistry.getHolder(threadPoolId);
    ThreadPoolExecutorProperties originalProperties = holder.getExecutorProperties();
    holder.setExecutorProperties(remoteProperties);
    ​
    // 发送线程池配置变更消息通知
    sendThreadPoolConfigChangeMessage(originalProperties, remoteProperties);
    ​
    // 打印线程池配置变更日志
    log.info(CHANGE_THREAD_POOL_TEXT,
            threadPoolId,
            String.format(CHANGE_DELIMITER, originalProperties.getCorePoolSize(), remoteProperties.getCorePoolSize()),
            String.format(CHANGE_DELIMITER, originalProperties.getMaximumPoolSize(), remoteProperties.getMaximumPoolSize()),
            String.format(CHANGE_DELIMITER, originalProperties.getQueueCapacity(), remoteProperties.getQueueCapacity()),
            String.format(CHANGE_DELIMITER, originalProperties.getKeepAliveTime(), remoteProperties.getKeepAliveTime()),
            String.format(CHANGE_DELIMITER, originalProperties.getRejectedHandler(), remoteProperties.getRejectedHandler()),
            String.format(CHANGE_DELIMITER, originalProperties.getAllowCoreThreadTimeOut(), remoteProperties.getAllowCoreThreadTimeOut()));
              
如何保障线程池配置刷新的并发安全？
-----------------

在动态线程池的配置刷新过程中，我们需要支持多个线程同时触发配置变更（比如配置中心推送受网络影响重复推送、多用户手动调用重复、定时校验等），但必须**保证同一个线程池的参数刷新是串行、原子、安全的** 。

否则就可能导致：

*  
参数错乱：两个线程同时修改 corePoolSize 和 maximumPoolSize，最终值不可预期；  
*  
日志混乱：原始值和新值打印错位；  
*  
队列扩容失败：并发修改 `ResizableCapacityLinkedBlockingQueue` 会抛异常；  
*  
  不一致性：通知发送和实际线程池状态不一致。

在处理并发安全问题时，我考虑了两种方案：

* 1.  
**对整个方法加锁** ：实现最简单，能快速解决线程安全问题；  
* 2.  
  **仅对指定的线程池ID加锁** ：粒度更细，性能更优。

虽然第一种方式实现起来最省事，但**独属于程序员的"强迫症"让我不太能接受这种处理方式** 。因此，最终我选择了第二种方案，对线程池 ID 维度加锁。

### 1. 使用 synchronized (id) {#1-synchronized-id}

代码示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    synchronized (threadPoolId) {
        // do refresh
    }
              
存在的问题：

*  
如果 `threadPoolId` 是从对象字段获取的（例如 `.getThreadPoolId()`），多个对象即使返回相同内容，也可能是**不同的String实例** 。  
*  
  JVM 会对不同的引用分配不同的锁 → **锁不生效** ，并发冲突依然会发生。

### 2. 使用 ConcurrentHashMap\<String, ReentrantLock\> {#2-concurrent-hash-map-string-reentrant-lock}

代码示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private static final ConcurrentMap<String, ReentrantLock> lockMap = new ConcurrentHashMap<>();
    ​
    ReentrantLock lock = lockMap.computeIfAbsent(threadPoolId, k -> new ReentrantLock());
    lock.lock();
    try {
        // do refresh
    } finally {
        lock.unlock();
    }
              
存在的问题：

*  
需要维护锁生命周期，比如什么时候释放锁内存？  
*  
  对于中小项目或轻量组件来说有点"重"。

### 3. 使用 .intern() 基于内容值构建锁 {#3-intern}

代码示例：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    String threadPoolId = remoteProperties.getThreadPoolId();
    synchronized (threadPoolId.intern()) {
        // do refresh
    }
              
优势：

*  
任何内容相同的字符串，调用 `.intern()` 后都会返回**同一个对象引用** ；  
*  
不依赖外部锁表，**零依赖、线程安全** ；  
*  
  锁粒度以线程池 ID 为单位，天然支持并发刷新多个线程池。

#### 3.1 .intern() 原理 {#3-1-intern}

`.intern()` 是 Java 提供的**字符串常量池机制** 的一部分。

*  
它会将当前字符串加入 JVM 的字符串池（String Pool）中；  
*  
如果字符串池中已有相同内容的字符串，则直接返回那一份对象引用；  
*  
  这就确保了**同内容字符串→同引用对象** ，从而让 `synchronized` 生效。

举个例子：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    String s1 = new String("abc");
    String s2 = new String("abc");
    ​
    System.out.println(s1 == s2);                // false，不同引用
    System.out.println(s1.intern() == s2.intern());  // true，intern 后相同引用
              
所以，在做字符串内容级别的同步控制时，**只有`.intern()`能够在不同对象间复用同一锁对象** 。

![image-20250718154039085.png](https://article-images.zsxq.com/FtPYd1R2h_PFdJxxfdAusRRqGAUp "image-20250718154039085.png")

#### 3.2 刷新代码实战 {#3-2}

实际代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 刷新动态线程池对象核心参数
    for (ThreadPoolExecutorProperties remoteProperties : refresherProperties.getExecutors()) {
        String threadPoolId = remoteProperties.getThreadPoolId();
        // 以线程池 ID 为粒度加锁，避免多个线程同时刷新同一个线程池
        synchronized (threadPoolId.intern()) {
            // 检查线程池配置是否发生变化（与当前内存中的配置对比）
            boolean changed = hasThreadPoolConfigChanged(remoteProperties);
            if (!changed) {
                continue;
            }
            // ......
        }
    }
              
#### 3.3 并发单元测试 {#3-3}

大家感兴趣可以试试这这个单元测试，分别是加 `.intern()` 和不加的方案。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;

    public class InternFromObjectPropertyTest {

        public static void main(String[] args) {
            ExecutorService executor = Executors.newFixedThreadPool(5);

            for (int i = 0; i < 5; i++) {
                executor.submit(() -> {
                    // 每个线程构造一个独立对象，但 threadPoolId 内容相同
                    ThreadPoolExecutorProperties props = new ThreadPoolExecutorProperties("core-biz-pool");

                    // ❌ 不加 intern：锁失效
                    // Object lock = props.getThreadPoolId();

                    // ✅ 加 intern：同内容，同锁对象
                    Object lock = props.getThreadPoolId().intern();

                    synchronized (lock) {
                        String threadName = Thread.currentThread().getName();
                        System.out.printf("[%d] %s 正在刷新线程池 %s%n",
                                System.currentTimeMillis(), threadName, props.getThreadPoolId());

                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }

                        System.out.printf("[%d] %s 刷新完成%n",
                                System.currentTimeMillis(), threadName);
                    }
                });
            }

            executor.shutdown();
        }
    }

    @Data
    public class ThreadPoolExecutorProperties {

        private String threadPoolId;

        /**
         * 如果大家对手动 new String（强制创建新对象）有疑惑
         * 可以把这行代码加到动态线程池配置刷新的流程里，查看每一次相同的 threadPoolId HashCode 值是否相同
         * System.out.println(System.identityHashCode(threadPoolId));
         * <p>
         * 因为每次配置中心的字符串都是重新创建的，所以这里为了贴合实际场景，所以是直接 new String
         */
        public ThreadPoolExecutorProperties(String threadPoolId) {
            // 模拟内容相同，但引用不同
            this.threadPoolId = new String(threadPoolId);
        }
    }
              
文末总结
----

动态线程池组件的实现看似简单，实则暗藏诸多细节与挑战。从远程配置感知、差异化比对、线程池参数安全刷新，到注册中心的元数据更新与可观测日志输出，每一步都需要严谨设计。而在实际开发中，还必须考虑多种因素：不同版本的 JDK 对线程池行为的差异、阻塞队列是否支持扩容、拒绝策略的可插拔性、配置变更的幂等性、并发场景下的线程安全等。

写一个"能用"的组件很容易，但要写一个"健壮、通用、可维护"的组件，就必须在设计层面下足功夫。希望本文的拆解，能为大家学习线程池治理体系提供一些经验。

完结，撒花 🎉

