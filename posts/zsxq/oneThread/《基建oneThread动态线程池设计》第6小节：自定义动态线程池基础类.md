2025年07月12日 22:46  
自定义动态线程池基础类，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：核心设计的提前规划与整体构思，是构建高质量动态线程池框架的基石，能够有效避免后期大幅返工，同时为系统的灵活扩展和维护奠定坚实基础。

课程目录如下所示：

*  
核心设计概述  
*  
关键类详解  
*  
Builder 设计模式  
*  
  文末总结

核心设计概览
------

### 1. 设计思想 {#1}

在上一篇文章中，我们已经初步了解了我在设计基础组件库时秉持的核心理念：**尽可能减少对三方框架的依赖** 。

秉持这一设计思路，在最初开发 `oneThread` 框架时，我将大部分与第三方框架无关的通用逻辑，统一抽象封装在了 `onethread-core` 模块中。

该模块主要承担以下核心职责：

*  
定义动态线程池的基础抽象类；  
*  
执行线程池运行时的告警扫描逻辑；  
*  
提供线程池运行状态的指标采集能力；  
*  
  支持监听配置中心变更，并动态调整线程池参数及触发告警通知。

下面是 `onethread-core` 模块的 `pom.xml` 依赖，可以看到它**并未强制依赖任何Spring相关的包** ，保持了良好的独立性和可移植性：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>com.nageoffer.onethread</groupId>
            <artifactId>onethread-all</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </parent>
        <artifactId>onethread-core</artifactId>
    ​
        <dependencies>
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>com.alibaba.fastjson2</groupId>
                <artifactId>fastjson2</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>org.yaml</groupId>
                <artifactId>snakeyaml</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
    ​
            <dependency>
                <groupId>ch.qos.logback</groupId>
                <artifactId>logback-classic</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>io.micrometer</groupId>
                <artifactId>micrometer-core</artifactId>
            </dependency>
        </dependencies>
    </project>
              
有同学可能会疑惑：**为什么上述依赖中没有显式指定版本号？**

这是因为我们在项目的根目录（通常是 `parent` 项目的 `pom.xml`）中，已经通过 `<dependencyManagement>` 统一定义了各依赖的版本号。因此，在 `onethread-core` 模块中只需声明依赖的坐标，无需重复指定版本，Maven 会自动从父模块继承对应的版本信息。这样不仅提高了依赖管理的统一性，也有助于减少版本冲突风险。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.0.7</spring-boot.version>
        <spring-cloud.version>2022.0.3</spring-cloud.version>
        <spring-cloud-alibaba.version>2022.0.0.0-RC2</spring-cloud-alibaba.version>
        <hutool-all.version>5.8.37</hutool-all.version>
        <apollo-client-config-data.version>2.4.0</apollo-client-config-data.version>
        <fastjson2.version>2.0.57</fastjson2.version>
        <spotless-maven-plugin.version>2.22.1</spotless-maven-plugin.version>
        <maven-compiler-plugin.version>3.6.1</maven-compiler-plugin.version>
    </properties>
    ​
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    ​
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    ​
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    ​
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>${hutool-all.version}</version>
            </dependency>
    ​
            <dependency>
                <groupId>com.ctrip.framework.apollo</groupId>
                <artifactId>apollo-client-config-data</artifactId>
                <version>${apollo-client-config-data.version}</version>
            </dependency>
    ​
            <dependency>
                <groupId>com.alibaba.fastjson2</groupId>
                <artifactId>fastjson2</artifactId>
                <version>${fastjson2.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
              
### 2. package 说明 {#2-package}

在前面的文章中，我们已经介绍过 `oneThread` 框架中各个模块（Module）所承担的职责。但当时并未展开到更细粒度的 **package层级结构** 。

本文将进一步说明 `onethread-core` 模块中各个包的具体作用，帮助大家更清晰地理解其内部结构与设计意图。

具体包划分与功能如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    ├── src
    │   ├── main
    │   │   └── java
    │   │       └── com
    │   │           └── nageoffer
    │   │               └── onethread
    │   │                   └── core
    │   │                       ├── alarm # 告警扫描
    │   │                       ├── config # 组件库基础配置
    │   │                       ├── constant # 常量类
    │   │                       ├── executor # 最最核心的线程池基础包
    │   │                       │   └── support # 核心基础包至上增强的功能，如果不拆分这个也行，但是类会比较多
    │   │                       ├── monitor # 运行时指标监控
    │   │                       ├── notification # 线程池配置变更和告警通知
    │   │                       │   ├── dto
    │   │                       │   └── service
    │   │                       ├── parser # 解析配置内容字符串为键值对 Map
    │   │                       └── toolkit # 工具包，比如：线程池和线程工厂构建者
              
大家最需要关注的是 alarm、executor、monitor、notification 四个包。

### 3. 类图设计 {#3}

在 `core` 包中，有四个最核心的类，它们构成了整个动态线程池功能的基础。其关系如下所示：

![iShot_2025-06-26_20.53.54.png](https://article-images.zsxq.com/FkqM-rk3f8UxBzTRHg1YzyUgfMrb "iShot_2025-06-26_20.53.54.png")

关键类详解
-----

### 1. OneThreadExecutor {#1-one-thread-executor}

有同学可能会问：**为什么要单独定义一个线程池基类，而不是直接使用原生的`ThreadPoolExecutor`呢？**

原因有很多，但为了循序渐进地展开说明，这里先讲其中一个最关键的点：
> 原生线程池并没有线程池 ID 或名称的概念。

在实际业务中，我们往往需要对线程池进行运行时变更、指标监控、告警通知等操作。而缺乏唯一标识，会让我们在面对多个线程池时难以准确定位、识别和管理。

因此，为了支持更强的可观测性与可运维性，我们抽象出了具备线程池标识能力的基类，作为后续扩展的统一入口。

值得一提的是，`ThreadPoolExecutor` 本身并不排斥继承，反而在设计上为继承扩展预留了几个关键的扩展点方法。通过继承并重写这些方法，我们可以轻松注入自定义行为，构建更智能的线程池体系：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 任务执行前钩子（可用于记录开始时间、设置线程上下文等）
     */
    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        // 自定义逻辑
    }
    ​
    /**
     * 任务执行后钩子（可用于统计执行耗时、清理资源、日志记录等）
     */
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        // 自定义逻辑
    }
    ​
    /**
     * 线程池终止钩子（可用于清理资源、发送通知等）
     */
    @Override
    protected void terminated() {
        super.terminated();
        // 自定义逻辑
    }
    ​
              
下面是 `oneThread` 的线程池基类，目前我们只在其中**集成了线程池唯一标识** 这一核心能力。

在阅读源码时，建议大家暂时聚焦这一功能点。至于类中其他相对复杂的逻辑，如果一时看不太懂也没关系，**与其陷入细节，不如先跟着Breathe一起按"主线思路"一步步深入理解。**  
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
        /**
         * 线程池唯一标识，用来动态变更参数等
         */
        @Getter
        private final String threadPoolId;
        // ......
    ​
        /**
         * Creates a new {@code ExtensibleThreadPoolExecutor} with the given initial parameters.
         *
         * @param threadPoolId           thread-pool id
         * @param corePoolSize           the number of threads to keep in the pool, even
         *                               if they are idle, unless {@code allowCoreThreadTimeOut} is set
         * @param maximumPoolSize        the maximum number of threads to allow in the
         *                               pool
         * @param keepAliveTime          when the number of threads is greater than
         *                               the core, this is the maximum time that excess idle threads
         *                               will wait for new tasks before terminating.
         * @param unit                   the time unit for the {@code keepAliveTime} argument
         * @param workQueue              the queue to use for holding tasks before they are
         *                               executed.  This queue will hold only the {@code Runnable}
         *                               tasks submitted by the {@code execute} method.
         * @param threadFactory          the factory to use when the executor
         *                               creates a new thread
         * @param handler                the handler to use when execution is blocked
         *                               because the thread bounds and queue capacities are reached
         * @param awaitTerminationMillis the maximum time to wait
         * @throws IllegalArgumentException if one of the following holds:<br>
         *                                  {@code corePoolSize < 0}<br>
         *                                  {@code keepAliveTime < 0}<br>
         *                                  {@code maximumPoolSize <= 0}<br>
         *                                  {@code maximumPoolSize < corePoolSize}
         * @throws NullPointerException     if {@code workQueue} or {@code unit}
         *                                  or {@code threadFactory} or {@code handler} is null
         */
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
            // ......
            // 设置动态线程池扩展属性：线程池 ID 标识
            this.threadPoolId = threadPoolId;
            // ......
        }
    ​
        // ......
    }
              
### 2. ThreadPoolExecutorProperties {#2-thread-pool-executor-properties}

有同学可能会问：**线程池的参数明明已经在`OneThreadExecutor`中定义了，为什么还要单独拆分出一个属性实体类？**

这是因为线程池中的参数通常**分散在多个字段中** ，并且像**告警阈值、通知规则等配置** 在原生线程池中是根本不存在的。为了更好地支持动态配置和统一管理，我们通过 `threadPoolId` 将这些"外围参数"与线程池进行关联，形成一套完整的配置体系，便于在运行时查看、存储、变更和追踪。

另外，为了Breathe写代码方便，告警和通知相关的参数类目前是以**静态内部类** 的形式存在。大家在实际开发中，也可以根据需要将其**拆分为独立的外部类** ，以提升可复用性。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池属性参数
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @Accessors(chain = true)
    public class ThreadPoolExecutorProperties {
    ​
        /**
         * 线程池唯一标识
         */
        private String threadPoolId;
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
         * 队列容量
         */
        private Integer queueCapacity;
    ​
        /**
         * 阻塞队列类型
         */
        private String workQueue;
    ​
        /**
         * 拒绝策略类型
         */
        private String rejectedHandler;
    ​
        /**
         * 线程空闲存活时间（单位：秒）
         */
        private Long keepAliveTime;
    ​
        /**
         * 是否允许核心线程超时
         */
        private Boolean allowCoreThreadTimeOut;
    ​
        /**
         * 通知配置
         */
        private NotifyConfig notify;
    ​
        /**
         * 报警配置，默认设置
         */
        private AlarmConfig alarm = new AlarmConfig();
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
    ​
            /**
             * 告警间隔，单位分钟
             */
            private Integer interval = 5;
        }
    ​
        @Data
        @NoArgsConstructor
        @AllArgsConstructor
        public static class AlarmConfig {
    ​
            /**
             * 默认开启报警配配置
             */
            private Boolean enable = Boolean.TRUE;
    ​
            /**
             * 队列阈值
             */
            private Integer queueThreshold = 80;
    ​
            /**
             * 活跃线程阈值
             */
            private Integer activeThreshold = 80;
        }
    }
              
### 3. ThreadPoolExecutorHolder {#3-thread-pool-executor-holder}

线程池实例对象和线程池参数配置对象本质上是两个独立的实体。但在实际使用中，我们往往需要**同时操作这两部分信息** ，例如在执行动态变更、状态监控或告警处理时。

为此，我们引入了 `Holder` 的设计思路，将线程池实例与其对应的属性配置，通过 `threadPoolId` 进行绑定和聚合，封装成一个统一的结构，便于后续统一管理与访问。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池执行器持有者对象
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    @Data
    @AllArgsConstructor
    public class ThreadPoolExecutorHolder {
    ​
        /**
         * 线程池唯一标识
         */
        private String threadPoolId;
    ​
        /**
         * 线程池
         */
        private ThreadPoolExecutor executor;
    ​
        /**
         * 线程池属性参数
         */
        private ThreadPoolExecutorProperties executorProperties;
    }
              
### 4. OneThreadRegistry {#4-one-thread-registry}

线程池实例与其参数配置的聚合问题解决后，接下来我们还需要考虑一个现实问题：**单个项目中往往不止一个动态线程池实例** 。

因此，我们需要一个统一的**线程池管理容器** 来进行集中存储和管理，这就是 `OneThreadRegistry` 的由来。

这个注册中心的设计非常简洁，主要提供了以下三类核心能力：

*  
**新增线程池实例** ；  
*  
**根据线程池ID获取指定实例** ；  
*  
  **获取当前所有线程池实例** 。

通过这一管理器，我们就可以以统一的方式对所有线程池进行注册、查询和维护。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池管理器，用于统一管理线程池实例
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    public class OneThreadRegistry {
    ​
        /**
         * 线程池持有者缓存，key 为线程池唯一标识，value 为线程池包装类
         */
        private static final Map<String, ThreadPoolExecutorHolder> HOLDER_MAP = new ConcurrentHashMap<>();
    ​
        /**
         * 注册线程池到管理器
         *
         * @param threadPoolId 线程池唯一标识
         * @param executor     线程池执行器实例
         * @param properties   线程池参数配置
         */
        public static void putHolder(String threadPoolId, ThreadPoolExecutor executor, ThreadPoolExecutorProperties properties) {
            ThreadPoolExecutorHolder executorHolder = new ThreadPoolExecutorHolder(threadPoolId, executor, properties);
            HOLDER_MAP.put(threadPoolId, executorHolder);
        }
    ​
        /**
         * 根据线程池 ID 获取对应的线程池包装对象
         *
         * @param threadPoolId 线程池唯一标识
         * @return 线程池持有者对象
         */
        public static ThreadPoolExecutorHolder getHolder(String threadPoolId) {
            return HOLDER_MAP.get(threadPoolId);
        }
    ​
        /**
         * 获取所有线程池集合
         *
         * @return 线程池集合
         */
        public static Collection<ThreadPoolExecutorHolder> getAllHolders() {
            return HOLDER_MAP.values();
        }
    }
              
以上这四个类构成了动态线程池体系中最核心的基础代码，后续的所有功能设计几乎都是围绕它们展开的。

当大家对这些基础设施有了清晰的理解之后，再去阅读其他基于这些对象构建的 API，就能更容易把握其背后的上下文语义与设计动机，从而真正做到"知其然，也知其所以然"。

Builder 设计模式 {#builder}
-----------------------

### 1. 什么是 Builder 设计模式？ {#1-builder}

在 `core` 包中，我们也引入了一种常见的设计模式 ------ **Builder模式** 。

随着线程池功能逐步增强，我们需要引入更多自定义参数，例如：动态线程池标识、项目关闭时等待任务完成的最大时长等。这些参数在构建过程中不仅需要进行合理性校验，还可能影响最终构造出的线程池类型（普通或动态线程池）。

为了解决上述构建灵活性与可维护性的问题，我们设计了 `ThreadPoolExecutorBuilder`，它采用了经典的 **Builder模式** ，将线程池的各项配置参数封装为链式调用形式。
> Builder 模式作用域：**如果类的属性之间有一定的依赖关系或者约束条件（源自设计模式之美）** 。

![iShot_2025-06-26_20.53.55.png](https://article-images.zsxq.com/FqPzWrqg4pFJajwZ0RxNziYVAziT "iShot_2025-06-26_20.53.55.png")


<br />


关于 Builder 模式在线程池中的应用，其实在 12306 项目中已有一篇文章专门讲解了：《Hutool 如何使用 Builder 模式创建线程池》，这里就不再赘述，感兴趣的同学可以前往阅读。
> 🔗 访问方式：进入星球专栏，查看 **12306专题** ，文章链接收录在对应的语雀文档中。
>
> 此外，后续我也计划对星球内容进行整体重构，将所有文章统一迁移到 [nageoffer.com](https://nageoffer.com)，并将一些通用的设计模式与工程设计亮点抽象整理成独立专栏，方便大家系统性学习与查阅。

### 2. 常见问题答疑 {#2}

可能有同学会疑惑：**Lombok不是也提供了`@Builder`注解吗？那为什么我们还要手动实现Builder模式呢？**

为了帮助大家更直观地理解两者的差异，我整理了一张对比表格，列出了手动实现 Builder 与 Lombok `@Builder` 注解在使用上的主要区别和各自的适用场景：  

|      对比项      |     常规 Builder 模式     |   Lombok `@Builder` 注解    |
|---------------|-----------------------|---------------------------|
| **实现方式**      | 手动编写 `Builder` 类和链式方法 | 自动生成 `Builder` 类、构造器和链式方法 |
| **代码控制力**     | 高：可以自由控制构造逻辑、参数校验、默认值 | 低：逻辑复杂时不易插入中间处理           |
| **可读性/IDE体验** | 明确结构清晰，容易调试和导航        | 快速生成，省代码，但不易阅读、调试         |
| **默认值支持**     | 易于通过构造器逻辑注入默认值        | 只能依赖字段定义处的初始化（或后处理）       |
| **继承支持**      | 支持更灵活的继承和扩展           | 不支持继承 Builder，复杂场景限制较多    |
| **方法重载控制**    | 可精细设计多个构造路径           | 不支持多种构造方式，容易受限            |
| **生成额外构造器**   | 自己控制                  | 会自动生成 `builder()` 方法      |
| **学习/使用成本**   | 高一些，但逻辑清晰，便于维护        | 非常低，上手快，适合简单场景            |

Lombok 提供的 `@Builder` 注解确实很方便，它的最大优势在于**省代码、上手快** ，非常适合用于构建简单的 Java 对象。但相比之下，手写的 Builder 模式则具有更强的**灵活性和可控性** ，能够**支持复杂构建逻辑、参数校验、默认值注入等扩展场景** 。

以我们当前的线程池构建器为例，如果单纯使用 Lombok 的 `@Builder`，只能构造出一个静态对象，**无法根据构建过程中的参数动态决定是创建普通线程池还是动态线程池** ，也**无法在构建过程中插入定制化校验或初始化逻辑** 。

而在我们手写的构建器中，最终 `build()` 方法就是扩展能力的核心体现。目前我们只验证了部分核心参数，实际上还可以在这里加入更多校验逻辑，例如线程池容量是否匹配、拒绝策略是否合法、等待时长是否超长等。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 构建线程池实例
     */
    public ThreadPoolExecutor build() {
        BlockingQueue<Runnable> blockingQueue = BlockingQueueTypeEnum.createBlockingQueue(workQueueType.getName(), workQueueCapacity);
        RejectedExecutionHandler rejectedHandler = Optional.ofNullable(this.rejectedHandler)
                .orElseGet(() -> new ThreadPoolExecutor.AbortPolicy());
    ​
        Assert.notNull(threadFactory, "The thread factory cannot be null.");
    ​
        ThreadPoolExecutor threadPoolExecutor;
        if (dynamicPool) {
            threadPoolExecutor = new OneThreadExecutor(
                    threadPoolId,
                    corePoolSize,
                    maximumPoolSize,
                    keepAliveTime,
                    TimeUnit.SECONDS,
                    blockingQueue,
                    threadFactory,
                    rejectedHandler,
                    awaitTerminationMillis
            );
        } else {
            threadPoolExecutor = new ThreadPoolExecutor(
                    corePoolSize,
                    maximumPoolSize,
                    keepAliveTime,
                    TimeUnit.SECONDS,
                    blockingQueue,
                    threadFactory,
                    rejectedHandler
            );
        }
    ​
        threadPoolExecutor.allowCoreThreadTimeOut(allowCoreThreadTimeOut);
        return threadPoolExecutor;
    }
              
可能有同学会问：**像"等待时长是否超出限制"这样的参数校验，我在设置参数的时候直接校验不就行了吗？**

从某些简单场景来看，这确实是可行的。但一旦出现**参数之间存在依赖关系** ，就不适合在每个 Setter 方法中逐一校验了。
> 还有一点，有可能用户没有调用这个参数方法，但是最终是需要必填的。

举个典型例子：**核心线程数不能大于最大线程数** 。如果你在设置 `corePoolSize` 时就进行校验，而这时 `maximumPoolSize` 还未设置，显然校验是无效或错误的。这种跨字段的逻辑依赖，只有在 **所有参数都配置完毕、准备构建实例的最后一步（即`build()`方法）** 时才能统一判断。

还有一点也值得注意：**有些参数是必填的，但用户可能根本没有调用对应的Setter方法进行设置** 。

比如线程池的 `threadFactory` 或 `threadPoolId`，在某些场景下是构造线程池所必需的核心参数。如果不在最终的 `build()` 方法中统一校验这些"必填但可能被遗漏"的字段，就可能导致线程池在运行时抛出异常或出现不可预期的行为。

因此，**`build()`方法不仅是构建实例的入口，也是做"最终兜底校验"的最佳位置** ，可以最大程度地确保构造出来的对象是完整、合法、可用的。

文末总结
----

通过本章的学习，我们深入剖析了 `oneThread` 动态线程池框架中核心基础类的设计理念与实现方式，涵盖线程池基类、参数封装、注册中心、构建器等关键模块。这些设计不仅提升了线程池的可观测性和可运维性，也为后续功能扩展（如运行时调参、告警通知）打下坚实基础。

建议大家在理解这些基础结构后，再去阅读上层组件代码，能更好把握整体架构脉络，实现从"知其然"到"知其所以然"的转变。

完结，撒花 🎉  
自定义动态线程池基础类，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：核心设计的提前规划与整体构思，是构建高质量动态线程池框架的基石，能够有效避免后期大幅返工，同时为系统的灵活扩展和维护奠定坚实基础。

课程目录如下所示：

*  
核心设计概述  
*  
关键类详解  
*  
Builder 设计模式  
*  
  文末总结

核心设计概览
------

### 1. 设计思想 {#1}

在上一篇文章中，我们已经初步了解了我在设计基础组件库时秉持的核心理念：**尽可能减少对三方框架的依赖** 。

秉持这一设计思路，在最初开发 `oneThread` 框架时，我将大部分与第三方框架无关的通用逻辑，统一抽象封装在了 `onethread-core` 模块中。

该模块主要承担以下核心职责：

*  
定义动态线程池的基础抽象类；  
*  
执行线程池运行时的告警扫描逻辑；  
*  
提供线程池运行状态的指标采集能力；  
*  
  支持监听配置中心变更，并动态调整线程池参数及触发告警通知。

下面是 `onethread-core` 模块的 `pom.xml` 依赖，可以看到它**并未强制依赖任何Spring相关的包** ，保持了良好的独立性和可移植性：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
            <groupId>com.nageoffer.onethread</groupId>
            <artifactId>onethread-all</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </parent>
        <artifactId>onethread-core</artifactId>
    ​
        <dependencies>
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>com.alibaba.fastjson2</groupId>
                <artifactId>fastjson2</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>org.yaml</groupId>
                <artifactId>snakeyaml</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
    ​
            <dependency>
                <groupId>ch.qos.logback</groupId>
                <artifactId>logback-classic</artifactId>
            </dependency>
    ​
            <dependency>
                <groupId>io.micrometer</groupId>
                <artifactId>micrometer-core</artifactId>
            </dependency>
        </dependencies>
    </project>
              
有同学可能会疑惑：**为什么上述依赖中没有显式指定版本号？**

这是因为我们在项目的根目录（通常是 `parent` 项目的 `pom.xml`）中，已经通过 `<dependencyManagement>` 统一定义了各依赖的版本号。因此，在 `onethread-core` 模块中只需声明依赖的坐标，无需重复指定版本，Maven 会自动从父模块继承对应的版本信息。这样不仅提高了依赖管理的统一性，也有助于减少版本冲突风险。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    <properties>
        <java.version>17</java.version>
        <spring-boot.version>3.0.7</spring-boot.version>
        <spring-cloud.version>2022.0.3</spring-cloud.version>
        <spring-cloud-alibaba.version>2022.0.0.0-RC2</spring-cloud-alibaba.version>
        <hutool-all.version>5.8.37</hutool-all.version>
        <apollo-client-config-data.version>2.4.0</apollo-client-config-data.version>
        <fastjson2.version>2.0.57</fastjson2.version>
        <spotless-maven-plugin.version>2.22.1</spotless-maven-plugin.version>
        <maven-compiler-plugin.version>3.6.1</maven-compiler-plugin.version>
    </properties>
    ​
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>${spring-boot.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    ​
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    ​
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
    ​
            <dependency>
                <groupId>cn.hutool</groupId>
                <artifactId>hutool-all</artifactId>
                <version>${hutool-all.version}</version>
            </dependency>
    ​
            <dependency>
                <groupId>com.ctrip.framework.apollo</groupId>
                <artifactId>apollo-client-config-data</artifactId>
                <version>${apollo-client-config-data.version}</version>
            </dependency>
    ​
            <dependency>
                <groupId>com.alibaba.fastjson2</groupId>
                <artifactId>fastjson2</artifactId>
                <version>${fastjson2.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
              
### 2. package 说明 {#2-package}

在前面的文章中，我们已经介绍过 `oneThread` 框架中各个模块（Module）所承担的职责。但当时并未展开到更细粒度的 **package层级结构** 。

本文将进一步说明 `onethread-core` 模块中各个包的具体作用，帮助大家更清晰地理解其内部结构与设计意图。

具体包划分与功能如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    .
    ├── src
    │   ├── main
    │   │   └── java
    │   │       └── com
    │   │           └── nageoffer
    │   │               └── onethread
    │   │                   └── core
    │   │                       ├── alarm # 告警扫描
    │   │                       ├── config # 组件库基础配置
    │   │                       ├── constant # 常量类
    │   │                       ├── executor # 最最核心的线程池基础包
    │   │                       │   └── support # 核心基础包至上增强的功能，如果不拆分这个也行，但是类会比较多
    │   │                       ├── monitor # 运行时指标监控
    │   │                       ├── notification # 线程池配置变更和告警通知
    │   │                       │   ├── dto
    │   │                       │   └── service
    │   │                       ├── parser # 解析配置内容字符串为键值对 Map
    │   │                       └── toolkit # 工具包，比如：线程池和线程工厂构建者
              
大家最需要关注的是 alarm、executor、monitor、notification 四个包。

### 3. 类图设计 {#3}

在 `core` 包中，有四个最核心的类，它们构成了整个动态线程池功能的基础。其关系如下所示：

![iShot_2025-06-26_20.53.54.png](https://article-images.zsxq.com/FkqM-rk3f8UxBzTRHg1YzyUgfMrb "iShot_2025-06-26_20.53.54.png")

关键类详解
-----

### 1. OneThreadExecutor {#1-one-thread-executor}

有同学可能会问：**为什么要单独定义一个线程池基类，而不是直接使用原生的`ThreadPoolExecutor`呢？**

原因有很多，但为了循序渐进地展开说明，这里先讲其中一个最关键的点：
> 原生线程池并没有线程池 ID 或名称的概念。

在实际业务中，我们往往需要对线程池进行运行时变更、指标监控、告警通知等操作。而缺乏唯一标识，会让我们在面对多个线程池时难以准确定位、识别和管理。

因此，为了支持更强的可观测性与可运维性，我们抽象出了具备线程池标识能力的基类，作为后续扩展的统一入口。

值得一提的是，`ThreadPoolExecutor` 本身并不排斥继承，反而在设计上为继承扩展预留了几个关键的扩展点方法。通过继承并重写这些方法，我们可以轻松注入自定义行为，构建更智能的线程池体系：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 任务执行前钩子（可用于记录开始时间、设置线程上下文等）
     */
    @Override
    protected void beforeExecute(Thread t, Runnable r) {
        super.beforeExecute(t, r);
        // 自定义逻辑
    }
    ​
    /**
     * 任务执行后钩子（可用于统计执行耗时、清理资源、日志记录等）
     */
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        super.afterExecute(r, t);
        // 自定义逻辑
    }
    ​
    /**
     * 线程池终止钩子（可用于清理资源、发送通知等）
     */
    @Override
    protected void terminated() {
        super.terminated();
        // 自定义逻辑
    }
    ​
              
下面是 `oneThread` 的线程池基类，目前我们只在其中**集成了线程池唯一标识** 这一核心能力。

在阅读源码时，建议大家暂时聚焦这一功能点。至于类中其他相对复杂的逻辑，如果一时看不太懂也没关系，**与其陷入细节，不如先跟着Breathe一起按"主线思路"一步步深入理解。**  
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
        /**
         * 线程池唯一标识，用来动态变更参数等
         */
        @Getter
        private final String threadPoolId;
        // ......
    ​
        /**
         * Creates a new {@code ExtensibleThreadPoolExecutor} with the given initial parameters.
         *
         * @param threadPoolId           thread-pool id
         * @param corePoolSize           the number of threads to keep in the pool, even
         *                               if they are idle, unless {@code allowCoreThreadTimeOut} is set
         * @param maximumPoolSize        the maximum number of threads to allow in the
         *                               pool
         * @param keepAliveTime          when the number of threads is greater than
         *                               the core, this is the maximum time that excess idle threads
         *                               will wait for new tasks before terminating.
         * @param unit                   the time unit for the {@code keepAliveTime} argument
         * @param workQueue              the queue to use for holding tasks before they are
         *                               executed.  This queue will hold only the {@code Runnable}
         *                               tasks submitted by the {@code execute} method.
         * @param threadFactory          the factory to use when the executor
         *                               creates a new thread
         * @param handler                the handler to use when execution is blocked
         *                               because the thread bounds and queue capacities are reached
         * @param awaitTerminationMillis the maximum time to wait
         * @throws IllegalArgumentException if one of the following holds:<br>
         *                                  {@code corePoolSize < 0}<br>
         *                                  {@code keepAliveTime < 0}<br>
         *                                  {@code maximumPoolSize <= 0}<br>
         *                                  {@code maximumPoolSize < corePoolSize}
         * @throws NullPointerException     if {@code workQueue} or {@code unit}
         *                                  or {@code threadFactory} or {@code handler} is null
         */
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
            // ......
            // 设置动态线程池扩展属性：线程池 ID 标识
            this.threadPoolId = threadPoolId;
            // ......
        }
    ​
        // ......
    }
              
### 2. ThreadPoolExecutorProperties {#2-thread-pool-executor-properties}

有同学可能会问：**线程池的参数明明已经在`OneThreadExecutor`中定义了，为什么还要单独拆分出一个属性实体类？**

这是因为线程池中的参数通常**分散在多个字段中** ，并且像**告警阈值、通知规则等配置** 在原生线程池中是根本不存在的。为了更好地支持动态配置和统一管理，我们通过 `threadPoolId` 将这些"外围参数"与线程池进行关联，形成一套完整的配置体系，便于在运行时查看、存储、变更和追踪。

另外，为了Breathe写代码方便，告警和通知相关的参数类目前是以**静态内部类** 的形式存在。大家在实际开发中，也可以根据需要将其**拆分为独立的外部类** ，以提升可复用性。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池属性参数
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    @Accessors(chain = true)
    public class ThreadPoolExecutorProperties {
    ​
        /**
         * 线程池唯一标识
         */
        private String threadPoolId;
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
         * 队列容量
         */
        private Integer queueCapacity;
    ​
        /**
         * 阻塞队列类型
         */
        private String workQueue;
    ​
        /**
         * 拒绝策略类型
         */
        private String rejectedHandler;
    ​
        /**
         * 线程空闲存活时间（单位：秒）
         */
        private Long keepAliveTime;
    ​
        /**
         * 是否允许核心线程超时
         */
        private Boolean allowCoreThreadTimeOut;
    ​
        /**
         * 通知配置
         */
        private NotifyConfig notify;
    ​
        /**
         * 报警配置，默认设置
         */
        private AlarmConfig alarm = new AlarmConfig();
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
    ​
            /**
             * 告警间隔，单位分钟
             */
            private Integer interval = 5;
        }
    ​
        @Data
        @NoArgsConstructor
        @AllArgsConstructor
        public static class AlarmConfig {
    ​
            /**
             * 默认开启报警配配置
             */
            private Boolean enable = Boolean.TRUE;
    ​
            /**
             * 队列阈值
             */
            private Integer queueThreshold = 80;
    ​
            /**
             * 活跃线程阈值
             */
            private Integer activeThreshold = 80;
        }
    }
              
### 3. ThreadPoolExecutorHolder {#3-thread-pool-executor-holder}

线程池实例对象和线程池参数配置对象本质上是两个独立的实体。但在实际使用中，我们往往需要**同时操作这两部分信息** ，例如在执行动态变更、状态监控或告警处理时。

为此，我们引入了 `Holder` 的设计思路，将线程池实例与其对应的属性配置，通过 `threadPoolId` 进行绑定和聚合，封装成一个统一的结构，便于后续统一管理与访问。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 线程池执行器持有者对象
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    @Data
    @AllArgsConstructor
    public class ThreadPoolExecutorHolder {
    ​
        /**
         * 线程池唯一标识
         */
        private String threadPoolId;
    ​
        /**
         * 线程池
         */
        private ThreadPoolExecutor executor;
    ​
        /**
         * 线程池属性参数
         */
        private ThreadPoolExecutorProperties executorProperties;
    }
              
### 4. OneThreadRegistry {#4-one-thread-registry}

线程池实例与其参数配置的聚合问题解决后，接下来我们还需要考虑一个现实问题：**单个项目中往往不止一个动态线程池实例** 。

因此，我们需要一个统一的**线程池管理容器** 来进行集中存储和管理，这就是 `OneThreadRegistry` 的由来。

这个注册中心的设计非常简洁，主要提供了以下三类核心能力：

*  
**新增线程池实例** ；  
*  
**根据线程池ID获取指定实例** ；  
*  
  **获取当前所有线程池实例** 。

通过这一管理器，我们就可以以统一的方式对所有线程池进行注册、查询和维护。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 动态线程池管理器，用于统一管理线程池实例
     * <p>
     * 作者：马丁
     * 加项目群：早加入就是优势！500人内部项目群，分享的知识总有你需要的 <a href="https://t.zsxq.com/cw7b9" />
     * 开发时间：2025-04-20
     */
    public class OneThreadRegistry {
    ​
        /**
         * 线程池持有者缓存，key 为线程池唯一标识，value 为线程池包装类
         */
        private static final Map<String, ThreadPoolExecutorHolder> HOLDER_MAP = new ConcurrentHashMap<>();
    ​
        /**
         * 注册线程池到管理器
         *
         * @param threadPoolId 线程池唯一标识
         * @param executor     线程池执行器实例
         * @param properties   线程池参数配置
         */
        public static void putHolder(String threadPoolId, ThreadPoolExecutor executor, ThreadPoolExecutorProperties properties) {
            ThreadPoolExecutorHolder executorHolder = new ThreadPoolExecutorHolder(threadPoolId, executor, properties);
            HOLDER_MAP.put(threadPoolId, executorHolder);
        }
    ​
        /**
         * 根据线程池 ID 获取对应的线程池包装对象
         *
         * @param threadPoolId 线程池唯一标识
         * @return 线程池持有者对象
         */
        public static ThreadPoolExecutorHolder getHolder(String threadPoolId) {
            return HOLDER_MAP.get(threadPoolId);
        }
    ​
        /**
         * 获取所有线程池集合
         *
         * @return 线程池集合
         */
        public static Collection<ThreadPoolExecutorHolder> getAllHolders() {
            return HOLDER_MAP.values();
        }
    }
              
以上这四个类构成了动态线程池体系中最核心的基础代码，后续的所有功能设计几乎都是围绕它们展开的。

当大家对这些基础设施有了清晰的理解之后，再去阅读其他基于这些对象构建的 API，就能更容易把握其背后的上下文语义与设计动机，从而真正做到"知其然，也知其所以然"。

Builder 设计模式 {#builder}
-----------------------

### 1. 什么是 Builder 设计模式？ {#1-builder}

在 `core` 包中，我们也引入了一种常见的设计模式 ------ **Builder模式** 。

随着线程池功能逐步增强，我们需要引入更多自定义参数，例如：动态线程池标识、项目关闭时等待任务完成的最大时长等。这些参数在构建过程中不仅需要进行合理性校验，还可能影响最终构造出的线程池类型（普通或动态线程池）。

为了解决上述构建灵活性与可维护性的问题，我们设计了 `ThreadPoolExecutorBuilder`，它采用了经典的 **Builder模式** ，将线程池的各项配置参数封装为链式调用形式。
> Builder 模式作用域：**如果类的属性之间有一定的依赖关系或者约束条件（源自设计模式之美）** 。

![iShot_2025-06-26_20.53.55.png](https://article-images.zsxq.com/FqPzWrqg4pFJajwZ0RxNziYVAziT "iShot_2025-06-26_20.53.55.png")


<br />


关于 Builder 模式在线程池中的应用，其实在 12306 项目中已有一篇文章专门讲解了：《Hutool 如何使用 Builder 模式创建线程池》，这里就不再赘述，感兴趣的同学可以前往阅读。
> 🔗 访问方式：进入星球专栏，查看 **12306专题** ，文章链接收录在对应的语雀文档中。
>
> 此外，后续我也计划对星球内容进行整体重构，将所有文章统一迁移到 [nageoffer.com](https://nageoffer.com)，并将一些通用的设计模式与工程设计亮点抽象整理成独立专栏，方便大家系统性学习与查阅。

### 2. 常见问题答疑 {#2}

可能有同学会疑惑：**Lombok不是也提供了`@Builder`注解吗？那为什么我们还要手动实现Builder模式呢？**

为了帮助大家更直观地理解两者的差异，我整理了一张对比表格，列出了手动实现 Builder 与 Lombok `@Builder` 注解在使用上的主要区别和各自的适用场景：  

|      对比项      |     常规 Builder 模式     |   Lombok `@Builder` 注解    |
|---------------|-----------------------|---------------------------|
| **实现方式**      | 手动编写 `Builder` 类和链式方法 | 自动生成 `Builder` 类、构造器和链式方法 |
| **代码控制力**     | 高：可以自由控制构造逻辑、参数校验、默认值 | 低：逻辑复杂时不易插入中间处理           |
| **可读性/IDE体验** | 明确结构清晰，容易调试和导航        | 快速生成，省代码，但不易阅读、调试         |
| **默认值支持**     | 易于通过构造器逻辑注入默认值        | 只能依赖字段定义处的初始化（或后处理）       |
| **继承支持**      | 支持更灵活的继承和扩展           | 不支持继承 Builder，复杂场景限制较多    |
| **方法重载控制**    | 可精细设计多个构造路径           | 不支持多种构造方式，容易受限            |
| **生成额外构造器**   | 自己控制                  | 会自动生成 `builder()` 方法      |
| **学习/使用成本**   | 高一些，但逻辑清晰，便于维护        | 非常低，上手快，适合简单场景            |

Lombok 提供的 `@Builder` 注解确实很方便，它的最大优势在于**省代码、上手快** ，非常适合用于构建简单的 Java 对象。但相比之下，手写的 Builder 模式则具有更强的**灵活性和可控性** ，能够**支持复杂构建逻辑、参数校验、默认值注入等扩展场景** 。

以我们当前的线程池构建器为例，如果单纯使用 Lombok 的 `@Builder`，只能构造出一个静态对象，**无法根据构建过程中的参数动态决定是创建普通线程池还是动态线程池** ，也**无法在构建过程中插入定制化校验或初始化逻辑** 。

而在我们手写的构建器中，最终 `build()` 方法就是扩展能力的核心体现。目前我们只验证了部分核心参数，实际上还可以在这里加入更多校验逻辑，例如线程池容量是否匹配、拒绝策略是否合法、等待时长是否超长等。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 构建线程池实例
     */
    public ThreadPoolExecutor build() {
        BlockingQueue<Runnable> blockingQueue = BlockingQueueTypeEnum.createBlockingQueue(workQueueType.getName(), workQueueCapacity);
        RejectedExecutionHandler rejectedHandler = Optional.ofNullable(this.rejectedHandler)
                .orElseGet(() -> new ThreadPoolExecutor.AbortPolicy());
    ​
        Assert.notNull(threadFactory, "The thread factory cannot be null.");
    ​
        ThreadPoolExecutor threadPoolExecutor;
        if (dynamicPool) {
            threadPoolExecutor = new OneThreadExecutor(
                    threadPoolId,
                    corePoolSize,
                    maximumPoolSize,
                    keepAliveTime,
                    TimeUnit.SECONDS,
                    blockingQueue,
                    threadFactory,
                    rejectedHandler,
                    awaitTerminationMillis
            );
        } else {
            threadPoolExecutor = new ThreadPoolExecutor(
                    corePoolSize,
                    maximumPoolSize,
                    keepAliveTime,
                    TimeUnit.SECONDS,
                    blockingQueue,
                    threadFactory,
                    rejectedHandler
            );
        }
    ​
        threadPoolExecutor.allowCoreThreadTimeOut(allowCoreThreadTimeOut);
        return threadPoolExecutor;
    }
              
可能有同学会问：**像"等待时长是否超出限制"这样的参数校验，我在设置参数的时候直接校验不就行了吗？**

从某些简单场景来看，这确实是可行的。但一旦出现**参数之间存在依赖关系** ，就不适合在每个 Setter 方法中逐一校验了。
> 还有一点，有可能用户没有调用这个参数方法，但是最终是需要必填的。

举个典型例子：**核心线程数不能大于最大线程数** 。如果你在设置 `corePoolSize` 时就进行校验，而这时 `maximumPoolSize` 还未设置，显然校验是无效或错误的。这种跨字段的逻辑依赖，只有在 **所有参数都配置完毕、准备构建实例的最后一步（即`build()`方法）** 时才能统一判断。

还有一点也值得注意：**有些参数是必填的，但用户可能根本没有调用对应的Setter方法进行设置** 。

比如线程池的 `threadFactory` 或 `threadPoolId`，在某些场景下是构造线程池所必需的核心参数。如果不在最终的 `build()` 方法中统一校验这些"必填但可能被遗漏"的字段，就可能导致线程池在运行时抛出异常或出现不可预期的行为。

因此，**`build()`方法不仅是构建实例的入口，也是做"最终兜底校验"的最佳位置** ，可以最大程度地确保构造出来的对象是完整、合法、可用的。

文末总结
----

通过本章的学习，我们深入剖析了 `oneThread` 动态线程池框架中核心基础类的设计理念与实现方式，涵盖线程池基类、参数封装、注册中心、构建器等关键模块。这些设计不仅提升了线程池的可观测性和可运维性，也为后续功能扩展（如运行时调参、告警通知）打下坚实基础。

建议大家在理解这些基础结构后，再去阅读上层组件代码，能更好把握整体架构脉络，实现从"知其然"到"知其所以然"的转变。

完结，撒花 🎉

