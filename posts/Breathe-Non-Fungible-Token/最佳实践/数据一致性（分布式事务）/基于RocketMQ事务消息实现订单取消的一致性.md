为了保证订单取消的时候，可以让订单的关单、库存的回退在同一个事务中，能够保证最终一致性，本项目中采用 RocketMQ 的事务消息。



![[9bbd1cfe-cfbd-421c-99b1-5878c667150c.jpeg]]

也可以参考八股文中的，rocketmq的事务消息是如何实现的？



正常情况情况下，如果一切顺利，那么就是本地事务和发送消息都成功，然后下游把消息也消费成功。



如果是非正常情况下，那么则可能发生以下过程：



情况1、半消息发送失败，则不需要执行本地事务，直接返回失败即可。



情况2、订单服务的本地事务执行失败，则发送 rollback 消息给 MQ，下游无需执行



情况3、第二个半消息发送失败，则 MQ 回查订单服务的事务执行结果，进行自己的半消息的提交或者回滚。



情况4、如果 MQ 发送 commit 消息失败，或者藏品服务执行过程中失败，则依赖 MQ 的重投继续重试即可。



以下是代码实现，其中取消订单和超时关单两个方法，都是通过这个方案实现的。


![[dca774c5-fec0-413c-8acd-fe993ef86a6d.png]]



接着看 sendTransactionMsgForClose的逻辑：

```java
@NotNull
private OrderResponse sendTransactionMsgForClose(BaseOrderUpdateRequest request) {
    boolean result = streamProducer.send("orderClose-out-0", null, JSON.toJSONString(request), "CLOSE_TYPE", request.getOrderEvent().name());
    OrderResponse orderResponse = new OrderResponse();
    if (result) {
        orderResponse.setSuccess(true);
    } else {
        orderResponse.setSuccess(false);
    }
    return orderResponse;
}
```

这里其实很简单，只是发了一个消息而已。

然后在消息监听者这里是重点。cyou.breathe.nft.order.listener.OrderCloseTransactionListener是这个`orderClose-out-0`的处理器，配置如下：

```yaml
spring:
  cloud:
    stream:
      rocketmq:
        bindings:
          orderClose-out-0:
            producer:
              producerType: Trans
              transactionListener: orderCloseTransactionListener
```

这样，就会在发消息的过程中，通过orderCloseTransactionListener来进行。这个类继承自`**TransactionListener**`，这是 RocketMQ 事务消息的根本。



这个接口中有两个方法：



```java
public interface TransactionListener {
    /**
     * When send transactional prepare(half) message succeed, this method will be invoked to execute local transaction.
     *
     * @param msg Half(prepare) message
     * @param arg Custom business parameter
     * @return Transaction state
     */
    LocalTransactionState executeLocalTransaction(final Message msg, final Object arg);

    /**
     * When no response to prepare(half) message. broker will send check message to check the transaction status, and this
     * method will be invoked to get local transaction status.
     *
     * @param msg Check message
     * @return Transaction state
     */
    LocalTransactionState checkLocalTransaction(final MessageExt msg);
}
```

executeLocalTransaction中执行本地事务，RocketMQ会在发送第一个半消息之后，调用这个方法。



checkLocalTransaction是检查本地事务是否成功的回调。



比如我们的关单操作的executeLocalTransaction代码如下，主要就是去执行本地的订单关单操作：ji



```java
public LocalTransactionState executeLocalTransaction(Message message, Object o) {
    try {
        Map<String, String> headers = message.getProperties();
        String closeType = headers.get("CLOSE_TYPE");

        OrderResponse response = null;
        if (TradeOrderEvent.CANCEL.name().equals(closeType)) {
            OrderCancelRequest cancelRequest = JSON.parseObject(JSON.parseObject(message.getBody()).getString("body"), OrderCancelRequest.class);
            logger.info("executeLocalTransaction , baseOrderUpdateRequest = {} , closeType = {}", JSON.toJSONString(cancelRequest), closeType);
            response = orderManageService.cancel(cancelRequest);
        } else if (TradeOrderEvent.TIME_OUT.name().equals(closeType)) {
            OrderTimeoutRequest timeoutRequest = JSON.parseObject(JSON.parseObject(message.getBody()).getString("body"), OrderTimeoutRequest.class);
            logger.info("executeLocalTransaction , baseOrderUpdateRequest = {} , closeType = {}", JSON.toJSONString(timeoutRequest), closeType);
            response = orderManageService.timeout(timeoutRequest);
        } else {
            throw new UnsupportedOperationException("unsupported closeType " + closeType);
        }

        if (response.getSuccess()) {
            return LocalTransactionState.COMMIT_MESSAGE;
        } else {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    } catch (Exception e) {
        logger.error("executeLocalTransaction error, message = {}", message, e);
        return LocalTransactionState.ROLLBACK_MESSAGE;
    }
}
```



而checkLocalTransaction则检查这一步的本地事务是否成功。即判断订单是否已经被关单，如果被关了，则任务消息需要 COMMIT 了。



```java
@Override
public LocalTransactionState checkLocalTransaction(MessageExt messageExt) {
    BaseOrderUpdateRequest baseOrderUpdateRequest = JSON.parseObject(messageExt.getBody(), BaseOrderUpdateRequest.class);

    TradeOrder tradeOrder = orderReadService.getOrder(baseOrderUpdateRequest.getOrderId());

    if (tradeOrder.getOrderState() == TradeOrderState.CLOSED) {
        return LocalTransactionState.COMMIT_MESSAGE;
    }

    return LocalTransactionState.ROLLBACK_MESSAGE;
}
```

