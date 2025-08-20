2025年07月20日 19:34  
阻塞队列容量热更新策略下的"坑"，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本篇文章围绕阻塞队列容量动态调整的可行性展开，结合 Java 原生 `LinkedBlockingQueue` 存在的反射修改问题，深入剖析了其底层设计的局限性，并介绍了 RabbitMQ 的解决方案 ------ `VariableLinkedBlockingQueue`。

课程目录如下所示：

*  
前言  
*  
反射方案的问题  
*  
RabbitMQ 如何解决热更新问题  
*  
  文末总结

前言
---

阻塞队列动态更新不止是咱们需要，RabbitMQ 同样需要。在实际运行中，RabbitMQ 的工作线程池会处理来自大量客户端的请求，这些操作的压力往往**具有明显的波峰波谷特征** ，例如在早晚高峰、电商秒杀等期间，通过**动态扩容任务队列**，防止因短期流量冲击造成系统雪崩。RabbitMQ 运行过程中，会监控如线程池活跃线程数、队列使用情况、任务阻塞率等指标，达到设置阈值自动进行扩容。
> RabbitMQ 我没用过不太熟，客户端通过自动扩容这里大家有兴趣可以具体看看。

RabbitMQ 团队在面临同样需求时，选择了一个更优雅的做法：**直接复制并修改 `LinkedBlockingQueue` 源码，做成可变容量版本**。

本章节我们先通过讲解反射带来的问题，然后再看 RabbitMQ 怎么做的。

反射方案的问题
-------

虽然我们可以通过反射手段绕过 `final` 限制，修改 `LinkedBlockingQueue` 中的 `capacity` 字段，但**仅仅修改这个字段值并不能真正实现预期行为**。我们来看两个非常典型的问题场景：

### 1. 队列已满修改容量无效 {#1}

当队列已满，调用线程正阻塞在 `put()` 方法上等待空位：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() == capacity) {
        notFull.await();  // 阻塞在这里
    }
              
此时，我们通过反射动态将 `capacity` 从 10 修改为 100，期望 unblock 等待线程。但线程仍然阻塞在 `notFull.await()`，无法及时感知容量变化！

下方截图取自 `LinkedBlockingQueue#put` 方法：

![image-20250720184805923.png](https://article-images.zsxq.com/FkWNABLsEUlsqDXxJRdiavnEOYNF)

`capacity` 虽然被改大了，但线程已经卡在了 `await()` 上，如果没有人手动调用 `signalNotFull()`，它永远不会被唤醒。
> ⚠️ 注意：**反射只改字段，不会自动触发条件变量通知（Condition#signal）**！

以下是一个测试用例，我放在了 `core` 模块 test 目录下，展示通过反射修改 LinkedBlockingQueue 容量的尝试及其问题：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ResizableCapacityLinkedBlockingQueueV1Test {
    ​
        public static void main(String[] args) throws Exception {
            ResizableCapacityLinkedBlockingQueueV1<String> queue = new ResizableCapacityLinkedBlockingQueueV1<>(2);
    ​
            // 填充队列至满
            queue.put("Element 1");
            System.out.println("入队列成功，当前大小：" + queue.size());
            queue.put("Element 2");
            System.out.println("入队列成功，当前大小：" + queue.size());
    ​
            // 尝试添加第三个元素，预期阻塞
            ExecutorService executor = Executors.newSingleThreadExecutor();
            executor.submit(() -> {
                try {
                    System.out.println("尝试添加 Element 3，队列已满，线程将被阻塞");
                    queue.put("Element 3");
                    System.out.println("成功添加 Element 3，队列大小：" + queue.size());
                } catch (InterruptedException e) {
                    System.out.println("添加 Element 3 失败");
                }
            });
    ​
            // 等待 2 秒，确保线程阻塞
            TimeUnit.SECONDS.sleep(2);
    ​
            // 通过反射修改容量
            try {
                queue.setCapacity(3);
                System.out.println("通过反射修改容量为：3");
            } catch (Exception e) {
                System.out.println("反射修改容量失败：" + e.getMessage());
            }
    ​
            // 等待 2 秒，观察是否成功添加
            TimeUnit.SECONDS.sleep(2);
    ​
            executor.shutdownNow();
        }
    }
              
> ⚠️ 注意：如果想在 IDEA 里跑这个单元测试，需要在 IDEA 单元测试中设置 VM 参数：  
> textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy
>
>                     --add-opens java.base/java.util.concurrent=ALL-UNNAMED
>               
我们期望的日志输出如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    队列已满，当前大小：1
    队列已满，当前大小：2
    尝试添加 Element 3，队列已满，线程将被阻塞
    18:51:28.979 [main] INFO com.nageoffer.onethread.core.executor.support.ResizableCapacityLinkedBlockingQueueV1 -- 成功修改阻塞队列容量为: 3
    通过反射修改容量为：3
    成功添加 Element 3，队列大小：3
              
理想很丰满，但是现实就比较骨感了，实际输出如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    入队列成功，当前大小：1
    入队列成功，当前大小：2
    尝试添加 Element 3，队列已满，线程将被阻塞
    18:26:34.881 [main] INFO com.nageoffer.onethread.core.executor.support.ResizableCapacityLinkedBlockingQueueV1 -- 成功修改阻塞队列容量为: 3
    通过反射修改容量为：3
    添加 Element 3 失败
              
### 2. 容量变小后无法阻塞 {#2}

假设当前队列已存入 8 个元素，容量为 10。我们通过反射将容量缩小为 5，期望此后入队操作会被阻塞。

但你会发现：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    queue.put(new Task());  // 居然成功入队！
              
仍然可以成功插入------根本没阻塞！

这是因为队列当前 `count.get() = 8`，我们虽然把 `capacity` 改成了 5，但 JDK 的 `put()` 判断仍然是：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() == capacity) {
        notFull.await();
    }
              
此时 `count.get() == 8`，`capacity == 5`，不相等，所以不会阻塞，**直接 bypass**。

队列实际元素数量已经超过了新容量，却还允许插入，这是违背预期的行为。属于是**违反了容量限制**的本意。

测试用例如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ResizableCapacityLinkedBlockingQueueV1Test2 {
    ​
        public static void main(String[] args) throws Exception {
            ResizableCapacityLinkedBlockingQueueV1<String> queue = new ResizableCapacityLinkedBlockingQueueV1<>(10);
            for (int i = 0; i < 8; i++) {
                queue.put("Element "+ i);
                System.out.println("入队列成功，当前大小：" + queue.size());
            }
    ​
            // 通过反射修改容量
            try {
                queue.setCapacity(5);
                System.out.println("通过反射修改容量为：5");
            } catch (Exception e) {
                System.out.println("反射修改容量失败：" + e.getMessage());
            }
    ​
            ExecutorService executor = Executors.newSingleThreadExecutor();
            executor.submit(() -> {
                try {
                    System.out.println("尝试添加 Element 9，队列已满，线程将被阻塞");
                    queue.put("Element 9");
                    System.out.println("成功添加 Element 9，队列大小：" + queue.size());
                } catch (InterruptedException e) {
                    System.out.println("添加 Element 9 失败");
                }
            });
    ​
            // 等待 2 秒，确保线程阻塞
            TimeUnit.SECONDS.sleep(2);
    ​
            executor.shutdownNow();
    ​
            System.out.println("🔍 最终队列元素数量：" + queue.size());
        }
    }
              
单元测试执行日志如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    入队列成功，当前大小：1
    入队列成功，当前大小：2
    入队列成功，当前大小：3
    入队列成功，当前大小：4
    入队列成功，当前大小：5
    入队列成功，当前大小：6
    入队列成功，当前大小：7
    入队列成功，当前大小：8
    19:11:58.834 [main] INFO com.nageoffer.onethread.core.executor.support.ResizableCapacityLinkedBlockingQueueV1 -- 成功修改阻塞队列容量为: 5
    通过反射修改容量为：5
    尝试添加 Element 9，队列已满，线程将被阻塞
    成功添加 Element 3，队列大小：9
    🔍 最终队列元素数量：9
              
`LinkedBlockingQueue` 使用 `==` 是因为 `capacity` 固定，队列满时 `count` 等于 `capacity`。

相信大家对于使用反射所存在的问题都已经清楚了，那接下来我们着手看 RabbitMQ 是如何解决这个问题的。

RabbitMQ 如何解决热更新问题？ {#rabbit-mq}
--------------------------------

`VariableLinkedBlockingQueue` 是 `java.util.concurrent.LinkedBlockingQueue` 的一个克隆版本，**扩展支持了运行时修改容量的能力** 。与 JDK 原生实现不同，它提供了一个公开的 `setCapacity(int)` 方法，允许我们在队列运行过程中**动态调整其容量上限**，而无需重建队列或重启线程池。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class VariableLinkedBlockingQueue<E> extends AbstractQueue<E>
            implements BlockingQueue<E>, java.io.Serializable {
        // ......
    }
              
同时，在保障容量动态更新的基础上，通过以下设计解决了动态容量调整的问题：

### 1. setCapacity() 中自动唤醒等待线程 {#1-set-capacity}

`VariableLinkedBlockingQueue` 的 `setCapacity` 方法在修改容量后，主动触发 `notFull.signalAll()` 来唤醒阻塞线程：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void setCapacity(int capacity) {
        final int oldCapacity = this.capacity;
        this.capacity = capacity;
        final int size = count.get();
        if (capacity > size && size >= oldCapacity) {
            signalNotFull();
        }
    }
              
当新容量大于当前队列大小（`capacity > size`）且队列在旧容量下已满或接近满（`size >= oldCapacity`），说明可能有线程因队列满而阻塞在 put 方法上。调用 signalNotFull() 唤醒这些线程，确保它们重新检查容量条件。

仅在 `capacity > size` 且 `size >= oldCapacity` 时触发 `signalNotFull()`，避免无意义的信号开销。

### 2. put() 使用 \>= 判断 {#2-put}

`LinkedBlockingQueue` 的 `put` 方法在队列满时阻塞：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() == capacity) {
        notFull.await();
    }
              
这里的条件是 `count.get() == capacity`，表示队列恰好满时才阻塞。`VariableLinkedBlockingQueue` 则使用：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() >= capacity) {
        notFull.await();
    }
              
使用 `>=` 是为了应对动态调整容量时的边界情况。如果在 put 操作期间，capacity 被动态缩小到小于当前 count（例如，队列有 1000 个元素，容量缩小到 500），`count >= capacity` 确保线程继续阻塞，直到队列元素数减少到新容量以下。

文末总结
----

`VariableLinkedBlockingQueue` 通过非 `final` 的 `capacity` 字段、改进的 `put` 方法（`count.get() >= capacity`）和 `setCapacity` 的唤醒逻辑，解决了 `LinkedBlockingQueue` 无法动态调整容量的问题。其关键设计要点包括：

*  
**动态容量支持** ：使用非 `final` 字段，消除反射的兼容性和性能问题。  
*  
**健壮的阻塞逻辑** ：`put` 方法使用 `>=` 条件，适应容量动态缩小场景。  
*  
**线程安全唤醒** ：`setCapacity` 在扩容时触发 `notFull.signalAll()`，确保阻塞线程及时恢复。  
*  
  **高兼容性**：不依赖 JDK 内部实现，适配多版本 JDK。

通过测试用例，我们验证了 `VariableLinkedBlockingQueue` 在队列满、动态扩容和线程恢复等场景下的正确性。对于需要动态调优的任务队列场景，`VariableLinkedBlockingQueue` 是一个更灵活、可靠的替代方案。

Breathe，为什么 RabbitMQ 叫 `VariableLinkedBlockingQueue`，咱们代码里是 `ResizableCapacityLinkedBlockingQueue`？

实话实说，因为当时看美团的文章里叫的这个，严格说起来两个名字表达的意思是一样的，都是指支持运行时动态调整容量的 `LinkedBlockingQueue`，只是命名风格略有差异，后面也是延续叫下来了。

完结，撒花 🎉  
阻塞队列容量热更新策略下的"坑"，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★★☆☆ - 较难  
*  
  视频地址：文档先行视频次之



*** ** * ** ***

内容摘要：本篇文章围绕阻塞队列容量动态调整的可行性展开，结合 Java 原生 `LinkedBlockingQueue` 存在的反射修改问题，深入剖析了其底层设计的局限性，并介绍了 RabbitMQ 的解决方案 ------ `VariableLinkedBlockingQueue`。

课程目录如下所示：

*  
前言  
*  
反射方案的问题  
*  
RabbitMQ 如何解决热更新问题  
*  
  文末总结

前言
---

阻塞队列动态更新不止是咱们需要，RabbitMQ 同样需要。在实际运行中，RabbitMQ 的工作线程池会处理来自大量客户端的请求，这些操作的压力往往**具有明显的波峰波谷特征** ，例如在早晚高峰、电商秒杀等期间，通过**动态扩容任务队列**，防止因短期流量冲击造成系统雪崩。RabbitMQ 运行过程中，会监控如线程池活跃线程数、队列使用情况、任务阻塞率等指标，达到设置阈值自动进行扩容。
> RabbitMQ 我没用过不太熟，客户端通过自动扩容这里大家有兴趣可以具体看看。

RabbitMQ 团队在面临同样需求时，选择了一个更优雅的做法：**直接复制并修改 `LinkedBlockingQueue` 源码，做成可变容量版本**。

本章节我们先通过讲解反射带来的问题，然后再看 RabbitMQ 怎么做的。

反射方案的问题
-------

虽然我们可以通过反射手段绕过 `final` 限制，修改 `LinkedBlockingQueue` 中的 `capacity` 字段，但**仅仅修改这个字段值并不能真正实现预期行为**。我们来看两个非常典型的问题场景：

### 1. 队列已满修改容量无效 {#1}

当队列已满，调用线程正阻塞在 `put()` 方法上等待空位：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() == capacity) {
        notFull.await();  // 阻塞在这里
    }
              
此时，我们通过反射动态将 `capacity` 从 10 修改为 100，期望 unblock 等待线程。但线程仍然阻塞在 `notFull.await()`，无法及时感知容量变化！

下方截图取自 `LinkedBlockingQueue#put` 方法：

![image-20250720184805923.png](https://article-images.zsxq.com/FkWNABLsEUlsqDXxJRdiavnEOYNF)

`capacity` 虽然被改大了，但线程已经卡在了 `await()` 上，如果没有人手动调用 `signalNotFull()`，它永远不会被唤醒。
> ⚠️ 注意：**反射只改字段，不会自动触发条件变量通知（Condition#signal）**！

以下是一个测试用例，我放在了 `core` 模块 test 目录下，展示通过反射修改 LinkedBlockingQueue 容量的尝试及其问题：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ResizableCapacityLinkedBlockingQueueV1Test {
    ​
        public static void main(String[] args) throws Exception {
            ResizableCapacityLinkedBlockingQueueV1<String> queue = new ResizableCapacityLinkedBlockingQueueV1<>(2);
    ​
            // 填充队列至满
            queue.put("Element 1");
            System.out.println("入队列成功，当前大小：" + queue.size());
            queue.put("Element 2");
            System.out.println("入队列成功，当前大小：" + queue.size());
    ​
            // 尝试添加第三个元素，预期阻塞
            ExecutorService executor = Executors.newSingleThreadExecutor();
            executor.submit(() -> {
                try {
                    System.out.println("尝试添加 Element 3，队列已满，线程将被阻塞");
                    queue.put("Element 3");
                    System.out.println("成功添加 Element 3，队列大小：" + queue.size());
                } catch (InterruptedException e) {
                    System.out.println("添加 Element 3 失败");
                }
            });
    ​
            // 等待 2 秒，确保线程阻塞
            TimeUnit.SECONDS.sleep(2);
    ​
            // 通过反射修改容量
            try {
                queue.setCapacity(3);
                System.out.println("通过反射修改容量为：3");
            } catch (Exception e) {
                System.out.println("反射修改容量失败：" + e.getMessage());
            }
    ​
            // 等待 2 秒，观察是否成功添加
            TimeUnit.SECONDS.sleep(2);
    ​
            executor.shutdownNow();
        }
    }
              
> ⚠️ 注意：如果想在 IDEA 里跑这个单元测试，需要在 IDEA 单元测试中设置 VM 参数：  
> textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy
>
>                     --add-opens java.base/java.util.concurrent=ALL-UNNAMED
>               
我们期望的日志输出如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    队列已满，当前大小：1
    队列已满，当前大小：2
    尝试添加 Element 3，队列已满，线程将被阻塞
    18:51:28.979 [main] INFO com.nageoffer.onethread.core.executor.support.ResizableCapacityLinkedBlockingQueueV1 -- 成功修改阻塞队列容量为: 3
    通过反射修改容量为：3
    成功添加 Element 3，队列大小：3
              
理想很丰满，但是现实就比较骨感了，实际输出如下：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    入队列成功，当前大小：1
    入队列成功，当前大小：2
    尝试添加 Element 3，队列已满，线程将被阻塞
    18:26:34.881 [main] INFO com.nageoffer.onethread.core.executor.support.ResizableCapacityLinkedBlockingQueueV1 -- 成功修改阻塞队列容量为: 3
    通过反射修改容量为：3
    添加 Element 3 失败
              
### 2. 容量变小后无法阻塞 {#2}

假设当前队列已存入 8 个元素，容量为 10。我们通过反射将容量缩小为 5，期望此后入队操作会被阻塞。

但你会发现：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    queue.put(new Task());  // 居然成功入队！
              
仍然可以成功插入------根本没阻塞！

这是因为队列当前 `count.get() = 8`，我们虽然把 `capacity` 改成了 5，但 JDK 的 `put()` 判断仍然是：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() == capacity) {
        notFull.await();
    }
              
此时 `count.get() == 8`，`capacity == 5`，不相等，所以不会阻塞，**直接 bypass**。

队列实际元素数量已经超过了新容量，却还允许插入，这是违背预期的行为。属于是**违反了容量限制**的本意。

测试用例如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class ResizableCapacityLinkedBlockingQueueV1Test2 {
    ​
        public static void main(String[] args) throws Exception {
            ResizableCapacityLinkedBlockingQueueV1<String> queue = new ResizableCapacityLinkedBlockingQueueV1<>(10);
            for (int i = 0; i < 8; i++) {
                queue.put("Element "+ i);
                System.out.println("入队列成功，当前大小：" + queue.size());
            }
    ​
            // 通过反射修改容量
            try {
                queue.setCapacity(5);
                System.out.println("通过反射修改容量为：5");
            } catch (Exception e) {
                System.out.println("反射修改容量失败：" + e.getMessage());
            }
    ​
            ExecutorService executor = Executors.newSingleThreadExecutor();
            executor.submit(() -> {
                try {
                    System.out.println("尝试添加 Element 9，队列已满，线程将被阻塞");
                    queue.put("Element 9");
                    System.out.println("成功添加 Element 9，队列大小：" + queue.size());
                } catch (InterruptedException e) {
                    System.out.println("添加 Element 9 失败");
                }
            });
    ​
            // 等待 2 秒，确保线程阻塞
            TimeUnit.SECONDS.sleep(2);
    ​
            executor.shutdownNow();
    ​
            System.out.println("🔍 最终队列元素数量：" + queue.size());
        }
    }
              
单元测试执行日志如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    入队列成功，当前大小：1
    入队列成功，当前大小：2
    入队列成功，当前大小：3
    入队列成功，当前大小：4
    入队列成功，当前大小：5
    入队列成功，当前大小：6
    入队列成功，当前大小：7
    入队列成功，当前大小：8
    19:11:58.834 [main] INFO com.nageoffer.onethread.core.executor.support.ResizableCapacityLinkedBlockingQueueV1 -- 成功修改阻塞队列容量为: 5
    通过反射修改容量为：5
    尝试添加 Element 9，队列已满，线程将被阻塞
    成功添加 Element 3，队列大小：9
    🔍 最终队列元素数量：9
              
`LinkedBlockingQueue` 使用 `==` 是因为 `capacity` 固定，队列满时 `count` 等于 `capacity`。

相信大家对于使用反射所存在的问题都已经清楚了，那接下来我们着手看 RabbitMQ 是如何解决这个问题的。

RabbitMQ 如何解决热更新问题？ {#rabbit-mq}
--------------------------------

`VariableLinkedBlockingQueue` 是 `java.util.concurrent.LinkedBlockingQueue` 的一个克隆版本，**扩展支持了运行时修改容量的能力** 。与 JDK 原生实现不同，它提供了一个公开的 `setCapacity(int)` 方法，允许我们在队列运行过程中**动态调整其容量上限**，而无需重建队列或重启线程池。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public class VariableLinkedBlockingQueue<E> extends AbstractQueue<E>
            implements BlockingQueue<E>, java.io.Serializable {
        // ......
    }
              
同时，在保障容量动态更新的基础上，通过以下设计解决了动态容量调整的问题：

### 1. setCapacity() 中自动唤醒等待线程 {#1-set-capacity}

`VariableLinkedBlockingQueue` 的 `setCapacity` 方法在修改容量后，主动触发 `notFull.signalAll()` 来唤醒阻塞线程：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    public void setCapacity(int capacity) {
        final int oldCapacity = this.capacity;
        this.capacity = capacity;
        final int size = count.get();
        if (capacity > size && size >= oldCapacity) {
            signalNotFull();
        }
    }
              
当新容量大于当前队列大小（`capacity > size`）且队列在旧容量下已满或接近满（`size >= oldCapacity`），说明可能有线程因队列满而阻塞在 put 方法上。调用 signalNotFull() 唤醒这些线程，确保它们重新检查容量条件。

仅在 `capacity > size` 且 `size >= oldCapacity` 时触发 `signalNotFull()`，避免无意义的信号开销。

### 2. put() 使用 \>= 判断 {#2-put}

`LinkedBlockingQueue` 的 `put` 方法在队列满时阻塞：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() == capacity) {
        notFull.await();
    }
              
这里的条件是 `count.get() == capacity`，表示队列恰好满时才阻塞。`VariableLinkedBlockingQueue` 则使用：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    while (count.get() >= capacity) {
        notFull.await();
    }
              
使用 `>=` 是为了应对动态调整容量时的边界情况。如果在 put 操作期间，capacity 被动态缩小到小于当前 count（例如，队列有 1000 个元素，容量缩小到 500），`count >= capacity` 确保线程继续阻塞，直到队列元素数减少到新容量以下。

文末总结
----

`VariableLinkedBlockingQueue` 通过非 `final` 的 `capacity` 字段、改进的 `put` 方法（`count.get() >= capacity`）和 `setCapacity` 的唤醒逻辑，解决了 `LinkedBlockingQueue` 无法动态调整容量的问题。其关键设计要点包括：

*  
**动态容量支持** ：使用非 `final` 字段，消除反射的兼容性和性能问题。  
*  
**健壮的阻塞逻辑** ：`put` 方法使用 `>=` 条件，适应容量动态缩小场景。  
*  
**线程安全唤醒** ：`setCapacity` 在扩容时触发 `notFull.signalAll()`，确保阻塞线程及时恢复。  
*  
  **高兼容性**：不依赖 JDK 内部实现，适配多版本 JDK。

通过测试用例，我们验证了 `VariableLinkedBlockingQueue` 在队列满、动态扩容和线程恢复等场景下的正确性。对于需要动态调优的任务队列场景，`VariableLinkedBlockingQueue` 是一个更灵活、可靠的替代方案。

Breathe，为什么 RabbitMQ 叫 `VariableLinkedBlockingQueue`，咱们代码里是 `ResizableCapacityLinkedBlockingQueue`？

实话实说，因为当时看美团的文章里叫的这个，严格说起来两个名字表达的意思是一样的，都是指支持运行时动态调整容量的 `LinkedBlockingQueue`，只是命名风格略有差异，后面也是延续叫下来了。

完结，撒花 🎉  

