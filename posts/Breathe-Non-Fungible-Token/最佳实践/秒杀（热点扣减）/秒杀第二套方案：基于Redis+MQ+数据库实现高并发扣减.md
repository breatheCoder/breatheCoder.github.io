我们的项目中，秒杀有3套方案，一种是基于InventoryHint 的方案，还有一种是不用 InventoryHint，而是用 RocketMQ 的方案。



为了可以抗更高的并发，我们把数据库中的库存扣减，放到了 Redis 中，在 Redis 进行扣减，但是，最终还是要同步到数据库的，因为 Redis 不可靠，所以我们还是要在数据库做真正的扣减。



只不过在数据库中扣减因为前面有 Redis 拦了一道了，真正过来到数据库中流量会小很多，所以数据库就能扛得住了。



那么，有什么办法可以让 Redis 扣减成功后，数据库也能进行扣减，并且可以扛得住剩余的高并发的热点行更新，并且能够在数据库扣减失败后重试。那就是 MQ 了。



我们在项目中引入 MQ，帮我们做下单环节的高并发流量的削峰填谷。整体流程如下：



![[69782887-17bb-476d-9404-46297e1024f6.png]]



1、用户秒杀请求过来后，先在 Redis 中进行扣减，利用 Redis 的单线程、高性能机制，可以保证在有库存的时候多个用户按顺序扣减。如果库存没有了，则失败。



2、对于在 Redis 中扣减成功的请求，说明他们下单的时候库存还有，那么就可以把这个用户的流量放过。这时候就发一个 MQ，告诉数据库，这个用户是可以下单的。



3、数据库在接受到MQ 的消息后，在进行数据库层面的库存扣减。这里的流量就小很多了，而且可以做限流和降级，因为 MQ 本身可以重试，也能控制速率。



详细的过程如下：



![[9898beef-0d53-4b29-8d9c-ba0844a48103.svg]]



newBuy 是基于 MQ 的方案的秒杀方案入口：

```java
@PostMapping("/newBuy")
public Result<String> newBuy(@Valid @RequestBody BuyParam buyParam) {
    OrderCreateRequest orderCreateRequest = getOrderCreateRequest(buyParam);

    try {
        orderPreValidatorChain.validate(orderCreateRequest);
    } catch (OrderException e) {
        throw new TradeException(e.getErrorCode().getMessage(), ORDER_CREATE_PRE_VALID_FAILED);
    }

    //消息监听：NewBuyMsgListener or NewBuyBatchMsgListener
    boolean result = streamProducer.send("newBuy-out-0", buyParam.getGoodsType(), JSON.toJSONString(orderCreateRequest));

    if (!result) {
        throw new TradeException(TradeErrorCode.ORDER_CREATE_FAILED);
    }

    //因为不管本地事务是否成功，只要一阶段消息发成功都会返回 true，所以这里需要确认是否成功
    InventoryRequest inventoryRequest = new InventoryRequest(orderCreateRequest);
    SingleResponse<String> response = inventoryFacadeService.getInventoryDecreaseLog(inventoryRequest);

    if (response.getSuccess() && response.getData() != null) {
        return Result.success(orderCreateRequest.getOrderId());
    }

    throw new TradeException(TradeErrorCode.ORDER_CREATE_FAILED);
}
```



这里，为了保证 Redis 扣减成功后，MQ 一定可以发成功，我们用了RocketMQ 的事务消息。newBuy是一个事务消息，可以先发半消息，再执行本地事务，成功后再发送另外一个半消息。



事务消息原理：

[[基于RocketMQ事务消息实现订单取消的一致性]]


配置如下：

![[af8b4f35-a43a-4a9c-abcc-c7fdda9ae4e3.png]]

![[9513fe1a-dd7f-488e-8b21-52dbbad59cdb.png]]



在本地事务操作中。我们直接去 Redis 做扣减



![[5a61acfd-a74a-407f-a4d2-338d2910c448.png]]



同时在Redis 扣减成功后 commit 这个消息，失败的话，则 rollback 这个消息。



并且提供一个供 MQ 反查的方法，当消息丢失时也可以知道要不要提交：



![[ed888767-0190-4578-b4cb-509d3ecb79d7.png]]



实现也很简单，就是去 Redis 中查询是否有扣减流水，如果有，则说明扣减成功了。如果没有，这说明没扣减成功。



只有 Redis 扣减成功，最终这个事务消息就会发出去，然后在 order 模块中就可以监听这个消息，做订单创建以及库存扣减了。

![[a08a5189-63a7-4cd8-9e7f-d10c63cb9b19.png]]



这里的createAndConfirm方法，就是具体的订单创建、以及库存扣减的过程了。流程和用 Hint 的方案差不多，只不过库存扣减的 SQL 上是无 Hint 的：



![[6aebe3e8-aa87-4b90-9aeb-5ad3ccb1fcf3.png]]



```java
<!--  库存预扣减-无hint版  -->
<update id="trySaleWithoutHint">
    UPDATE collection
    SET saleable_inventory = saleable_inventory - #{quantity}, lock_version = lock_version + 1,gmt_modified = now()
    WHERE id = #{id} and <![CDATA[saleable_inventory >= #{quantity}]]>
</update>
```



另外，有个地方需要注意，RocketMQ的事务消息，不管本地事务是否成功，只要一阶段消息发成功都会返回 true，所以，在发消息之后，需要确认下本地事务是否执行成功了。即在newBuy方法中增加以下校验：



![[431ca7a3-0e99-456b-a56a-10697c523828.png]]



这里之所以可以直接反查，是因为在RocketMQ的事务消息中，本地事务的操作（InventoryDecreaseTransactionListener#executeLocalTransaction）是同步执行的，所以只要同步调用成功了，这里一定能查得到。

