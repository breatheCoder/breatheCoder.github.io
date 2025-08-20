在订单完成支付后，我们会接受到外部支付渠道的支付成功消息。接收到消息之后需要做以下事情：

```latex
正常支付成功：
    1.查询订单状态
    2.推进订单状态到支付成功
    3.藏品库存真正扣减
    4.创建真正的藏品
    5.推进支付状态到支付成功
    6.持有的藏品上链（mint）

支付幂等成功：
    1.查询订单状态
    2.推进支付状态到支付成功

重复支付：
    1.查询订单状态
    2.创建退款单
    3.重试退款直到成功
```



三种情况，支付成功，重复支付，以及支付幂等。其中支付成功这个子流程，可以看到我们操作了很多i模块，包括了订单，支付，藏品，链等等。



那么，我们如何保证了这个过程中的一致性呢？这里我们用到了seata的分布式事务，用的是AT模式，之所以用这个模式，是因为他的侵入性低，而且性能也不差。



![[eda17b70-02ae-43ef-8a09-274ad845de9f.png]]



在项目接入Seata以后，其实使用seata的AT模式还是比较简单的，只需要在事务发起处加一个@GlobalException，这样seata就会开启一个全局事务，就会通过和我们搭建好的seata服务端进行交互，进行整个分布式事务的协调。



Seata中包含了三个组件：

+ TC（Transaction Coordinator）：事务协调器，负责管理全局事务的生命周期，包括开始事务，提交事务和回滚事务。
+ TM（Transaction Manager）：事务管理器，定义事务的范围，负责开启和结束全局事务。
+ RM（Resource Manager）：资源管理器，管理资源对象，如数据库连接，负责资源的注册与释放。



在我们的场景中，TC就是我们搭建好的Seata的服务端，和我们的代码没关系。TM就是我们的支付模块，RM就是我们的订单模块，藏品模块等。



用了Seata以后，一次分布式事务的大致流程如下：



1. TM（支付服务）在接收到支付成功的回调请求之后，会先调用TC（Seata服务器）创建一个全局事务，并且从TC处获取到他生成的XID。
2. TM（支付服务）还是通过Dubbo调用各个RM，调用过程中需要把XID同时传递过去。
3. RM（订单，藏品等服务）通过其接收到的XID，将其所管理的资源且被该调用所使用到的资源注册为一个事务分支（Branch Transaction）
4. 当该请求的调用链全部结束的时候，TM（支付服务）根据本次调用是否有失败的情况，如果所有调用都成功，则决议commit，如果有超时或失败，则决议rollback。
5. TM（支付服务）将事务的决议结果告诉TC（Seata服务器)，TC（Seata服务器）将协调所有RM（订单，藏品等服务）进行事务的二阶段操作，该回滚回滚，该提交提交。



回滚的实现是通过我们的数据库配置的一个undo_log所实现的，所以，如果我们多个模块，用的是不同的数据库，都需要单独在数据库中创建一个undo_log，这个undolog和innodb的那个undolog不是一回事，需要根据seata的部署文档创建的。

## 代码实现


```java
@GlobalTransactional(rollbackFor = Exception.class)
public boolean paySuccess(PaySuccessEvent paySuccessEvent) {

    PayOrder payOrder = payOrderService.queryByOrderId(paySuccessEvent.getPayOrderId());
    if (payOrder.isPaid()) {
        return true;
    }

    SingleResponse<TradeOrderVO> response = orderFacadeService.getTradeOrder(payOrder.getBizNo());
    TradeOrderVO tradeOrderVO = response.getData();

    OrderPayRequest orderPayRequest = getOrderPayRequest(paySuccessEvent, payOrder);
    OrderResponse orderResponse = RemoteCallWrapper.call(req -> orderFacadeService.pay(req), orderPayRequest, "orderFacadeService.pay");
    if (orderResponse.getResponseCode() != null && orderResponse.getResponseCode().equals(OrderErrorCode.ORDER_ALREADY_PAID.getCode())) {
        doChargeBack(paySuccessEvent);
        return true;
    }

    if (!orderResponse.getSuccess()) {
        log.error("orderFacadeService.pay error, response = {}", JSON.toJSONString(orderResponse));
        return false;
    }

    CollectionSaleRequest collectionSaleRequest = getCollectionSaleRequest(tradeOrderVO);
    CollectionSaleResponse collectionSaleResponse = RemoteCallWrapper.call(req -> collectionFacadeService.confirmSale(req), collectionSaleRequest, "collectionFacadeService.confirmSale");

    TransactionHookManager.registerHook(new PaySuccessTransactionHook(collectionSaleResponse.getHeldCollectionId()));

    Boolean result = payOrderService.paySuccess(paySuccessEvent);
    Assert.isTrue(result, "payOrderService.paySuccess failed");

    return true;
}
```



我们在支付服务的paySuccess方法上添加了@GlobalTransactionException注解，代码执行到这一行的时候就会自动开去全局事务。



然后在这个方法中通过Dubbo去远程调用各个其他服务，并且这里我们大量用到了Assert，来判断远程服务的结果，如果有某个服务失败了，则抛出异常。然后seata会拦截到异常，然后进行全局事务的回滚动作。



各个参与事务的服务，如订单服务、藏品服务等，如果用了支持seata的rpc框架，则不需要做任何事情，只需要同样接入seata，他么就能自动的创建分支事务、以及进行事务的提交和回滚。



但是如果用到了shardingjdbc，则需要做一些配置和改动，可以参考：

[[Seata 接入]]


```java
/**
 * 订单支付
 *
 * @param request
 * @return
 */
@Transactional(rollbackFor = Exception.class)
@ShardingSphereTransactionType(TransactionType.BASE)
public OrderResponse pay(OrderPayRequest request) {
    return doExecuteWithOutTrans(request, tradeOrder -> tradeOrder.pay(request));
}
```



这里就是多用了一个@ShardingSphereTransactionType(TransactionType.BASE)。并且在doExecuteWithOutTrans里面不需要再加其他的事务了。其他的doExecute方法我们为了减少事务粒度，用了编程式事务，这里不能用了，所以就单独加了个doExecuteWithOutTrans。

