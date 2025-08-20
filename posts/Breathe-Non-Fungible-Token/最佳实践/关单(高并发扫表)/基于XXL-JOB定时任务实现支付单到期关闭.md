[[基于XXL-JOB的分片实现分库分表后的扫表]]

定时任务的调度部分和上文一致, 这里就不再重复了, 只介绍一下关单操作的具体内容.

这里关单的时候做一次判断, 查询支付单, 判断是否有支付过的, 避免渠道支付回调已经回来了, 但是还没来得及通知订单模块, 导致误关单的情况.

![[Pasted image 20250816105933.png]]

也就是上面第一步成功以后, 第三步没来得及通知, 第二步先执行了, 如果不做判断的话, 就会导致第三步再过来的时候, 订单已经被关的情况, 所以在这里反查一下, 减少这个情况发生的概率.(但是也会存在渠道侧成功了, 但是还没来得及通知支付模块的情况, 这种再回来退款就行了)

```java
private void executeTimeoutSingle(TradeOrder tradeOrder) {  
    //查询支付单，判断是否已经支付成功。  
    PayQueryRequest request = new PayQueryRequest();  
    request.setPayerId(tradeOrder.getBuyerId());  
    request.setPayOrderState(PayOrderState.PAID);  
    PayQueryByBizNo payQueryByBizNo = new PayQueryByBizNo();  
    payQueryByBizNo.setBizNo(tradeOrder.getOrderId());  
    payQueryByBizNo.setBizType(BizOrderType.TRADE_ORDER.name());  
    request.setPayQueryCondition(payQueryByBizNo);  
    MultiResponse<PayOrderVO> payQueryResponse = payFacadeService.queryPayOrders(request);  
  
    if (payQueryResponse.getSuccess() && CollectionUtils.isEmpty(payQueryResponse.getDatas())) {  
        LOG.info("start to execute order timeout , orderId is {}", tradeOrder.getOrderId());  
        OrderTimeoutRequest orderTimeoutRequest = new OrderTimeoutRequest();  
        orderTimeoutRequest.setOrderId(tradeOrder.getOrderId());  
        orderTimeoutRequest.setOperateTime(new Date());  
        orderTimeoutRequest.setOperator(UserType.PLATFORM.name());  
        orderTimeoutRequest.setOperatorType(UserType.PLATFORM);  
        orderTimeoutRequest.setIdentifier(tradeOrder.getOrderId());  
        orderFacadeService.timeout(orderTimeoutRequest);  
    }  
}
```


timeout的实现的接口如下:

```java
@Override  
@Facade  
public OrderResponse timeout(OrderTimeoutRequest request) {  
    return sendTransactionMsgForClose(request);  
}
```

这里依赖的是rocketmq的事务消息来实现的, 具体参考:

[[基于RocketMQ事务消息实现订单取消的一致性]]
