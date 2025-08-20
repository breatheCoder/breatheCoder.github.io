# 典型回答


Java 的 JUC（`java.util.concurrent`）是 Java 并发编程的核心工具包，几乎所有高性能、多线程的 Java 应用都会用到它。从 JDK 1.5 开始引入，由 Doug Lea 主导开发。



<font style="color:rgb(64, 64, 64);">在 JUC 出现之前，Java 开发者主要依赖 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">synchronized</font>**`<font style="color:rgb(64, 64, 64);"> 和 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">Object</font>**`<font style="color:rgb(64, 64, 64);"> 的 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">wait()/notify()</font>**`<font style="color:rgb(64, 64, 64);"> 等方式进行线程之间的同步（保证并发安全）。但是有很多缺点：</font>

1. **<font style="color:rgb(64, 64, 64);">粒度粗：</font>**<font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">synchronized</font>**`<font style="color:rgb(64, 64, 64);"> </font><font style="color:rgb(64, 64, 64);">锁住的是整个对象或类，容易导致不必要的线程阻塞，降低并发性能。</font>
2. **<font style="color:rgb(64, 64, 64);">功能有限：</font>**<font style="color:rgb(64, 64, 64);"> 难以实现复杂的同步需求，如读写锁、信号量、屏障等。</font>
3. **<font style="color:rgb(64, 64, 64);">易出错：</font>**<font style="color:rgb(64, 64, 64);"> 手动管理 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">wait()</font>**`<font style="color:rgb(64, 64, 64);"> 和 </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">notify()</font>**`<font style="color:rgb(64, 64, 64);"> 容易导致死锁等问题，代码难以编写和维护。</font>
4. **<font style="color:rgb(64, 64, 64);">性能瓶颈：</font>**<font style="color:rgb(64, 64, 64);"> </font>`**<font style="color:rgb(64, 64, 64);background-color:rgb(236, 236, 236);">synchronized</font>**`<font style="color:rgb(64, 64, 64);">是重量级锁，他的性能可能成为瓶颈。</font>



于是就出现了JUC，说白了就是提供了很多并发安全的工具类，帮助开发者来在并发场景下使用。**JUC中的工具类会****<font style="color:rgb(64, 64, 64);">通过细粒度锁、CAS、并发数据结构等大幅提升高并发场景下的吞吐量和响应速度。</font>**

<font style="color:rgb(64, 64, 64);"></font>

<font style="color:rgb(64, 64, 64);">JUC中主要包含以下工具类：</font>

<font style="color:rgb(64, 64, 64);"></font>

## <font style="color:rgb(64, 64, 64);"></font>**线程池与任务执行框架**
<font style="color:rgb(64, 64, 64);">这部分是 JUC 中最常用的一部分，统一了线程的创建、调度和销毁。</font>

+ **核心接口**
    - `<font style="color:rgb(64, 64, 64);">Executor</font>`<font style="color:rgb(64, 64, 64);">：执行任务的最顶层接口（任务用 </font>`<font style="color:rgb(64, 64, 64);">Runnable</font>`<font style="color:rgb(64, 64, 64);"> 表示）。</font>
    - `<font style="color:rgb(64, 64, 64);">ExecutorService</font>`<font style="color:rgb(64, 64, 64);">：扩展了 </font>`<font style="color:rgb(64, 64, 64);">Executor</font>`<font style="color:rgb(64, 64, 64);">，提供了生命周期管理（</font>`<font style="color:rgb(64, 64, 64);">shutdown</font>`<font style="color:rgb(64, 64, 64);">）和任务提交（</font>`<font style="color:rgb(64, 64, 64);">submit</font>`<font style="color:rgb(64, 64, 64);">）等方法。</font>
    - `<font style="color:rgb(64, 64, 64);">ScheduledExecutorService</font>`<font style="color:rgb(64, 64, 64);">：支持延时与周期性任务调度。</font>
+ **主要实现类**
    - `<font style="color:rgb(64, 64, 64);">ThreadPoolExecutor</font>`<font style="color:rgb(64, 64, 64);">：可配置的线程池，支持核心线程数、最大线程数、队列、拒绝策略等。</font>
    - `<font style="color:rgb(64, 64, 64);">ScheduledThreadPoolExecutor</font>`<font style="color:rgb(64, 64, 64);">：定时任务线程池。</font>
    - **工具类**`<font style="color:rgb(64, 64, 64);">Executors</font>`<font style="color:rgb(64, 64, 64);">：提供常用线程池工厂方法（但生产环境推荐自己用 </font>`<font style="color:rgb(64, 64, 64);">ThreadPoolExecutor</font>`<font style="color:rgb(64, 64, 64);"> 构造，避免资源问题）。</font>
    - `<font style="color:rgb(64, 64, 64);">ForkJoinPool</font>`<font style="color:rgb(64, 64, 64);"> / </font>`<font style="color:rgb(64, 64, 64);">ForkJoinTask</font>`<font style="color:rgb(64, 64, 64);">：分而治之的并行计算框架，支持工作窃取算法（</font>`<font style="color:rgb(64, 64, 64);">parallelStream()</font>`<font style="color:rgb(64, 64, 64);"> 底层就是它）。</font>
+ **新特性**
    - <font style="color:rgb(64, 64, 64);">JDK 8 引入 </font>`<font style="color:rgb(64, 64, 64);">CompletableFuture</font>`<font style="color:rgb(64, 64, 64);">：支持链式异步编程。</font>
    - <font style="color:rgb(64, 64, 64);">JDK 21 可结合虚拟线程 </font>`<font style="color:rgb(64, 64, 64);">Executors.newVirtualThreadPerTaskExecutor()</font>`<font style="color:rgb(64, 64, 64);">。</font>



## **同步器**
<font style="color:rgb(64, 64, 64);">这些是比 </font>`<font style="color:rgb(64, 64, 64);">synchronized</font>`<font style="color:rgb(64, 64, 64);"> 更灵活、更高性能的线程协作工具。他们是AQS的具体实现。</font>

<font style="color:rgb(64, 64, 64);"></font>

[[✅如何理解AQS？]]

<font style="color:rgb(64, 64, 64);"></font>

+ **锁**
    - `<font style="color:rgb(64, 64, 64);">ReentrantLock</font>`<font style="color:rgb(64, 64, 64);">：可重入互斥锁，支持公平/非公平模式，支持条件变量 </font>`<font style="color:rgb(64, 64, 64);">Condition</font>`<font style="color:rgb(64, 64, 64);">。</font>
    - `<font style="color:rgb(64, 64, 64);">ReentrantReadWriteLock</font>`<font style="color:rgb(64, 64, 64);">：读写分离锁，读锁共享、写锁独占。</font>
    - `<font style="color:rgb(64, 64, 64);">StampedLock</font>`<font style="color:rgb(64, 64, 64);">：支持乐观读，性能更高（但 API 相对复杂）。</font>
+ **并发控制工具**
    - `<font style="color:rgb(64, 64, 64);">Semaphore</font>`<font style="color:rgb(64, 64, 64);">：计数信号量，限制同时访问资源的线程数。</font>
    - `<font style="color:rgb(64, 64, 64);">CountDownLatch</font>`<font style="color:rgb(64, 64, 64);">：倒计时锁存器，等待一组操作完成再继续。</font>
    - `<font style="color:rgb(64, 64, 64);">CyclicBarrier</font>`<font style="color:rgb(64, 64, 64);">：循环屏障，一组线程到达屏障点后一起继续。</font>
    - `<font style="color:rgb(64, 64, 64);">Phaser</font>`<font style="color:rgb(64, 64, 64);">：更灵活的阶段同步器（</font>`<font style="color:rgb(64, 64, 64);">CyclicBarrier</font>`<font style="color:rgb(64, 64, 64);"> + </font>`<font style="color:rgb(64, 64, 64);">CountDownLatch</font>`<font style="color:rgb(64, 64, 64);"> 的超集）。</font>
+ **底层支持类**
    - `<font style="color:rgb(64, 64, 64);">AbstractQueuedSynchronizer</font>`<font style="color:rgb(64, 64, 64);">（AQS）：几乎所有 JUC 锁和同步器的基础框架，基于 FIFO 队列和 CAS 实现。</font>



## **并发集合**
<font style="color:rgb(64, 64, 64);">这些集合在多线程环境下是线程安全的，性能远高于 </font>`<font style="color:rgb(64, 64, 64);">Collections.synchronizedXXX</font>`<font style="color:rgb(64, 64, 64);">。</font>

<font style="color:rgb(64, 64, 64);"></font>

+ **常用容器**
    - `<font style="color:rgb(64, 64, 64);">ConcurrentHashMap</font>`<font style="color:rgb(64, 64, 64);">：高性能并发哈希表（JDK 8 以后用 CAS + 分段桶链表/红黑树）。</font>
    - `<font style="color:rgb(64, 64, 64);">ConcurrentLinkedQueue</font>`<font style="color:rgb(64, 64, 64);"> / </font>`<font style="color:rgb(64, 64, 64);">ConcurrentLinkedDeque</font>`<font style="color:rgb(64, 64, 64);">：无界非阻塞链表队列。</font>
    - `<font style="color:rgb(64, 64, 64);">CopyOnWriteArrayList</font>`<font style="color:rgb(64, 64, 64);"> / </font>`<font style="color:rgb(64, 64, 64);">CopyOnWriteArraySet</font>`<font style="color:rgb(64, 64, 64);">：写时复制，适合读多写少场景。</font>



[[✅什么是COW，如何保证的线程安全？]]



+ **阻塞队列**
    - `<font style="color:rgb(64, 64, 64);">ArrayBlockingQueue</font>`<font style="color:rgb(64, 64, 64);">：有界阻塞队列（基于数组）。</font>
    - `<font style="color:rgb(64, 64, 64);">LinkedBlockingQueue</font>`<font style="color:rgb(64, 64, 64);">：有界/无界链表阻塞队列。</font>
    - `<font style="color:rgb(64, 64, 64);">PriorityBlockingQueue</font>`<font style="color:rgb(64, 64, 64);">：支持优先级的阻塞队列。</font>
    - `<font style="color:rgb(64, 64, 64);">DelayQueue</font>`<font style="color:rgb(64, 64, 64);">：延时队列，任务到期后才能取出。</font>
    - `<font style="color:rgb(64, 64, 64);">SynchronousQueue</font>`<font style="color:rgb(64, 64, 64);">：无缓冲队列，交替交付任务（常用于直接交接任务给线程池）。</font>
    - `<font style="color:rgb(64, 64, 64);">LinkedTransferQueue</font>`<font style="color:rgb(64, 64, 64);">：高性能转移队列，支持直接传递元素。</font>



## **原子变量**
<font style="color:rgb(64, 64, 64);">提供</font>**无锁线程安全**<font style="color:rgb(64, 64, 64);">的原子操作，基于 CPU 的 CAS 指令。</font>

<font style="color:rgb(64, 64, 64);"></font>

+ **原子类**
    - <font style="color:rgb(64, 64, 64);">基本类型：</font>`<font style="color:rgb(64, 64, 64);">AtomicInteger</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">AtomicLong</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">AtomicBoolean</font>`<font style="color:rgb(64, 64, 64);">。</font>
    - <font style="color:rgb(64, 64, 64);">数组类型：</font>`<font style="color:rgb(64, 64, 64);">AtomicIntegerArray</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">AtomicLongArray</font>`<font style="color:rgb(64, 64, 64);">。</font>
    - <font style="color:rgb(64, 64, 64);">引用类型：</font>`<font style="color:rgb(64, 64, 64);">AtomicReference</font>`<font style="color:rgb(64, 64, 64);">、</font>`<font style="color:rgb(64, 64, 64);">AtomicStampedReference</font>`<font style="color:rgb(64, 64, 64);">（解决 ABA 问题）。</font>
    - <font style="color:rgb(64, 64, 64);">字段更新器：</font>`<font style="color:rgb(64, 64, 64);">AtomicIntegerFieldUpdater</font>`<font style="color:rgb(64, 64, 64);"> 等（基于反射直接更新对象字段）。</font>



[[✅LongAdder和AtomicLong的区别？]]

