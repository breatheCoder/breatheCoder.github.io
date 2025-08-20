项目中有一个NewBuyMsgListener监听器，用于监听下单消息进行创单的，他是通过SpringCloud Stream的方式配置的监听，这个监听器用起来很方便。

具体实现参考：

[[NewBuyBatchMsgListener和NewBuyMsgListener有什么关系]]

但是这个方式有个问题，就是他无法批量消费消息，只能单条消费，但是单条消费速度比较慢，会导致消息堆积，于是我们考虑搞一个批量消费。

但是很遗憾，Spring Cloud Stream对RocketMQ的支持中，不包含对批量消费的支持。所以没办法，就需要用RocketMQ原生的方式实现，那就是我们的NewBuyBatchMsgListener了。他的监听配置主要在：

```yaml
rocketmq: 
	consumer: 
		group: trade-group 
		# 一次拉取消息最大值，注意是拉取消息的最大值而非消费最大值 
		pull-batch-size: 64 
		consume-message-batch-max-size: 32
```

然后在bean上也要有@RocketMQMessageListener(topic = "new-buy-topic", consumerGroup = "trade-group")这样的配置，指明监听的topic。

然后重写其中的prepareStart方法，在其中设置批量消费的参数：

```java
consumer.setPullInterval(1000);
consumer.setConsumeMessageBatchMaxSize(128);
consumer.setPullBatchSize(64);
```

紧接着就可以用MessageListenerConcurrently来批量处理消息了。

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

这里面还用到了一个CompletionService，主要原因参考：

[[通过CompletionService编排并发消费任务，解决串行执行的速度瓶颈。]]

