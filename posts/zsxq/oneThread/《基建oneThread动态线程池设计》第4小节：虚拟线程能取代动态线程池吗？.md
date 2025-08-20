2025年07月06日 18:00  
虚拟线程能取代动态线程池吗？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：技术博客讲解，无



*** ** * ** ***

内容摘要：自从 Go 进入大众视野以来，其协程特性带来的并发能力一直备受推崇。相比之下，Java 在这方面一度显得力不从心，几乎被"按在地上摩擦"。所幸，随着 JDK 19 到 21 推出了虚拟线程，Java 在并发领域迎来了重要的技术革新，也为自己扳回了一局。

很多同学可能会疑惑：在 JDK 引入虚拟线程之后，动态线程池是否还有存在的必要？本文将围绕虚拟线程与线程池（动态）展开讨论，分析虚拟线程的优缺点，并探讨线程池（动态）在实际场景中的应用价值。

为确保严谨性，本文章发布时间为 2025-07-06，JDK 最高调研到 21。

课程目录如下所示：

*  
背景概要  
*  
什么是虚拟线程？  
*  
虚拟线程 vs 动态线程池  
*  
Web 容器如何关联虚拟线程？  
*  
虚拟线程的缺点  
*  
常见问题答疑  
*  
巨人的肩膀  
*  
  文末总结

> 本文中有不少内容借助了 AI 生成，但整体框架和观点，都是我在查阅几十篇文章、结合自己的实战经验以及各大厂的文章说明后，整理和总结出来的。

背景概要
----

Java 传统的并发模型是**一请求一线程** （Thread-per-request），每个 `java.lang.Thread` 对应一个操作系统线程（平台线程），每个线程占用约 1MB 栈空间。在高并发场景下，线程数受到内存和操作系统线程数量的限制，一旦线程数过多就会耗尽资源，线程创建的成本很高，不能"无限"增加。同时，随着 CPU 调度的线程数增加，会导致更严重的资源争用，宝贵的 CPU 资源被损耗在上下文切换上。

为此，开发者常使用**线程池** 复用线程，来减少创建和销毁的开销。但静态配置的线程池大小难以调优：线程太多会浪费内存，线程太少又会导致吞吐率下降。传统线程池还缺乏运行时的监控和告警机制，线程阻塞时难以定位问题。

为了解决这些问题，出现了动态线程池框架（如 [Hippo4j](https://github.com/opengoofy/hippo4j)）。它通过运行时**动态调整参数** 、**监控告警** 等手段，来增强线程池的能力。不过，它的底层仍依赖平台线程，性能瓶颈依然存在。

面对这些挑战，Java 21（19 中首次作为预览版本出现）引入了**虚拟线程** （Virtual Threads），期望以轻量级的线程实现大规模并发。

一句话总结：**为了突破传统平台线程在内存占用、上下文切换以及线程池参数难以评估的瓶颈，Java引入了虚拟线程，以更轻量的方式实现高并发，解决"一请求一线程"模式下资源消耗过大的问题** 。

什么是虚拟线程？
--------

### 1. 基本概念 {#1}

Java 平台引入的**虚拟线程** 是一种由 JVM 管理的轻量级线程实现。它依然是 `java.lang.Thread` 的实例，但不再一一对应 OS 线程；当虚拟线程执行阻塞操作（如 I/O、`Thread.sleep` 等）时，JVM 会自动将其挂起并释放底层的 OS 线程供其他虚拟线程使用。虚拟线程具有以下特性：**创建/销毁开销极低** 、**内存占用少** （栈空间通常只有几百字节，存放于 Java 堆中）；**可支持数百万级别的并发线程；** 与现有线程 API 完全兼容，迁移现有代码无需大幅改动。官方文档将其描述为"轻量级线程，可以降低编写高吞吐并发应用的难度"。

虚拟线程的底层实现基于 **ProjectLoom** 的"**continuation** "机制。
> [Loom](https://openjdk.org/projects/loom/) 是 **OpenJDK的一个官方项目** ，旨在在 Java 平台上引入更轻量的并发模型。很多人会把两者混为一谈，但严格来说 **Loom** → 一个项目，包含虚拟线程、continuations、structured concurrency 等一揽子技术。虚拟线程是 Project Loom 里最核心、最先落地的成果。
>
> 简单理解：最早 loom 是单独的项目，实现效果非常哇塞，后面就被直接并购进 JDK 内部实现啦。

Continuation（有限定的协程，可类比 Go 中的协程）可以在执行过程中自行挂起和恢复，类似于将线程的调用栈保存到堆上，当线程阻塞时挂起，待事件完成时从堆中恢复执行。这种设计使得虚拟线程与平台线程不同：平台线程始终**占用一个独立的OS线程** ，而虚拟线程则是**多对一调度** （M:N 模式），大量虚拟线程可以映射到少量 OS 线程上执行。当虚拟线程遇到阻塞时，它被从底层线程上卸载（非阻塞态），释放 OS 线程去执行其它虚拟线程，从而避免了操作系统级的上下文切换开销。此外，虚拟线程仍支持线程局部变量（`ThreadLocal`）等特性，以保证兼容性。

![iShot_2025-06-26_20.53.53.png](https://article-images.zsxq.com/FtAys7l9gt6VxvK-VNIoG0ev9CFg "iShot_2025-06-26_20.53.53.png")

> 图片引用自得物技术文章[虚拟线程原理及性能分析](https://tech.dewu.com/article?id=89)。

与平台线程（平常大家使用的线程 Thread）相比，虚拟线程的**主要区别** 包括：

*  
**调度关系：** 平台线程是一种轻包装的 OS 线程，任何时候都会独占一个内核线程；虚拟线程不固定绑定 OS 线程，多对一共享执行。  
*  
**内存占用：** 平台线程需要较大的栈（默认约1MB），启动时开销显著；虚拟线程的栈初始只有几百字节且可按需扩展，内存开销极低。因此 JVM 可以同时创建和管理数以百万计的虚拟线程。  
*  
**适用场景：** 虚拟线程适合处理大量**阻塞型/IO密集型任务** ，因为挂起一个虚拟线程不会阻塞底层 OS 线程；而对于**CPU密集型任务** ，虚拟线程并不会带来速度提升，它与平台线程执行速度相当。  
*  
  **兼容性：** 虚拟线程完全兼容 `Thread` 和 `Executors` API，已有多线程代码几乎无需改动即可运行在虚拟线程上。现有的调试与诊断工具（如 `jstack`）也可以识别和显示虚拟线程。

虚拟线程并不是传统意义上完全独立的线程，而是由 JDK 的调度器管理，并在底层由线程池（如默认的 `ForkJoinPool`）承载执行任务。传统情况下，如果线程在阻塞操作（比如 I/O）时，会一直占用操作系统线程，既消耗内存栈空间，又浪费宝贵的并发资源。而实际上，这段等待的时间完全可以被用来处理其他任务。类似 Netty 的事件驱动模型通过非阻塞 I/O 实现了对资源的高效利用，而虚拟线程则通过挂起和恢复栈帧，让阻塞式编程在高并发场景下重新变得可行，从而应运而生。

### 2. 简单示例 {#2}

下面是一个简单示例，展示如何在 JDK21 中创建并启动虚拟线程：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class TestVirtualThread {
        public static void main(String[] args) throws InterruptedException {
            // 创建虚拟线程的几种方式
    ​
            // 方式1: 使用Thread.ofVirtual()
            Thread.ofVirtual()
                    .start(() -> System.out.println("Running in virtual thread: " + Thread.currentThread()));
    ​
            // 方式2: 使用Executors.newVirtualThreadPerTaskExecutor()
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                executor.submit(() -> System.out.println("Task in virtual thread: " + Thread.currentThread()));
            }
    ​
            // 方式3: 直接启动
            Thread.startVirtualThread(() -> {
                System.out.println("Simple virtual thread: " + Thread.currentThread());
            });
        }
    }
              
以上代码中，`Thread.ofVirtual()` 和 `Executors.newVirtualThreadPerTaskExecutor()` 会创建虚拟线程来执行任务，调用 `join()` 或 `Future.get()` 可以等待任务完成。

可以看输出日志，进一步印证 ForkJoinPool 理论：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Running in virtual thread: VirtualThread[#22]/runnable@ForkJoinPool-1-worker-1
    Task in virtual thread: VirtualThread[#25]/runnable@ForkJoinPool-1-worker-1
    Simple virtual thread: VirtualThread[#26]/runnable@ForkJoinPool-1-worker-1
              
虚拟线程 vs 动态线程池 {#vs}
-------------------

### 1. 创建成本 {#1}

在**创建成本** 方面，虚拟线程的开销非常低：创建一个虚拟线程仅需少量资源，可以瞬时创建成千上万的线程；而动态线程池使用的仍然是平台线程（OS 线程），启动时需要较大的栈空间和系统调用，开销较高。不过动态线程池在应用启动时通常已经创建好一定数量的线程，并在运行时复用它们，这样后续提交任务时无需反复创建销毁线程，摊薄了创建成本。

### 2. 调度效率 {#2}

在**调度效率** 方面，虚拟线程由 JVM 调度。当虚拟线程遇到阻塞操作时，它被挂起并释放底层 OS 线程，让其他虚拟线程继续运行。这意味着在 I/O 密集型场景下，虚拟线程可以显著提高并发度，因为一个线程的阻塞不会影响其他线程运行。而动态线程池的线程属于平台线程，一旦执行阻塞调用，该线程就被占用无法执行其他任务（只能通过增加线程池规模来提升吞吐）。因此虚拟线程在高并发阻塞场景下的**上下文切换成本** 几乎为零，而平台线程池中的线程切换需要依赖操作系统，开销更大。

### 3. 资源占用 {#3}

在**资源占用** 方面，虚拟线程单个实例占用内存极少（几 KB 级别的栈），因此即使同时存在上百万个虚拟线程，也只消耗少量内存。平台线程每个通常占用约1MB栈空间，若线程池规模很大，则内存消耗巨大。动态线程池的线程数量可动态调整，但底层仍是平台线程，其每个线程固定内存开销不可避免。此外，动态线程池还维护任务队列和监控数据，这些也占用一定内存和 CPU 资源。

### 4. 使用复杂度 {#4}

在**使用复杂度** 方面，虚拟线程的编程模型与传统线程基本相同，开发者只需将 `Thread.ofVirtual()` 或 `newVirtualThreadPerTaskExecutor()` 替换原有线程池/线程创建代码即可，无需额外库或复杂配置。而使用动态线程池则需引入第三方框架（如 Hippo4j、oneThread 等）并进行配置中心或服务部署，同时编写配置文件或代码来管理线程池参数，学习成本和运维成本较高。

虚拟线程在 JDK21+ 环境下可直接使用，旧版本（JDK 19/20）则需要开启预览特性；动态线程池对 JDK 版本没有特殊要求，但需要额外的框架依赖和运行环境。

### 5. 调试与观测性 {#5}

在**调试与可观测性** 方面，虚拟线程可以借助现有的 Java 调试工具进行跟踪。例如，`jstack` 或 IDE 调试都能看到虚拟线程信息。由于虚拟线程不使用统一的任务队列，无法像线程池那样监控队列长度或活跃线程数等指标；但可以通过第三方埋点或 JVM 提供的线程 MXBean 查询当前活跃虚拟线程数。相比之下，动态线程池（以 Hippo4j 为例）提供了可视化控制台和报警系统，可实时查看每个线程池的运行状态、线程数、任务队列长度等信息。Hippo4j 还支持将线程池运行时数据推送到 Prometheus、InfluxDB 等监控系统，便于建立告警和趋势分析。总的来说，动态线程池在可观测性和运维支持方面更为丰富，而虚拟线程则偏向于简化开发、提升吞吐。

Web 容器如何关联虚拟线程？ {#web}
----------------------

### 1. Tomcat 如何启动虚拟线程？ {#1-tomcat}

网上很多举例都是错的，包括很多 AI 也是在胡言乱语。

我是通过 JDK21 和 SpringBoot 3.2.0 尝试，加上下述配置，将 SpringBoot Tomcat 虚拟线程启动成功。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    spring:
      threads:
        virtual:
          enabled: true
              
可以创建一个简单的 Controller，在其中打印当前线程的信息。虚拟线程的名称和类型与传统的平台线程有明显区别。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;
    ​
    @RestController
    public class ThreadInfoController {
    ​
        private static final Logger log = LoggerFactory.getLogger(ThreadInfoController.class);
    ​
        @GetMapping("/hello-virtual")
        public void getThreadInfo() {
            // Thread.currentThread() 会返回当前线程的引用
            Thread currentThread = Thread.currentThread();
            String threadInfo = "Response from thread: " + currentThread;
            
            // isVirtual() 方法可以明确判断是否为虚拟线程
            boolean isVirtual = currentThread.isVirtual();
    ​
            log.info(threadInfo);
            log.info("Is this a virtual thread? {}", isVirtual);
        }
    }
              
控制台看到类似以下的日志：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    2025-07-06T16:13:47.190+08:00  INFO 18047 --- [omcat-handler-0] c.n.t.v.ThreadInfoController             : Response from thread: VirtualThread[#53,tomcat-handler-0]/runnable@ForkJoinPool-1-worker-1
    2025-07-06T16:13:47.191+08:00  INFO 18047 --- [omcat-handler-0] c.n.t.v.ThreadInfoController             : Is this a virtual thread? true
              
线程名以 `VirtualThread` 开头，并且 `isVirtual()` 返回 `true`。

### 2. 虚拟线程和工作线程关系 {#2}

在 SpringBoot 3.2.0 中，没有较多说明如果配置了虚拟线程，默认 Web 容器的工作线程数量是否还有效？比如下面这个配置：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    server:
      tomcat:
        threads:
          max: 200
          min-spare: 10
              
后续我又把 SpringBoot 版本升级到了 3.5.0，配置属性说明中增加了描述：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    {
      "name": "server.tomcat.threads.max",
      "type": "java.lang.Integer",
      "description": "Maximum amount of worker threads. Doesn't have an effect if virtual threads are enabled.",
      "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat$Threads",
      "defaultValue": 200
    }
    {
      "name": "server.tomcat.threads.min-spare",
      "type": "java.lang.Integer",
      "description": "Minimum amount of worker threads. Doesn't have an effect if virtual threads are enabled.",
      "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat$Threads",
      "defaultValue": 10
    }
              
大致意思是：一旦启用了虚拟线程，Tomcat 的这两个线程相关参数就会失效。

不过虽然目前已经对虚拟线程的机制有了初步认识，但仍存在一些疑问，比如虚拟线程毕竟运行在平台线程之上，默认情况下 Web 容器虚拟线程和业务代码中的虚拟线程是否可能混用平台线程？这一点目前还没有深入验证，后续可以再细看相关实现。

### 3. Web 容器配置规范思考 {#3-web}

其实这里我还有点小疑惑：为什么 Tomcat、Jetty 等 Web 容器的虚拟线程开关，要放在 `spring` 前缀的配置里？官方文档里暂时没看到特别明确的解释。

我能想到的可能原因，是 SpringBoot 想做 **统一管理** 、**简化配置** 、**保持一致性** 。毕竟 SpringBoot 一直主张"约定优于配置"，把虚拟线程当作一种应用层的能力，而不仅仅是容器里的底层细节，也就不奇怪了。

不过因为还得忙别的资料，我还没仔细去研究这块源码。有兴趣的小伙伴可以去挖一挖，说不定能找到更深入的答案。

虚拟线程的缺点
-------

### 1. 虚拟线程固定（Pinned）问题 {#1-pinned}

#### 1.1 什么是虚拟线程的 Pinned 问题？ {#1-1-pinned}

**虚拟线程的Pin问题** 指的是当虚拟线程被"固定"在其承载线程上，无法卸载时的情形。一旦虚拟线程被固定（pinned），它即使遇到阻塞操作也无法释放承载线程，导致底层的操作系统线程也被长时间占用。根据 Oracle 官方文档的定义：当虚拟线程处于以下场景时，就会被视为已"固定"在承载线程上。

以下是一些常见的会导致虚拟线程被固定的场景：

*  
**同步块或同步方法（`synchronized`）** ：当虚拟线程在执行 `synchronized` 关键字控制的代码块或方法时，它会被认为已固定在当前承载线程上，直到离开该同步块。换言之，在同步块内部发生阻塞（例如 I/O 或 `Thread.sleep` 等）会引起固定。  
*  
**对象的`wait()/notify()`等同步阻塞** ：如果虚拟线程在持有对象监视器（在同步方法内）时调用 `wait()`，则线程会在 JVM 内部阻塞，承载线程也被固定，直到收到通知并重新获得监视器。  
*  
  **其他潜在的native阻塞调用** ：任何直接在底层进行阻塞、且 JVM 无法自动卸载的操作，都可能导致线程固定。例如，一些底层的 I/O 实现或特殊库调用，若不使用可挂载的 I/O 机制，也会固定虚拟线程。

**注意** ：并非所有同步操作都会立即固定虚拟线程。根据最新改进（JEP 491），在未来版本中虚拟线程进入 `synchronized` 时遇到阻塞会**尝试自动释放承载线程** 。但在当前 JDK（21/22/23）中，上述同步与本地调用场景均会导致固定。

#### 1.2 Pin 对伸缩性和性能的影响 {#1-2-pin}

虚拟线程之所以引入并发优势，是因为它们可以"挂载/卸载"在少量承载线程上实现高并发。但是当线程被固定时，这种优势就会受损：固定的虚拟线程在阻塞时**不能释放承载线程资源** ，导致其它虚拟线程无法使用该承载线程。举例而言，如果一个虚拟线程在 `synchronized` 方法中执行 Socket 读操作并被阻塞，它所在的承载线程将被占用，无法调度其他虚拟线程使用该线程。当大量虚拟线程因固定而无法卸载时，就需要更多的平台线程去支撑并发，这增加了上下文切换成本和系统开销，从而显著降低虚拟线程的伸缩性。长期来看，频繁且长时间的固定甚至可能导致资源饥饿或死锁：所有承载线程都被锁定在虚拟线程上时，系统将无法调度新的任务。

简而言之，被固定的虚拟线程"抹煞"了虚拟线程的轻量级优势：它们被迫像传统平台线程一样长期占用内核线程资源。这会导致虚拟线程在高并发场景下性能下降，吞吐量降低，反而不如使用普通线程池。因此，在采用虚拟线程时，应尽量避免出现会触发固定的编程模式。

#### 1.3 避免和解决 Pin 问题的实践建议 {#1-3-pin}

要减少虚拟线程固定带来的问题，可以从编程习惯和架构设计上做出调整，主要有以下建议：

*  
**避免在同步块中执行长时间阻塞操作** ：尽量不要在 `synchronized` 块或同步方法内调用耗时 I/O、`Thread.sleep()` 等阻塞操作。如果必须进行阻塞调用，可将其从同步块中移出，或拆分代码逻辑，缩短同步块的范围。示例中，将内部的 `Thread.sleep` 和计数操作用 `ReentrantLock` 实现，可以让线程及时释放锁，缩短固定持续时间。总体原则是：**监视器锁中只保护共享状态，而不进行阻塞I/O** 。  
*  
**使用`java.util.concurrent`锁而非`synchronized`** ：像 `ReentrantLock`、`Semaphore` 等并发工具不会导致虚拟线程固定。例如，用 `ReentrantLock` 包裹临界区，当线程遇到阻塞时可以先释放底层锁。许多库已经将同步结构改为并发锁，以避免固定问题。  
*  
**更新依赖和库** ：使用对虚拟线程友好的库版本。例如，一些底层网络或数据库驱动会针对 Loom 优化，减少使用同步锁或提供异步 API。TheServerSide 建议更新相关依赖至专门优化过的版本，或在应用层使用异步/非阻塞的替代方案，以彻底消除固定源头。  
*  
  **关注JVM新特性** ：JDK 24+（已通过 JEP 491）将改进 `synchronized` 实现，使得虚拟线程在大多数情况下即使在同步方法内部也能挂载/卸载，从而"几乎消除"固定。这意味着后续版本中，开发者只要遵循好的编程实践，通常无需特意回避同步。但在当前 JDK 21--23 版本中，上述做法依旧很有必要。

### 2. 避免用 ThreadLocal 池化资源 {#2-thread-local}

虚拟线程完全支持 `ThreadLocal` 和 `InheritableThreadLocal`，因此可以兼容原本依赖 ThreadLocal 的现有代码。但与平台线程不同，虚拟线程的设计初衷是"一线程一任务，用完即销毁"，而不是在线程池中反复复用线程来执行多个任务。

这意味着，在虚拟线程中使用 `ThreadLocal` 池化资源不再能带来节省开销的好处，反而可能导致每个虚拟线程都单独分配内存，从而增加内存占用和垃圾回收压力，尤其是池化较大的对象时风险更高。

此外，由于虚拟线程可能会在不同的承载线程之间迁移，也不能依赖 carrier thread 的 `ThreadLocal` 数据。因此，虚拟线程场景下，不建议使用 `ThreadLocal` 来池化资源，而是应当直接创建短生命周期对象，或在确实需要复用大对象时采用显式的对象池。

为适配虚拟线程，JDK 自身也在逐步移除基础库中过多依赖 `ThreadLocal` 的用法，以降低在高并发虚拟线程场景下的内存占用。

不止 JDK 自身，像一些开源软件也在逐渐替代 ThreadLocal，比如 Sentinel 通过 DateTimeFormatter 替换 ThreadLocal\<SimpleDateFormat\> 以更好支持虚拟线程。

参考链接：[Sentinel：替换ThreadLocal\<SimpleDateFormat\>以适配Java21虚拟线程](https://github.com/alibaba/Sentinel/issues/3321)

### 3. 无标准使用规范 {#3}

**再好的新技术，盲目上也可能带来意想不到的风险** 。像当初线程池或者其他新框架出来时，通常很快就会有大量教程、视频、文档、规范跟上。可反观虚拟线程，虽然是一项创新且里程碑式的技术，但目前各大公司还没太多系统的讲解规范，也缺少真实的生产落地案例。
> 根据我查阅大量网上资料，目前虚拟线程的应用，大多还停留在 Demo 级别，或者仅在少数公司中进行尝试，距离形成大规模的技术"虹吸效应"还有不小的距离。这也可以理解，毕竟虚拟线程面临一个很现实的问题：跨越的 JDK 版本跨度太大。在当前国内的技术环境下，许多企业依然难以完成 JDK 升级，因此"升不动"几乎成了常态。
>
> 如果，假设如果，虚拟线程在 JDK10 就已经出现，那么绝不会出现如今"任你发布新版本，我照样坚守 Java 8"的局面。

这里参考网上一篇分享 [虚拟线程实际应用案例](https://juejin.cn/post/7514242912373866496) 的文章，给大家一些思考。

#### 3.1 事故表现 {#3-1}

在系统上线几周后，伴随公司运营部门发起的大促活动，流量迅速攀升。

起初，一切运转正常。然而，当并发量达到平时的 3 倍时，系统开始陆续出现异常：

*  
**18:23** --- 监控告警：部分订单处理超时  
*  
**18:25** --- 数据库连接池告警：连接数异常增长  
*  
**18:27** --- 系统整体响应时间飙升至 5 秒以上  
*  
**18:30** --- 服务开始返回 500 错误  
*  
  **18:35** --- 系统陷入完全不可用状态

#### 3.2 问题定位 {#3-2}

起初，怀疑是数据库性能瓶颈所致，但通过阿里云监控面板观察，数据库本身运行正常。

真正的问题出在连接数激增，业务日志中频繁出现大量数据库连接超时的错误：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    2025-05-15 16:26:33.245 ERROR [virtual-thread-1234] c.e.OrderService - 
    获取数据库连接超时: HikariPool-1 - Connection is not available, 
    request timed out after 30000ms.
              
业务代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 问题代码片段
    @Service
    public class OrderService {
        
        @Async  // 使用虚拟线程执行
        public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
            // 数据库查询
            Order order = orderRepository.findByOrderNo(request.getOrderNo());
            
            // 调用外部支付接口
            PaymentResult paymentResult = paymentClient.processPayment(request);
            
            // 更新订单状态
            order.setStatus(paymentResult.getStatus());
            orderRepository.save(order);
            
            return CompletableFuture.completedFuture(new OrderResult(order));
        }
    }
              
这里使用了 `@Async` 注解，底层线程池做了改造。代码如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 改造前：传统线程池
    @Configuration
    public class ThreadPoolConfig {

        @Bean
        public TaskExecutor taskExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(200);
            executor.setMaxPoolSize(500);
            executor.setQueueCapacity(1000);
            return executor;
        }
    }

    // 改造后：虚拟线程
    @Configuration
    public class VirtualThreadConfig {

        @Bean
        public TaskExecutor taskExecutor() {
            return new TaskExecutor() {
                private final ExecutorService virtualExecutor = 
                    Executors.newVirtualThreadPerTaskExecutor();
                
                @Override
                public void execute(Runnable task) {
                    virtualExecutor.submit(task);
                }
            };
        }
    }
              
#### 3.3 原因分析 {#3-3}

传统线程池与数据库连接池的设计有一个前提：**并发线程数量是有限且可控的** 。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 传统配置
    // 关于数据源这里仅举例，更多是在配置文件中设置
    @Configuration
    public class DataSourceConfig {

        @Bean
        public DataSource dataSource() {
            HikariConfig config = new HikariConfig();
            config.setMaximumPoolSize(100);  // 连接池大小
            config.setConnectionTimeout(30000);
            return new HikariDataSource(config);
        }
    }

    // 对应的线程池配置
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setMaxPoolSize(200);  // 线程数 vs 连接数比例约为 2:1
              
虚拟线程的优势在于可以创建数百万个线程而不耗尽内存，但这也意味着：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 虚拟线程场景下的问题
    public void demonstrateProblem() {
        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
        
        // 瞬间创建 10000 个虚拟线程
        for (int i = 0; i < 10000; i++) {
            executor.submit(() -> {
                // 每个虚拟线程都试图获取数据库连接
                try (Connection conn = dataSource.getConnection()) {
                    // 执行数据库操作
                    performDatabaseOperation(conn);
                }
            });
        }
    }
              
在高并发场景下，成千上万的虚拟线程同时竞争有限的数据库连接，导致连接池迅速耗尽。

而且，传统的 JVM 监控工具对虚拟线程的可观测性支持还不够完善：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 传统监控代码无法准确反映虚拟线程数量
    ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
    int threadCount = threadBean.getThreadCount();  // 只显示载体线程数量
    System.out.println("活跃线程数: " + threadCount);  // 误导性信息
              
#### 3.4 解决方案 {#3-4}

具体方案大家可以看下原文，这里简单整理下：

*  
针对虚拟线程场景，需要重新评估各种资源池的配置，比如加大连接池中连接的数量。  
*  
作者通过引入信号量（Semaphore）来控制虚拟线程对数据库等资源的并发度。不过这里在评论区有一些不同意见，觉得数据库连接池已经控制了并发度，加上信号量后续又多了一个可能调整的参数。作者回复：加信号量只是更细致的流控手段，加一层保护，防止极端情况下资源争抢。  
*  
  构建虚拟线程专用的监控，监控虚拟线程的活跃数量、执行时间等关键参数。

整体来看，这只是生产系统可能出现问题的众多环节之一。对于业务流程复杂且核心的重要系统来说，几乎没有人敢赌上线的新特性一定万无一失。

实际上，绝大多数对性能有要求的应用，在稳定性与性能之间的取舍，往往是通过简单地增加部署节点来解决的。很多公司的核心系统至今仍在使用 Spring Boot 1.5，本质上也是出于同样的考量。

常见问题答疑
------

### 1. 如何选择？ {#1}

上面讲了关于动态线程池和虚拟线程之间的区别，从使用角度上给大家列下：

*  
对于**IO密集型、高并发** 的场景（如高并发 HTTP 服务、RPC 调用、大量数据库查询等），虚拟线程能极大简化代码逻辑，直接使用一请求一线程的编程模型即可满足高吞吐需求。有测试表明，将阻塞的 RestTemplate 调用放在虚拟线程中执行，吞吐与复杂的响应式（WebFlux）代码相当，而开发成本更低。  
*  
对于**CPU密集型** 任务，虚拟线程并不会比平台线程更快。在这类场景下，还是需要限制线程数以避免过度切换，使用线程池并根据 CPU 核数合理设置核心/最大线程数更合适。此时可以考虑**结合使用** ：在业务逻辑上用虚拟线程管理阻塞操作，但对关键资源（如数据库连接、CPU 核）使用信号量或限流来控制并发上限。  
*  
  如果需要**灵活的线程池参数调整、细粒度监控或故障告警** ，动态线程池框架依然有优势。例如，在生产环境中可以借助 Hippo4j 实时增减线程数、调整队列大小，并设置线程利用率或拒绝率告警。对于现有无法迁移到 JDK21 的老系统，也可继续使用动态线程池来提升可观测性和运维便利。

总之，虚拟线程和动态线程池各有侧重：虚拟线程解决了**并发规模** 的问题，动态线程池解决了**管控和监控** 的问题。两者并非完全互斥，而是可以根据业务需求和技术栈灵活选择或组合使用。
> 还有一点，现在 JDK21 的市场占有率低，绝大部分生产应用还都在 JDK8 上，所以动态线程池依然是有市场需求。

### 2. 虚拟线程所依赖平台线程数量如何设置？ {#2}

可能有同学会有疑问，既然虚拟线程依托于平台线程运行，那平台线程创建多少合适？

*  
`-Djdk.virtualThreadScheduler.parallelism=N` 这个参数是并行线程数的设置，虚拟机会在大部分情况下维持的工作线程数量与 N 一致，这个参数需要用户根据自己的机器决定，一般情况下与机器的 CPU 核数一致（或略小于机器 CPU 核数）。  
*  
  `-Djdk.virtualThreadScheduler.maxPoolSize=M` 这个参数是最大允许创建的线程数量，实际应用中可以将 M 设置比较大，（M\>\>N，比如设置成M=1024），遇到 pin 情况后触发的补偿机制在当前线程数量未超过 M 时都会生效。在补偿结束后，线程池会尽量恢复到 N 个线程。这个补偿机制主要的目的是解决虚拟线程中的 pin 问题，这种问题遇到的概率本身非常低，且处理此情况的补偿机制带来的开销几乎可以忽略不计，因此用户无需担心该参数带来的负面影响。

此外，`-Djdk.virtualThreadScheduler.minRunnable` 已经开放，默认是 `max(parallelism / 2, 1)`。其余的参数暂时没有开放给用户。目前虚拟线程的工作线程池默认的 corePoolSize 实际与 parallelism 一致，keepAliveTime 是30s。

### 3. 为什么虚拟线程这么晚出来？ {#3}

为什么 Java 迟迟没有引入协程，而是直到今天才推出虚拟线程？我个人认为，这背后有几方面原因：

一方面，Java 的发展风格可以说是"稳健"，也有人觉得它显得保守甚至有些"缺乏进取心"。另一方面，虽然 Java 在并发编程上存在不少痛点，但过去它的工具箱一直"能用"，只是"没那么好用"。

实际上，Java 语言本身已经相当成熟，更重要的是，它拥有一个庞大的生态，各种框架在不同阶段不断为它补齐短板。比如：

* 1.  
创建和销毁线程的开销很大？Java 提供了线程池，前提是你需要清楚如何合理配置线程池。  
* 2.  
线程池配合 one-thread-per-connection 的 BIO 模式，在遇到大量连接和阻塞操作时难以支撑高并发。于是 Java 引入了 NIO，通过 I/O 多路复用，让少量线程就能处理海量连接。  
* 3.  
不过原生 NIO API 过于复杂、使用门槛高，因此诞生了 Netty，如今已经成为 Java 世界的通用网络 I/O 库。  
* 4.  
  即使使用 NIO，编写异步 + 回调风格的代码仍然很繁琐。为此，Java 社区又探索了响应式编程，比如 RxJava、Project Reactor，当然还有 Vert.x 等框架，进一步提升了异步编程的可用性。

所以，在过去很长一段时间里，Java 借助各种工具和框架，间接解决了协程缺位的问题。直到现在，JDK 才正式引入虚拟线程，从语言层面简化高并发编程，这既是顺应时代发展的必然，也体现了 Java 一贯的谨慎态度。

### 4. 有了 RxJava 或 Vert.x 等异步 API 为什么还要有虚拟线程？ {#4-rx-java-vert-x-api}

这么一看，确实没有什么问题是**只能** 通过虚拟线程才能解决的。多线程开发中常见的并发问题，比如共享变量的使用，在虚拟线程里同样存在。这也能理解，为什么 Java 在推动这件事情上一直显得动力不足。

不过，正如 OpenJDK 官网 Project Loom 提案《Project Loom: Fibers and Continuations for the Java Virtual Machine》所说：
> 我们使用这些异步 API，并不是因为它们更容易理解或编写------实际上它们更难；不是因为它们更容易调试或分析------实际上它们往往不会产生有意义的堆栈跟踪；不是因为它们比同步 API 编写得更好------它们通常更不优雅；也不是因为它们更适合这门语言或与现有代码更好地集成------事实上，它们更不适合。归根结底，原因在于，线程作为 Java 并发编程的基础单元，从内存占用和性能角度来看，远远不够高效。

为了最大化性能，Java 开发者过去不得不付出极高的复杂度成本。这就引出一个问题：是否有可能"既要、又要、还要"？也就是------既保留同步编程简洁、易读的风格，又能获得异步编程带来的高性能。

虚拟线程带来了这样的希望：用同步编程的方式，写出接近异步编程性能的代码。

### 5. 虚拟线程是否对响应式编程有影响？ {#5}

这里抛砖引玉提一句，虚拟线程的出现在一定程度上挑战了响应式（比如 Reactor、RxJava）编程的地位。过去，为了提升并发性能，开发者往往需要采用复杂的异步或响应式编程模型，而虚拟线程则让同步阻塞式编程在高并发场景下重新变得可行。这可能削弱了响应式编程在部分场景中的优势。当然，这一变化仍在持续演进，值得长期关注。

巨人的肩膀
-----

非常感谢阿里、得物以及技术爱好者所分享的经验，以下文章值得大家都看看。

*  
[JDK21有没有什么稳定、简单又强势的特性？](https://mp.weixin.qq.com/s/aoFo74SSXoaEIywu-pX-Ow)  
*  
[虚拟线程原理及性能分析](https://tech.dewu.com/article?id=89)  
*  
[✨JDK21✨虚拟线程彻底杀死响应式编程](https://zhuanlan.zhihu.com/p/22562251084)  
*  
[尝鲜JDK21虚拟线程](https://meethigher.top/blog/2023/jdk21-virtual-thread/)  
*  
[Java21手册（一）：虚拟线程 Virtual Threads](https://juejin.cn/post/7280746515526058038)  
*  
[虚拟线程：Java的新利器？](https://mp.weixin.qq.com/s/Mssgn3DLs1xjC_N_jzFAYA)  
*  
[虚拟线程生产事故复盘：警惕高性能背后的陷阱](https://juejin.cn/post/7514242912373866496)  
*  
  [虚拟线程 - VirtualThread源码透视](https://blog.51cto.com/throwable/5736388)

文末总结
----

虚拟线程的出现为 Java 并发编程带来了新的可能。它保留了传统"一线程对应一请求"的编程风格，却将线程的底层实现改为轻量级映射，大大提高了系统的并发吞吐。在许多 I/O 密集型场景下，采用虚拟线程往往能实现接近异步框架的吞吐量，同时保持了代码的简单和可调试性。

但是，虚拟线程并不会自动为所有场景提供最佳解决方案。对于需要严格限制并发数量、复杂监控或运行时动态调整的场景，传统的动态线程池（如 Hippo4j）仍然有其价值。

在实际应用中，可以根据业务特点选择合适的方案：对绝大多数阻塞型并发任务，推荐使用虚拟线程来简化开发；而在对线程池管理和故障可观测有严格要求的系统中，动态线程池则可以提供更丰富的运维支持。两者各有场景，并存并用，才能发挥最大优势。

另外也需要考虑新老项目，如果是新项目且公司愿意做技术尝试，可以切换到 JDK21 尝鲜虚拟线程。

完结，撒花 🎉  
虚拟线程能取代动态线程池吗？元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：技术博客讲解，无



*** ** * ** ***

内容摘要：自从 Go 进入大众视野以来，其协程特性带来的并发能力一直备受推崇。相比之下，Java 在这方面一度显得力不从心，几乎被"按在地上摩擦"。所幸，随着 JDK 19 到 21 推出了虚拟线程，Java 在并发领域迎来了重要的技术革新，也为自己扳回了一局。

很多同学可能会疑惑：在 JDK 引入虚拟线程之后，动态线程池是否还有存在的必要？本文将围绕虚拟线程与线程池（动态）展开讨论，分析虚拟线程的优缺点，并探讨线程池（动态）在实际场景中的应用价值。

为确保严谨性，本文章发布时间为 2025-07-06，JDK 最高调研到 21。

课程目录如下所示：

*  
背景概要  
*  
什么是虚拟线程？  
*  
虚拟线程 vs 动态线程池  
*  
Web 容器如何关联虚拟线程？  
*  
虚拟线程的缺点  
*  
常见问题答疑  
*  
巨人的肩膀  
*  
  文末总结

> 本文中有不少内容借助了 AI 生成，但整体框架和观点，都是我在查阅几十篇文章、结合自己的实战经验以及各大厂的文章说明后，整理和总结出来的。

背景概要
----

Java 传统的并发模型是**一请求一线程** （Thread-per-request），每个 `java.lang.Thread` 对应一个操作系统线程（平台线程），每个线程占用约 1MB 栈空间。在高并发场景下，线程数受到内存和操作系统线程数量的限制，一旦线程数过多就会耗尽资源，线程创建的成本很高，不能"无限"增加。同时，随着 CPU 调度的线程数增加，会导致更严重的资源争用，宝贵的 CPU 资源被损耗在上下文切换上。

为此，开发者常使用**线程池** 复用线程，来减少创建和销毁的开销。但静态配置的线程池大小难以调优：线程太多会浪费内存，线程太少又会导致吞吐率下降。传统线程池还缺乏运行时的监控和告警机制，线程阻塞时难以定位问题。

为了解决这些问题，出现了动态线程池框架（如 [Hippo4j](https://github.com/opengoofy/hippo4j)）。它通过运行时**动态调整参数** 、**监控告警** 等手段，来增强线程池的能力。不过，它的底层仍依赖平台线程，性能瓶颈依然存在。

面对这些挑战，Java 21（19 中首次作为预览版本出现）引入了**虚拟线程** （Virtual Threads），期望以轻量级的线程实现大规模并发。

一句话总结：**为了突破传统平台线程在内存占用、上下文切换以及线程池参数难以评估的瓶颈，Java引入了虚拟线程，以更轻量的方式实现高并发，解决"一请求一线程"模式下资源消耗过大的问题** 。

什么是虚拟线程？
--------

### 1. 基本概念 {#1}

Java 平台引入的**虚拟线程** 是一种由 JVM 管理的轻量级线程实现。它依然是 `java.lang.Thread` 的实例，但不再一一对应 OS 线程；当虚拟线程执行阻塞操作（如 I/O、`Thread.sleep` 等）时，JVM 会自动将其挂起并释放底层的 OS 线程供其他虚拟线程使用。虚拟线程具有以下特性：**创建/销毁开销极低** 、**内存占用少** （栈空间通常只有几百字节，存放于 Java 堆中）；**可支持数百万级别的并发线程；** 与现有线程 API 完全兼容，迁移现有代码无需大幅改动。官方文档将其描述为"轻量级线程，可以降低编写高吞吐并发应用的难度"。

虚拟线程的底层实现基于 **ProjectLoom** 的"**continuation** "机制。
> [Loom](https://openjdk.org/projects/loom/) 是 **OpenJDK的一个官方项目** ，旨在在 Java 平台上引入更轻量的并发模型。很多人会把两者混为一谈，但严格来说 **Loom** → 一个项目，包含虚拟线程、continuations、structured concurrency 等一揽子技术。虚拟线程是 Project Loom 里最核心、最先落地的成果。
>
> 简单理解：最早 loom 是单独的项目，实现效果非常哇塞，后面就被直接并购进 JDK 内部实现啦。

Continuation（有限定的协程，可类比 Go 中的协程）可以在执行过程中自行挂起和恢复，类似于将线程的调用栈保存到堆上，当线程阻塞时挂起，待事件完成时从堆中恢复执行。这种设计使得虚拟线程与平台线程不同：平台线程始终**占用一个独立的OS线程** ，而虚拟线程则是**多对一调度** （M:N 模式），大量虚拟线程可以映射到少量 OS 线程上执行。当虚拟线程遇到阻塞时，它被从底层线程上卸载（非阻塞态），释放 OS 线程去执行其它虚拟线程，从而避免了操作系统级的上下文切换开销。此外，虚拟线程仍支持线程局部变量（`ThreadLocal`）等特性，以保证兼容性。

![iShot_2025-06-26_20.53.53.png](https://article-images.zsxq.com/FtAys7l9gt6VxvK-VNIoG0ev9CFg "iShot_2025-06-26_20.53.53.png")

> 图片引用自得物技术文章[虚拟线程原理及性能分析](https://tech.dewu.com/article?id=89)。

与平台线程（平常大家使用的线程 Thread）相比，虚拟线程的**主要区别** 包括：

*  
**调度关系：** 平台线程是一种轻包装的 OS 线程，任何时候都会独占一个内核线程；虚拟线程不固定绑定 OS 线程，多对一共享执行。  
*  
**内存占用：** 平台线程需要较大的栈（默认约1MB），启动时开销显著；虚拟线程的栈初始只有几百字节且可按需扩展，内存开销极低。因此 JVM 可以同时创建和管理数以百万计的虚拟线程。  
*  
**适用场景：** 虚拟线程适合处理大量**阻塞型/IO密集型任务** ，因为挂起一个虚拟线程不会阻塞底层 OS 线程；而对于**CPU密集型任务** ，虚拟线程并不会带来速度提升，它与平台线程执行速度相当。  
*  
  **兼容性：** 虚拟线程完全兼容 `Thread` 和 `Executors` API，已有多线程代码几乎无需改动即可运行在虚拟线程上。现有的调试与诊断工具（如 `jstack`）也可以识别和显示虚拟线程。

虚拟线程并不是传统意义上完全独立的线程，而是由 JDK 的调度器管理，并在底层由线程池（如默认的 `ForkJoinPool`）承载执行任务。传统情况下，如果线程在阻塞操作（比如 I/O）时，会一直占用操作系统线程，既消耗内存栈空间，又浪费宝贵的并发资源。而实际上，这段等待的时间完全可以被用来处理其他任务。类似 Netty 的事件驱动模型通过非阻塞 I/O 实现了对资源的高效利用，而虚拟线程则通过挂起和恢复栈帧，让阻塞式编程在高并发场景下重新变得可行，从而应运而生。

### 2. 简单示例 {#2}

下面是一个简单示例，展示如何在 JDK21 中创建并启动虚拟线程：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class TestVirtualThread {
        public static void main(String[] args) throws InterruptedException {
            // 创建虚拟线程的几种方式
    ​
            // 方式1: 使用Thread.ofVirtual()
            Thread.ofVirtual()
                    .start(() -> System.out.println("Running in virtual thread: " + Thread.currentThread()));
    ​
            // 方式2: 使用Executors.newVirtualThreadPerTaskExecutor()
            try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
                executor.submit(() -> System.out.println("Task in virtual thread: " + Thread.currentThread()));
            }
    ​
            // 方式3: 直接启动
            Thread.startVirtualThread(() -> {
                System.out.println("Simple virtual thread: " + Thread.currentThread());
            });
        }
    }
              
以上代码中，`Thread.ofVirtual()` 和 `Executors.newVirtualThreadPerTaskExecutor()` 会创建虚拟线程来执行任务，调用 `join()` 或 `Future.get()` 可以等待任务完成。

可以看输出日志，进一步印证 ForkJoinPool 理论：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Running in virtual thread: VirtualThread[#22]/runnable@ForkJoinPool-1-worker-1
    Task in virtual thread: VirtualThread[#25]/runnable@ForkJoinPool-1-worker-1
    Simple virtual thread: VirtualThread[#26]/runnable@ForkJoinPool-1-worker-1
              
虚拟线程 vs 动态线程池 {#vs}
-------------------

### 1. 创建成本 {#1}

在**创建成本** 方面，虚拟线程的开销非常低：创建一个虚拟线程仅需少量资源，可以瞬时创建成千上万的线程；而动态线程池使用的仍然是平台线程（OS 线程），启动时需要较大的栈空间和系统调用，开销较高。不过动态线程池在应用启动时通常已经创建好一定数量的线程，并在运行时复用它们，这样后续提交任务时无需反复创建销毁线程，摊薄了创建成本。

### 2. 调度效率 {#2}

在**调度效率** 方面，虚拟线程由 JVM 调度。当虚拟线程遇到阻塞操作时，它被挂起并释放底层 OS 线程，让其他虚拟线程继续运行。这意味着在 I/O 密集型场景下，虚拟线程可以显著提高并发度，因为一个线程的阻塞不会影响其他线程运行。而动态线程池的线程属于平台线程，一旦执行阻塞调用，该线程就被占用无法执行其他任务（只能通过增加线程池规模来提升吞吐）。因此虚拟线程在高并发阻塞场景下的**上下文切换成本** 几乎为零，而平台线程池中的线程切换需要依赖操作系统，开销更大。

### 3. 资源占用 {#3}

在**资源占用** 方面，虚拟线程单个实例占用内存极少（几 KB 级别的栈），因此即使同时存在上百万个虚拟线程，也只消耗少量内存。平台线程每个通常占用约1MB栈空间，若线程池规模很大，则内存消耗巨大。动态线程池的线程数量可动态调整，但底层仍是平台线程，其每个线程固定内存开销不可避免。此外，动态线程池还维护任务队列和监控数据，这些也占用一定内存和 CPU 资源。

### 4. 使用复杂度 {#4}

在**使用复杂度** 方面，虚拟线程的编程模型与传统线程基本相同，开发者只需将 `Thread.ofVirtual()` 或 `newVirtualThreadPerTaskExecutor()` 替换原有线程池/线程创建代码即可，无需额外库或复杂配置。而使用动态线程池则需引入第三方框架（如 Hippo4j、oneThread 等）并进行配置中心或服务部署，同时编写配置文件或代码来管理线程池参数，学习成本和运维成本较高。

虚拟线程在 JDK21+ 环境下可直接使用，旧版本（JDK 19/20）则需要开启预览特性；动态线程池对 JDK 版本没有特殊要求，但需要额外的框架依赖和运行环境。

### 5. 调试与观测性 {#5}

在**调试与可观测性** 方面，虚拟线程可以借助现有的 Java 调试工具进行跟踪。例如，`jstack` 或 IDE 调试都能看到虚拟线程信息。由于虚拟线程不使用统一的任务队列，无法像线程池那样监控队列长度或活跃线程数等指标；但可以通过第三方埋点或 JVM 提供的线程 MXBean 查询当前活跃虚拟线程数。相比之下，动态线程池（以 Hippo4j 为例）提供了可视化控制台和报警系统，可实时查看每个线程池的运行状态、线程数、任务队列长度等信息。Hippo4j 还支持将线程池运行时数据推送到 Prometheus、InfluxDB 等监控系统，便于建立告警和趋势分析。总的来说，动态线程池在可观测性和运维支持方面更为丰富，而虚拟线程则偏向于简化开发、提升吞吐。

Web 容器如何关联虚拟线程？ {#web}
----------------------

### 1. Tomcat 如何启动虚拟线程？ {#1-tomcat}

网上很多举例都是错的，包括很多 AI 也是在胡言乱语。

我是通过 JDK21 和 SpringBoot 3.2.0 尝试，加上下述配置，将 SpringBoot Tomcat 虚拟线程启动成功。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    spring:
      threads:
        virtual:
          enabled: true
              
可以创建一个简单的 Controller，在其中打印当前线程的信息。虚拟线程的名称和类型与传统的平台线程有明显区别。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import org.slf4j.Logger;
    import org.slf4j.LoggerFactory;
    import org.springframework.web.bind.annotation.GetMapping;
    import org.springframework.web.bind.annotation.RestController;
    ​
    @RestController
    public class ThreadInfoController {
    ​
        private static final Logger log = LoggerFactory.getLogger(ThreadInfoController.class);
    ​
        @GetMapping("/hello-virtual")
        public void getThreadInfo() {
            // Thread.currentThread() 会返回当前线程的引用
            Thread currentThread = Thread.currentThread();
            String threadInfo = "Response from thread: " + currentThread;
            
            // isVirtual() 方法可以明确判断是否为虚拟线程
            boolean isVirtual = currentThread.isVirtual();
    ​
            log.info(threadInfo);
            log.info("Is this a virtual thread? {}", isVirtual);
        }
    }
              
控制台看到类似以下的日志：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    2025-07-06T16:13:47.190+08:00  INFO 18047 --- [omcat-handler-0] c.n.t.v.ThreadInfoController             : Response from thread: VirtualThread[#53,tomcat-handler-0]/runnable@ForkJoinPool-1-worker-1
    2025-07-06T16:13:47.191+08:00  INFO 18047 --- [omcat-handler-0] c.n.t.v.ThreadInfoController             : Is this a virtual thread? true
              
线程名以 `VirtualThread` 开头，并且 `isVirtual()` 返回 `true`。

### 2. 虚拟线程和工作线程关系 {#2}

在 SpringBoot 3.2.0 中，没有较多说明如果配置了虚拟线程，默认 Web 容器的工作线程数量是否还有效？比如下面这个配置：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    server:
      tomcat:
        threads:
          max: 200
          min-spare: 10
              
后续我又把 SpringBoot 版本升级到了 3.5.0，配置属性说明中增加了描述：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    {
      "name": "server.tomcat.threads.max",
      "type": "java.lang.Integer",
      "description": "Maximum amount of worker threads. Doesn't have an effect if virtual threads are enabled.",
      "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat$Threads",
      "defaultValue": 200
    }
    {
      "name": "server.tomcat.threads.min-spare",
      "type": "java.lang.Integer",
      "description": "Minimum amount of worker threads. Doesn't have an effect if virtual threads are enabled.",
      "sourceType": "org.springframework.boot.autoconfigure.web.ServerProperties$Tomcat$Threads",
      "defaultValue": 10
    }
              
大致意思是：一旦启用了虚拟线程，Tomcat 的这两个线程相关参数就会失效。

不过虽然目前已经对虚拟线程的机制有了初步认识，但仍存在一些疑问，比如虚拟线程毕竟运行在平台线程之上，默认情况下 Web 容器虚拟线程和业务代码中的虚拟线程是否可能混用平台线程？这一点目前还没有深入验证，后续可以再细看相关实现。

### 3. Web 容器配置规范思考 {#3-web}

其实这里我还有点小疑惑：为什么 Tomcat、Jetty 等 Web 容器的虚拟线程开关，要放在 `spring` 前缀的配置里？官方文档里暂时没看到特别明确的解释。

我能想到的可能原因，是 SpringBoot 想做 **统一管理** 、**简化配置** 、**保持一致性** 。毕竟 SpringBoot 一直主张"约定优于配置"，把虚拟线程当作一种应用层的能力，而不仅仅是容器里的底层细节，也就不奇怪了。

不过因为还得忙别的资料，我还没仔细去研究这块源码。有兴趣的小伙伴可以去挖一挖，说不定能找到更深入的答案。

虚拟线程的缺点
-------

### 1. 虚拟线程固定（Pinned）问题 {#1-pinned}

#### 1.1 什么是虚拟线程的 Pinned 问题？ {#1-1-pinned}

**虚拟线程的Pin问题** 指的是当虚拟线程被"固定"在其承载线程上，无法卸载时的情形。一旦虚拟线程被固定（pinned），它即使遇到阻塞操作也无法释放承载线程，导致底层的操作系统线程也被长时间占用。根据 Oracle 官方文档的定义：当虚拟线程处于以下场景时，就会被视为已"固定"在承载线程上。

以下是一些常见的会导致虚拟线程被固定的场景：

*  
**同步块或同步方法（`synchronized`）** ：当虚拟线程在执行 `synchronized` 关键字控制的代码块或方法时，它会被认为已固定在当前承载线程上，直到离开该同步块。换言之，在同步块内部发生阻塞（例如 I/O 或 `Thread.sleep` 等）会引起固定。  
*  
**对象的`wait()/notify()`等同步阻塞** ：如果虚拟线程在持有对象监视器（在同步方法内）时调用 `wait()`，则线程会在 JVM 内部阻塞，承载线程也被固定，直到收到通知并重新获得监视器。  
*  
  **其他潜在的native阻塞调用** ：任何直接在底层进行阻塞、且 JVM 无法自动卸载的操作，都可能导致线程固定。例如，一些底层的 I/O 实现或特殊库调用，若不使用可挂载的 I/O 机制，也会固定虚拟线程。

**注意** ：并非所有同步操作都会立即固定虚拟线程。根据最新改进（JEP 491），在未来版本中虚拟线程进入 `synchronized` 时遇到阻塞会**尝试自动释放承载线程** 。但在当前 JDK（21/22/23）中，上述同步与本地调用场景均会导致固定。

#### 1.2 Pin 对伸缩性和性能的影响 {#1-2-pin}

虚拟线程之所以引入并发优势，是因为它们可以"挂载/卸载"在少量承载线程上实现高并发。但是当线程被固定时，这种优势就会受损：固定的虚拟线程在阻塞时**不能释放承载线程资源** ，导致其它虚拟线程无法使用该承载线程。举例而言，如果一个虚拟线程在 `synchronized` 方法中执行 Socket 读操作并被阻塞，它所在的承载线程将被占用，无法调度其他虚拟线程使用该线程。当大量虚拟线程因固定而无法卸载时，就需要更多的平台线程去支撑并发，这增加了上下文切换成本和系统开销，从而显著降低虚拟线程的伸缩性。长期来看，频繁且长时间的固定甚至可能导致资源饥饿或死锁：所有承载线程都被锁定在虚拟线程上时，系统将无法调度新的任务。

简而言之，被固定的虚拟线程"抹煞"了虚拟线程的轻量级优势：它们被迫像传统平台线程一样长期占用内核线程资源。这会导致虚拟线程在高并发场景下性能下降，吞吐量降低，反而不如使用普通线程池。因此，在采用虚拟线程时，应尽量避免出现会触发固定的编程模式。

#### 1.3 避免和解决 Pin 问题的实践建议 {#1-3-pin}

要减少虚拟线程固定带来的问题，可以从编程习惯和架构设计上做出调整，主要有以下建议：

*  
**避免在同步块中执行长时间阻塞操作** ：尽量不要在 `synchronized` 块或同步方法内调用耗时 I/O、`Thread.sleep()` 等阻塞操作。如果必须进行阻塞调用，可将其从同步块中移出，或拆分代码逻辑，缩短同步块的范围。示例中，将内部的 `Thread.sleep` 和计数操作用 `ReentrantLock` 实现，可以让线程及时释放锁，缩短固定持续时间。总体原则是：**监视器锁中只保护共享状态，而不进行阻塞I/O** 。  
*  
**使用`java.util.concurrent`锁而非`synchronized`** ：像 `ReentrantLock`、`Semaphore` 等并发工具不会导致虚拟线程固定。例如，用 `ReentrantLock` 包裹临界区，当线程遇到阻塞时可以先释放底层锁。许多库已经将同步结构改为并发锁，以避免固定问题。  
*  
**更新依赖和库** ：使用对虚拟线程友好的库版本。例如，一些底层网络或数据库驱动会针对 Loom 优化，减少使用同步锁或提供异步 API。TheServerSide 建议更新相关依赖至专门优化过的版本，或在应用层使用异步/非阻塞的替代方案，以彻底消除固定源头。  
*  
  **关注JVM新特性** ：JDK 24+（已通过 JEP 491）将改进 `synchronized` 实现，使得虚拟线程在大多数情况下即使在同步方法内部也能挂载/卸载，从而"几乎消除"固定。这意味着后续版本中，开发者只要遵循好的编程实践，通常无需特意回避同步。但在当前 JDK 21--23 版本中，上述做法依旧很有必要。

### 2. 避免用 ThreadLocal 池化资源 {#2-thread-local}

虚拟线程完全支持 `ThreadLocal` 和 `InheritableThreadLocal`，因此可以兼容原本依赖 ThreadLocal 的现有代码。但与平台线程不同，虚拟线程的设计初衷是"一线程一任务，用完即销毁"，而不是在线程池中反复复用线程来执行多个任务。

这意味着，在虚拟线程中使用 `ThreadLocal` 池化资源不再能带来节省开销的好处，反而可能导致每个虚拟线程都单独分配内存，从而增加内存占用和垃圾回收压力，尤其是池化较大的对象时风险更高。

此外，由于虚拟线程可能会在不同的承载线程之间迁移，也不能依赖 carrier thread 的 `ThreadLocal` 数据。因此，虚拟线程场景下，不建议使用 `ThreadLocal` 来池化资源，而是应当直接创建短生命周期对象，或在确实需要复用大对象时采用显式的对象池。

为适配虚拟线程，JDK 自身也在逐步移除基础库中过多依赖 `ThreadLocal` 的用法，以降低在高并发虚拟线程场景下的内存占用。

不止 JDK 自身，像一些开源软件也在逐渐替代 ThreadLocal，比如 Sentinel 通过 DateTimeFormatter 替换 ThreadLocal\<SimpleDateFormat\> 以更好支持虚拟线程。

参考链接：[Sentinel：替换ThreadLocal\<SimpleDateFormat\>以适配Java21虚拟线程](https://github.com/alibaba/Sentinel/issues/3321)

### 3. 无标准使用规范 {#3}

**再好的新技术，盲目上也可能带来意想不到的风险** 。像当初线程池或者其他新框架出来时，通常很快就会有大量教程、视频、文档、规范跟上。可反观虚拟线程，虽然是一项创新且里程碑式的技术，但目前各大公司还没太多系统的讲解规范，也缺少真实的生产落地案例。
> 根据我查阅大量网上资料，目前虚拟线程的应用，大多还停留在 Demo 级别，或者仅在少数公司中进行尝试，距离形成大规模的技术"虹吸效应"还有不小的距离。这也可以理解，毕竟虚拟线程面临一个很现实的问题：跨越的 JDK 版本跨度太大。在当前国内的技术环境下，许多企业依然难以完成 JDK 升级，因此"升不动"几乎成了常态。
>
> 如果，假设如果，虚拟线程在 JDK10 就已经出现，那么绝不会出现如今"任你发布新版本，我照样坚守 Java 8"的局面。

这里参考网上一篇分享 [虚拟线程实际应用案例](https://juejin.cn/post/7514242912373866496) 的文章，给大家一些思考。

#### 3.1 事故表现 {#3-1}

在系统上线几周后，伴随公司运营部门发起的大促活动，流量迅速攀升。

起初，一切运转正常。然而，当并发量达到平时的 3 倍时，系统开始陆续出现异常：

*  
**18:23** --- 监控告警：部分订单处理超时  
*  
**18:25** --- 数据库连接池告警：连接数异常增长  
*  
**18:27** --- 系统整体响应时间飙升至 5 秒以上  
*  
**18:30** --- 服务开始返回 500 错误  
*  
  **18:35** --- 系统陷入完全不可用状态

#### 3.2 问题定位 {#3-2}

起初，怀疑是数据库性能瓶颈所致，但通过阿里云监控面板观察，数据库本身运行正常。

真正的问题出在连接数激增，业务日志中频繁出现大量数据库连接超时的错误：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    2025-05-15 16:26:33.245 ERROR [virtual-thread-1234] c.e.OrderService - 
    获取数据库连接超时: HikariPool-1 - Connection is not available, 
    request timed out after 30000ms.
              
业务代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 问题代码片段
    @Service
    public class OrderService {
        
        @Async  // 使用虚拟线程执行
        public CompletableFuture<OrderResult> processOrder(OrderRequest request) {
            // 数据库查询
            Order order = orderRepository.findByOrderNo(request.getOrderNo());
            
            // 调用外部支付接口
            PaymentResult paymentResult = paymentClient.processPayment(request);
            
            // 更新订单状态
            order.setStatus(paymentResult.getStatus());
            orderRepository.save(order);
            
            return CompletableFuture.completedFuture(new OrderResult(order));
        }
    }
              
这里使用了 `@Async` 注解，底层线程池做了改造。代码如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 改造前：传统线程池
    @Configuration
    public class ThreadPoolConfig {

        @Bean
        public TaskExecutor taskExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(200);
            executor.setMaxPoolSize(500);
            executor.setQueueCapacity(1000);
            return executor;
        }
    }

    // 改造后：虚拟线程
    @Configuration
    public class VirtualThreadConfig {

        @Bean
        public TaskExecutor taskExecutor() {
            return new TaskExecutor() {
                private final ExecutorService virtualExecutor = 
                    Executors.newVirtualThreadPerTaskExecutor();
                
                @Override
                public void execute(Runnable task) {
                    virtualExecutor.submit(task);
                }
            };
        }
    }
              
#### 3.3 原因分析 {#3-3}

传统线程池与数据库连接池的设计有一个前提：**并发线程数量是有限且可控的** 。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 传统配置
    // 关于数据源这里仅举例，更多是在配置文件中设置
    @Configuration
    public class DataSourceConfig {

        @Bean
        public DataSource dataSource() {
            HikariConfig config = new HikariConfig();
            config.setMaximumPoolSize(100);  // 连接池大小
            config.setConnectionTimeout(30000);
            return new HikariDataSource(config);
        }
    }

    // 对应的线程池配置
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setMaxPoolSize(200);  // 线程数 vs 连接数比例约为 2:1
              
虚拟线程的优势在于可以创建数百万个线程而不耗尽内存，但这也意味着：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 虚拟线程场景下的问题
    public void demonstrateProblem() {
        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
        
        // 瞬间创建 10000 个虚拟线程
        for (int i = 0; i < 10000; i++) {
            executor.submit(() -> {
                // 每个虚拟线程都试图获取数据库连接
                try (Connection conn = dataSource.getConnection()) {
                    // 执行数据库操作
                    performDatabaseOperation(conn);
                }
            });
        }
    }
              
在高并发场景下，成千上万的虚拟线程同时竞争有限的数据库连接，导致连接池迅速耗尽。

而且，传统的 JVM 监控工具对虚拟线程的可观测性支持还不够完善：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 传统监控代码无法准确反映虚拟线程数量
    ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
    int threadCount = threadBean.getThreadCount();  // 只显示载体线程数量
    System.out.println("活跃线程数: " + threadCount);  // 误导性信息
              
#### 3.4 解决方案 {#3-4}

具体方案大家可以看下原文，这里简单整理下：

*  
针对虚拟线程场景，需要重新评估各种资源池的配置，比如加大连接池中连接的数量。  
*  
作者通过引入信号量（Semaphore）来控制虚拟线程对数据库等资源的并发度。不过这里在评论区有一些不同意见，觉得数据库连接池已经控制了并发度，加上信号量后续又多了一个可能调整的参数。作者回复：加信号量只是更细致的流控手段，加一层保护，防止极端情况下资源争抢。  
*  
  构建虚拟线程专用的监控，监控虚拟线程的活跃数量、执行时间等关键参数。

整体来看，这只是生产系统可能出现问题的众多环节之一。对于业务流程复杂且核心的重要系统来说，几乎没有人敢赌上线的新特性一定万无一失。

实际上，绝大多数对性能有要求的应用，在稳定性与性能之间的取舍，往往是通过简单地增加部署节点来解决的。很多公司的核心系统至今仍在使用 Spring Boot 1.5，本质上也是出于同样的考量。

常见问题答疑
------

### 1. 如何选择？ {#1}

上面讲了关于动态线程池和虚拟线程之间的区别，从使用角度上给大家列下：

*  
对于**IO密集型、高并发** 的场景（如高并发 HTTP 服务、RPC 调用、大量数据库查询等），虚拟线程能极大简化代码逻辑，直接使用一请求一线程的编程模型即可满足高吞吐需求。有测试表明，将阻塞的 RestTemplate 调用放在虚拟线程中执行，吞吐与复杂的响应式（WebFlux）代码相当，而开发成本更低。  
*  
对于**CPU密集型** 任务，虚拟线程并不会比平台线程更快。在这类场景下，还是需要限制线程数以避免过度切换，使用线程池并根据 CPU 核数合理设置核心/最大线程数更合适。此时可以考虑**结合使用** ：在业务逻辑上用虚拟线程管理阻塞操作，但对关键资源（如数据库连接、CPU 核）使用信号量或限流来控制并发上限。  
*  
  如果需要**灵活的线程池参数调整、细粒度监控或故障告警** ，动态线程池框架依然有优势。例如，在生产环境中可以借助 Hippo4j 实时增减线程数、调整队列大小，并设置线程利用率或拒绝率告警。对于现有无法迁移到 JDK21 的老系统，也可继续使用动态线程池来提升可观测性和运维便利。

总之，虚拟线程和动态线程池各有侧重：虚拟线程解决了**并发规模** 的问题，动态线程池解决了**管控和监控** 的问题。两者并非完全互斥，而是可以根据业务需求和技术栈灵活选择或组合使用。
> 还有一点，现在 JDK21 的市场占有率低，绝大部分生产应用还都在 JDK8 上，所以动态线程池依然是有市场需求。

### 2. 虚拟线程所依赖平台线程数量如何设置？ {#2}

可能有同学会有疑问，既然虚拟线程依托于平台线程运行，那平台线程创建多少合适？

*  
`-Djdk.virtualThreadScheduler.parallelism=N` 这个参数是并行线程数的设置，虚拟机会在大部分情况下维持的工作线程数量与 N 一致，这个参数需要用户根据自己的机器决定，一般情况下与机器的 CPU 核数一致（或略小于机器 CPU 核数）。  
*  
  `-Djdk.virtualThreadScheduler.maxPoolSize=M` 这个参数是最大允许创建的线程数量，实际应用中可以将 M 设置比较大，（M\>\>N，比如设置成M=1024），遇到 pin 情况后触发的补偿机制在当前线程数量未超过 M 时都会生效。在补偿结束后，线程池会尽量恢复到 N 个线程。这个补偿机制主要的目的是解决虚拟线程中的 pin 问题，这种问题遇到的概率本身非常低，且处理此情况的补偿机制带来的开销几乎可以忽略不计，因此用户无需担心该参数带来的负面影响。

此外，`-Djdk.virtualThreadScheduler.minRunnable` 已经开放，默认是 `max(parallelism / 2, 1)`。其余的参数暂时没有开放给用户。目前虚拟线程的工作线程池默认的 corePoolSize 实际与 parallelism 一致，keepAliveTime 是30s。

### 3. 为什么虚拟线程这么晚出来？ {#3}

为什么 Java 迟迟没有引入协程，而是直到今天才推出虚拟线程？我个人认为，这背后有几方面原因：

一方面，Java 的发展风格可以说是"稳健"，也有人觉得它显得保守甚至有些"缺乏进取心"。另一方面，虽然 Java 在并发编程上存在不少痛点，但过去它的工具箱一直"能用"，只是"没那么好用"。

实际上，Java 语言本身已经相当成熟，更重要的是，它拥有一个庞大的生态，各种框架在不同阶段不断为它补齐短板。比如：

* 1.  
创建和销毁线程的开销很大？Java 提供了线程池，前提是你需要清楚如何合理配置线程池。  
* 2.  
线程池配合 one-thread-per-connection 的 BIO 模式，在遇到大量连接和阻塞操作时难以支撑高并发。于是 Java 引入了 NIO，通过 I/O 多路复用，让少量线程就能处理海量连接。  
* 3.  
不过原生 NIO API 过于复杂、使用门槛高，因此诞生了 Netty，如今已经成为 Java 世界的通用网络 I/O 库。  
* 4.  
  即使使用 NIO，编写异步 + 回调风格的代码仍然很繁琐。为此，Java 社区又探索了响应式编程，比如 RxJava、Project Reactor，当然还有 Vert.x 等框架，进一步提升了异步编程的可用性。

所以，在过去很长一段时间里，Java 借助各种工具和框架，间接解决了协程缺位的问题。直到现在，JDK 才正式引入虚拟线程，从语言层面简化高并发编程，这既是顺应时代发展的必然，也体现了 Java 一贯的谨慎态度。

### 4. 有了 RxJava 或 Vert.x 等异步 API 为什么还要有虚拟线程？ {#4-rx-java-vert-x-api}

这么一看，确实没有什么问题是**只能** 通过虚拟线程才能解决的。多线程开发中常见的并发问题，比如共享变量的使用，在虚拟线程里同样存在。这也能理解，为什么 Java 在推动这件事情上一直显得动力不足。

不过，正如 OpenJDK 官网 Project Loom 提案《Project Loom: Fibers and Continuations for the Java Virtual Machine》所说：
> 我们使用这些异步 API，并不是因为它们更容易理解或编写------实际上它们更难；不是因为它们更容易调试或分析------实际上它们往往不会产生有意义的堆栈跟踪；不是因为它们比同步 API 编写得更好------它们通常更不优雅；也不是因为它们更适合这门语言或与现有代码更好地集成------事实上，它们更不适合。归根结底，原因在于，线程作为 Java 并发编程的基础单元，从内存占用和性能角度来看，远远不够高效。

为了最大化性能，Java 开发者过去不得不付出极高的复杂度成本。这就引出一个问题：是否有可能"既要、又要、还要"？也就是------既保留同步编程简洁、易读的风格，又能获得异步编程带来的高性能。

虚拟线程带来了这样的希望：用同步编程的方式，写出接近异步编程性能的代码。

### 5. 虚拟线程是否对响应式编程有影响？ {#5}

这里抛砖引玉提一句，虚拟线程的出现在一定程度上挑战了响应式（比如 Reactor、RxJava）编程的地位。过去，为了提升并发性能，开发者往往需要采用复杂的异步或响应式编程模型，而虚拟线程则让同步阻塞式编程在高并发场景下重新变得可行。这可能削弱了响应式编程在部分场景中的优势。当然，这一变化仍在持续演进，值得长期关注。

巨人的肩膀
-----

非常感谢阿里、得物以及技术爱好者所分享的经验，以下文章值得大家都看看。

*  
[JDK21有没有什么稳定、简单又强势的特性？](https://mp.weixin.qq.com/s/aoFo74SSXoaEIywu-pX-Ow)  
*  
[虚拟线程原理及性能分析](https://tech.dewu.com/article?id=89)  
*  
[✨JDK21✨虚拟线程彻底杀死响应式编程](https://zhuanlan.zhihu.com/p/22562251084)  
*  
[尝鲜JDK21虚拟线程](https://meethigher.top/blog/2023/jdk21-virtual-thread/)  
*  
[Java21手册（一）：虚拟线程 Virtual Threads](https://juejin.cn/post/7280746515526058038)  
*  
[虚拟线程：Java的新利器？](https://mp.weixin.qq.com/s/Mssgn3DLs1xjC_N_jzFAYA)  
*  
[虚拟线程生产事故复盘：警惕高性能背后的陷阱](https://juejin.cn/post/7514242912373866496)  
*  
  [虚拟线程 - VirtualThread源码透视](https://blog.51cto.com/throwable/5736388)

文末总结
----

虚拟线程的出现为 Java 并发编程带来了新的可能。它保留了传统"一线程对应一请求"的编程风格，却将线程的底层实现改为轻量级映射，大大提高了系统的并发吞吐。在许多 I/O 密集型场景下，采用虚拟线程往往能实现接近异步框架的吞吐量，同时保持了代码的简单和可调试性。

但是，虚拟线程并不会自动为所有场景提供最佳解决方案。对于需要严格限制并发数量、复杂监控或运行时动态调整的场景，传统的动态线程池（如 Hippo4j）仍然有其价值。

在实际应用中，可以根据业务特点选择合适的方案：对绝大多数阻塞型并发任务，推荐使用虚拟线程来简化开发；而在对线程池管理和故障可观测有严格要求的系统中，动态线程池则可以提供更丰富的运维支持。两者各有场景，并存并用，才能发挥最大优势。

另外也需要考虑新老项目，如果是新项目且公司愿意做技术尝试，可以切换到 JDK21 尝鲜虚拟线程。

完结，撒花 🎉

