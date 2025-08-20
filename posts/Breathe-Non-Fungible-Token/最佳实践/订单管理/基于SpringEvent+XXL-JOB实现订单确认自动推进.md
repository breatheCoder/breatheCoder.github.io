- [x] 基于SpringEvent+XXL-JOB实现订单确认自动推进  [completion:: 2025-08-16]

以下是我们的用户下单功能的顺序图:

![[Pasted image 20250816111558.png]]

从图中可以看到, 在我们项目的下单场景下, 先是通过redis进行库存的预扣减, 然后开始创建订单, 订单状态为CREATE.

这时候我们会在异步链路上做订单的确认, 也就是两件事情.

1. 将在redis中扣减的库存, 持久化到MySQL中.
	- 调用collectionService的trySale方法.
2. 将订单的状态从CREATE推进到CONFIRM
	－　调用orderService的confirm方法.

那么这个订单确认功能, 其实需要满足两点功能, 即解耦 + 异步.

解耦: 订单确认是一个自动的过程, 他是在下单过程中, 即订单的创建完成后的一个后置操作, 我们需要把他从订单创建的方法中解耦出来, 因为他严格来说并不是订单创建的一部分. 至少在OrderManageService中, create和confirm是两个完全不相关的方法.

异步: 这个比较好理解, 就是他并不在订单创建方法的主流程中, 不要阻塞主流程的执行, 所以需要做异步处理.

能起到异步 + 解耦的, 就是事件机制了, 常见的事件机制有好几个方案:

1. 通过rocketmq发一个消息, 然后再接收这个消息进行处理.
	- 好处是基于MQ进行处理, 可以做到更好的削峰填谷.
	- 缺点是MQ投递会有延迟.
2. 通过事件机制, 如SpringEvent进行应用内的异步处理
	- 好处是延迟的问题要比 MQ 好很多，并且可以配合线程池提升性能
	- 缺点1：无法做到削峰填谷，可能会给数据库的热点扣减带来压力
	- 缺点2：任务没有持久化，应用重启后任务就没了

一般来说用第一个方案的比较多, 尤其是在秒杀这个场景中, 基于MQ来做削峰填谷, 利用MQ的队列机制来减少数据库的压力. 但是我们的项目中并没有直接用这个方案, 而是采用了方案2.

主要原因是我们的做数据库扣减这里，依赖了阿里云上 DMS 提供的 Inventory Hint 的机制，说白了就是这里不用 MQ 做缓冲数据库也能扛得住，所以就干脆不用了。

[[秒杀第一套方案：基于InventoryHint实现库存的热点扣减]]

所以，就采用了方案2这样主要以 SpringEvent 做异步处理，但是，方案2有个致命的缺点，那就是没办法做持久化，一旦运行过程中应用重启了，这个任务就没了。所以我们在这个方案的基础上引入了 XXL-JOB进行补偿。

所以, 整个方案是基于SpringEvent和xxl-job实现订单确认的自动推进, 其实是分为两步:

第一步: 通过springEvent的事件推动, 把订单确认的事件流程丢到异步线程池中进行.

第二步: 通过xxl-job查找未处理的订单, 通过补偿任务的形式进行推进, 从而保证订单确认的处理完成.

## 创建event

首先我们需要定义一个事件, 并且这个事件是需要实现spring的applicationEvent类.

```java
public class OrderCreateEvent extends ApplicationEvent {
	public OrderCreateEvent(TradeOrder tradeOrder) {
		super(tradeOrder);
	}
}
```

## 在业务逻辑中发布event

然后需要在订单创建结束之后，把这个事件发出来：

```java
@Transactional(rollbackFor = Exception.class)  
public OrderResponse createAndAsyncConfirm(OrderCreateRequest request) {  
    //幂等，一查二判三更新  
    TradeOrder existOrder = orderMapper.selectByIdentifier(request.getIdentifier(), request.getBuyerId());  
    if (existOrder != null) {  
        return new OrderResponse.OrderResponseBuilder().orderId(existOrder.getOrderId()).buildSuccess();  
    }  
  
    TradeOrder tradeOrder = doCreate(request);  
    //发布事件，进行异步confirm  
    applicationContext.publishEvent(new OrderCreateEvent(tradeOrder));  
    return new OrderResponse.OrderResponseBuilder().orderId(tradeOrder.getOrderId()).buildSuccess();  
}
```

这里通过applicationContext实现发布.

## 定义事件监视器

同时我们定义一个事件的监视器, 用来处理这个事件

```java
@Component  
public class OrderEventListener {  
  
    @Autowired  
    private OrderFacadeService orderFacadeService;  
  
    @TransactionalEventListener(value = OrderCreateEvent.class)  
    @Async("orderListenExecutor")  
    public void onApplicationEvent(OrderCreateEvent event) {  
  
        TradeOrder tradeOrder = (TradeOrder) event.getSource();  
        OrderConfirmRequest confirmRequest = new OrderConfirmRequest();  
        confirmRequest.setOperator(UserType.PLATFORM.name());  
        confirmRequest.setOperatorType(UserType.PLATFORM);  
        confirmRequest.setOrderId(tradeOrder.getOrderId());  
        confirmRequest.setIdentifier(tradeOrder.getIdentifier());  
        confirmRequest.setOperateTime(new Date());  
        confirmRequest.setBuyerId(tradeOrder.getBuyerId());  
        confirmRequest.setItemCount(tradeOrder.getItemCount());  
        confirmRequest.setGoodsType(tradeOrder.getGoodsType());  
        confirmRequest.setGoodsId(tradeOrder.getGoodsId());  
        orderFacadeService.confirm(confirmRequest);  
    }  
}
```

@EventListener(OrderCreateEvent.class)表示接收并处理OrderCreateEvent事件

@Async("orderListenExecutor") 表示将使用名称为orderListenExecutor的自定义线程池来异步执行该方法

完成以上配置后，订单创建过程中就会发布一个OrderCreateEvent，这个事件会被OrderEventListener给订阅到，然后基于我们自定义的线程池进行异步处理。

## 配置xxl-job定时任务

为了做失败的补偿，我们定义一个 xxl-job 的回调任务，这个任务也是借助了 XXL-JOB 的分片任务，以及生产者消费者模式来实现的，主要就是为了提升性能。

原理和下面这篇文章中讲的是一样的：

[[基于生产者消费者+线程池实现并发关闭订单]]

处理核心逻辑：
- 通过买家id尾号 orderReadService.pageQueryNeedConfirmOrders 分页查询需要处理的待确认数据
- 把它放入阻塞队列中，然后把阻塞队列放入forkJoinPool中进行处理
- executeConfirm核心是从阻塞队列一直取并通过executeConfirmSingle处理，取到POISON表示已经没有待处理数据，然后结束

