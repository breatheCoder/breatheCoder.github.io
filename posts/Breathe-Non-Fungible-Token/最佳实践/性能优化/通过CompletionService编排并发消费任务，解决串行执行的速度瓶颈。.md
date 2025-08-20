在RocketMQ使用了批量消费之后，会一次性拉取一批消息过来消费。

但是这里有个问题，如果把这个一批消息直接都交给线程池执行，那么如果异步线程执行失败了，MQ的Broker是无法知道的，我们需要有个办法可以在主线程中知道每个子线程的执行结果。

如果所有线程都成功了，则返回CONSUME_SUCCESS给到RocketMQ，他就可以提交这些消息的偏移量了，如果有一条失败了，则返回RECONSUME_LATER，则让RocketMQ整体重新发送。

这里我们引入了CompletionService帮我们做编排。CompletionService 是 Java 并发编程中的一个核心接口（位于 java.util.concurrent 包），主要设计用于批量管理异步任务并高效处理任务结果。

那么我们就可以按照以下3个步骤让代码执行：

1、 提交所有任务到线程池。

2、循环检查结果，如果发现有失败的，则标记整体失败。

3、检查结果，如果都成功了，返回CONSUME_SUCCESS，否则返回RECONSUME_LATER

代码逻辑如下：

```java
@Override  
public void prepareStart(DefaultMQPushConsumer consumer) {  
    consumer.setPullInterval(500);  
    consumer.setConsumeMessageBatchMaxSize(64);  
    consumer.setPullBatchSize(64);  
    consumer.registerMessageListener((MessageListenerConcurrently) (msgs, context) -> {  
        log.warn("NewBuyBatchMsgListener receive message size: {}", msgs.size());  
  
        CompletionService<Boolean> completionService = new ExecutorCompletionService<>(newBuyConsumePool);  
        List<Future<Boolean>> futures = new ArrayList<>();  
  
        // 1. 提交所有任务  
        msgs.forEach(messageExt -> {  
            Callable<Boolean> task = () -> {  
                try {  
                    OrderCreateRequest orderCreateRequest = JSON.parseObject(JSON.parseObject(messageExt.getBody()).getString("body"), OrderCreateRequest.class);  
                    return doNewBuyExecute(orderCreateRequest);  
                } catch (Exception e) {  
                    log.error("Task failed", e);  
                    return false; // 标记失败  
                }  
            };  
            futures.add(completionService.submit(task));  
        });  
  
        // 2. 检查结果  
        boolean allSuccess = true;  
        try {  
            for (int i = 0; i < msgs.size(); i++) {  
                Future<Boolean> future = completionService.take();  
                if (!future.get()) { // 3.发现一个失败立即终止  
                    allSuccess = false;  
                    break;  
                }  
            }        } catch (Exception e) {  
            allSuccess = false;  
        }  
  
        // 3. 根据结果返回消费状态  
        return allSuccess ? ConsumeConcurrentlyStatus.CONSUME_SUCCESS  
                : ConsumeConcurrentlyStatus.RECONSUME_LATER;  
    });  
}
```

并发任务提交:

```java
CompletionService<Boolean> completionService = new ExecutorCompletionService<>(executor);

msgs.forEach(messageExt -> {  
    Callable<Boolean> task = () -> {  
        try {  
            OrderCreateRequest orderCreateRequest = JSON.parseObject(JSON.parseObject(messageExt.getBody()).getString("body"), OrderCreateRequest.class);  
            return doNewBuyExecute(orderCreateRequest);  
        } catch (Exception e) {  
            log.error("Task failed", e);  
            return false; // 标记失败  
        }  
    };  
    futures.add(completionService.submit(task));  
});


```

1、completionService.submit()提交任务，并交给线程池执行。

结果聚合检查：

```java
boolean allSuccess = true;  
try {  
    for (int i = 0; i < msgs.size(); i++) {  
        Future<Boolean> future = completionService.take();  
        if (!future.get()) { // 3.发现一个失败立即终止  
            allSuccess = false;  
            break;  
        }  
    }} catch (Exception e) {  
    allSuccess = false;  
}
```

1、检查所有任务必须成功
2、利用 CompletionService.take() 按任务完成顺序获取结果