2025年07月01日 23:13  
深度解析线程池底层原理，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：线程池通过复用线程、管理任务队列与智能调度策略，大幅提升系统吞吐并降低资源消耗。其核心在于`ThreadPoolExecutor`的三大组件：**工作线程（Worker）实现任务执行与复用，阻塞队列缓冲任务洪峰，位运算变量（ctl）统一管控状态与线程数**。这种设计完美平衡了响应速度与资源利用率，是高并发场景的底层基石。本篇文章跟着Breathe一起看下线程池底层设计之美。

课程目录如下所示：

*  
线程池介绍  
*  
线程池概要  
*  
状态控制  
*  
线程池状态  
*  
任务调度  
*  
  文末总结

线程池介绍
-----

通常当我们提到"线程池"时，狭义上指的是 `ThreadPoolExectutor` 及其子类，而广义上则指整个 `Executor` 大家族：

![1718264475803-2cef03c1-5f3f-46e1-a483-0d7c32c267f1.png](https://article-images.zsxq.com/FtZdWFrxRJdSPMZ1Y3-SASG8DVrD "1718264475803-2cef03c1-5f3f-46e1-a483-0d7c32c267f1.png")

*  
`Executor`：整个体系的最上级接口，定义了 execute 方法。  
*  
`ExecutorService`：它在 `Executor` 接口的基础上，定义了 submit、shutdown 与 shutdownNow 等方法，完善了对 Future 接口的支持。  
*  
`AbstractExecutorService`：实现了 `ExecutorService` 中关于任务提交的方法，将这部分逻辑统一为基于 execute 方法完成，使得实现类只需要关系 execute 方法的实现逻辑即可。  
*  
  `ThreadPoolExecutor`：线程池实现类，完善了线程状态管理与任务调度等具体的逻辑，实现了上述所有的接口。

`ThreadPoolExecutor` 作为 `Executor` 体系下最通用的实现基本可以满足日常的大部分需求，不过实际上也有不少定制的扩展实现，比如：

*  
JDK 基于 `ThreadPoolExecutor` 实现了 `ScheduledThreadPoolExecutor` 用于支持任务调度。  
*  
Tomcat 基于 `ThreadPoolExecutor` 实现了一个同名的线程池，用于处理 Web 请求。  
*  
  Spring 基于 `ExecutorService` 接口提供了一个 `ThreadPoolTaskExecutor` 实现，它仍然基于内置的 `ThreadPoolExecutor` 运行，在这个基础上提供了不少便捷的方法。

`ThreadPoolExecutor` 基于生产者与消费者模型实现，从功能上可以分为三个部分：

*  
**线程池本体** ：负责维护运行状态、管理工作线程以及调度任务。  
*  
**工作队列** ：即在构造函数中指定的阻塞队列，它扮演者生产者消费者模型中缓冲区的角色。工作线程将会不断的从队列中获取并执行任务。  
*  
  **工作线程** ：即持有 `Thread` 对象的内部类 `Worker`，当一个 `Wroker` 被创建并启动以后，它将会不断的从工作队列中获取并执行任务，直到它因为获取任务超时、任务执行异常或线程池停机后才会终止运行。

当我们向线程池**提交任务** 时，线程池将根据下述逻辑处理任务：

*  
如果当前工作线程数**小于** 核心线程数，则**启动一个工作线程** 执行任务。  
*  
如果当前工作线程数**大于等于** 核心线程数，且阻塞队列**未满** ，则将任务**添加到阻塞队列** 。  
*  
如果当前工作线程数**大于等于** 核心线程数，且阻塞队列**已满** ，则**启动一个工作线程** 执行任务。  
*  
  如果当前工作线程数已达**最大值** ，且阻塞队列已满，则触发拒绝策略。

![1718263071002-c2c40c59-1389-4199-9555-c62837fecead.png](https://article-images.zsxq.com/FjX-Fcaq6pPcAhf2E4LN0aJcJoUi "1718263071002-c2c40c59-1389-4199-9555-c62837fecead.png")

而当一个**工作线程启动** 以后，它将会在一个 while 循环中重复执行下述逻辑：

* 1.  
通过 `getTask` 方法从工作队列中获取任务，如果拿不到任务就阻塞一段时间，直到超时或者获取到任务。如果成功获取到任务就进入下一步，否则就直接进入线程退出流程；  
* 2.  
调用 `Worker` 的 `lock` 方法加锁，保证一个线程只被一个任务占用；  
* 3.  
调用 `beforeExecute` 回调方法，随后开始执行任务，如果在执行任务的过程中发生异常则会被捕获；  
* 4.  
任务执行完毕或者因为异常中断，此后调用一次 `afterExecute` 回调方法，然后调用 `unlock` 方法解锁；  
* 5.  
  如果线程是因为异常中断，那么进入线程退出流程，否则回到步骤 1 进入下一次循环。

![1718123052179-1b426288-ee7f-4aee-85e4-b68ffe75c9f4.png](https://article-images.zsxq.com/FqToDBHUCM4QkPkPCCTlhrwQ-ONH "1718123052179-1b426288-ee7f-4aee-85e4-b68ffe75c9f4.png")

线程池概要
-----

### 1. 构造函数的参数 {#1}

`ThreadPoolExecutor` 类一共提供了四个构造方法，我们基于参数最完整构造方法了解一下线程池创建所需要的变量：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                              int maximumPoolSize, // 最大线程数
                              long keepAliveTime, // 非核心线程闲置存活时间
                              TimeUnit unit, // 时间单位
                              BlockingQueue<Runnable> workQueue, // 工作队列
                              ThreadFactory threadFactory, // 创建线程使用的线程工厂
                              RejectedExecutionHandler handler // 拒绝策略) {
    }
              
*  
**核心线程数** ：即长期存在的线程数，当线程池中运行线程未达到核心线程数时会优先创建新线程\*\*。\*\*  
*  
**最大线程数** ：当核心线程已满，工作队列已满，同时线程池中线程总数未超过最大线程数，会创建非核心线程。  
*  
**超时时间** ：非核心线程闲置存活时间，当非核心线程闲置的时的最大存活时间。  
*  
**时间单位** ：非核心线程闲置存活时间的时间单位。  
*  
**任务队列** ：当核心线程满后，任务会优先加入工作队列，等待核心线程消费。  
*  
**线程工厂** ：线程池创建新线程时使用的线程工厂。  
*  
  **拒绝策略** ：当工作队列已满，且线程池中线程数已经达到最大线程数时，执行的兜底策略。

线程池每个参数的作用算是一个老生常谈的问题了，这里我们不过多赘述，你只需大概了解这几个参数即可，在下文我们会结合源码和具体的场景进一步的带你了解他们具体含义。

### 2. 工作线程 Worker {#2-worker}

线程池的核心在于工作线程，在 `ThreadPoolExecutor` 中，每个工作线程都对应的一个内部类 `Worker`，它们都存放在一个 `HashSet` 中：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final HashSet<Worker> workers = new HashSet<Worker>();
              
`Worker` 类的大致结构如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
    ​
        // 线程对象
        final Thread thread;
        // 首个执行的任务，一般执行完任务后就保持为空
        Runnable firstTask;
        // 该工作线程已经完成的任务数
        volatile long completedTasks;
        
        Worker(Runnable firstTask) {
            // 默认状态为 -1，禁止中断直到线程启动为止
            setState(-1);
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
    ​
        public void run() {
            runWorker(this);
        }
    }
              
`Worker` 本身实现了 `Runnable` 接口，当创建一个 `Worker` 实例时，构造函数会通过我们**在创建线程池时指定的线程工厂创建一个Thread对象，并把当前的Worker对象作为一个Runnable绑定到线程里面** 。当调用它的 `run` 方法时，它会通过调用线程池的 `runWorker`反过来启动线程，此时 Worker 就开始运行了。

`Worker` 类继承了 `AbstractQueuedSynchronizer`，也就是我们一般说的 AQS，这意味着当我们操作 Worker 的时候，**它会通过AQS的同步机制来保证对工作线程的访问是线程安全** 。比如当工作线程开始执行任务时，就会"加锁"，直到任务执行结束以后才会"解锁"。

### 3. 主锁 mainLock {#3-main-lock}

在上文介绍工作线程的时候，我们会注意到，线程池直接使用一个 `HashSet` 来存储 `Worker` 示例，而 `HashSet` 本身却并非线程安全的，那在并发场景下要如何保证线程安全呢？

实际上，除了 `workers` 以外，线程池中还有大量非线程安全的变量，这里再举几个例子：

*  
`ctl`：记录线程池状态与工作线程数。  
*  
`largestPoolSize`/`corePoolSize`：最大/核心工作线程数。  
*  
`completedTaskCount`：已完成任务数。  
*  
  `keepAliveTime`：核心线程超时时间。

**这些变量实际上环环相扣，因此很难通过分别将它们改为原子变量/并发容器来保证线程安全** ，因此 `ThreadPoolExecutor` 选择为整个线程池提供一把主锁 `mainLock`，每次操作或读取这种全局性变量的时候，都需要获取主锁才能进行：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final ReentrantLock mainLock = new ReentrantLock();
              
比如获取当前工作线程数的时候：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public int getPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 如果线程已经开始停机，则返回 0，否则返回工作线程数量
            return runStateAtLeast(ctl.get(), TIDYING) ? 0
                : workers.size();
        } finally {
            mainLock.unlock();
        }
    }
              
总的来说，**线程池通过mainLock来保证全局配置的线程安全，而每个工作线程再通过AQS来保证工作线程自己的线程安全。**

状态控制
----

### 1. ctl {#1-ctl}

线程池拥有一个 `AtomicInteger` 类型的成员变量 ctl ，它是 control 的缩写，线程池分别通过 ctl 的高位低位来管理两部分状态信息：

*  
第一部分为高 3 位，用来记录线程池当前的运行状态。  
*  
  第二部分为低 29 位，用来记录线程池中的工作线程数。

![1718263851031-2e9267bc-ccfd-4970-a063-f8d90442e215.png](https://article-images.zsxq.com/Fu4YyGzrjLHpjPpJPL6K-bNFaKFK)

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    ​
    ​
    // 29（32-3）
    private static final int COUNT_BITS = Integer.SIZE - 3;
    ​
    // ======== 线程数相关常量 ========
    // 允许的最大工作线程（2^29-1 约5亿）
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    ​
    // ======== 线程状态相关常量 ========
    // 运行状态。线程池接受并处理新任务
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 关闭状态。线程池不能接受新任务，处理完剩余任务后关闭。调用shutdown()方法会进入该状态。
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 停止状态。线程池不能接受新任务，并且尝试中断旧任务。调用shutdownNow()方法会进入该状态。
    private static final int STOP       =  1 << COUNT_BITS;
    // 整理状态。由关闭状态转变，线程池任务队列为空时进入该状态，会调用terminated()方法。
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 终止状态。terminated()方法执行完毕后进入该状态，线程池彻底停止。
    private static final int TERMINATED =  3 << COUNT_BITS;
              
在实际使用时，线程池将会通过位运算从 ctl 变量中解析出所需要的部分，并且做出相应的修改。

### 2. 通过 clt 计算当前状态 {#2-clt}

在开始前，我们需要理解一下关于补码的知识：在计算机中，二进制数中的首位是符号位，即第一位为 0 时表示正数，为 1 时表示负数。当我们需要表示一个负数时，则需要对它对应的正数按位取反再 + 1（也就是先取反码再获得补码）。

举个例子，如果我们假设在代码中使用 1 字节 ------ 也就是 8 bit，即 8 位 ------ 来表示一个数字，那么这种情况下，1 的二进制表示方式为 0000 0001，而 -1 则为 1111 1111

![1718017083081-d985f3b1-edce-4b03-bb8d-6b034bdc7a06.png](https://article-images.zsxq.com/FjgQ0nNoTIIpFC76TK660vcFN5ic)

现在我们对补码有了基础的了解，那么就可以尝试着理解线程池是如何通过 `-1 << COUNT_BITS`这行代码来表示 `RUNNING` 这个状态的：

* 1.  
Java 中 int 是 4 个字节，即 32 bit，按上述过程 -1 这个值转为二进制即为 1 111......1111（32个1）；  
* 2.  
`COUNT_BITS`是 29，`-1 << COUNT_BITS`这行代码表示让 -1 左移 29 位。  
* 3.  
  我们对 -1 左移 29 位后得到了一个 32 位的 int 值，它转为二进制就是 1110...0000，即前 3 位为 1，其余 29 位都为 0。

同理，计算其他的几种状态，最终可知五种状态对应的二进制表示分别是：  

|     状态     |           二进制           |
|------------|-------------------------|
| RUNNING    | 1110...0....00（可记为 111） |
| SHUTDOWN   | 0000...0....00（可记为 000） |
| STOP       | 0010...0....00（可记为 001） |
| TIDYING    | 0100...0....00（可记为 010） |
| TERMINATED | 0110...0....00（可记为 011） |

有意思的地方在于，**RUNNING的符号位是1，说明它转为十进制以后是个负数，而除它以外其他的状态的符号位都是0，转为十进制之后都是正数** ，也就是说，我们可以这么认为：

**小于SHUTDOWN的就是RUNNING，大于SHUTDOWN就是停止中或者已停止。**

这也是后面状态计算的一些写法的基础。比如 `isRunning()`方法：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
              
### 3. 通过 ctl 计算工作线程数 {#3-ctl}

在代码中，我们会经常看到线程池通过这一段代码来获取状态：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 29（32-3）
    private static final int COUNT_BITS = Integer.SIZE - 3;
    ​
    // 允许的最大工作线程（2^29-1 约5亿）
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    ​
    private static int workerCountOf(int c)  {
        return c & CAPACITY;
    }
              
这个运算过程其实与上文运行状态的运算过程基本一致：

* 1.  
在 Java 中，int 值 1 的二进制表示为 0000......1，即除了最后一位为 1 以外其他都为 0；  
* 2.  
然后 `1 << COUNT_BITS` 表示让 1 左移 29 位，即得到 0010......0，即除了第三位为 1 以外其他位都为 0；  
* 3.  
随后对该值再 -1，即得到 0001......1，即除了前三位为 0 以外，其余 29 位皆为 1，此时该值实际上就是一个掩码了；  
* 4.  
  此后，我们执行 `c & CAPACITY`，实际上就会得到低 29 位的值，即当前线程中的线程数量。

这里我们举个例子，假如当前线程池处于 RUNNING 状态，且有 1 个工作线程，那么此时 ctl 值为 1110......0001，即前三位和最后一位为 1，其余位数都为 0，然后与 CAPACITY 进行与运算后，高三位全变为 0，此时 ctl 即为 0000......0001，也就是 1。

线程池状态
-----

### 1. 状态的流转 {#1}

在前文，我们提到线程池通过 ctl 一共可以表示**五种状态** ：

*  
**RUNNING** ：运行状态。线程池接受并处理新任务。  
*  
**SHUTDOWN** ：关闭状态。线程池不能接受新任务，处理完剩余任务后关闭。调用 `shutdown` 方法会进入该状态。  
*  
**STOP** ：停止状态。线程池不能接受新任务，并且尝试中断旧任务。调用 `shutdownNow` 方法会进入该状态。  
*  
**TIDYING** ：整理状态。由关闭状态转变，线程池任务队列为空且没有任何工作线程时时进入该状态，会调用 `terminated` 方法。  
*  
  **TERMINATED** ：终止状态。`terminated` 方法执行完毕后进入该状态，线程池彻底停止。

它们具体的流转关系可以参考下图：

![1718119274178-74c3b599-4c0a-4c71-9a0f-bcde6b3a5b98.png](https://article-images.zsxq.com/FrLOeMLJ_vp6HtQHIrDZll24wIG5)

除了这种运行状态，线程池还提供了一些方法让我们可以获取其他的指标信息，比如核心线程数、最大线程数等，这个咱们后面会详细讲。

### 2. 如何触发停机？ {#2}

当我们要停止运行一个线程池时，可以调用下述两个方法：

*  
**shutdown** ：中断线程池，不再添加新任务，同时**等待当前进行和队列中的任务完成** 。  
*  
  **shutdownNow** ：立即中断线程池，不再添加新任务，**同时中断所有工作中的任务，不再处理任务队列中任务** 。

#### 2.1 shutdown {#2-1-shutdown}

`shutdown` 是正常关闭，这个方法主要做了这几件事：

* 1.  
改变当前线程池状态为 `SHUTDOWN`；  
* 2.  
将线程池中的**空闲工作线程** 标记为中断；  
* 3.  
完成上述过程后将线程池状态改为 `TIDYING`；  
* 4.  
  此后等到最后一个线程也退出后则将状态改为 `TERMINATED`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        // 加锁
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 将线程池状态改为 SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断线程池中的所有空闲线程
            interruptIdleWorkers();
            // 钩子函数，默认空实现
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
              
#### 2.2 shutdownNow {#2-2-shutdown-now}

`shutdownNow` 与 `shutdown` 流程类似，不过它是立即停机，因此在细节上又有点区别：

* 1.  
改变当前线程池状态为 `STOP`；  
* 2.  
将线程池中的**所有工作线程** 标记为中断；  
* 3.  
**将任务队列中的任务全部移除** ；  
* 4.  
完成上述过程后将线程池状态改为 `TIDYING`；  
* 5.  
  此后等到最后一个线程也退出后则将状态改为 `TERMINATED`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 将线程池状态改为 STOP
            advanceRunState(STOP);
            // 中断线程池中的所有线程
            interruptWorkers();
            // 删除任务队列中的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
              
此外，而在 `addWorker` 或者`getTask` 等处理任务的相关方法里，会针对 `STOP` 或更进一步的状态做区分，比如在 `getTask` 方法中，如果线程池进入了 `STOP` 或更进一步的状态，则会直接返回而不会继续申请任务。

### 3. 真正的停机 {#3}

我们已经知道通过 `shutdownNow` 与 `shutdown`可以触发停机流程，当两个方法执行完毕后，线程池将会进入 `STOP` 或者 `SHUTDOWN` 状态。

但是此时线程池**并未真正的停机** ，真正的停机逻辑，需要等到线程通过 `processWorkerExit` 方法退出时，里面调用的 `tryTerminate` 方法：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 如果线程池不处于预停机状态，则不进行停机
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 如果当前还有工作线程，则不进行停机
            if (workerCountOf(c) != 0) {
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            // 线程现在处于预停机状态，尝试进行停机
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 尝试通过 CAS 将线程池状态修改为 TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        // 尝试通过 CAS 将线程池状态修改为 TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // 进入下一次循环
        }
    }
              
简单的来说，由于在当我们调用了停机方法时，实际上工作线程仍然还在执行任务，我们可能并没有办法立刻终止线程的执行。

因此，每个线程执行完任务并且开始退出时，它都有可能是线程池中最后一个线程，此时它就需要承担起后续的收尾工作：

* 1.  
将线程池状态修改为 `TIDYING`；  
* 2.  
调用 `terminated` 回调方法，触发自定义停机逻辑；  
* 3.  
将线程池状态修改为 `TERMINATED`；  
* 4.  
  唤醒通过 `awaitTerminated` 阻塞的外部线程。

至此，线程池就真正的停机了。

任务调度
----

### 1. 任务的提交 {#1}

当我们向线程池提交一个任务时 ------ 无论是通过 `execute`、`submit` 还是 `invokeAll/invokerAny` 方法 ------ 最终都会走到 `execute` 方法上来：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        // 如果当前工作线程数小于核心线程数，则启动一个新工作线程执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 如果当前工作线程数大于核心线程数，则尝试将任务添加到工作队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 如果任务添加成功，但是发现线程池已经不运行了，则移除任务并且触发拒绝策略
            if (!isRunning(recheck) && remove(command))
                reject(command);
            // 如果任务添加成功，但是线程池中已经没有工作线程，则添加一个新的工作线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
            
        // 如果当前工作线程数大于核心线程数，且工作队列已满，则启动一个新工作线程执行任务，
        // 如果工作线程数已经达到最大值，则直接触发拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
              
我们可以将上述逻辑简化并梳理为下图：

![1718121774197-e021bf5b-1f57-4bf2-ad29-8bbd395a489e.png](https://article-images.zsxq.com/FsJkW9FCnrbUXI3c1qjrSN5yVaXc)

当然，我们还可以进一步简化，将整段代码归类为四种分支逻辑：

*  
如果当前工作线程数小于核心线程数，则启动一个工作线程执行任务。  
*  
如果当前工作线程数大于等于核心线程数，且阻塞队列未满，则将任务添加到阻塞队列。  
*  
如果当前工作线程数大于等于核心线程数，且阻塞队列已满，则启动一个工作线程执行任务。  
*  
  如果当前工作线程数已达最大值，且阻塞队列已满，则触发拒绝策略。

根据上述逻辑，我们不难意识到，如果阻塞队列设置的比较大而难以打满，那么线程池中的线程数就会一直与核心线程数保持一致从而匀速消费。

### 2. 任务执行 {#2}

#### 2.1 线程复用 {#2-1}

当线程池通过 `addWorker` 方法创建了运行线程 `Worker` 后，线程会通过 `runWorker` 去运行相关任务。**该线程运行任务完成后并不会直接销毁** ，因为 `runWorker` 方法会让线程在一个 while 循环中，反复调用 getTask 方法从阻塞队列中获取新的任务并执行。

通过该机制保障了在满足工作线程不销毁的前提下，让工作线程**不断从阻塞队列中拿到新的任务并运行** ，保障了线程池线程复用机制。

![1714298457545-0a4cab65-6e00-4aaf-9cfd-82e1b6afacb9.png](https://article-images.zsxq.com/FtmtagdRssbc9wFJFpPAFrs5RBZS)

线程池中的执行线程被包装为了 `Worker` 对象，并通过内部字段 thread 运行。可以看到 `Worker` 对象实现了 `Runnable` 接口，重写 `run` 方法调用了 `runWorker` 方法。

能保障线程池中线程复用的精髓都在这个方法以及调用的 `getTask` 方法中，我将相关代码精简出来，会省略部分非核心逻辑，方便大家更容易理解逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ThreadPoolExecutor extends AbstractExecutorService {
        
        private final class Worker extends AbstractQueuedSynchronizer
            implements Runnable
        {
            // ......
            /**
             * 执行线程池中任务的线程
             */
            final Thread thread;
            
            public void run() {
                runWorker(this);
            }
            // ......
        }
        final void runWorker(Worker w) {
            Thread wt = Thread.currentThread();
            Runnable task = w.firstTask;
            w.firstTask = null;
            w.unlock(); // 新创建的Worker默认state为-1，AQS的unlock方法会将其改为0，此后允许使用interruptIfStarted()方法进行中断

            // 完成任务以后是否需要移除当前Worker，即当前任务是否意外退出
            boolean completedAbruptly = true;

            try {
                // 循环获取任务
                while (task != null || (task = getTask()) != null) {
                    // 加锁，防止 shutdown 时中断正在运行的任务
                    w.lock();
                    // 如果线程池状态为 STOP 或更后面的状态，中断线程任务
                    if ((runStateAtLeast(ctl.get(), STOP) ||
                         (Thread.interrupted() &&
                          runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                        wt.interrupt();
                    try {
                        // 钩子方法，默认空实现，需要自己提供
                        beforeExecute(wt, task);
                        Throwable thrown = null;
                        try {
                            // 执行任务
                            task.run();
                        } catch (RuntimeException x) {
                            thrown = x; throw x;
                        } catch (Error x) {
                            thrown = x; throw x;
                        } catch (Throwable x) {
                            thrown = x; throw new Error(x);
                        } finally {
                            // 钩子方法
                            afterExecute(task, thrown);
                        }
                    } finally {
                        task = null;
                        // 任务执行完毕
                        w.completedTasks++;
                        w.unlock();
                    }
                }

                completedAbruptly = false;
            } finally {
                // 当线程获取任务超时、或者线程异常退出时都会调用此方法移除当前线程，这两种情况的主要区别在于，
                // 如果是异常退出，那么只要线程池还在运行状态，就会立刻启动一个新线程；
                // 而如果不是异常退出，那么将会根据情况来是否要启动新线程，以保证线程池中保持足够数量的线程。
                processWorkerExit(w, completedAbruptly);
            }
        }
    }
              
简单的来说，这一段代码**在一个while循环中** 做了这样一件事情：

* 1.  
通过 `getTask` 方法从工作队列中获取任务，如果拿不到任务就阻塞一段时间，直到超时或者获取到任务。如果成功获取到任务就进入下一步，否则就直接调用 `processWorkerExit` 进入线程退出流程；  
* 2.  
调用 `Worker` 的 `lock` 方法加锁，保证一个线程只被一个任务占用；  
* 3.  
调用 `beforeExecute` 回调方法，随后开始执行任务，如果在执行任务的过程中发生异常则会被捕获；  
* 4.  
任务执行完毕或者因为异常中断，此后调用一次 `afterExecute` 回调方法，然后调用 `unlock` 方法解锁；  
* 5.  
  如果线程是因为异常中断，那么调用 `processWorkerExit` 进入线程退出流程，否则回到步骤 1 进入下一次循环。

我们可以将上述逻辑简化为下图：

![1718123052179-1b426288-ee7f-4aee-85e4-b68ffe75c9f4-20250701165704131.png](https://article-images.zsxq.com/FqToDBHUCM4QkPkPCCTlhrwQ-ONH)

#### 2.2 线程超时回收 {#2-2}

其中，`getTask` 中又涉及到线程的超时回收机制，按照上面的逻辑，那线程池里的线程岂不是会一直在 while 循环中拿阻塞队列的任务并运行？如何实现线程池中设置了 keepAliveTime 的空闲回收机制呢？

线程池中线程从阻塞队列中获取任务的 `getTask` 方法有两种机制，第一种是无限期等待阻塞队列中有任务了返回，还有一种是等待设置 `keepAliveTime` 时间之后返回，如果说等待了 `keepAliveTime` 时间之后阻塞队列中还是没有任务，那么会返回空，并在 `getTask` 方法中返回空，最终在 `runWorkerask` 方法中执行线程销毁逻辑，这个过程也就是超时回收。

![1714315541458-c7f9e393-6e59-496a-ae36-86da26333648.png](https://article-images.zsxq.com/FsfJZ61ru0MWnYZ0dLjc6d-2Jqdo)

超时回收重点在于 `getTask` 方法，其中有两行核心代码我标记了下。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 从阻塞队列中获取任务
     */ 
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // ......
            int wc = workerCountOf(c);

            // 核心1：判断是否需要带超时时间去阻塞队列中获取任务
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 核心2：是否需要带超时方式获取
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
              
1）如何判断是否需要带超时时间？

虽然代码里两个逻辑判断是反着的，但是我感觉这样讲解会更容易理解些。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    boolean timed = wc > corePoolSize || allowCoreThreadTimeOut;
              
`wc > corePoolSize` wc（工作线程数量）大于 corePoolSize（核心线程数），则 timed 的值为 true。这个比较容易理解，毕竟 `keepAliveTime` 参数的语意就是当线程池内的线程超过核心线程数后，多余的线程就会在 `keepAliveTime` 空余时间后进行销毁。

`allowCoreThreadTimeOut` 是线程池的一个配置选项，用于控制是否允许核心线程超时销毁。在标准的线程池实现中，**核心线程是线程池中一直保持活动状态的线程，不会被销毁------即使它们处于空闲状态** 。这样可以保证线程池能够快速响应任务的执行请求，避免线程的创建和销毁开销。

然而，有些情况下，**当线程池中的任务量较少或任务执行时间较长时，保持大量的核心线程可能会浪费系统资源** 。这时，可以通过将 allowCoreThreadTimeOut 设置为 true，**允许核心线程也受到空闲超时时间的影响，即使它们是核心线程也可能会被销毁** 。线程池只在乎 "核心线程"的数量而不会真的去区分线程是不是 "核心线程"。

只要走这个逻辑的时候，工作线程数小于核心线程数，那么它就是一个核心线程，等它再回到这个地方的时候，如果工作线程数已经大于核心线程数了，那么它就是一个非核心线程。

所以，两个条件任意为 true 则 timed 都会设置 true。

2）如何使用超时时间获取阻塞队列任务？

阻塞队列 `BlockingQueue` 接口提供了 take 和 poll 两个公共用于让线程阻塞的从队列中获取元素：

*  
take：不设置超时时间，线程会一直阻塞直到成功从队列里获取一个元素为止。  
*  
  poll：只阻塞指定时间，如果超时了还是没法从队列中获取一个元素就放弃。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface BlockingQueue<E> extends Queue<E> {

        /**
         * 检索并删除此队列的头部，一直等待到指定元素可用的等待时间
         */
        E poll(long timeout, TimeUnit unit)
            throws InterruptedException;

        /**
         * 检索并删除此队列的头部，必要时等待直到元素可用为止
         */
        E take() throws InterruptedException;
    }
              
这两个方法分别对应 `getTask` 方法中的核心 2 流程。上文有说，如果获取获取不到元素，那么就意味着线程已经空闲了一段时间，按照默认场景下，如果线程池的线程数多余核心线程数，而且又获取不到任务，那么迎接它的将是被销毁。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
    if (r != null)
        return r;
              
如果 `getTask` 方法返回空的 task，while 循环将会结束，执行 `processWorkerExit` 销毁逻辑。

3）如何销毁工作线程？

首先解析下 `completedAbruptly` 参数，对应两种场景：

*  
如果 `completedAbruptly` 为 true，表示线程执行任务时因为抛出了异常而退出。在这种情况下，会调用 `decrementWorkerCount` 方法来减少工作线程的计数。  
*  
  为 false 则继续执行正常工作线程销毁逻辑。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            // 移除工作线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        // 兜底策略，避免线程池中的线程全部被清除
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
              
最下面的这段逻辑，做了一层兜底。简单的来说，如果线程是因为抛出异常而退出的，那么会立刻让工作线程数量减一，并且立刻添加一个新的工作线程。而如果线程是因为在规定的超时时间内获取不到任务而退出，那么就需要分情况考虑是否要添加一个新的工作线程。

当核心线程设置是否超时情况下，有两种处理策略：

*  
**允许核心线程超时** ：那么就需要看看工作队列中是否有未消费的任务，如果队列已空，那么不需要补偿新的工作线程，如果工作队列未空，则仅在当前线程池没有任何工作线程时才补充一个新的工作线程。  
*  
  **不允许核心线程超时** ：那么就需要确认当前工作线程数是否已经大于核心线程数，如果已经大于则不需要补充新的工作线程，否则则需要。

4）keepAliveTime为零还会超时回收么？

如果将 `keepAliveTime` 设置为零，这意味着非核心线程在空闲后可以立即被回收，而不需要等待超时时间。

也就是这段代码中 poll 方法将等同于没有阻塞等待时间，如果阻塞队列中没有元素将立即返回。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
    if (r != null)
        return r;
              
线程池的 `keepAliveTime` 设置为零有什么问题呢？

*  
**线程资源浪费** ：线程池中的非核心线程在任务执行完毕后会立即被回收。这种设置可能导致线程频繁地创建和销毁，增加了线程管理的开销。  
*  
  **不利于线程重用** ：如果无法以保证工作队列始终非空，那么线程池中的线程就有可能因为经常无法从工作队列获取到任务而频繁的超时退出。这种情况下，一旦又无法保证以比较均匀的速度向线程池提交任务，那么线程池就会频繁的创建或者销毁线程，这无疑会增加线程池的压力，造成线程池的性能下降。

5）如何让线程不被回收？

可以通过将线程池的核心线程数和最大线程数设置为相同的值，可以确保线程池中的所有线程都被视为核心线程。这样，即使线程处于空闲状态，它们也不会被回收。

但有一点需要注意，如果将线程池中的线程设置为不可回收，这可能会导致线程资源的浪费，特别是当线程处于空闲状态时。

### 3. 任务缓冲 {#3}

线程池是一个非常典型的生产者和消费者模型，在这个模型中，由于生产者生产速度与消费者消费速度可能不一致，所以不会让消费者直接消费生产者的数据，而是通过一个缓冲区来进行解耦削峰。

在线程池中这个缓冲区即为工作队列，也就是我们在线程池的构造函数中指定的阻塞队列。

![1718264253293-d54ed1a5-86f7-4558-9c94-0a4ca29ea71d.png](https://article-images.zsxq.com/FoDP_L7op00ZGPVYrl8KmVgTrxLY)

JDK 默认提供了以下几种阻塞队列实现：

![1718170081903-5b1ed76d-8227-4edb-8509-bd65a4baf292.jpeg](https://article-images.zsxq.com/FjpjZVjp3URXJITRriS-jNW29kJN)

通过更换阻塞队列，配合调整线程池的核心线程数和最大线程数，我们就可以搭配出一些适用于特定场景的线程池，比如 JDK 默认在 `Executors` 提供的几种线程池：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 固定大小的线程池，核心线程数与最大线程数一致，阻塞队列无限大小
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    // 核心线程数为0，最大线程数不设限的线程池，阻塞队列容量为0（同步队列）
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
              
### 4. 拒绝策略 {#4}

当任务队列已满，并且线程池中线程也到达最大线程数的时候，就会调用拒绝策略。也就是 `reject()` 方法：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
              
默认支持的拒绝策略共四种：

*  
**AbortPolicy** ：拒绝策略，直接抛出异常，默认策略。  
*  
**CallerRunsPolicy** ：调用者运行策略，用调用者所在的线程来执行任务。  
*  
**DiscardOldestPolicy** ：弃老策略，无声无息的丢弃阻塞队列中靠最前的任务，并执行当前任务。  
*  
  **DiscardPolicy** ：丢弃策略，直接无声无息的丢弃任务。

当然，我们也可以实现 `RejectedExecutionHandler` 接口来定义自己的拒绝策略。比如在 oneThread 中我们就实现一个可以发送告警并记录拒绝次数的策略，从而为线程池的监控提供支持。

文末总结
----

纵观线程池的底层架构，其精妙之处在于将复杂的并发问题分解为三个维度的协同：**状态管理（通过ctl的高效位运算实现无锁化流转）、资源调度（核心/非核心线程的动态扩缩容与超时回收机制）、任务生命周期管控（从submit到afterExecute的闭环钩子）** 。这种设计不仅解决了高频线程创建的性能瓶颈，更通过拒绝策略与队列选型为系统提供柔性容错能力。

大家在学习和使用过程中，唯有深入理解 Worker 的 AQS 锁机制、任务缓冲队列的选型逻辑（如 LinkedBlockingQueue 与 SynchronousQueue 的场景差异）、以及 shutdownNow 与 shutdown的停机本质，才能更好地驾驭线程池这座"并发引擎"，在资源约束与性能需求间找到精准平衡点。

完结，撒花 🎉  
深度解析线程池底层原理，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：线程池通过复用线程、管理任务队列与智能调度策略，大幅提升系统吞吐并降低资源消耗。其核心在于`ThreadPoolExecutor`的三大组件：**工作线程（Worker）实现任务执行与复用，阻塞队列缓冲任务洪峰，位运算变量（ctl）统一管控状态与线程数**。这种设计完美平衡了响应速度与资源利用率，是高并发场景的底层基石。本篇文章跟着Breathe一起看下线程池底层设计之美。

课程目录如下所示：

*  
线程池介绍  
*  
线程池概要  
*  
状态控制  
*  
线程池状态  
*  
任务调度  
*  
  文末总结

线程池介绍
-----

通常当我们提到"线程池"时，狭义上指的是 `ThreadPoolExectutor` 及其子类，而广义上则指整个 `Executor` 大家族：

![1718264475803-2cef03c1-5f3f-46e1-a483-0d7c32c267f1.png](https://article-images.zsxq.com/FtZdWFrxRJdSPMZ1Y3-SASG8DVrD "1718264475803-2cef03c1-5f3f-46e1-a483-0d7c32c267f1.png")

*  
`Executor`：整个体系的最上级接口，定义了 execute 方法。  
*  
`ExecutorService`：它在 `Executor` 接口的基础上，定义了 submit、shutdown 与 shutdownNow 等方法，完善了对 Future 接口的支持。  
*  
`AbstractExecutorService`：实现了 `ExecutorService` 中关于任务提交的方法，将这部分逻辑统一为基于 execute 方法完成，使得实现类只需要关系 execute 方法的实现逻辑即可。  
*  
  `ThreadPoolExecutor`：线程池实现类，完善了线程状态管理与任务调度等具体的逻辑，实现了上述所有的接口。

`ThreadPoolExecutor` 作为 `Executor` 体系下最通用的实现基本可以满足日常的大部分需求，不过实际上也有不少定制的扩展实现，比如：

*  
JDK 基于 `ThreadPoolExecutor` 实现了 `ScheduledThreadPoolExecutor` 用于支持任务调度。  
*  
Tomcat 基于 `ThreadPoolExecutor` 实现了一个同名的线程池，用于处理 Web 请求。  
*  
  Spring 基于 `ExecutorService` 接口提供了一个 `ThreadPoolTaskExecutor` 实现，它仍然基于内置的 `ThreadPoolExecutor` 运行，在这个基础上提供了不少便捷的方法。

`ThreadPoolExecutor` 基于生产者与消费者模型实现，从功能上可以分为三个部分：

*  
**线程池本体** ：负责维护运行状态、管理工作线程以及调度任务。  
*  
**工作队列** ：即在构造函数中指定的阻塞队列，它扮演者生产者消费者模型中缓冲区的角色。工作线程将会不断的从队列中获取并执行任务。  
*  
  **工作线程** ：即持有 `Thread` 对象的内部类 `Worker`，当一个 `Wroker` 被创建并启动以后，它将会不断的从工作队列中获取并执行任务，直到它因为获取任务超时、任务执行异常或线程池停机后才会终止运行。

当我们向线程池**提交任务** 时，线程池将根据下述逻辑处理任务：

*  
如果当前工作线程数**小于** 核心线程数，则**启动一个工作线程** 执行任务。  
*  
如果当前工作线程数**大于等于** 核心线程数，且阻塞队列**未满** ，则将任务**添加到阻塞队列** 。  
*  
如果当前工作线程数**大于等于** 核心线程数，且阻塞队列**已满** ，则**启动一个工作线程** 执行任务。  
*  
  如果当前工作线程数已达**最大值** ，且阻塞队列已满，则触发拒绝策略。

![1718263071002-c2c40c59-1389-4199-9555-c62837fecead.png](https://article-images.zsxq.com/FjX-Fcaq6pPcAhf2E4LN0aJcJoUi "1718263071002-c2c40c59-1389-4199-9555-c62837fecead.png")

而当一个**工作线程启动** 以后，它将会在一个 while 循环中重复执行下述逻辑：

* 1.  
通过 `getTask` 方法从工作队列中获取任务，如果拿不到任务就阻塞一段时间，直到超时或者获取到任务。如果成功获取到任务就进入下一步，否则就直接进入线程退出流程；  
* 2.  
调用 `Worker` 的 `lock` 方法加锁，保证一个线程只被一个任务占用；  
* 3.  
调用 `beforeExecute` 回调方法，随后开始执行任务，如果在执行任务的过程中发生异常则会被捕获；  
* 4.  
任务执行完毕或者因为异常中断，此后调用一次 `afterExecute` 回调方法，然后调用 `unlock` 方法解锁；  
* 5.  
  如果线程是因为异常中断，那么进入线程退出流程，否则回到步骤 1 进入下一次循环。

![1718123052179-1b426288-ee7f-4aee-85e4-b68ffe75c9f4.png](https://article-images.zsxq.com/FqToDBHUCM4QkPkPCCTlhrwQ-ONH "1718123052179-1b426288-ee7f-4aee-85e4-b68ffe75c9f4.png")

线程池概要
-----

### 1. 构造函数的参数 {#1}

`ThreadPoolExecutor` 类一共提供了四个构造方法，我们基于参数最完整构造方法了解一下线程池创建所需要的变量：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public ThreadPoolExecutor(int corePoolSize, // 核心线程数
                              int maximumPoolSize, // 最大线程数
                              long keepAliveTime, // 非核心线程闲置存活时间
                              TimeUnit unit, // 时间单位
                              BlockingQueue<Runnable> workQueue, // 工作队列
                              ThreadFactory threadFactory, // 创建线程使用的线程工厂
                              RejectedExecutionHandler handler // 拒绝策略) {
    }
              
*  
**核心线程数** ：即长期存在的线程数，当线程池中运行线程未达到核心线程数时会优先创建新线程\*\*。\*\*  
*  
**最大线程数** ：当核心线程已满，工作队列已满，同时线程池中线程总数未超过最大线程数，会创建非核心线程。  
*  
**超时时间** ：非核心线程闲置存活时间，当非核心线程闲置的时的最大存活时间。  
*  
**时间单位** ：非核心线程闲置存活时间的时间单位。  
*  
**任务队列** ：当核心线程满后，任务会优先加入工作队列，等待核心线程消费。  
*  
**线程工厂** ：线程池创建新线程时使用的线程工厂。  
*  
  **拒绝策略** ：当工作队列已满，且线程池中线程数已经达到最大线程数时，执行的兜底策略。

线程池每个参数的作用算是一个老生常谈的问题了，这里我们不过多赘述，你只需大概了解这几个参数即可，在下文我们会结合源码和具体的场景进一步的带你了解他们具体含义。

### 2. 工作线程 Worker {#2-worker}

线程池的核心在于工作线程，在 `ThreadPoolExecutor` 中，每个工作线程都对应的一个内部类 `Worker`，它们都存放在一个 `HashSet` 中：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final HashSet<Worker> workers = new HashSet<Worker>();
              
`Worker` 类的大致结构如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {
    ​
        // 线程对象
        final Thread thread;
        // 首个执行的任务，一般执行完任务后就保持为空
        Runnable firstTask;
        // 该工作线程已经完成的任务数
        volatile long completedTasks;
        
        Worker(Runnable firstTask) {
            // 默认状态为 -1，禁止中断直到线程启动为止
            setState(-1);
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
    ​
        public void run() {
            runWorker(this);
        }
    }
              
`Worker` 本身实现了 `Runnable` 接口，当创建一个 `Worker` 实例时，构造函数会通过我们**在创建线程池时指定的线程工厂创建一个Thread对象，并把当前的Worker对象作为一个Runnable绑定到线程里面** 。当调用它的 `run` 方法时，它会通过调用线程池的 `runWorker`反过来启动线程，此时 Worker 就开始运行了。

`Worker` 类继承了 `AbstractQueuedSynchronizer`，也就是我们一般说的 AQS，这意味着当我们操作 Worker 的时候，**它会通过AQS的同步机制来保证对工作线程的访问是线程安全** 。比如当工作线程开始执行任务时，就会"加锁"，直到任务执行结束以后才会"解锁"。

### 3. 主锁 mainLock {#3-main-lock}

在上文介绍工作线程的时候，我们会注意到，线程池直接使用一个 `HashSet` 来存储 `Worker` 示例，而 `HashSet` 本身却并非线程安全的，那在并发场景下要如何保证线程安全呢？

实际上，除了 `workers` 以外，线程池中还有大量非线程安全的变量，这里再举几个例子：

*  
`ctl`：记录线程池状态与工作线程数。  
*  
`largestPoolSize`/`corePoolSize`：最大/核心工作线程数。  
*  
`completedTaskCount`：已完成任务数。  
*  
  `keepAliveTime`：核心线程超时时间。

**这些变量实际上环环相扣，因此很难通过分别将它们改为原子变量/并发容器来保证线程安全** ，因此 `ThreadPoolExecutor` 选择为整个线程池提供一把主锁 `mainLock`，每次操作或读取这种全局性变量的时候，都需要获取主锁才能进行：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final ReentrantLock mainLock = new ReentrantLock();
              
比如获取当前工作线程数的时候：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public int getPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 如果线程已经开始停机，则返回 0，否则返回工作线程数量
            return runStateAtLeast(ctl.get(), TIDYING) ? 0
                : workers.size();
        } finally {
            mainLock.unlock();
        }
    }
              
总的来说，**线程池通过mainLock来保证全局配置的线程安全，而每个工作线程再通过AQS来保证工作线程自己的线程安全。**

状态控制
----

### 1. ctl {#1-ctl}

线程池拥有一个 `AtomicInteger` 类型的成员变量 ctl ，它是 control 的缩写，线程池分别通过 ctl 的高位低位来管理两部分状态信息：

*  
第一部分为高 3 位，用来记录线程池当前的运行状态。  
*  
  第二部分为低 29 位，用来记录线程池中的工作线程数。

![1718263851031-2e9267bc-ccfd-4970-a063-f8d90442e215.png](https://article-images.zsxq.com/Fu4YyGzrjLHpjPpJPL6K-bNFaKFK)

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    ​
    ​
    // 29（32-3）
    private static final int COUNT_BITS = Integer.SIZE - 3;
    ​
    // ======== 线程数相关常量 ========
    // 允许的最大工作线程（2^29-1 约5亿）
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    ​
    // ======== 线程状态相关常量 ========
    // 运行状态。线程池接受并处理新任务
    private static final int RUNNING    = -1 << COUNT_BITS;
    // 关闭状态。线程池不能接受新任务，处理完剩余任务后关闭。调用shutdown()方法会进入该状态。
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 停止状态。线程池不能接受新任务，并且尝试中断旧任务。调用shutdownNow()方法会进入该状态。
    private static final int STOP       =  1 << COUNT_BITS;
    // 整理状态。由关闭状态转变，线程池任务队列为空时进入该状态，会调用terminated()方法。
    private static final int TIDYING    =  2 << COUNT_BITS;
    // 终止状态。terminated()方法执行完毕后进入该状态，线程池彻底停止。
    private static final int TERMINATED =  3 << COUNT_BITS;
              
在实际使用时，线程池将会通过位运算从 ctl 变量中解析出所需要的部分，并且做出相应的修改。

### 2. 通过 clt 计算当前状态 {#2-clt}

在开始前，我们需要理解一下关于补码的知识：在计算机中，二进制数中的首位是符号位，即第一位为 0 时表示正数，为 1 时表示负数。当我们需要表示一个负数时，则需要对它对应的正数按位取反再 + 1（也就是先取反码再获得补码）。

举个例子，如果我们假设在代码中使用 1 字节 ------ 也就是 8 bit，即 8 位 ------ 来表示一个数字，那么这种情况下，1 的二进制表示方式为 0000 0001，而 -1 则为 1111 1111

![1718017083081-d985f3b1-edce-4b03-bb8d-6b034bdc7a06.png](https://article-images.zsxq.com/FjgQ0nNoTIIpFC76TK660vcFN5ic)

现在我们对补码有了基础的了解，那么就可以尝试着理解线程池是如何通过 `-1 << COUNT_BITS`这行代码来表示 `RUNNING` 这个状态的：

* 1.  
Java 中 int 是 4 个字节，即 32 bit，按上述过程 -1 这个值转为二进制即为 1 111......1111（32个1）；  
* 2.  
`COUNT_BITS`是 29，`-1 << COUNT_BITS`这行代码表示让 -1 左移 29 位。  
* 3.  
  我们对 -1 左移 29 位后得到了一个 32 位的 int 值，它转为二进制就是 1110...0000，即前 3 位为 1，其余 29 位都为 0。

同理，计算其他的几种状态，最终可知五种状态对应的二进制表示分别是：  

|     状态     |           二进制           |
|------------|-------------------------|
| RUNNING    | 1110...0....00（可记为 111） |
| SHUTDOWN   | 0000...0....00（可记为 000） |
| STOP       | 0010...0....00（可记为 001） |
| TIDYING    | 0100...0....00（可记为 010） |
| TERMINATED | 0110...0....00（可记为 011） |

有意思的地方在于，**RUNNING的符号位是1，说明它转为十进制以后是个负数，而除它以外其他的状态的符号位都是0，转为十进制之后都是正数** ，也就是说，我们可以这么认为：

**小于SHUTDOWN的就是RUNNING，大于SHUTDOWN就是停止中或者已停止。**

这也是后面状态计算的一些写法的基础。比如 `isRunning()`方法：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }
              
### 3. 通过 ctl 计算工作线程数 {#3-ctl}

在代码中，我们会经常看到线程池通过这一段代码来获取状态：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 29（32-3）
    private static final int COUNT_BITS = Integer.SIZE - 3;
    ​
    // 允许的最大工作线程（2^29-1 约5亿）
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
    ​
    private static int workerCountOf(int c)  {
        return c & CAPACITY;
    }
              
这个运算过程其实与上文运行状态的运算过程基本一致：

* 1.  
在 Java 中，int 值 1 的二进制表示为 0000......1，即除了最后一位为 1 以外其他都为 0；  
* 2.  
然后 `1 << COUNT_BITS` 表示让 1 左移 29 位，即得到 0010......0，即除了第三位为 1 以外其他位都为 0；  
* 3.  
随后对该值再 -1，即得到 0001......1，即除了前三位为 0 以外，其余 29 位皆为 1，此时该值实际上就是一个掩码了；  
* 4.  
  此后，我们执行 `c & CAPACITY`，实际上就会得到低 29 位的值，即当前线程中的线程数量。

这里我们举个例子，假如当前线程池处于 RUNNING 状态，且有 1 个工作线程，那么此时 ctl 值为 1110......0001，即前三位和最后一位为 1，其余位数都为 0，然后与 CAPACITY 进行与运算后，高三位全变为 0，此时 ctl 即为 0000......0001，也就是 1。

线程池状态
-----

### 1. 状态的流转 {#1}

在前文，我们提到线程池通过 ctl 一共可以表示**五种状态** ：

*  
**RUNNING** ：运行状态。线程池接受并处理新任务。  
*  
**SHUTDOWN** ：关闭状态。线程池不能接受新任务，处理完剩余任务后关闭。调用 `shutdown` 方法会进入该状态。  
*  
**STOP** ：停止状态。线程池不能接受新任务，并且尝试中断旧任务。调用 `shutdownNow` 方法会进入该状态。  
*  
**TIDYING** ：整理状态。由关闭状态转变，线程池任务队列为空且没有任何工作线程时时进入该状态，会调用 `terminated` 方法。  
*  
  **TERMINATED** ：终止状态。`terminated` 方法执行完毕后进入该状态，线程池彻底停止。

它们具体的流转关系可以参考下图：

![1718119274178-74c3b599-4c0a-4c71-9a0f-bcde6b3a5b98.png](https://article-images.zsxq.com/FrLOeMLJ_vp6HtQHIrDZll24wIG5)

除了这种运行状态，线程池还提供了一些方法让我们可以获取其他的指标信息，比如核心线程数、最大线程数等，这个咱们后面会详细讲。

### 2. 如何触发停机？ {#2}

当我们要停止运行一个线程池时，可以调用下述两个方法：

*  
**shutdown** ：中断线程池，不再添加新任务，同时**等待当前进行和队列中的任务完成** 。  
*  
  **shutdownNow** ：立即中断线程池，不再添加新任务，**同时中断所有工作中的任务，不再处理任务队列中任务** 。

#### 2.1 shutdown {#2-1-shutdown}

`shutdown` 是正常关闭，这个方法主要做了这几件事：

* 1.  
改变当前线程池状态为 `SHUTDOWN`；  
* 2.  
将线程池中的**空闲工作线程** 标记为中断；  
* 3.  
完成上述过程后将线程池状态改为 `TIDYING`；  
* 4.  
  此后等到最后一个线程也退出后则将状态改为 `TERMINATED`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        // 加锁
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 将线程池状态改为 SHUTDOWN
            advanceRunState(SHUTDOWN);
            // 中断线程池中的所有空闲线程
            interruptIdleWorkers();
            // 钩子函数，默认空实现
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
              
#### 2.2 shutdownNow {#2-2-shutdown-now}

`shutdownNow` 与 `shutdown` 流程类似，不过它是立即停机，因此在细节上又有点区别：

* 1.  
改变当前线程池状态为 `STOP`；  
* 2.  
将线程池中的**所有工作线程** 标记为中断；  
* 3.  
**将任务队列中的任务全部移除** ；  
* 4.  
完成上述过程后将线程池状态改为 `TIDYING`；  
* 5.  
  此后等到最后一个线程也退出后则将状态改为 `TERMINATED`。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            // 将线程池状态改为 STOP
            advanceRunState(STOP);
            // 中断线程池中的所有线程
            interruptWorkers();
            // 删除任务队列中的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
              
此外，而在 `addWorker` 或者`getTask` 等处理任务的相关方法里，会针对 `STOP` 或更进一步的状态做区分，比如在 `getTask` 方法中，如果线程池进入了 `STOP` 或更进一步的状态，则会直接返回而不会继续申请任务。

### 3. 真正的停机 {#3}

我们已经知道通过 `shutdownNow` 与 `shutdown`可以触发停机流程，当两个方法执行完毕后，线程池将会进入 `STOP` 或者 `SHUTDOWN` 状态。

但是此时线程池**并未真正的停机** ，真正的停机逻辑，需要等到线程通过 `processWorkerExit` 方法退出时，里面调用的 `tryTerminate` 方法：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 如果线程池不处于预停机状态，则不进行停机
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 如果当前还有工作线程，则不进行停机
            if (workerCountOf(c) != 0) {
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            // 线程现在处于预停机状态，尝试进行停机
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 尝试通过 CAS 将线程池状态修改为 TIDYING
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        // 尝试通过 CAS 将线程池状态修改为 TERMINATED
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // 进入下一次循环
        }
    }
              
简单的来说，由于在当我们调用了停机方法时，实际上工作线程仍然还在执行任务，我们可能并没有办法立刻终止线程的执行。

因此，每个线程执行完任务并且开始退出时，它都有可能是线程池中最后一个线程，此时它就需要承担起后续的收尾工作：

* 1.  
将线程池状态修改为 `TIDYING`；  
* 2.  
调用 `terminated` 回调方法，触发自定义停机逻辑；  
* 3.  
将线程池状态修改为 `TERMINATED`；  
* 4.  
  唤醒通过 `awaitTerminated` 阻塞的外部线程。

至此，线程池就真正的停机了。

任务调度
----

### 1. 任务的提交 {#1}

当我们向线程池提交一个任务时 ------ 无论是通过 `execute`、`submit` 还是 `invokeAll/invokerAny` 方法 ------ 最终都会走到 `execute` 方法上来：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        // 如果当前工作线程数小于核心线程数，则启动一个新工作线程执行任务
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        // 如果当前工作线程数大于核心线程数，则尝试将任务添加到工作队列
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // 如果任务添加成功，但是发现线程池已经不运行了，则移除任务并且触发拒绝策略
            if (!isRunning(recheck) && remove(command))
                reject(command);
            // 如果任务添加成功，但是线程池中已经没有工作线程，则添加一个新的工作线程
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
            
        // 如果当前工作线程数大于核心线程数，且工作队列已满，则启动一个新工作线程执行任务，
        // 如果工作线程数已经达到最大值，则直接触发拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
              
我们可以将上述逻辑简化并梳理为下图：

![1718121774197-e021bf5b-1f57-4bf2-ad29-8bbd395a489e.png](https://article-images.zsxq.com/FsJkW9FCnrbUXI3c1qjrSN5yVaXc)

当然，我们还可以进一步简化，将整段代码归类为四种分支逻辑：

*  
如果当前工作线程数小于核心线程数，则启动一个工作线程执行任务。  
*  
如果当前工作线程数大于等于核心线程数，且阻塞队列未满，则将任务添加到阻塞队列。  
*  
如果当前工作线程数大于等于核心线程数，且阻塞队列已满，则启动一个工作线程执行任务。  
*  
  如果当前工作线程数已达最大值，且阻塞队列已满，则触发拒绝策略。

根据上述逻辑，我们不难意识到，如果阻塞队列设置的比较大而难以打满，那么线程池中的线程数就会一直与核心线程数保持一致从而匀速消费。

### 2. 任务执行 {#2}

#### 2.1 线程复用 {#2-1}

当线程池通过 `addWorker` 方法创建了运行线程 `Worker` 后，线程会通过 `runWorker` 去运行相关任务。**该线程运行任务完成后并不会直接销毁** ，因为 `runWorker` 方法会让线程在一个 while 循环中，反复调用 getTask 方法从阻塞队列中获取新的任务并执行。

通过该机制保障了在满足工作线程不销毁的前提下，让工作线程**不断从阻塞队列中拿到新的任务并运行** ，保障了线程池线程复用机制。

![1714298457545-0a4cab65-6e00-4aaf-9cfd-82e1b6afacb9.png](https://article-images.zsxq.com/FtmtagdRssbc9wFJFpPAFrs5RBZS)

线程池中的执行线程被包装为了 `Worker` 对象，并通过内部字段 thread 运行。可以看到 `Worker` 对象实现了 `Runnable` 接口，重写 `run` 方法调用了 `runWorker` 方法。

能保障线程池中线程复用的精髓都在这个方法以及调用的 `getTask` 方法中，我将相关代码精简出来，会省略部分非核心逻辑，方便大家更容易理解逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ThreadPoolExecutor extends AbstractExecutorService {
        
        private final class Worker extends AbstractQueuedSynchronizer
            implements Runnable
        {
            // ......
            /**
             * 执行线程池中任务的线程
             */
            final Thread thread;
            
            public void run() {
                runWorker(this);
            }
            // ......
        }
        final void runWorker(Worker w) {
            Thread wt = Thread.currentThread();
            Runnable task = w.firstTask;
            w.firstTask = null;
            w.unlock(); // 新创建的Worker默认state为-1，AQS的unlock方法会将其改为0，此后允许使用interruptIfStarted()方法进行中断

            // 完成任务以后是否需要移除当前Worker，即当前任务是否意外退出
            boolean completedAbruptly = true;

            try {
                // 循环获取任务
                while (task != null || (task = getTask()) != null) {
                    // 加锁，防止 shutdown 时中断正在运行的任务
                    w.lock();
                    // 如果线程池状态为 STOP 或更后面的状态，中断线程任务
                    if ((runStateAtLeast(ctl.get(), STOP) ||
                         (Thread.interrupted() &&
                          runStateAtLeast(ctl.get(), STOP))) &&
                        !wt.isInterrupted())
                        wt.interrupt();
                    try {
                        // 钩子方法，默认空实现，需要自己提供
                        beforeExecute(wt, task);
                        Throwable thrown = null;
                        try {
                            // 执行任务
                            task.run();
                        } catch (RuntimeException x) {
                            thrown = x; throw x;
                        } catch (Error x) {
                            thrown = x; throw x;
                        } catch (Throwable x) {
                            thrown = x; throw new Error(x);
                        } finally {
                            // 钩子方法
                            afterExecute(task, thrown);
                        }
                    } finally {
                        task = null;
                        // 任务执行完毕
                        w.completedTasks++;
                        w.unlock();
                    }
                }

                completedAbruptly = false;
            } finally {
                // 当线程获取任务超时、或者线程异常退出时都会调用此方法移除当前线程，这两种情况的主要区别在于，
                // 如果是异常退出，那么只要线程池还在运行状态，就会立刻启动一个新线程；
                // 而如果不是异常退出，那么将会根据情况来是否要启动新线程，以保证线程池中保持足够数量的线程。
                processWorkerExit(w, completedAbruptly);
            }
        }
    }
              
简单的来说，这一段代码**在一个while循环中** 做了这样一件事情：

* 1.  
通过 `getTask` 方法从工作队列中获取任务，如果拿不到任务就阻塞一段时间，直到超时或者获取到任务。如果成功获取到任务就进入下一步，否则就直接调用 `processWorkerExit` 进入线程退出流程；  
* 2.  
调用 `Worker` 的 `lock` 方法加锁，保证一个线程只被一个任务占用；  
* 3.  
调用 `beforeExecute` 回调方法，随后开始执行任务，如果在执行任务的过程中发生异常则会被捕获；  
* 4.  
任务执行完毕或者因为异常中断，此后调用一次 `afterExecute` 回调方法，然后调用 `unlock` 方法解锁；  
* 5.  
  如果线程是因为异常中断，那么调用 `processWorkerExit` 进入线程退出流程，否则回到步骤 1 进入下一次循环。

我们可以将上述逻辑简化为下图：

![1718123052179-1b426288-ee7f-4aee-85e4-b68ffe75c9f4-20250701165704131.png](https://article-images.zsxq.com/FqToDBHUCM4QkPkPCCTlhrwQ-ONH)

#### 2.2 线程超时回收 {#2-2}

其中，`getTask` 中又涉及到线程的超时回收机制，按照上面的逻辑，那线程池里的线程岂不是会一直在 while 循环中拿阻塞队列的任务并运行？如何实现线程池中设置了 keepAliveTime 的空闲回收机制呢？

线程池中线程从阻塞队列中获取任务的 `getTask` 方法有两种机制，第一种是无限期等待阻塞队列中有任务了返回，还有一种是等待设置 `keepAliveTime` 时间之后返回，如果说等待了 `keepAliveTime` 时间之后阻塞队列中还是没有任务，那么会返回空，并在 `getTask` 方法中返回空，最终在 `runWorkerask` 方法中执行线程销毁逻辑，这个过程也就是超时回收。

![1714315541458-c7f9e393-6e59-496a-ae36-86da26333648.png](https://article-images.zsxq.com/FsfJZ61ru0MWnYZ0dLjc6d-2Jqdo)

超时回收重点在于 `getTask` 方法，其中有两行核心代码我标记了下。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    /**
     * 从阻塞队列中获取任务
     */ 
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();

            // ......
            int wc = workerCountOf(c);

            // 核心1：判断是否需要带超时时间去阻塞队列中获取任务
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                // 核心2：是否需要带超时方式获取
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
              
1）如何判断是否需要带超时时间？

虽然代码里两个逻辑判断是反着的，但是我感觉这样讲解会更容易理解些。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    boolean timed = wc > corePoolSize || allowCoreThreadTimeOut;
              
`wc > corePoolSize` wc（工作线程数量）大于 corePoolSize（核心线程数），则 timed 的值为 true。这个比较容易理解，毕竟 `keepAliveTime` 参数的语意就是当线程池内的线程超过核心线程数后，多余的线程就会在 `keepAliveTime` 空余时间后进行销毁。

`allowCoreThreadTimeOut` 是线程池的一个配置选项，用于控制是否允许核心线程超时销毁。在标准的线程池实现中，**核心线程是线程池中一直保持活动状态的线程，不会被销毁------即使它们处于空闲状态** 。这样可以保证线程池能够快速响应任务的执行请求，避免线程的创建和销毁开销。

然而，有些情况下，**当线程池中的任务量较少或任务执行时间较长时，保持大量的核心线程可能会浪费系统资源** 。这时，可以通过将 allowCoreThreadTimeOut 设置为 true，**允许核心线程也受到空闲超时时间的影响，即使它们是核心线程也可能会被销毁** 。线程池只在乎 "核心线程"的数量而不会真的去区分线程是不是 "核心线程"。

只要走这个逻辑的时候，工作线程数小于核心线程数，那么它就是一个核心线程，等它再回到这个地方的时候，如果工作线程数已经大于核心线程数了，那么它就是一个非核心线程。

所以，两个条件任意为 true 则 timed 都会设置 true。

2）如何使用超时时间获取阻塞队列任务？

阻塞队列 `BlockingQueue` 接口提供了 take 和 poll 两个公共用于让线程阻塞的从队列中获取元素：

*  
take：不设置超时时间，线程会一直阻塞直到成功从队列里获取一个元素为止。  
*  
  poll：只阻塞指定时间，如果超时了还是没法从队列中获取一个元素就放弃。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public interface BlockingQueue<E> extends Queue<E> {

        /**
         * 检索并删除此队列的头部，一直等待到指定元素可用的等待时间
         */
        E poll(long timeout, TimeUnit unit)
            throws InterruptedException;

        /**
         * 检索并删除此队列的头部，必要时等待直到元素可用为止
         */
        E take() throws InterruptedException;
    }
              
这两个方法分别对应 `getTask` 方法中的核心 2 流程。上文有说，如果获取获取不到元素，那么就意味着线程已经空闲了一段时间，按照默认场景下，如果线程池的线程数多余核心线程数，而且又获取不到任务，那么迎接它的将是被销毁。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
    if (r != null)
        return r;
              
如果 `getTask` 方法返回空的 task，while 循环将会结束，执行 `processWorkerExit` 销毁逻辑。

3）如何销毁工作线程？

首先解析下 `completedAbruptly` 参数，对应两种场景：

*  
如果 `completedAbruptly` 为 true，表示线程执行任务时因为抛出了异常而退出。在这种情况下，会调用 `decrementWorkerCount` 方法来减少工作线程的计数。  
*  
  为 false 则继续执行正常工作线程销毁逻辑。

textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            // 移除工作线程
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        // 兜底策略，避免线程池中的线程全部被清除
        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
              
最下面的这段逻辑，做了一层兜底。简单的来说，如果线程是因为抛出异常而退出的，那么会立刻让工作线程数量减一，并且立刻添加一个新的工作线程。而如果线程是因为在规定的超时时间内获取不到任务而退出，那么就需要分情况考虑是否要添加一个新的工作线程。

当核心线程设置是否超时情况下，有两种处理策略：

*  
**允许核心线程超时** ：那么就需要看看工作队列中是否有未消费的任务，如果队列已空，那么不需要补偿新的工作线程，如果工作队列未空，则仅在当前线程池没有任何工作线程时才补充一个新的工作线程。  
*  
  **不允许核心线程超时** ：那么就需要确认当前工作线程数是否已经大于核心线程数，如果已经大于则不需要补充新的工作线程，否则则需要。

4）keepAliveTime为零还会超时回收么？

如果将 `keepAliveTime` 设置为零，这意味着非核心线程在空闲后可以立即被回收，而不需要等待超时时间。

也就是这段代码中 poll 方法将等同于没有阻塞等待时间，如果阻塞队列中没有元素将立即返回。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    Runnable r = timed ?
        workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
        workQueue.take();
    if (r != null)
        return r;
              
线程池的 `keepAliveTime` 设置为零有什么问题呢？

*  
**线程资源浪费** ：线程池中的非核心线程在任务执行完毕后会立即被回收。这种设置可能导致线程频繁地创建和销毁，增加了线程管理的开销。  
*  
  **不利于线程重用** ：如果无法以保证工作队列始终非空，那么线程池中的线程就有可能因为经常无法从工作队列获取到任务而频繁的超时退出。这种情况下，一旦又无法保证以比较均匀的速度向线程池提交任务，那么线程池就会频繁的创建或者销毁线程，这无疑会增加线程池的压力，造成线程池的性能下降。

5）如何让线程不被回收？

可以通过将线程池的核心线程数和最大线程数设置为相同的值，可以确保线程池中的所有线程都被视为核心线程。这样，即使线程处于空闲状态，它们也不会被回收。

但有一点需要注意，如果将线程池中的线程设置为不可回收，这可能会导致线程资源的浪费，特别是当线程处于空闲状态时。

### 3. 任务缓冲 {#3}

线程池是一个非常典型的生产者和消费者模型，在这个模型中，由于生产者生产速度与消费者消费速度可能不一致，所以不会让消费者直接消费生产者的数据，而是通过一个缓冲区来进行解耦削峰。

在线程池中这个缓冲区即为工作队列，也就是我们在线程池的构造函数中指定的阻塞队列。

![1718264253293-d54ed1a5-86f7-4558-9c94-0a4ca29ea71d.png](https://article-images.zsxq.com/FoDP_L7op00ZGPVYrl8KmVgTrxLY)

JDK 默认提供了以下几种阻塞队列实现：

![1718170081903-5b1ed76d-8227-4edb-8509-bd65a4baf292.jpeg](https://article-images.zsxq.com/FjpjZVjp3URXJITRriS-jNW29kJN)

通过更换阻塞队列，配合调整线程池的核心线程数和最大线程数，我们就可以搭配出一些适用于特定场景的线程池，比如 JDK 默认在 `Executors` 提供的几种线程池：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    // 固定大小的线程池，核心线程数与最大线程数一致，阻塞队列无限大小
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }

    // 核心线程数为0，最大线程数不设限的线程池，阻塞队列容量为0（同步队列）
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
              
### 4. 拒绝策略 {#4}

当任务队列已满，并且线程池中线程也到达最大线程数的时候，就会调用拒绝策略。也就是 `reject()` 方法：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    final void reject(Runnable command) {
        handler.rejectedExecution(command, this);
    }
              
默认支持的拒绝策略共四种：

*  
**AbortPolicy** ：拒绝策略，直接抛出异常，默认策略。  
*  
**CallerRunsPolicy** ：调用者运行策略，用调用者所在的线程来执行任务。  
*  
**DiscardOldestPolicy** ：弃老策略，无声无息的丢弃阻塞队列中靠最前的任务，并执行当前任务。  
*  
  **DiscardPolicy** ：丢弃策略，直接无声无息的丢弃任务。

当然，我们也可以实现 `RejectedExecutionHandler` 接口来定义自己的拒绝策略。比如在 oneThread 中我们就实现一个可以发送告警并记录拒绝次数的策略，从而为线程池的监控提供支持。

文末总结
----

纵观线程池的底层架构，其精妙之处在于将复杂的并发问题分解为三个维度的协同：**状态管理（通过ctl的高效位运算实现无锁化流转）、资源调度（核心/非核心线程的动态扩缩容与超时回收机制）、任务生命周期管控（从submit到afterExecute的闭环钩子）** 。这种设计不仅解决了高频线程创建的性能瓶颈，更通过拒绝策略与队列选型为系统提供柔性容错能力。

大家在学习和使用过程中，唯有深入理解 Worker 的 AQS 锁机制、任务缓冲队列的选型逻辑（如 LinkedBlockingQueue 与 SynchronousQueue 的场景差异）、以及 shutdownNow 与 shutdown的停机本质，才能更好地驾驭线程池这座"并发引擎"，在资源约束与性能需求间找到精准平衡点。

完结，撒花 🎉

