2025年06月30日 23:02  
JDK线程池有哪些应用场景，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文详细介绍了线程池的概念、核心优势和应用场景。重点分析了两种常见应用场景：快速响应用户请求（如电商查询商品详情）和快速处理批量任务（如批量发送短信），并通过代码示例对比了串行和并行实现的性能差异。

课程目录如下所示：

*  
线程池介绍  
*  
  线程池应用场景

线程池介绍
-----

线程池是一种基于池化思想管理线程的工具，使用线程池可以**减少创建销毁线程的开销** ，避免线程过多导致系统资源耗尽。充分利用池内计算资源，等待分配并发执行任务，**提高系统性能和响应能力**。

在业务系统开发过程中，线程池有两个常见的应用场景，分别是：**快速响应用户请求和快速处理批量任务**。

线程池应用场景
-------

### 1. 快速响应用户请求 {#1}

以电商中的查询商品详情接口举例，从用户发起请求开始，想要获取到商品全部信息，可能会包括获取商品基本信息、库存信息、优惠券以及评论等多个查询逻辑，假设每个查询是 50ms，**如果是串行化查询则需要 200ms**，查询性能一般。

![1710916579007-154bc12d-da4f-41f7-bd3b-3bd4453b59ce.png](https://article-images.zsxq.com/Fjz_2Izwdpwjw6JSe7Jx8BuYH-ls)

而如果说通过线程池的方式并行查询，那查询全部商品信息的时间就取决于多个流程中最慢的那一条。

假设优惠信息流程查询时间 80ms，其他流程查询时间 50ms，经过线程池并行优化后，商品详情接口响应时间就是 80ms，**通过并行缩短了整体查询时间**。
> 线程池种并行提交任务的完成时间，取决于这些任务中执行时间最慢的流程。

![1710916862386-3a8a4df9-a55d-4791-b9c3-20afb7d2de94.png](https://article-images.zsxq.com/FgrMg8xoPkJ4MWZEwNHahw9PExQP)

这种场景想要达到的效果是**最快时间将结果响应给用户** ，我们在创建线程池时**不应该使用阻塞队列去缓冲任务**，而是可以尝试适当调大核心线程数和最大线程数，提高任务并行执行的性能。

通过串行化获取商品详情信息，代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    ​
    import lombok.SneakyThrows;
    ​
    public class Main {
    ​
        public static void main(String[] args) {
            long startTime = System.currentTimeMillis();
            getProductInventory();
            getProductPromotions();
            getProductReviews();
            System.out.println("查询商品详情耗时：" + (System.currentTimeMillis() - startTime));
        }
    ​
        /**
         * 获取商品库存信息
         */
        @SneakyThrows
        private static Object getProductInventory() {
            Thread.sleep(50);
            return new Object();
        }
    ​
        /**
         * 获取商品优惠信息
         */
        @SneakyThrows
        private static Object getProductPromotions() {
            Thread.sleep(80);
            return new Object();
        }
    ​
        /**
         * 获取商品评论信息
         */
        @SneakyThrows
        private static Object getProductReviews() {
            Thread.sleep(50);
            return new Object();
        }
    }
              
上面程序输出耗时大概在 180ms 到 200ms 之间，因为程序是串行化，所以要将所有方法耗时进行累加，同时要加上计算机运行打印和获取当前时间等步骤。

如果希望通过线程池的方式并行查询，需要重构下相关逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import lombok.SneakyThrows;
    ​
    import java.util.ArrayList;
    import java.util.List;
    import java.util.concurrent.Future;
    import java.util.concurrent.SynchronousQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    ​
    public class Main {
    ​
        private static final ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                6,
                9,
                1024,
                TimeUnit.SECONDS,
                new SynchronousQueue<>());
    ​
        @SneakyThrows
        public static void main(String[] args) {
            long startTime = System.currentTimeMillis();
            List<Future<Object>> results = new ArrayList<>();
            Future<Object> getProductInventory = threadPoolExecutor.submit(Main::getProductInventory);
            results.add(getProductInventory);
            Future<Object> getProductPromotions = threadPoolExecutor.submit(Main::getProductPromotions);
            results.add(getProductPromotions);
            Future<Object> getProductReviews = threadPoolExecutor.submit(Main::getProductReviews);
            results.add(getProductReviews);
            for (Future<Object> result : results) {
                result.get();
            }
            System.out.println("查询商品详情耗时：" + (System.currentTimeMillis() - startTime));
            // 优雅编码，这里如果不进行停止线程池，测试方法不能主动结束
            threadPoolExecutor.shutdown();
        }
    ​
        /**
         * 获取商品库存信息
         */
        @SneakyThrows
        private static Object getProductInventory() {
            Thread.sleep(50);
            return new Object();
        }
    ​
        /**
         * 获取商品优惠信息
         */
        @SneakyThrows
        private static Object getProductPromotions() {
            Thread.sleep(80);
            return new Object();
        }
    ​
        /**
         * 获取商品评论信息
         */
        @SneakyThrows
        private static Object getProductReviews() {
            Thread.sleep(50);
            return new Object();
        }
    }
              
并发执行的程序耗时大概在 80ms 到 100ms 之间，因为上面有说，线程池种并行提交任务的完成时间，取决于这些任务中执行时间最慢的流程。

### 2. 快速处理批量任务 {#2}

在工作中，快速处理批量任务场景比较多，包括不限于以下举得例子：

*  
公司举办周年庆，需要给每个员工发送邮件说明。  
*  
  短信平台后台通过上传 Excel 给一批用户发送短信。

我们以发送批量短信举例子，假设需要给 100 万用户发送短信通知，一条短信通知流程需要 50ms，初步计算如果要给所有用户发送成功，执行时间大概需要 13 小时左右。

![1710924369659-ac0388a5-aca6-43d4-ae06-b1f71ca81cc1.png](https://article-images.zsxq.com/Fl62UWxawjC4WjqcSy37FUMqMTo0)

处理批量任务和快速响应用户请求不一致，在后者的使用场景中，主线程需要阻塞的等待所有任务执行完成，因此必须要尽可能减少等待时间，而前者的使用场景中则完全不需要考虑这个问题，因此我们可以设置一个合适的阻塞队列用来缓冲任务。
> 大量任务堆积线程池阻塞队列场景下，可能会遇到项目发布重启或者意外宕机等情况，进而导致任务丢失风险。

如果我们通过线程池并发执行，假设线程池内线程数是 10 个，那么执行时间将会大幅度缩减，最佳情况会缩短时间时间接近 10 倍的性能，约等于 1 小时 38 分钟左右。

![1710924383065-d72a3b22-e146-4ba3-bef6-03916142d308.png](https://article-images.zsxq.com/FrbYdbCjGzYynz80GYCHB5ROplVo)

通过串行化批量发送用户短信通知，代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import cn.hutool.core.collection.ListUtil;
    import lombok.SneakyThrows;
    ​
    import java.util.List;
    ​
    public class Main {
    ​
        public static void main(String[] args) {
            List<String> phones = ListUtil.of("1560116xxxx", "1560116xxxx", "1560116xxxx", "1560116xxxx");
            for (String phone : phones) {
                sendPhoneSms(phone);
            }
        }
    ​
        /**
         * 发送手机短信流程
         */
        @SneakyThrows
        private static void sendPhoneSms(String phone) {
            Thread.sleep(50);
        }
    }
              
在这种情况下，我们使用阻塞队列来缓冲任务。提交任务的线程将任务提交到线程池，然后线程池中的线程会从阻塞队列中获取任务并执行。

这种执行方式允许任务的提交和执行解耦，提高了系统的灵活性和并发处理能力。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import cn.hutool.core.collection.ListUtil;
    import lombok.SneakyThrows;
    ​
    import java.util.List;
    import java.util.concurrent.LinkedBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    ​
    public class Main {
    ​
        private static final ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                10,
                10,
                1024,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(100000));
    ​
        public static void main(String[] args) {
            List<String> phones = ListUtil.of("1560116xxxx", "1560116xxxx", "1560116xxxx", "1560116xxxx");
            for (String phone : phones) {
                threadPoolExecutor.execute(() -> sendPhoneSms(phone));
            }
        }
    ​
        /**
         * 发送手机短信流程
         */
        @SneakyThrows
        private static void sendPhoneSms(String phone) {
            Thread.sleep(50);
        }
    }
              
绝知此事要躬行，非常建议大家能够实际本地代码运行下线程池，入门并发编程不二之选。

完结，撒花 🎉  
JDK线程池有哪些应用场景，元数据信息：

*  
什么是线程池oneThread：<https://t.zsxq.com/5GfrN>  
*  
代码仓库：<https://gitcode.net/nageoffer/onethread> ------ 申请项目权限参考上述线程池项目链接  
*  
章节难度：★★☆☆☆ - 中等  
*  
  视频地址：本章节内容简单，无



*** ** * ** ***

内容摘要：本文详细介绍了线程池的概念、核心优势和应用场景。重点分析了两种常见应用场景：快速响应用户请求（如电商查询商品详情）和快速处理批量任务（如批量发送短信），并通过代码示例对比了串行和并行实现的性能差异。

课程目录如下所示：

*  
线程池介绍  
*  
  线程池应用场景

线程池介绍
-----

线程池是一种基于池化思想管理线程的工具，使用线程池可以**减少创建销毁线程的开销** ，避免线程过多导致系统资源耗尽。充分利用池内计算资源，等待分配并发执行任务，**提高系统性能和响应能力**。

在业务系统开发过程中，线程池有两个常见的应用场景，分别是：**快速响应用户请求和快速处理批量任务**。

线程池应用场景
-------

### 1. 快速响应用户请求 {#1}

以电商中的查询商品详情接口举例，从用户发起请求开始，想要获取到商品全部信息，可能会包括获取商品基本信息、库存信息、优惠券以及评论等多个查询逻辑，假设每个查询是 50ms，**如果是串行化查询则需要 200ms**，查询性能一般。

![1710916579007-154bc12d-da4f-41f7-bd3b-3bd4453b59ce.png](https://article-images.zsxq.com/Fjz_2Izwdpwjw6JSe7Jx8BuYH-ls)

而如果说通过线程池的方式并行查询，那查询全部商品信息的时间就取决于多个流程中最慢的那一条。

假设优惠信息流程查询时间 80ms，其他流程查询时间 50ms，经过线程池并行优化后，商品详情接口响应时间就是 80ms，**通过并行缩短了整体查询时间**。
> 线程池种并行提交任务的完成时间，取决于这些任务中执行时间最慢的流程。

![1710916862386-3a8a4df9-a55d-4791-b9c3-20afb7d2de94.png](https://article-images.zsxq.com/FgrMg8xoPkJ4MWZEwNHahw9PExQP)

这种场景想要达到的效果是**最快时间将结果响应给用户** ，我们在创建线程池时**不应该使用阻塞队列去缓冲任务**，而是可以尝试适当调大核心线程数和最大线程数，提高任务并行执行的性能。

通过串行化获取商品详情信息，代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    ​
    import lombok.SneakyThrows;
    ​
    public class Main {
    ​
        public static void main(String[] args) {
            long startTime = System.currentTimeMillis();
            getProductInventory();
            getProductPromotions();
            getProductReviews();
            System.out.println("查询商品详情耗时：" + (System.currentTimeMillis() - startTime));
        }
    ​
        /**
         * 获取商品库存信息
         */
        @SneakyThrows
        private static Object getProductInventory() {
            Thread.sleep(50);
            return new Object();
        }
    ​
        /**
         * 获取商品优惠信息
         */
        @SneakyThrows
        private static Object getProductPromotions() {
            Thread.sleep(80);
            return new Object();
        }
    ​
        /**
         * 获取商品评论信息
         */
        @SneakyThrows
        private static Object getProductReviews() {
            Thread.sleep(50);
            return new Object();
        }
    }
              
上面程序输出耗时大概在 180ms 到 200ms 之间，因为程序是串行化，所以要将所有方法耗时进行累加，同时要加上计算机运行打印和获取当前时间等步骤。

如果希望通过线程池的方式并行查询，需要重构下相关逻辑。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import lombok.SneakyThrows;
    ​
    import java.util.ArrayList;
    import java.util.List;
    import java.util.concurrent.Future;
    import java.util.concurrent.SynchronousQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    ​
    public class Main {
    ​
        private static final ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                6,
                9,
                1024,
                TimeUnit.SECONDS,
                new SynchronousQueue<>());
    ​
        @SneakyThrows
        public static void main(String[] args) {
            long startTime = System.currentTimeMillis();
            List<Future<Object>> results = new ArrayList<>();
            Future<Object> getProductInventory = threadPoolExecutor.submit(Main::getProductInventory);
            results.add(getProductInventory);
            Future<Object> getProductPromotions = threadPoolExecutor.submit(Main::getProductPromotions);
            results.add(getProductPromotions);
            Future<Object> getProductReviews = threadPoolExecutor.submit(Main::getProductReviews);
            results.add(getProductReviews);
            for (Future<Object> result : results) {
                result.get();
            }
            System.out.println("查询商品详情耗时：" + (System.currentTimeMillis() - startTime));
            // 优雅编码，这里如果不进行停止线程池，测试方法不能主动结束
            threadPoolExecutor.shutdown();
        }
    ​
        /**
         * 获取商品库存信息
         */
        @SneakyThrows
        private static Object getProductInventory() {
            Thread.sleep(50);
            return new Object();
        }
    ​
        /**
         * 获取商品优惠信息
         */
        @SneakyThrows
        private static Object getProductPromotions() {
            Thread.sleep(80);
            return new Object();
        }
    ​
        /**
         * 获取商品评论信息
         */
        @SneakyThrows
        private static Object getProductReviews() {
            Thread.sleep(50);
            return new Object();
        }
    }
              
并发执行的程序耗时大概在 80ms 到 100ms 之间，因为上面有说，线程池种并行提交任务的完成时间，取决于这些任务中执行时间最慢的流程。

### 2. 快速处理批量任务 {#2}

在工作中，快速处理批量任务场景比较多，包括不限于以下举得例子：

*  
公司举办周年庆，需要给每个员工发送邮件说明。  
*  
  短信平台后台通过上传 Excel 给一批用户发送短信。

我们以发送批量短信举例子，假设需要给 100 万用户发送短信通知，一条短信通知流程需要 50ms，初步计算如果要给所有用户发送成功，执行时间大概需要 13 小时左右。

![1710924369659-ac0388a5-aca6-43d4-ae06-b1f71ca81cc1.png](https://article-images.zsxq.com/Fl62UWxawjC4WjqcSy37FUMqMTo0)

处理批量任务和快速响应用户请求不一致，在后者的使用场景中，主线程需要阻塞的等待所有任务执行完成，因此必须要尽可能减少等待时间，而前者的使用场景中则完全不需要考虑这个问题，因此我们可以设置一个合适的阻塞队列用来缓冲任务。
> 大量任务堆积线程池阻塞队列场景下，可能会遇到项目发布重启或者意外宕机等情况，进而导致任务丢失风险。

如果我们通过线程池并发执行，假设线程池内线程数是 10 个，那么执行时间将会大幅度缩减，最佳情况会缩短时间时间接近 10 倍的性能，约等于 1 小时 38 分钟左右。

![1710924383065-d72a3b22-e146-4ba3-bef6-03916142d308.png](https://article-images.zsxq.com/FrbYdbCjGzYynz80GYCHB5ROplVo)

通过串行化批量发送用户短信通知，代码如下所示：  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import cn.hutool.core.collection.ListUtil;
    import lombok.SneakyThrows;
    ​
    import java.util.List;
    ​
    public class Main {
    ​
        public static void main(String[] args) {
            List<String> phones = ListUtil.of("1560116xxxx", "1560116xxxx", "1560116xxxx", "1560116xxxx");
            for (String phone : phones) {
                sendPhoneSms(phone);
            }
        }
    ​
        /**
         * 发送手机短信流程
         */
        @SneakyThrows
        private static void sendPhoneSms(String phone) {
            Thread.sleep(50);
        }
    }
              
在这种情况下，我们使用阻塞队列来缓冲任务。提交任务的线程将任务提交到线程池，然后线程池中的线程会从阻塞队列中获取任务并执行。

这种执行方式允许任务的提交和执行解耦，提高了系统的灵活性和并发处理能力。  
textjavascripttypescriptcsshtmlbashjsonmarkdownpythonjavaccpprubygorustphpsqlyaml Copy

                    import cn.hutool.core.collection.ListUtil;
    import lombok.SneakyThrows;
    ​
    import java.util.List;
    import java.util.concurrent.LinkedBlockingQueue;
    import java.util.concurrent.ThreadPoolExecutor;
    import java.util.concurrent.TimeUnit;
    ​
    public class Main {
    ​
        private static final ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                10,
                10,
                1024,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(100000));
    ​
        public static void main(String[] args) {
            List<String> phones = ListUtil.of("1560116xxxx", "1560116xxxx", "1560116xxxx", "1560116xxxx");
            for (String phone : phones) {
                threadPoolExecutor.execute(() -> sendPhoneSms(phone));
            }
        }
    ​
        /**
         * 发送手机短信流程
         */
        @SneakyThrows
        private static void sendPhoneSms(String phone) {
            Thread.sleep(50);
        }
    }
              
绝知此事要躬行，非常建议大家能够实际本地代码运行下线程池，入门并发编程不二之选。

完结，撒花 🎉  

