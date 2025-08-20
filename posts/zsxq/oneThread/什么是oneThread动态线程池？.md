2025年07月16日 21:30  

🚀oneThread 轮子项目 {#one-thread}
------------------------------

### 1. 基础介绍 {#1}

oneThread 是基于 **配置中心** 构建的 动态可观测 Java 线程池框架，弥补了 JDK 原生线程池 **参数配置不灵活的不足** ，支持核心参数的 **在线动态调整** 、**运行时状态监控** 与 **阈值告警** ，有效提升系统的稳定性与可运维性。框架兼容主流配置中心如 `Nacos`、`Apollo`，实现线程池参数的 **热更新与统一管理**。

![image-20250626140245388 (1).png](https://article-images.zsxq.com/FhFm8y2CCuum97fGvPbVCcvEF3l4)

目前后端核心代码（包含注释）`1.34w` 行，仅 Java 后端纯代码 `4.99k` 行。

![image-20250524210252860.png](https://article-images.zsxq.com/FjceMoExzAo3X08iMhR98lgm_x6T)

### 2. 技术栈 {#2}

Spring、Spring Boot、JUC、Tomcat、Jetty、Nacos、Apollo、Prometheus、Grafana

### 3. 为什么会有动态线程池？ {#3}

> 参考自：[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

#### 3.1 任务执行拒绝 {#3-1}

**事故描述**：XX页面展示接口产生大量调用降级，数量级在几十到上百。

**事故原因** ：该服务展示接口内部逻辑使用线程池做并行计算，由于没有预估好调用的流量，导致最大核心数**设置偏小** ，大量抛出 `RejectedExecutionException`，触发接口降级条件，示意图如下：

![1df932840b31f41931bb69e16be2932844240.png](https://article-images.zsxq.com/Fj9WxNJcAaZk9V9xHa5hgE7Uj1cB)

#### 3.2 任务执行时间过长 {#3-2}

**事故描述**：XX业务提供的服务执行时间过长，作为上游服务整体超时，大量下游服务调用失败。

**事故原因** ：该服务处理请求内部逻辑使用线程池做资源隔离，由于**队列设置过长**，最大线程数设置失效，导致请求数量增加时，大量任务堆积在队列中，任务执行时间过长，最终导致下游服务的大量调用超时失败。示意图如下：

![668e3c90f4b918bfcead2f4280091e9757284.png](https://article-images.zsxq.com/FsV0s4_15rNzkjgG-eg8gs57ZCc5)

### 4. 教学亮点 \& 技术实现------ {#4-and}

#### 4.1 动态参数刷新能力 {#4-1}

基于配置中心实现线程池核心参数的动态刷新能力，采用 模板方法模式复用刷新逻辑，支持**多种配置中心** （如 `Nacos`、`Apollo`）监听事件的统一注册与处理，提升代码复用和扩展性。

线程池动态变更原理如下图所示：

![640 (5).png](https://article-images.zsxq.com/FueYIfLGYH0RBvqC5LxhQkrrDts0)

线程池参数在配置中心定义如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      nacos:
        data-id: onethread-nacos-cloud-example-ding-ma.yaml
        group: DEFAULT_GROUP
      config-file-type: YAML
      web:
        core-pool-size: 20
        maximum-pool-size: 400
        keep-alive-time: 60
        notify:
          receives: xxxxxx
      notify-platforms:
        platform: DING
        url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxx
      executors:
      - thread-pool-id: onethread-producer
        core-pool-size: 12
        maximum-pool-size: 18
        queue-capacity: 2000
        work-queue: ResizableCapacityLinkedBlockingQueue
        rejected-handler: CallerRunsPolicy
        keep-alive-time: 60
        allow-core-thread-time-out: false
        notify:
          receives: xxxxxx
          interval: 10
        alarm:
          enable: true
          queue-threshold: 80
          active-threshold: 80
              
发起参数变更后：

![image-20250525170844026 (1).png](https://article-images.zsxq.com/FrPA3pcnT-8l7n0l-2cL7O50rZBI)

#### 4.2 拒绝策略增强机制 {#4-2}

基于 JDK 的 `InvocationHandler` 实现 动态代理机制，对线程池拒绝策略进行代理扩展，拦截拒绝任务事件并触发告警机制，实现任务拒绝的实时响应。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @AllArgsConstructor
    public class RejectedProxyInvocationHandler implements InvocationHandler {

        private final Object target;
        private final AtomicLong rejectCount;

        private static final String REJECT_METHOD = "rejectedExecution";

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (REJECT_METHOD.equals(method.getName()) &&
                    args != null &&
                    args.length == 2 &&
                    args[0] instanceof Runnable &&
                    args[1] instanceof ThreadPoolExecutor) {
                rejectCount.incrementAndGet();
            }

            if (method.getName().equals("toString") && method.getParameterCount() == 0) {
                return target.getClass().getSimpleName();
            }

            try {
                return method.invoke(target, args);
            } catch (InvocationTargetException ex) {
                throw ex.getCause();
            }
        }
    }
              
监控线程池运行状态异常告警如下所示：

![image-20250525171313388 (2).png](https://article-images.zsxq.com/FrbsER6iTn8Nfi5HcaYzE63a9XCx)

#### 4.3 ...... {#4-3}

### 5. 项目结构 {#5}

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
              
💡项目质量怎么样？
----------

### 1. 编码质量 {#1}

因为Breathe之前写过 Hippo4j，所以知道动态线程池是为了解决什么问题，以及核心功能应该是什么。为此，我把这部分经验带到了这个项目，毫不夸张的说，oneThread 就是一个生产级可用的动态线程池框架。

通过一个思维导图梳理了系统中大部分亮点，如下图所示：

![image-20250626195856877.png](https://article-images.zsxq.com/Fpzrls5jIZ0TynyKAh7DcaSAatiQ)

如果上面这种缩略图大家看着不能总结亮点，简单说几点：

*  
合理分层：项目结构围绕核心能力进行解耦，划分为 `core`（核心能力）、`starter`（可插拔配置中心组件）、`spring-base`（基础能力封装）、`example`（使用示例）、`dashboard-dev`（可视化控制台）等模块，实现了典型的"核心逻辑 + 接入层 + 展示层"分层模式，方便维护与扩展。  
*  
高质量编码：遵循统一的代码规范，核心模块代码注释率高，逻辑清晰；大量使用接口 + 抽象类进行功能定义和通用处理；配置项设计灵活，支持用户自定义线程池、告警策略、拒绝处理器等，具备良好的扩展性和可插拔性。  
*  
设计模式：项目广泛应用了多种经典设计模式，如模板方法模式（刷新逻辑复用）、观察者模式（监听配置变化）、构建者模式（线程池参数构造）、代理模式（增强拒绝处理器）等，增强了代码的可读性、可扩展性和解耦能力。  
*  
  基础架构：对接主流配置中心（Nacos、Apollo）实现动态参数管理，结合 Prometheus + Grafana 实现运行状态采集与可视化监控；提供统一告警能力（钉钉通知），具备完整的配置 - 执行 - 监控 - 告警链路，体现了高可运维性与稳定性设计。

### 2. 精美控制台 UI {#2-ui}

在牛券项目中，我们最初专注于后端架构与技术实现，但过程中有不少同学反馈：
> Breathe，能不能也提供前端部分，方便本地联调和整体理解？

考虑到大家的学习体验，我们也在持续打磨项目配套能力。这次，动态线程池框架 oneThread 已全面配套上线可视化前端控制台，支持线程池状态查看、参数动态调整及告警信息展示，进一步提升了项目的实用性与完整度。
> 大家都知道的，我是不能听到大家提需求的，你敢提我就敢实现 😎

动态线程池管理之线程池列表页面：

![image-20250626202656145.png](https://article-images.zsxq.com/FsK8KqvI1SebkeWBQbB8WX7dXVoE)

线程池参数编辑功能：

![image-20250626203006767 (1).png](https://article-images.zsxq.com/FjpWFmojHVMZ_nU193xKATYY7n0F)

动态线程池监控页面，内嵌了 Grafana 中间件：

![image-20250626203039058 (1).png](https://article-images.zsxq.com/Fj-PgaFZFD199gqN5K9oNgAtN6c7)

因为篇幅重点是介绍后端架构的，所以关于前端的章节就先到这里。大家可以参考星球文章本地部署下。

📝项目教程
------

关于 oneThread 动态线程池项目的项目教程，配有从零到一的文档教程，同时会提供核心逻辑的视频讲解，逐行带着大家 Debug 源代码，加深项目理解。

![image-20250626205514732 (1).png](https://article-images.zsxq.com/FmDkPsffIyGSEH2tIAoi8cNu91Tp)

📘常见问题答疑
--------

### 1. 能够学到什么？ {#1}

*  
基于 SpringBoot Starter 构建线程池组件，支持快速接入。  
*  
动态刷新线程池参数，兼容多种配置中心。  
*  
定时监控线程池状态，支持异常告警。  
*  
扩展拒绝策略，拦截任务并告警。  
*  
接入监控系统，支持指标采集与可视化。  
*  
优雅关闭线程池，避免任务丢失。  
*  
模板、构建者、动态代理以及观察者等设计模式。  
*  
  ......

> 从本质上讲，通过学习 oneThread 动态线程池这个轮子项目，相当于一只脚踏出了纯业务开发的思维圈，开始接触并理解框架设计和运行机制的底层逻辑。这是从"会用"走向"会写"的重要一步，也是在向真正掌握系统底层能力迈进。

### 2. 适合人群？ {#2}

*  
开发或学习过业务系统开发，比如拿个offer社群任意项目，或黑马点评、瑞吉外卖等。  
*  
对并发编程有过了解，使用过线程池、阻塞队列等并发包下技术。  

🚀oneThread 轮子项目 {#one-thread}
------------------------------

### 1. 基础介绍 {#1}

oneThread 是基于 **配置中心** 构建的 动态可观测 Java 线程池框架，弥补了 JDK 原生线程池 **参数配置不灵活的不足** ，支持核心参数的 **在线动态调整** 、**运行时状态监控** 与 **阈值告警** ，有效提升系统的稳定性与可运维性。框架兼容主流配置中心如 `Nacos`、`Apollo`，实现线程池参数的 **热更新与统一管理**。

![image-20250626140245388 (1).png](https://article-images.zsxq.com/FhFm8y2CCuum97fGvPbVCcvEF3l4)

目前后端核心代码（包含注释）`1.34w` 行，仅 Java 后端纯代码 `4.99k` 行。

![image-20250524210252860.png](https://article-images.zsxq.com/FjceMoExzAo3X08iMhR98lgm_x6T)

### 2. 技术栈 {#2}

Spring、Spring Boot、JUC、Tomcat、Jetty、Nacos、Apollo、Prometheus、Grafana

### 3. 为什么会有动态线程池？ {#3}

> 参考自：[Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

#### 3.1 任务执行拒绝 {#3-1}

**事故描述**：XX页面展示接口产生大量调用降级，数量级在几十到上百。

**事故原因** ：该服务展示接口内部逻辑使用线程池做并行计算，由于没有预估好调用的流量，导致最大核心数**设置偏小** ，大量抛出 `RejectedExecutionException`，触发接口降级条件，示意图如下：

![1df932840b31f41931bb69e16be2932844240.png](https://article-images.zsxq.com/Fj9WxNJcAaZk9V9xHa5hgE7Uj1cB)

#### 3.2 任务执行时间过长 {#3-2}

**事故描述**：XX业务提供的服务执行时间过长，作为上游服务整体超时，大量下游服务调用失败。

**事故原因** ：该服务处理请求内部逻辑使用线程池做资源隔离，由于**队列设置过长**，最大线程数设置失效，导致请求数量增加时，大量任务堆积在队列中，任务执行时间过长，最终导致下游服务的大量调用超时失败。示意图如下：

![668e3c90f4b918bfcead2f4280091e9757284.png](https://article-images.zsxq.com/FsV0s4_15rNzkjgG-eg8gs57ZCc5)

### 4. 教学亮点 \& 技术实现------ {#4-and}

#### 4.1 动态参数刷新能力 {#4-1}

基于配置中心实现线程池核心参数的动态刷新能力，采用 模板方法模式复用刷新逻辑，支持**多种配置中心** （如 `Nacos`、`Apollo`）监听事件的统一注册与处理，提升代码复用和扩展性。

线程池动态变更原理如下图所示：

![640 (5).png](https://article-images.zsxq.com/FueYIfLGYH0RBvqC5LxhQkrrDts0)

线程池参数在配置中心定义如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    onethread:
      nacos:
        data-id: onethread-nacos-cloud-example-ding-ma.yaml
        group: DEFAULT_GROUP
      config-file-type: YAML
      web:
        core-pool-size: 20
        maximum-pool-size: 400
        keep-alive-time: 60
        notify:
          receives: xxxxxx
      notify-platforms:
        platform: DING
        url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxx
      executors:
      - thread-pool-id: onethread-producer
        core-pool-size: 12
        maximum-pool-size: 18
        queue-capacity: 2000
        work-queue: ResizableCapacityLinkedBlockingQueue
        rejected-handler: CallerRunsPolicy
        keep-alive-time: 60
        allow-core-thread-time-out: false
        notify:
          receives: xxxxxx
          interval: 10
        alarm:
          enable: true
          queue-threshold: 80
          active-threshold: 80
              
发起参数变更后：

![image-20250525170844026 (1).png](https://article-images.zsxq.com/FrPA3pcnT-8l7n0l-2cL7O50rZBI)

#### 4.2 拒绝策略增强机制 {#4-2}

基于 JDK 的 `InvocationHandler` 实现 动态代理机制，对线程池拒绝策略进行代理扩展，拦截拒绝任务事件并触发告警机制，实现任务拒绝的实时响应。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    @AllArgsConstructor
    public class RejectedProxyInvocationHandler implements InvocationHandler {

        private final Object target;
        private final AtomicLong rejectCount;

        private static final String REJECT_METHOD = "rejectedExecution";

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if (REJECT_METHOD.equals(method.getName()) &&
                    args != null &&
                    args.length == 2 &&
                    args[0] instanceof Runnable &&
                    args[1] instanceof ThreadPoolExecutor) {
                rejectCount.incrementAndGet();
            }

            if (method.getName().equals("toString") && method.getParameterCount() == 0) {
                return target.getClass().getSimpleName();
            }

            try {
                return method.invoke(target, args);
            } catch (InvocationTargetException ex) {
                throw ex.getCause();
            }
        }
    }
              
监控线程池运行状态异常告警如下所示：

![image-20250525171313388 (2).png](https://article-images.zsxq.com/FrbsER6iTn8Nfi5HcaYzE63a9XCx)

#### 4.3 ...... {#4-3}

### 5. 项目结构 {#5}

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
              
💡项目质量怎么样？
----------

### 1. 编码质量 {#1}

因为Breathe之前写过 Hippo4j，所以知道动态线程池是为了解决什么问题，以及核心功能应该是什么。为此，我把这部分经验带到了这个项目，毫不夸张的说，oneThread 就是一个生产级可用的动态线程池框架。

通过一个思维导图梳理了系统中大部分亮点，如下图所示：

![image-20250626195856877.png](https://article-images.zsxq.com/Fpzrls5jIZ0TynyKAh7DcaSAatiQ)

如果上面这种缩略图大家看着不能总结亮点，简单说几点：

*  
合理分层：项目结构围绕核心能力进行解耦，划分为 `core`（核心能力）、`starter`（可插拔配置中心组件）、`spring-base`（基础能力封装）、`example`（使用示例）、`dashboard-dev`（可视化控制台）等模块，实现了典型的"核心逻辑 + 接入层 + 展示层"分层模式，方便维护与扩展。  
*  
高质量编码：遵循统一的代码规范，核心模块代码注释率高，逻辑清晰；大量使用接口 + 抽象类进行功能定义和通用处理；配置项设计灵活，支持用户自定义线程池、告警策略、拒绝处理器等，具备良好的扩展性和可插拔性。  
*  
设计模式：项目广泛应用了多种经典设计模式，如模板方法模式（刷新逻辑复用）、观察者模式（监听配置变化）、构建者模式（线程池参数构造）、代理模式（增强拒绝处理器）等，增强了代码的可读性、可扩展性和解耦能力。  
*  
  基础架构：对接主流配置中心（Nacos、Apollo）实现动态参数管理，结合 Prometheus + Grafana 实现运行状态采集与可视化监控；提供统一告警能力（钉钉通知），具备完整的配置 - 执行 - 监控 - 告警链路，体现了高可运维性与稳定性设计。

### 2. 精美控制台 UI {#2-ui}

在牛券项目中，我们最初专注于后端架构与技术实现，但过程中有不少同学反馈：
> Breathe，能不能也提供前端部分，方便本地联调和整体理解？

考虑到大家的学习体验，我们也在持续打磨项目配套能力。这次，动态线程池框架 oneThread 已全面配套上线可视化前端控制台，支持线程池状态查看、参数动态调整及告警信息展示，进一步提升了项目的实用性与完整度。
> 大家都知道的，我是不能听到大家提需求的，你敢提我就敢实现 😎

动态线程池管理之线程池列表页面：

![image-20250626202656145.png](https://article-images.zsxq.com/FsK8KqvI1SebkeWBQbB8WX7dXVoE)

线程池参数编辑功能：

![image-20250626203006767 (1).png](https://article-images.zsxq.com/FjpWFmojHVMZ_nU193xKATYY7n0F)

动态线程池监控页面，内嵌了 Grafana 中间件：

![image-20250626203039058 (1).png](https://article-images.zsxq.com/Fj-PgaFZFD199gqN5K9oNgAtN6c7)

因为篇幅重点是介绍后端架构的，所以关于前端的章节就先到这里。大家可以参考星球文章本地部署下。

📝项目教程
------

关于 oneThread 动态线程池项目的项目教程，配有从零到一的文档教程，同时会提供核心逻辑的视频讲解，逐行带着大家 Debug 源代码，加深项目理解。

![image-20250626205514732 (1).png](https://article-images.zsxq.com/FmDkPsffIyGSEH2tIAoi8cNu91Tp)

📘常见问题答疑
--------

### 1. 能够学到什么？ {#1}

*  
基于 SpringBoot Starter 构建线程池组件，支持快速接入。  
*  
动态刷新线程池参数，兼容多种配置中心。  
*  
定时监控线程池状态，支持异常告警。  
*  
扩展拒绝策略，拦截任务并告警。  
*  
接入监控系统，支持指标采集与可视化。  
*  
优雅关闭线程池，避免任务丢失。  
*  
模板、构建者、动态代理以及观察者等设计模式。  
*  
  ......

> 从本质上讲，通过学习 oneThread 动态线程池这个轮子项目，相当于一只脚踏出了纯业务开发的思维圈，开始接触并理解框架设计和运行机制的底层逻辑。这是从"会用"走向"会写"的重要一步，也是在向真正掌握系统底层能力迈进。

### 2. 适合人群？ {#2}

*  
开发或学习过业务系统开发，比如拿个offer社群任意项目，或黑马点评、瑞吉外卖等。  
*  
对并发编程有过了解，使用过线程池、阻塞队列等并发包下技术。  

