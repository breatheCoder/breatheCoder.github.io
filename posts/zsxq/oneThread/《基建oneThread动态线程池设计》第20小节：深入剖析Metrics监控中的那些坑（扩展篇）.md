2025年07月31日 22:55  
深入剖析 Metrics 监控中的那些坑（扩展篇），元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

课程目录如下所示：

*  
前言  
*  
Metrics 注册原理  
*  
为什么每次使用新传入的 runtimeInfo 没有报 NaN？  
*  
当前 Metrics 实现存在的问题  
*  
  Metrics 优化重构

前言
---

这是一篇扩展文章，起因是星球里有位同学提出了一个很有意思的问题，我们先来看看。

![image-20250731094334060.png](https://article-images.zsxq.com/Fl4v7uztjNnUdgtDt0o3TAJds8v_)

这是我们之前的代码实现，这位同学认为在 `Metrics.gauge` 注册指标时，应该传入被缓存的 `existingRuntimeInfo`，而不是直接使用 `runtimeInfo`。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void micrometerMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
        String threadPoolId = runtimeInfo.getThreadPoolId();
        ThreadPoolRuntimeInfo existingRuntimeInfo = micrometerMonitorCache.get(threadPoolId);
        if (existingRuntimeInfo != null) {
            BeanUtil.copyProperties(runtimeInfo, existingRuntimeInfo);
        } else {
            micrometerMonitorCache.put(threadPoolId, runtimeInfo);
        }
    ​
        Iterable<Tag> tags = CollectionUtil.newArrayList(
                Tag.of(DYNAMIC_THREAD_POOL_ID_TAG, threadPoolId),
                Tag.of(APPLICATION_NAME_TAG, ApplicationProperties.getApplicationName()));
    ​
        Metrics.gauge(metricName("core.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getCorePoolSize);
        Metrics.gauge(metricName("maximum.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getMaximumPoolSize);
        Metrics.gauge(metricName("current.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getCurrentPoolSize);
        Metrics.gauge(metricName("largest.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getLargestPoolSize);
        Metrics.gauge(metricName("active.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getActivePoolSize);
        Metrics.gauge(metricName("queue.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getWorkQueueSize);
        Metrics.gauge(metricName("queue.capacity"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getWorkQueueCapacity);
        Metrics.gauge(metricName("queue.remaining.capacity"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getWorkQueueRemainingCapacity);
        Metrics.gauge(metricName("completed.task.count"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getCompletedTaskCount);
        Metrics.gauge(metricName("reject.count"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getRejectCount);
    }
              
这篇文章就围绕这个问题，和大家深入探讨一下 `Metrics` 的注册机制，以及基于这个机制我们如何进行代码重构优化。

Metrics 注册原理 {#metrics}
-----------------------

### 1. 基于首次注册对象引用获取指标 {#1}

我们先来看看这行代码的实际作用：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Metrics.gauge(metricName("xxx"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getXxx);
              
这行代码背后的执行逻辑是这样的：

* 1.  
如果这个 metric（包括名字 + tag）是**第一次注册** ，Micrometer 会绑定 `runtimeInfo` 对象的**引用** ；  
* 2.  
如果 metric 已经注册过了，这次调用**不会更新引用** ，只是返回已存在的 Gauge（Micrometer 内部维护了一个缓存结构）；  
* 3.  
  因此，即使后续你传入了新的 `runtimeInfo` 对象，但这个新对象并不会被绑定，Micrometer 仍然会读取**最初绑定的老对象的引用值** 。

让我们看看 Micrometer 的 `Metrics.gauge(...)` 源码逻辑（以 `SimpleMeterRegistry` 为例）：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public <T> Gauge gauge(String name, Iterable<Tag> tags, T obj, ToDoubleFunction<T> f) {
        return register(
            new Gauge(name, tags, obj, f)
        );
    }
              
`register(...)` 方法会检查是否已存在相同 `name + tags` 的 Gauge，如果存在：

*  
直接返回已存在的 Gauge；  
*  
不会重新注册或更新引用；  
*  
  即使你传入的是新对象，也会被忽略，仍然使用原始绑定的对象。

![image-20250731172945646.png](https://article-images.zsxq.com/FsZuLxAFq__D1q2sHhzeIi7ETo5L)

换句话说，**第一次传入的`runtimeInfo1`被绑定后，之后无论你传入`runtimeInfo2、3、4`，Micrometer都会忽略，只会继续读取`runtimeInfo1`的字段值** 。
> 在之前的文章中，我们提到过如果没有缓存层会出现获取值 NaN 的情况，这正好对应了上面的第三点。这意味着 metric 获取的绑定对象被 GC 回收了，可能就会返回 NaN。

### 2. 注册的 Metric 指标什么时候更新？ {#2-metric}

既然已经注册的 Metric 指标需要更新最新的值，那是不是内部有个定时任务在收集数据呢？

其实不是的。Metric 内部并没有定时收集机制，而是当外部调用方请求指标列表时，才会从已绑定的对象中获取最新数据。这个调用方通常是 Prometheus，因为 Prometheus 会定时拉取数据。而我们 oneThread 内部的定时任务就是用来更新首次绑定对象的值。

![image-20250731172846824.png](https://article-images.zsxq.com/FuVxTPhiCfbOhZl7-CuN4KCaNd53)

为什么每次使用新传入的 runtimeInfo 没有报 NaN？ {#runtime-info-na-n}
-----------------------------------------------------

这段代码最初是我从 Hippo4j 项目中复制过来的，已经很长时间没仔细看过了。这位同学一提问，我自己也有点懵，按理说应该会有问题才对，怎么会正常运行呢？如果没问题，岂不是上面的结论不对？

让我们再仔细看看这段代码，看看有没有遗漏的地方：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void micrometerMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
        String threadPoolId = runtimeInfo.getThreadPoolId();
        ThreadPoolRuntimeInfo existingRuntimeInfo = micrometerMonitorCache.get(threadPoolId);
        if (existingRuntimeInfo != null) {
            BeanUtil.copyProperties(runtimeInfo, existingRuntimeInfo);
        } else {
            micrometerMonitorCache.put(threadPoolId, runtimeInfo);
        }
        // ......
        Metrics.gauge(metricName("reject.count"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getRejectCount);
    }
              
后来仔细分析了代码，发现逻辑其实很巧妙，数据流转过程如下：

* 1.  
第一次注册 runtimeInfo 对象时，Metrics.gauge 绑定该对象；  
* 2.  
第二次执行时，从 micrometerMonitorCache 中获取 threadPoolId 对应的对象不为空，这个 existingRuntimeInfo 其实就是第一次绑定的 runtimeInfo；  
* 3.  
  然后每次都会将新的 runtimeInfo2、3、x 对象的属性复制给 existingRuntimeInfo，这相当于间接更新了绑定的 runtimeInfo 对象。

![image-20250731174540149.png](https://article-images.zsxq.com/FnPbYqjrbUDE6aYJxxYJVC4dtpRr)

当前 Metrics 实现存在的问题 {#metrics}
-----------------------------

刚开始写 Hippo4j 的时候，经验还不够丰富。经过这位同学的提醒，我重新审视了这段代码，发现确实还有优化空间。经过一番思考和实践，最终发现了几个问题。

### 1. 重复注册 Gauge 的性能问题 {#1-gauge}

每轮采集都会执行以下代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Metrics.gauge(metricName("xxx"), tags, runtimeInfo, getter);
              
这在 Micrometer 中的实际效果是：

*  
检查是否已存在这个 `(name + tags)` 的 Gauge；  
*  
如果存在：**忽略注册，保持绑定的是第一次传入的对象引用** ；  
*  
  如果不存在：**绑定当前对象引用（即`runtimeInfo`）并注册Gauge** 。

虽然实际效果不大，但增加了不必要的性能开销，这部分可以优化。

### 2. 实际绑定对象和更新对象的语义不清晰 {#2}

尽管 Micrometer 自带缓存机制会忽略重复注册，但代码读起来：

*  
语义不够清晰；  
*  
  容易误导阅读代码的人：看上去每次都在绑定新对象，但实际上不会生效最新的 runtimeInfo。

### 3. 任务完成数量和拒绝策略指标展示不够直观 {#3}

我们在注册完成任务数量和拒绝策略数量时，指标每次都是累计递增的，展示的图表类似这样：


<br />


要么指标是平的，要么一直往上增长，这样不利于观察某个时间区间内的任务完成情况。

![image-20250731155052661.png](https://article-images.zsxq.com/FoLwQQGRtq8rrfsWGHyskuLhkPsF)

Metrics 优化重构 {#metrics}
-----------------------

基于上面发现的问题，我重构了 `ThreadPoolMonitor` 监控的核心方法。这里先放张流程图：

![image-20250731173227514.png](https://article-images.zsxq.com/Fu-Om5baiBwip0QLMWEEKLgnnMeF)

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    public class ThreadPoolMonitor {
    ​
        private ScheduledExecutorService scheduler;
        private Map<String, ThreadPoolRuntimeInfo> micrometerMonitorCache;
        private Map<String, DeltaWrapper> rejectCountDeltaMap;
        private Map<String, DeltaWrapper> completedTaskDeltaMap;
    ​
        // ......
    ​
        /**
         * 采集 Micrometer 指标
         */
        private void micrometerMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
            String threadPoolId = runtimeInfo.getThreadPoolId();
            ThreadPoolRuntimeInfo existingRuntimeInfo = micrometerMonitorCache.get(threadPoolId);
    ​
            // 只在首次注册时绑定 Gauge
            if (existingRuntimeInfo == null) {
                Iterable<Tag> tags = CollectionUtil.newArrayList(
                        Tag.of(DYNAMIC_THREAD_POOL_ID_TAG, threadPoolId),
                        Tag.of(APPLICATION_NAME_TAG, ApplicationProperties.getApplicationName())
                );
    ​
                ThreadPoolRuntimeInfo registerRuntimeInfo = BeanUtil.toBean(runtimeInfo, ThreadPoolRuntimeInfo.class);
                micrometerMonitorCache.put(threadPoolId, registerRuntimeInfo);
    ​
                // 注册总量指标
                Metrics.gauge(metricName("core.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getCorePoolSize);
                Metrics.gauge(metricName("maximum.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getMaximumPoolSize);
                Metrics.gauge(metricName("current.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getCurrentPoolSize);
                Metrics.gauge(metricName("largest.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getLargestPoolSize);
                Metrics.gauge(metricName("active.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getActivePoolSize);
                Metrics.gauge(metricName("queue.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getWorkQueueSize);
                Metrics.gauge(metricName("queue.capacity"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getWorkQueueCapacity);
                Metrics.gauge(metricName("queue.remaining.capacity"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getWorkQueueRemainingCapacity);
    ​
                // 注册 delta 指标
                DeltaWrapper completedDelta = new DeltaWrapper();
                completedTaskDeltaMap.put(threadPoolId, completedDelta);
                Metrics.gauge(metricName("completed.task.count"), tags, completedDelta, DeltaWrapper::getDelta);
    ​
                DeltaWrapper rejectDelta = new DeltaWrapper();
                rejectCountDeltaMap.put(threadPoolId, rejectDelta);
                Metrics.gauge(metricName("reject.count"), tags, rejectDelta, DeltaWrapper::getDelta);
            } else {
                // 更新属性（避免重新注册 Gauge）
                BeanUtil.copyProperties(runtimeInfo, existingRuntimeInfo);
            }
    ​
            // 每次都更新 delta 值
            completedTaskDeltaMap.get(threadPoolId).update(runtimeInfo.getCompletedTaskCount());
            rejectCountDeltaMap.get(threadPoolId).update(runtimeInfo.getRejectCount());
        }
        // ......
    }
              
这里新增了两个成员变量：

*  
rejectCountDeltaMap：保存每个线程池的拒绝次数增量计算器。  
*  
  completedTaskDeltaMap：保存每个线程池的任务完成增量计算器。

方法的执行逻辑如下：

* 1.  
获取 `threadPoolId` 和缓存的 runtime 对象，判断该线程池是否**已经注册过Gauge** 。如果没有，则进入首次注册逻辑。  
* 2.  
  首次注册逻辑（只执行一次）：
  * 1.  
  **复制一个独立对象`registerRuntimeInfo`** 用于绑定 Gauge（Micrometer 要求引用稳定）；  
  * 2.  
  放入缓存，后续只更新其字段，不更换引用。  
  * 3.  
  每个 Gauge 都会绑定 `registerRuntimeInfo` 的引用，并定期调用其 Getter 方法采样。  
  * 4.  
注册增量指标：使用 `DeltaWrapper` 包装类维护历史值，实现 "当前值 - 上次值" 的统计；将 `deltaMap` 存入缓存，便于后续更新。  
* 3.  
后续只更新绑定引用的字段值：保持 Micrometer Gauge 绑定对象不变，仅更新其属性值，确保指标能够刷新。  
* 4.  
  每轮采集都更新 delta 值：每次传入新的原始计数，`DeltaWrapper` 会自动计算出 delta（当前值 - 上次记录值）。

至此，我们解决了前面提到的三个问题，代码的实现也更加优雅了。

完结，撒花 🎉  
深入剖析 Metrics 监控中的那些坑（扩展篇），元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

课程目录如下所示：

*  
前言  
*  
Metrics 注册原理  
*  
为什么每次使用新传入的 runtimeInfo 没有报 NaN？  
*  
当前 Metrics 实现存在的问题  
*  
  Metrics 优化重构

前言
---

这是一篇扩展文章，起因是星球里有位同学提出了一个很有意思的问题，我们先来看看。

![image-20250731094334060.png](https://article-images.zsxq.com/Fl4v7uztjNnUdgtDt0o3TAJds8v_)

这是我们之前的代码实现，这位同学认为在 `Metrics.gauge` 注册指标时，应该传入被缓存的 `existingRuntimeInfo`，而不是直接使用 `runtimeInfo`。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void micrometerMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
        String threadPoolId = runtimeInfo.getThreadPoolId();
        ThreadPoolRuntimeInfo existingRuntimeInfo = micrometerMonitorCache.get(threadPoolId);
        if (existingRuntimeInfo != null) {
            BeanUtil.copyProperties(runtimeInfo, existingRuntimeInfo);
        } else {
            micrometerMonitorCache.put(threadPoolId, runtimeInfo);
        }
    ​
        Iterable<Tag> tags = CollectionUtil.newArrayList(
                Tag.of(DYNAMIC_THREAD_POOL_ID_TAG, threadPoolId),
                Tag.of(APPLICATION_NAME_TAG, ApplicationProperties.getApplicationName()));
    ​
        Metrics.gauge(metricName("core.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getCorePoolSize);
        Metrics.gauge(metricName("maximum.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getMaximumPoolSize);
        Metrics.gauge(metricName("current.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getCurrentPoolSize);
        Metrics.gauge(metricName("largest.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getLargestPoolSize);
        Metrics.gauge(metricName("active.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getActivePoolSize);
        Metrics.gauge(metricName("queue.size"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getWorkQueueSize);
        Metrics.gauge(metricName("queue.capacity"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getWorkQueueCapacity);
        Metrics.gauge(metricName("queue.remaining.capacity"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getWorkQueueRemainingCapacity);
        Metrics.gauge(metricName("completed.task.count"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getCompletedTaskCount);
        Metrics.gauge(metricName("reject.count"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getRejectCount);
    }
              
这篇文章就围绕这个问题，和大家深入探讨一下 `Metrics` 的注册机制，以及基于这个机制我们如何进行代码重构优化。

Metrics 注册原理 {#metrics}
-----------------------

### 1. 基于首次注册对象引用获取指标 {#1}

我们先来看看这行代码的实际作用：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Metrics.gauge(metricName("xxx"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getXxx);
              
这行代码背后的执行逻辑是这样的：

* 1.  
如果这个 metric（包括名字 + tag）是**第一次注册** ，Micrometer 会绑定 `runtimeInfo` 对象的**引用** ；  
* 2.  
如果 metric 已经注册过了，这次调用**不会更新引用** ，只是返回已存在的 Gauge（Micrometer 内部维护了一个缓存结构）；  
* 3.  
  因此，即使后续你传入了新的 `runtimeInfo` 对象，但这个新对象并不会被绑定，Micrometer 仍然会读取**最初绑定的老对象的引用值** 。

让我们看看 Micrometer 的 `Metrics.gauge(...)` 源码逻辑（以 `SimpleMeterRegistry` 为例）：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public <T> Gauge gauge(String name, Iterable<Tag> tags, T obj, ToDoubleFunction<T> f) {
        return register(
            new Gauge(name, tags, obj, f)
        );
    }
              
`register(...)` 方法会检查是否已存在相同 `name + tags` 的 Gauge，如果存在：

*  
直接返回已存在的 Gauge；  
*  
不会重新注册或更新引用；  
*  
  即使你传入的是新对象，也会被忽略，仍然使用原始绑定的对象。

![image-20250731172945646.png](https://article-images.zsxq.com/FsZuLxAFq__D1q2sHhzeIi7ETo5L)

换句话说，**第一次传入的`runtimeInfo1`被绑定后，之后无论你传入`runtimeInfo2、3、4`，Micrometer都会忽略，只会继续读取`runtimeInfo1`的字段值** 。
> 在之前的文章中，我们提到过如果没有缓存层会出现获取值 NaN 的情况，这正好对应了上面的第三点。这意味着 metric 获取的绑定对象被 GC 回收了，可能就会返回 NaN。

### 2. 注册的 Metric 指标什么时候更新？ {#2-metric}

既然已经注册的 Metric 指标需要更新最新的值，那是不是内部有个定时任务在收集数据呢？

其实不是的。Metric 内部并没有定时收集机制，而是当外部调用方请求指标列表时，才会从已绑定的对象中获取最新数据。这个调用方通常是 Prometheus，因为 Prometheus 会定时拉取数据。而我们 oneThread 内部的定时任务就是用来更新首次绑定对象的值。

![image-20250731172846824.png](https://article-images.zsxq.com/FuVxTPhiCfbOhZl7-CuN4KCaNd53)

为什么每次使用新传入的 runtimeInfo 没有报 NaN？ {#runtime-info-na-n}
-----------------------------------------------------

这段代码最初是我从 Hippo4j 项目中复制过来的，已经很长时间没仔细看过了。这位同学一提问，我自己也有点懵，按理说应该会有问题才对，怎么会正常运行呢？如果没问题，岂不是上面的结论不对？

让我们再仔细看看这段代码，看看有没有遗漏的地方：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void micrometerMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
        String threadPoolId = runtimeInfo.getThreadPoolId();
        ThreadPoolRuntimeInfo existingRuntimeInfo = micrometerMonitorCache.get(threadPoolId);
        if (existingRuntimeInfo != null) {
            BeanUtil.copyProperties(runtimeInfo, existingRuntimeInfo);
        } else {
            micrometerMonitorCache.put(threadPoolId, runtimeInfo);
        }
        // ......
        Metrics.gauge(metricName("reject.count"), tags, runtimeInfo, ThreadPoolRuntimeInfo::getRejectCount);
    }
              
后来仔细分析了代码，发现逻辑其实很巧妙，数据流转过程如下：

* 1.  
第一次注册 runtimeInfo 对象时，Metrics.gauge 绑定该对象；  
* 2.  
第二次执行时，从 micrometerMonitorCache 中获取 threadPoolId 对应的对象不为空，这个 existingRuntimeInfo 其实就是第一次绑定的 runtimeInfo；  
* 3.  
  然后每次都会将新的 runtimeInfo2、3、x 对象的属性复制给 existingRuntimeInfo，这相当于间接更新了绑定的 runtimeInfo 对象。

![image-20250731174540149.png](https://article-images.zsxq.com/FnPbYqjrbUDE6aYJxxYJVC4dtpRr)

当前 Metrics 实现存在的问题 {#metrics}
-----------------------------

刚开始写 Hippo4j 的时候，经验还不够丰富。经过这位同学的提醒，我重新审视了这段代码，发现确实还有优化空间。经过一番思考和实践，最终发现了几个问题。

### 1. 重复注册 Gauge 的性能问题 {#1-gauge}

每轮采集都会执行以下代码：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Metrics.gauge(metricName("xxx"), tags, runtimeInfo, getter);
              
这在 Micrometer 中的实际效果是：

*  
检查是否已存在这个 `(name + tags)` 的 Gauge；  
*  
如果存在：**忽略注册，保持绑定的是第一次传入的对象引用** ；  
*  
  如果不存在：**绑定当前对象引用（即`runtimeInfo`）并注册Gauge** 。

虽然实际效果不大，但增加了不必要的性能开销，这部分可以优化。

### 2. 实际绑定对象和更新对象的语义不清晰 {#2}

尽管 Micrometer 自带缓存机制会忽略重复注册，但代码读起来：

*  
语义不够清晰；  
*  
  容易误导阅读代码的人：看上去每次都在绑定新对象，但实际上不会生效最新的 runtimeInfo。

### 3. 任务完成数量和拒绝策略指标展示不够直观 {#3}

我们在注册完成任务数量和拒绝策略数量时，指标每次都是累计递增的，展示的图表类似这样：


<br />


要么指标是平的，要么一直往上增长，这样不利于观察某个时间区间内的任务完成情况。

![image-20250731155052661.png](https://article-images.zsxq.com/FoLwQQGRtq8rrfsWGHyskuLhkPsF)

Metrics 优化重构 {#metrics}
-----------------------

基于上面发现的问题，我重构了 `ThreadPoolMonitor` 监控的核心方法。这里先放张流程图：

![image-20250731173227514.png](https://article-images.zsxq.com/Fu-Om5baiBwip0QLMWEEKLgnnMeF)

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    public class ThreadPoolMonitor {
    ​
        private ScheduledExecutorService scheduler;
        private Map<String, ThreadPoolRuntimeInfo> micrometerMonitorCache;
        private Map<String, DeltaWrapper> rejectCountDeltaMap;
        private Map<String, DeltaWrapper> completedTaskDeltaMap;
    ​
        // ......
    ​
        /**
         * 采集 Micrometer 指标
         */
        private void micrometerMonitor(ThreadPoolRuntimeInfo runtimeInfo) {
            String threadPoolId = runtimeInfo.getThreadPoolId();
            ThreadPoolRuntimeInfo existingRuntimeInfo = micrometerMonitorCache.get(threadPoolId);
    ​
            // 只在首次注册时绑定 Gauge
            if (existingRuntimeInfo == null) {
                Iterable<Tag> tags = CollectionUtil.newArrayList(
                        Tag.of(DYNAMIC_THREAD_POOL_ID_TAG, threadPoolId),
                        Tag.of(APPLICATION_NAME_TAG, ApplicationProperties.getApplicationName())
                );
    ​
                ThreadPoolRuntimeInfo registerRuntimeInfo = BeanUtil.toBean(runtimeInfo, ThreadPoolRuntimeInfo.class);
                micrometerMonitorCache.put(threadPoolId, registerRuntimeInfo);
    ​
                // 注册总量指标
                Metrics.gauge(metricName("core.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getCorePoolSize);
                Metrics.gauge(metricName("maximum.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getMaximumPoolSize);
                Metrics.gauge(metricName("current.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getCurrentPoolSize);
                Metrics.gauge(metricName("largest.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getLargestPoolSize);
                Metrics.gauge(metricName("active.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getActivePoolSize);
                Metrics.gauge(metricName("queue.size"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getWorkQueueSize);
                Metrics.gauge(metricName("queue.capacity"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getWorkQueueCapacity);
                Metrics.gauge(metricName("queue.remaining.capacity"), tags, registerRuntimeInfo, ThreadPoolRuntimeInfo::getWorkQueueRemainingCapacity);
    ​
                // 注册 delta 指标
                DeltaWrapper completedDelta = new DeltaWrapper();
                completedTaskDeltaMap.put(threadPoolId, completedDelta);
                Metrics.gauge(metricName("completed.task.count"), tags, completedDelta, DeltaWrapper::getDelta);
    ​
                DeltaWrapper rejectDelta = new DeltaWrapper();
                rejectCountDeltaMap.put(threadPoolId, rejectDelta);
                Metrics.gauge(metricName("reject.count"), tags, rejectDelta, DeltaWrapper::getDelta);
            } else {
                // 更新属性（避免重新注册 Gauge）
                BeanUtil.copyProperties(runtimeInfo, existingRuntimeInfo);
            }
    ​
            // 每次都更新 delta 值
            completedTaskDeltaMap.get(threadPoolId).update(runtimeInfo.getCompletedTaskCount());
            rejectCountDeltaMap.get(threadPoolId).update(runtimeInfo.getRejectCount());
        }
        // ......
    }
              
这里新增了两个成员变量：

*  
rejectCountDeltaMap：保存每个线程池的拒绝次数增量计算器。  
*  
  completedTaskDeltaMap：保存每个线程池的任务完成增量计算器。

方法的执行逻辑如下：

* 1.  
获取 `threadPoolId` 和缓存的 runtime 对象，判断该线程池是否**已经注册过Gauge** 。如果没有，则进入首次注册逻辑。  
* 2.  
  首次注册逻辑（只执行一次）：
  * 1.  
  **复制一个独立对象`registerRuntimeInfo`** 用于绑定 Gauge（Micrometer 要求引用稳定）；  
  * 2.  
  放入缓存，后续只更新其字段，不更换引用。  
  * 3.  
  每个 Gauge 都会绑定 `registerRuntimeInfo` 的引用，并定期调用其 Getter 方法采样。  
  * 4.  
注册增量指标：使用 `DeltaWrapper` 包装类维护历史值，实现 "当前值 - 上次值" 的统计；将 `deltaMap` 存入缓存，便于后续更新。  
* 3.  
后续只更新绑定引用的字段值：保持 Micrometer Gauge 绑定对象不变，仅更新其属性值，确保指标能够刷新。  
* 4.  
  每轮采集都更新 delta 值：每次传入新的原始计数，`DeltaWrapper` 会自动计算出 delta（当前值 - 上次记录值）。

至此，我们解决了前面提到的三个问题，代码的实现也更加优雅了。

完结，撒花 🎉

