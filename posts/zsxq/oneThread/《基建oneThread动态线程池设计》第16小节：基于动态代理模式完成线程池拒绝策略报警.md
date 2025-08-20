2025年07月23日 20:11  
基于动态代理模式完成线程池拒绝策略报警，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



\***内容摘要：本文介绍如何基于** JDK 动态代理\*\*和 Lambda 轻量级静态代理，优雅地扩展线程池的拒绝策略行为，实现拒绝次数统计与准实时告警。

课程目录如下所示：

*  
前言  
*  
线程池拒绝任务场景  
*  
代理模式  
*  
动态代理  
*  
Lambda 轻量级静态代理  
*  
  文末总结

> 在阅读本章节前，大家先通过Breathe公众号的一篇文章 [MyBatis动态代理核心原理](https://mp.weixin.qq.com/s/_7NHd5asmo8DbGANyQM4Vw) 学习动态代理，下面文章将不再赘述。

前言
---

线程池作为任务并发处理的核心组件，其稳定性直接影响系统整体吞吐与响应能力。而线程池中的**拒绝策略** ，作为最后一道防线，往往代表了系统已出现短时瓶颈或配置不合理等问题。

为此，我们希望在拒绝任务发生的第一时间：

*  
上报报警，告知系统维护人员；  
*  
记录关键指标，便于事后分析与扩容调优；  
*  
  ......

现实比较骨感，JDK 线程池仅能完成拒绝基础逻辑，无法满足系统要求。下文将介绍如何优雅地使用代理模式，从静态代理逐步优化到动态代理，实现线程池拒绝任务的统计与实时告警功能。

线程池拒绝任务场景
---------

线程池拒绝任务有两个主要触发条件：

* 1.  
线程池状态非运行状态。  
* 2.  
  阻塞队列已满，且线程池线程数达到最大值。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
              
注意到上述方法为默认权限且被 `final` 修饰，因此不能直接扩展。

代理模式
----

尽管线程池的拒绝任务方法被设置为 `final` 且具有默认访问权限，导致我们**无法继承或重写该方法** ，但我们仍可以通过 **代理模式** 实现扩展功能。

**代理模式** （Proxy Design Pattern）是一种在**不修改原始类代码的前提下** ，通过引入代理对象对其行为进行增强的设计手段，非常适合用于功能增强、权限控制、延迟加载等场景。

![iShot_2025-07-19_13.07.31.png](https://article-images.zsxq.com/Fkg9PO-tIsoCBGRmPCOG--LP1SUj "iShot_2025-07-19_13.07.31.png")

接下来我们将逐步实现 **基于代理模式的拒绝策略扩展** ，包括拒绝次数统计与告警触发两个核心能力。

### 1. 扩展线程池 {#1}

静态代理模式需要创建多个具体实现类来增强原始拒绝策略，如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class SupportThreadPoolExecutor extends ThreadPoolExecutor {
    ​
        /** 拒绝次数统计 */
        private final AtomicInteger rejectCount = new AtomicInteger();
    ​
        public SupportThreadPoolExecutor(...) {
            super(...);
        }
    ​
        /** 拒绝次数自增 */
        public void incrementRejectCount() {
            rejectCount.incrementAndGet();
        }
    ​
        /** 获取当前拒绝次数 */
        public int getRejectCount() {
            return rejectCount.get();
        }
    }
              
### 2. 扩展拒绝策略 {#2}

接下来，我们通过定义一个通用的拒绝策略扩展接口，为后续具体策略提供统一的增强能力，包括：

*  
**拒绝次数统计** ；  
*  
  **报警通知触发** 。

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface SupportRejectedExecutionHandler extends RejectedExecutionHandler {
    ​
        /**
         * 拒绝策略前置处理逻辑：统计与告警。
         */
        default void beforeReject(ThreadPoolExecutor executor) {
            if (executor instanceof SupportThreadPoolExecutor) {
                SupportThreadPoolExecutor supportExecutor = (SupportThreadPoolExecutor) executor;
                // 拒绝次数自增
                supportExecutor.incrementRejectCount();
                // 执行告警逻辑（可替换为实际推送渠道）
                System.out.println("线程池触发了任务拒绝...");
            }
        }
    }
              
然后以 `AbortPolicy` 为例，实现一个具备扩展能力的拒绝策略类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class SupportAbortPolicyRejected extends ThreadPoolExecutor.AbortPolicy
            implements SupportRejectedExecutionHandler {
    ​
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            beforeReject(e); // 拒绝前执行扩展逻辑
            super.rejectedExecution(r, e); // 调用原始策略行为
        }
    }
              
### 3. 功能验证 {#3}

我们通过一个简单的测试用例，验证上述扩展拒绝策略是否能实现**拒绝统计+告警输出** 的预期功能：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @SneakyThrows
    public static void main(String[] args) {
        SupportThreadPoolExecutor executor = new SupportThreadPoolExecutor(
                1,
                1,
                1024,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(1),
                // 使用自定义的增强型拒绝策略
                new SupportAbortPolicyRejected()
        );
    ​
        // 提交 3 个任务，超过最大线程数和队列容量，触发拒绝
        for (int i = 0; i < 3; i++) {
            try {
                executor.execute(() -> Thread.sleep(Integer.MAX_VALUE));
            } catch (Exception ex) {
                // 忽略拒绝异常，专注验证统计与告警逻辑
            }
        }
    ​
        Thread.sleep(50);
        System.out.println(String.format("线程池拒绝次数统计 :: %d", executor.getRejectCount()));
    }
    ​
    // 控制台输出示例：
    线程池触发了任务拒绝...
    线程池拒绝次数统计 :: 1
              
从日志可以确认，**我们的扩展逻辑已成功生效** ：

*  
拒绝策略触发时，执行了 `beforeReject()` 中的统计与日志输出。  
*  
  线程池准确记录了被拒绝任务的次数。

### 4. 模式小结 {#4}

上述扩展方案采用的是一种经典的设计模式：**静态代理** 。建议大家在继续阅读之前，先在本地运行一遍示例代码，通过实践加深理解。

完成运行后，我们总结出一张图，帮助大家更直观地理解**静态代理的工作机制** ：

![iShot_2025-07-19_13.07.30.png](https://article-images.zsxq.com/FvtZIbjLfRueIB3hleGfdIK0M5xZ "iShot_2025-07-19_13.07.30.png")

通过拒绝策略的实战演练，大家对**静态代理的实现方式** 已经有了初步认识。但静态代理真的足够优雅吗？我们不妨冷静分析一下其局限性：

*  
**类爆炸问题** ：每一种拒绝策略都需要手动创建对应的代理类来实现增强逻辑。以 JDK 提供的四种默认策略为例，若都需要扩展，就得创建四个额外类，显然不符合**开闭原则** ，也不利于维护。  
*  
  **侵入性较高，增加系统复杂度** ：所有线程池都必须显式使用这些增强后的拒绝策略，一旦项目规模扩大或开发人员更替，容易遗漏代理逻辑或出现不一致实现，造成潜在的系统风险。

那么，如何在保证功能扩展的同时避免这些问题呢？答案是：**使用动态代理** 。

动态代理
----

**动态代理** 通过在**运行时动态生成代理类** ，实现对目标对象行为的增强，避免了对原始类的侵入式修改，从而更好地遵循了**开闭原则（对扩展开放，对修改关闭）** 。

### 1. 创建动态代理类 {#1}

实现了 `java.lang.reflect.InvocationHandler` 接口，是 JDK 动态代理机制的核心接口，用于定义代理方法的增强逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @AllArgsConstructor
    public class RejectedProxyInvocationHandler implements InvocationHandler {
    ​
        private final Object target;
        private final AtomicLong rejectCount;
    ​
        private static final String REJECT_METHOD = "rejectedExecution";
    ​
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (REJECT_METHOD.equals(method.getName()) &&
                    args != null &&
                    args.length == 2 &&
                    args[0] instanceof Runnable &&
                    args[1] instanceof ThreadPoolExecutor) {
                rejectCount.incrementAndGet();
            }
    ​
            if ("toString".equals(method.getName()) && method.getParameterCount() == 0) {
                return target.getClass().getSimpleName();
            }
    ​
            try {
                return method.invoke(target, args);
            } catch (InvocationTargetException ex) {
                throw ex.getCause();
            }
        }
    }
              
成员变量：

*  
`target`：被代理的对象，通常是某个具体的 `RejectedExecutionHandler` 实例。  
*  
`rejectCount`：线程安全的拒绝次数统计器，代理中每次拒绝都会自增。  
*  
  `REJECT_METHOD`：常量，用于快速判断是否是 `rejectedExecution` 方法。

核心方法 `invoke`：

* 1.  
  判断是否是拒绝方法：
  * 1.  
  判断当前调用的方法是否为 `rejectedExecution`；  
  * 2.  
  校验参数合法性：两个参数分别是 `Runnable` 和 `ThreadPoolExecutor`；  
  * 3.  
如果满足条件，表示线程池拒绝了一个任务，调用 `rejectCount.incrementAndGet()` 实现拒绝次数累加。  
* 2.  
  特殊处理 `toString` 方法：
  * 1.  
  为了避免动态代理对象打印出来是一堆代理类名，单独处理了 `toString()` 方法；  
  * 2.  
返回的是被代理对象的类名，便于日志或调试时识别原始拒绝策略类型。  
* 3.  
  反射调用原始逻辑：
  * 1.  
  最终通过反射调用原始 `RejectedExecutionHandler` 的方法，确保原有逻辑不被破坏；  
  * 2.  
    如果目标方法抛出异常，则抛出其原始异常（`getCause()`），保持行为一致。

如果使用了上述拒绝策略动态代理类，所有对 `rejectedExecution()` 的调用都被动态代理拦截，执行了额外的统计与告警逻辑。同时**保持了原生语义** ，最终还是会调用原始策略的逻辑（如 `AbortPolicy` 抛出异常），不影响线程池原有行为。

![iShot_2025-07-19_13.07.28.png](https://article-images.zsxq.com/FsXLrnU6eyA_amgbwFUu5MUptcbw "iShot_2025-07-19_13.07.28.png")

### 2. oneThread 线程池代理 {#2-one-thread}

在构建线程池时，我们依然保留用户自定义的原始拒绝策略，**不破坏其原有使用习惯** 。然后在 `oneThread` 的构造方法中，通过 **动态代理方式对拒绝策略进行增强处理** ，实现功能注入。

这种设计既保持了**用户体验的一致性** ，又实现了对线程池拒绝行为的**透明增强** ，兼顾了**兼容性与可扩展性** 。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 增强的动态、报警和受监控的线程池 oneThread
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    @Slf4j
    public class OneThreadExecutor extends ThreadPoolExecutor {
    ​
        // ......
    ​
        /**
         * 线程池拒绝策略执行次数
         */
        @Getter
        private final AtomicLong rejectCount = new AtomicLong();
    ​
        // ......
    ​
        public OneThreadExecutor(
                @NonNull String threadPoolId,
                int corePoolSize,
                int maximumPoolSize,
                long keepAliveTime,
                @NonNull TimeUnit unit,
                @NonNull BlockingQueue<Runnable> workQueue,
                @NonNull ThreadFactory threadFactory,
                @NonNull RejectedExecutionHandler handler,
                long awaitTerminationMillis) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    ​
            // 通过动态代理设置拒绝策略执行次数
            setRejectedExecutionHandler(handler);
    ​
            // ......
        }
    ​
        @Override
        public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
            RejectedExecutionHandler rejectedProxy = (RejectedExecutionHandler) Proxy
                    .newProxyInstance(
                            handler.getClass().getClassLoader(),
                            new Class[]{RejectedExecutionHandler.class},
                            new RejectedProxyInvocationHandler(handler, rejectCount)
                    );
            super.setRejectedExecutionHandler(rejectedProxy);
        }
    }
              
结构图如下所示：

![iShot_2025-07-19_13.07.29.png](https://article-images.zsxq.com/FpQc5Tah6WRWXLmXB5S073zPtdrN "iShot_2025-07-19_13.07.29.png")

### 3. 拒绝策略告警检查 {#3}

细心的同学可能已经注意到：前面的流程中，我们只是**统计了拒绝策略的执行次数** ，那么------**告警到底是在哪触发的呢** ？

在我最初设计 Hippo4j 时，采取的是一种"即时告警"的思路：**在拒绝策略被触发的那一刻，立即执行告警逻辑** 。

构思 oneThread 时，我开始反思：**真的需要这么高的实时性吗？** 答案是否定的。绝大多数情况下，我们更关注的是\*\*"有没有拒绝发生"\*\* ，而不是它发生的**精确时刻** 。

于是，我采用了另一种实现思路 ------**通过定时任务定期扫描拒绝次数** ：如果某个线程池的拒绝次数与上一次记录不一致，即视为出现了新的拒绝行为，再触发告警逻辑。  
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
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(
                1,
                ThreadFactoryBuilder.builder()
                        .namePrefix("scheduler_thread-pool_alarm_checker")
                        .build()
        );
        private final Map<String, Long> lastRejectCountMap = new ConcurrentHashMap<>();
    ​
        // ......  
    ​
        /**
         * 报警检查核心逻辑
         */
        private void checkAlarm() {
            Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
            for (ThreadPoolExecutorHolder holder : holders) {
                if (holder.getExecutorProperties().getAlarm().getEnable()) {
                    // ......
                    checkRejectCount(holder);
                }
            }
        }
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
    ​
        // ......
    }
              
Lambda 轻量级静态代理 {#lambda}
------------------------

### 1. 扩展实现 {#1}

上面的动态代理实现相信大家已经看懂了。在实现完成后，我也在思考一个问题：
> 我们的场景其实非常简单，远不如 MyBatis 那样需要支持用户定义多种 Mapper 接口、集成 Spring 等复杂能力，确实有必要搞到"动态代理"这么重吗？

毕竟我们只是想对固定的拒绝策略做一层增强逻辑。

于是我尝试用 Lambda 实现了一个更**轻量级** 的方案，不依赖反射，也不需要额外的代理类，仅通过一层静态包装就完成了功能增强。下面是具体实现，供大家参考：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    public class OneThreadExecutor extends ThreadPoolExecutor {
    ​
        // ......
    ​
        /**
         * 线程池拒绝策略执行次数
         */
        @Getter
        private final AtomicLong rejectCount = new AtomicLong();
    ​
        // ......
    ​
        public OneThreadExecutor(
                @NonNull String threadPoolId,
                int corePoolSize,
                int maximumPoolSize,
                long keepAliveTime,
                @NonNull TimeUnit unit,
                @NonNull BlockingQueue<Runnable> workQueue,
                @NonNull ThreadFactory threadFactory,
                @NonNull RejectedExecutionHandler handler,
                long awaitTerminationMillis) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    ​
            // 通过 Lambda 静态包装设置拒绝策略执行次数
            setRejectedExecutionHandler(handler);
    ​
            // ......
        }
    ​
        @Override
        public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
            super.setRejectedExecutionHandler((r, executor) -> {
                rejectCount.incrementAndGet();
                handler.rejectedExecution(r, executor);
            });
        }
    }
              
上面这种方案看着没问题，其实还有点小瑕疵，那就是告警时候获取拒绝策略名称是下面这种：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    com.nageoffer.onethread.core.executor.OneThreadExecutor$$Lambda$758/0x0000000801634fc0@54ceb23d
              
还需要做拒绝策略的 toString 方法重写，完整代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Override
    public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        RejectedExecutionHandler handlerWrapper = new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                rejectCount.incrementAndGet();
                handler.rejectedExecution(r, executor);
            }
    ​
            @Override
            public String toString() {
                return handler.getClass().getSimpleName();
            }
        };
    ​
        super.setRejectedExecutionHandler(handlerWrapper);
    }
              
### 2. Lambda vs 动态代理 {#2-lambda-vs}

在本场景中，使用动态代理的方式是合理的，它不仅满足功能需求，同时也具备一定的实战训练价值，属于"学习层面上的花活"。然而在实际工程落地过程中，我们始终坚持一个原则：**简单即正义，优先选择可读性强、维护成本低的实现方式** 。

因此，在 oneThread 的核心实现中，我们采用了更加简洁直观的 **Lambda静态包装方案** 来增强拒绝策略逻辑。与此同时，我们仍保留了基于动态代理的实现作为学习参考，集成在 `OneThreadExecutor` 类中，供有更高定制需求或对代理机制感兴趣的开发者深入研究和拓展。

### 3. Lambda 包装是静态代理么？ {#3-lambda}

Lambda 和静态代理本质上都做了**相同的事情** ：在调用真实逻辑（如 `rejectedExecution`）之前或之后，**插入一段增强逻辑** 。所以从行为角度来看，Lambda 实现的包装就是一种"静态代理"。

有相同点也有不同点：  

|  对比维度   |    传统静态代理     |               Lambda 静态包装               |
|---------|---------------|-----------------------------------------|
| 是否需要代理类 | ✅ 是（要写一个实现类）  | ❌ 否（用闭包或匿名函数直接实现）                       |
| 可复用性    | ✅ 高（代理类可用于多处） | ⚠ 限于作用域（适合局部包装）                         |
| 可读性     | ❌ 容易产生样板代码    | ✅ 简洁清晰                                  |
| 接口要求    | 可代理多个方法       | 仅适用于函数式接口（如 `RejectedExecutionHandler`） |

大家可以这么理解：Lambda 实现本质上是一种"轻量级的静态代理"，特别适用于函数式接口的包装增强场景。

文末总结
----

本文围绕线程池拒绝策略的扩展需求，从 JDK 限制出发，深入探讨了静态代理、动态代理与 Lambda 包装三种不同实现路径，逐步构建出一套既可增强拒绝统计与告警能力、又不破坏原有策略行为的通用方案。

在实践过程中，不仅讲述了代理模式的核心思想，也思考了设计复杂度与工程可维护性的权衡：动态代理适合通用封装与统一拦截，Lambda 则在轻量场景中具备更高性价比。最终，oneThread 选择将 Lambda 用于核心逻辑实现，保留动态代理作为学习方案，实现功能、性能与开发体验的平衡。希望这篇文章能帮助大家理解代理增强在实际项目中的应用价值。

完结，撒花 🎉  
基于动态代理模式完成线程池拒绝策略报警，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



\***内容摘要：本文介绍如何基于** JDK 动态代理\*\*和 Lambda 轻量级静态代理，优雅地扩展线程池的拒绝策略行为，实现拒绝次数统计与准实时告警。

课程目录如下所示：

*  
前言  
*  
线程池拒绝任务场景  
*  
代理模式  
*  
动态代理  
*  
Lambda 轻量级静态代理  
*  
  文末总结

> 在阅读本章节前，大家先通过Breathe公众号的一篇文章 [MyBatis动态代理核心原理](https://mp.weixin.qq.com/s/_7NHd5asmo8DbGANyQM4Vw) 学习动态代理，下面文章将不再赘述。

前言
---

线程池作为任务并发处理的核心组件，其稳定性直接影响系统整体吞吐与响应能力。而线程池中的**拒绝策略** ，作为最后一道防线，往往代表了系统已出现短时瓶颈或配置不合理等问题。

为此，我们希望在拒绝任务发生的第一时间：

*  
上报报警，告知系统维护人员；  
*  
记录关键指标，便于事后分析与扩容调优；  
*  
  ......

现实比较骨感，JDK 线程池仅能完成拒绝基础逻辑，无法满足系统要求。下文将介绍如何优雅地使用代理模式，从静态代理逐步优化到动态代理，实现线程池拒绝任务的统计与实时告警功能。

线程池拒绝任务场景
---------

线程池拒绝任务有两个主要触发条件：

* 1.  
线程池状态非运行状态。  
* 2.  
  阻塞队列已满，且线程池线程数达到最大值。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
              
注意到上述方法为默认权限且被 `final` 修饰，因此不能直接扩展。

代理模式
----

尽管线程池的拒绝任务方法被设置为 `final` 且具有默认访问权限，导致我们**无法继承或重写该方法** ，但我们仍可以通过 **代理模式** 实现扩展功能。

**代理模式** （Proxy Design Pattern）是一种在**不修改原始类代码的前提下** ，通过引入代理对象对其行为进行增强的设计手段，非常适合用于功能增强、权限控制、延迟加载等场景。

![iShot_2025-07-19_13.07.31.png](https://article-images.zsxq.com/Fkg9PO-tIsoCBGRmPCOG--LP1SUj "iShot_2025-07-19_13.07.31.png")

接下来我们将逐步实现 **基于代理模式的拒绝策略扩展** ，包括拒绝次数统计与告警触发两个核心能力。

### 1. 扩展线程池 {#1}

静态代理模式需要创建多个具体实现类来增强原始拒绝策略，如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class SupportThreadPoolExecutor extends ThreadPoolExecutor {
    ​
        /** 拒绝次数统计 */
        private final AtomicInteger rejectCount = new AtomicInteger();
    ​
        public SupportThreadPoolExecutor(...) {
            super(...);
        }
    ​
        /** 拒绝次数自增 */
        public void incrementRejectCount() {
            rejectCount.incrementAndGet();
        }
    ​
        /** 获取当前拒绝次数 */
        public int getRejectCount() {
            return rejectCount.get();
        }
    }
              
### 2. 扩展拒绝策略 {#2}

接下来，我们通过定义一个通用的拒绝策略扩展接口，为后续具体策略提供统一的增强能力，包括：

*  
**拒绝次数统计** ；  
*  
  **报警通知触发** 。

代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface SupportRejectedExecutionHandler extends RejectedExecutionHandler {
    ​
        /**
         * 拒绝策略前置处理逻辑：统计与告警。
         */
        default void beforeReject(ThreadPoolExecutor executor) {
            if (executor instanceof SupportThreadPoolExecutor) {
                SupportThreadPoolExecutor supportExecutor = (SupportThreadPoolExecutor) executor;
                // 拒绝次数自增
                supportExecutor.incrementRejectCount();
                // 执行告警逻辑（可替换为实际推送渠道）
                System.out.println("线程池触发了任务拒绝...");
            }
        }
    }
              
然后以 `AbortPolicy` 为例，实现一个具备扩展能力的拒绝策略类：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class SupportAbortPolicyRejected extends ThreadPoolExecutor.AbortPolicy
            implements SupportRejectedExecutionHandler {
    ​
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            beforeReject(e); // 拒绝前执行扩展逻辑
            super.rejectedExecution(r, e); // 调用原始策略行为
        }
    }
              
### 3. 功能验证 {#3}

我们通过一个简单的测试用例，验证上述扩展拒绝策略是否能实现**拒绝统计+告警输出** 的预期功能：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @SneakyThrows
    public static void main(String[] args) {
        SupportThreadPoolExecutor executor = new SupportThreadPoolExecutor(
                1,
                1,
                1024,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(1),
                // 使用自定义的增强型拒绝策略
                new SupportAbortPolicyRejected()
        );
    ​
        // 提交 3 个任务，超过最大线程数和队列容量，触发拒绝
        for (int i = 0; i < 3; i++) {
            try {
                executor.execute(() -> Thread.sleep(Integer.MAX_VALUE));
            } catch (Exception ex) {
                // 忽略拒绝异常，专注验证统计与告警逻辑
            }
        }
    ​
        Thread.sleep(50);
        System.out.println(String.format("线程池拒绝次数统计 :: %d", executor.getRejectCount()));
    }
    ​
    // 控制台输出示例：
    线程池触发了任务拒绝...
    线程池拒绝次数统计 :: 1
              
从日志可以确认，**我们的扩展逻辑已成功生效** ：

*  
拒绝策略触发时，执行了 `beforeReject()` 中的统计与日志输出。  
*  
  线程池准确记录了被拒绝任务的次数。

### 4. 模式小结 {#4}

上述扩展方案采用的是一种经典的设计模式：**静态代理** 。建议大家在继续阅读之前，先在本地运行一遍示例代码，通过实践加深理解。

完成运行后，我们总结出一张图，帮助大家更直观地理解**静态代理的工作机制** ：

![iShot_2025-07-19_13.07.30.png](https://article-images.zsxq.com/FvtZIbjLfRueIB3hleGfdIK0M5xZ "iShot_2025-07-19_13.07.30.png")

通过拒绝策略的实战演练，大家对**静态代理的实现方式** 已经有了初步认识。但静态代理真的足够优雅吗？我们不妨冷静分析一下其局限性：

*  
**类爆炸问题** ：每一种拒绝策略都需要手动创建对应的代理类来实现增强逻辑。以 JDK 提供的四种默认策略为例，若都需要扩展，就得创建四个额外类，显然不符合**开闭原则** ，也不利于维护。  
*  
  **侵入性较高，增加系统复杂度** ：所有线程池都必须显式使用这些增强后的拒绝策略，一旦项目规模扩大或开发人员更替，容易遗漏代理逻辑或出现不一致实现，造成潜在的系统风险。

那么，如何在保证功能扩展的同时避免这些问题呢？答案是：**使用动态代理** 。

动态代理
----

**动态代理** 通过在**运行时动态生成代理类** ，实现对目标对象行为的增强，避免了对原始类的侵入式修改，从而更好地遵循了**开闭原则（对扩展开放，对修改关闭）** 。

### 1. 创建动态代理类 {#1}

实现了 `java.lang.reflect.InvocationHandler` 接口，是 JDK 动态代理机制的核心接口，用于定义代理方法的增强逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @AllArgsConstructor
    public class RejectedProxyInvocationHandler implements InvocationHandler {
    ​
        private final Object target;
        private final AtomicLong rejectCount;
    ​
        private static final String REJECT_METHOD = "rejectedExecution";
    ​
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (REJECT_METHOD.equals(method.getName()) &&
                    args != null &&
                    args.length == 2 &&
                    args[0] instanceof Runnable &&
                    args[1] instanceof ThreadPoolExecutor) {
                rejectCount.incrementAndGet();
            }
    ​
            if ("toString".equals(method.getName()) && method.getParameterCount() == 0) {
                return target.getClass().getSimpleName();
            }
    ​
            try {
                return method.invoke(target, args);
            } catch (InvocationTargetException ex) {
                throw ex.getCause();
            }
        }
    }
              
成员变量：

*  
`target`：被代理的对象，通常是某个具体的 `RejectedExecutionHandler` 实例。  
*  
`rejectCount`：线程安全的拒绝次数统计器，代理中每次拒绝都会自增。  
*  
  `REJECT_METHOD`：常量，用于快速判断是否是 `rejectedExecution` 方法。

核心方法 `invoke`：

* 1.  
  判断是否是拒绝方法：
  * 1.  
  判断当前调用的方法是否为 `rejectedExecution`；  
  * 2.  
  校验参数合法性：两个参数分别是 `Runnable` 和 `ThreadPoolExecutor`；  
  * 3.  
如果满足条件，表示线程池拒绝了一个任务，调用 `rejectCount.incrementAndGet()` 实现拒绝次数累加。  
* 2.  
  特殊处理 `toString` 方法：
  * 1.  
  为了避免动态代理对象打印出来是一堆代理类名，单独处理了 `toString()` 方法；  
  * 2.  
返回的是被代理对象的类名，便于日志或调试时识别原始拒绝策略类型。  
* 3.  
  反射调用原始逻辑：
  * 1.  
  最终通过反射调用原始 `RejectedExecutionHandler` 的方法，确保原有逻辑不被破坏；  
  * 2.  
    如果目标方法抛出异常，则抛出其原始异常（`getCause()`），保持行为一致。

如果使用了上述拒绝策略动态代理类，所有对 `rejectedExecution()` 的调用都被动态代理拦截，执行了额外的统计与告警逻辑。同时**保持了原生语义** ，最终还是会调用原始策略的逻辑（如 `AbortPolicy` 抛出异常），不影响线程池原有行为。

![iShot_2025-07-19_13.07.28.png](https://article-images.zsxq.com/FsXLrnU6eyA_amgbwFUu5MUptcbw "iShot_2025-07-19_13.07.28.png")

### 2. oneThread 线程池代理 {#2-one-thread}

在构建线程池时，我们依然保留用户自定义的原始拒绝策略，**不破坏其原有使用习惯** 。然后在 `oneThread` 的构造方法中，通过 **动态代理方式对拒绝策略进行增强处理** ，实现功能注入。

这种设计既保持了**用户体验的一致性** ，又实现了对线程池拒绝行为的**透明增强** ，兼顾了**兼容性与可扩展性** 。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 增强的动态、报警和受监控的线程池 oneThread
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    @Slf4j
    public class OneThreadExecutor extends ThreadPoolExecutor {
    ​
        // ......
    ​
        /**
         * 线程池拒绝策略执行次数
         */
        @Getter
        private final AtomicLong rejectCount = new AtomicLong();
    ​
        // ......
    ​
        public OneThreadExecutor(
                @NonNull String threadPoolId,
                int corePoolSize,
                int maximumPoolSize,
                long keepAliveTime,
                @NonNull TimeUnit unit,
                @NonNull BlockingQueue<Runnable> workQueue,
                @NonNull ThreadFactory threadFactory,
                @NonNull RejectedExecutionHandler handler,
                long awaitTerminationMillis) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    ​
            // 通过动态代理设置拒绝策略执行次数
            setRejectedExecutionHandler(handler);
    ​
            // ......
        }
    ​
        @Override
        public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
            RejectedExecutionHandler rejectedProxy = (RejectedExecutionHandler) Proxy
                    .newProxyInstance(
                            handler.getClass().getClassLoader(),
                            new Class[]{RejectedExecutionHandler.class},
                            new RejectedProxyInvocationHandler(handler, rejectCount)
                    );
            super.setRejectedExecutionHandler(rejectedProxy);
        }
    }
              
结构图如下所示：

![iShot_2025-07-19_13.07.29.png](https://article-images.zsxq.com/FpQc5Tah6WRWXLmXB5S073zPtdrN "iShot_2025-07-19_13.07.29.png")

### 3. 拒绝策略告警检查 {#3}

细心的同学可能已经注意到：前面的流程中，我们只是**统计了拒绝策略的执行次数** ，那么------**告警到底是在哪触发的呢** ？

在我最初设计 Hippo4j 时，采取的是一种"即时告警"的思路：**在拒绝策略被触发的那一刻，立即执行告警逻辑** 。

构思 oneThread 时，我开始反思：**真的需要这么高的实时性吗？** 答案是否定的。绝大多数情况下，我们更关注的是\*\*"有没有拒绝发生"\*\* ，而不是它发生的**精确时刻** 。

于是，我采用了另一种实现思路 ------**通过定时任务定期扫描拒绝次数** ：如果某个线程池的拒绝次数与上一次记录不一致，即视为出现了新的拒绝行为，再触发告警逻辑。  
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
        private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(
                1,
                ThreadFactoryBuilder.builder()
                        .namePrefix("scheduler_thread-pool_alarm_checker")
                        .build()
        );
        private final Map<String, Long> lastRejectCountMap = new ConcurrentHashMap<>();
    ​
        // ......  
    ​
        /**
         * 报警检查核心逻辑
         */
        private void checkAlarm() {
            Collection<ThreadPoolExecutorHolder> holders = OneThreadRegistry.getAllHolders();
            for (ThreadPoolExecutorHolder holder : holders) {
                if (holder.getExecutorProperties().getAlarm().getEnable()) {
                    // ......
                    checkRejectCount(holder);
                }
            }
        }
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
    ​
        // ......
    }
              
Lambda 轻量级静态代理 {#lambda}
------------------------

### 1. 扩展实现 {#1}

上面的动态代理实现相信大家已经看懂了。在实现完成后，我也在思考一个问题：
> 我们的场景其实非常简单，远不如 MyBatis 那样需要支持用户定义多种 Mapper 接口、集成 Spring 等复杂能力，确实有必要搞到"动态代理"这么重吗？

毕竟我们只是想对固定的拒绝策略做一层增强逻辑。

于是我尝试用 Lambda 实现了一个更**轻量级** 的方案，不依赖反射，也不需要额外的代理类，仅通过一层静态包装就完成了功能增强。下面是具体实现，供大家参考：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Slf4j
    public class OneThreadExecutor extends ThreadPoolExecutor {
    ​
        // ......
    ​
        /**
         * 线程池拒绝策略执行次数
         */
        @Getter
        private final AtomicLong rejectCount = new AtomicLong();
    ​
        // ......
    ​
        public OneThreadExecutor(
                @NonNull String threadPoolId,
                int corePoolSize,
                int maximumPoolSize,
                long keepAliveTime,
                @NonNull TimeUnit unit,
                @NonNull BlockingQueue<Runnable> workQueue,
                @NonNull ThreadFactory threadFactory,
                @NonNull RejectedExecutionHandler handler,
                long awaitTerminationMillis) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    ​
            // 通过 Lambda 静态包装设置拒绝策略执行次数
            setRejectedExecutionHandler(handler);
    ​
            // ......
        }
    ​
        @Override
        public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
            super.setRejectedExecutionHandler((r, executor) -> {
                rejectCount.incrementAndGet();
                handler.rejectedExecution(r, executor);
            });
        }
    }
              
上面这种方案看着没问题，其实还有点小瑕疵，那就是告警时候获取拒绝策略名称是下面这种：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    com.nageoffer.onethread.core.executor.OneThreadExecutor$$Lambda$758/0x0000000801634fc0@54ceb23d
              
还需要做拒绝策略的 toString 方法重写，完整代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @Override
    public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        RejectedExecutionHandler handlerWrapper = new RejectedExecutionHandler() {
            @Override
            public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                rejectCount.incrementAndGet();
                handler.rejectedExecution(r, executor);
            }
    ​
            @Override
            public String toString() {
                return handler.getClass().getSimpleName();
            }
        };
    ​
        super.setRejectedExecutionHandler(handlerWrapper);
    }
              
### 2. Lambda vs 动态代理 {#2-lambda-vs}

在本场景中，使用动态代理的方式是合理的，它不仅满足功能需求，同时也具备一定的实战训练价值，属于"学习层面上的花活"。然而在实际工程落地过程中，我们始终坚持一个原则：**简单即正义，优先选择可读性强、维护成本低的实现方式** 。

因此，在 oneThread 的核心实现中，我们采用了更加简洁直观的 **Lambda静态包装方案** 来增强拒绝策略逻辑。与此同时，我们仍保留了基于动态代理的实现作为学习参考，集成在 `OneThreadExecutor` 类中，供有更高定制需求或对代理机制感兴趣的开发者深入研究和拓展。

### 3. Lambda 包装是静态代理么？ {#3-lambda}

Lambda 和静态代理本质上都做了**相同的事情** ：在调用真实逻辑（如 `rejectedExecution`）之前或之后，**插入一段增强逻辑** 。所以从行为角度来看，Lambda 实现的包装就是一种"静态代理"。

有相同点也有不同点：  

|  对比维度   |    传统静态代理     |               Lambda 静态包装               |
|---------|---------------|-----------------------------------------|
| 是否需要代理类 | ✅ 是（要写一个实现类）  | ❌ 否（用闭包或匿名函数直接实现）                       |
| 可复用性    | ✅ 高（代理类可用于多处） | ⚠ 限于作用域（适合局部包装）                         |
| 可读性     | ❌ 容易产生样板代码    | ✅ 简洁清晰                                  |
| 接口要求    | 可代理多个方法       | 仅适用于函数式接口（如 `RejectedExecutionHandler`） |

大家可以这么理解：Lambda 实现本质上是一种"轻量级的静态代理"，特别适用于函数式接口的包装增强场景。

文末总结
----

本文围绕线程池拒绝策略的扩展需求，从 JDK 限制出发，深入探讨了静态代理、动态代理与 Lambda 包装三种不同实现路径，逐步构建出一套既可增强拒绝统计与告警能力、又不破坏原有策略行为的通用方案。

在实践过程中，不仅讲述了代理模式的核心思想，也思考了设计复杂度与工程可维护性的权衡：动态代理适合通用封装与统一拦截，Lambda 则在轻量场景中具备更高性价比。最终，oneThread 选择将 Lambda 用于核心逻辑实现，保留动态代理作为学习方案，实现功能、性能与开发体验的平衡。希望这篇文章能帮助大家理解代理增强在实际项目中的应用价值。

完结，撒花 🎉

