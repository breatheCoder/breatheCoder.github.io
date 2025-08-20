这个项目没有完全按照DDD来做，因为DDD在业内使用的还是比较少的，大家也没有一个统一的共识，有人推崇，有人反对，不像ＭＶＣ架构



但是我们还是从DDD中吸收了很多精华去使用的，比如分层架构，充血模型，事件驱动和设计模式等等．



## 分层架构
DDD的分层架构是一个四层架构:

从上到下

![[1a604486-64fc-4b7b-8f73-b5338c89b516.png]]



层次之间的调用关系是上层调用下层，即用户接口层可以调用应用层，领域层及基础层。应用层可以调用领域层，基础层。但是你不能反着调用，各个层级之间是严格的单向调用的依赖关系

[[代码分层]]


## 充血模型
充血模型是一种面向对象的软件设计方法，他强调将业务逻辑封装在领域对象中。



假设有一个电商平台，需要对商品进行购买，库存管理等操作，在充血模型中，我们可以定义一个Product类来表示商品，Product类会包含商品的属性（例如名称，价格，库存等），并且也会包含一些行为（购买商品，更新库存等）这些行为是直接封装在Product类中的，便于对商品进行操作。

```java
public class Product{
	private String productName;
	private BigDecimal price;
	private Long stock;

	 public void purchase(int quantity) {
      if (quantity > stock) {
          throw new IllegalArgumentException("Not enough stock available");
      }
      stock -= quantity;
  }


}
```



充血模型的优点：

1. 面向对象设计，具有良好的封装性和可维护性
2. 领域对象自己包含业务逻辑，容易理解和扩展
3. 可以避免过度依赖外部服务，提高系统的稳定性



在项目中，TradeOrder就用到了该模型

```java
@Setter
@Getter
public class TradeOrder extends BaseEntity {

    /**
     * 默认超时时间
     */
    public static final int DEFAULT_TIME_OUT_MINUTES = 30;

    /**
     * 订单id
     */
    private String orderId;

    /**
     * 买家id
     */
    private String buyerId;

    /**
     * 买家 ID 的逆序
     */
    private String reverseBuyerId;

    /**
     * 买家id类型
     */
    private UserType buyerType;

    /**
     * 卖家id
     */
    private String sellerId;

    /**
     * 卖家id类型
     */
    private UserType sellerType;

    /**
     * 幂等号
     */
    private String identifier;

    /**
     * 订单金额
     */
    private BigDecimal orderAmount;

    /**
     * 商品数量
     */
    private int itemCount;

    /**
     * 商品单价
     */
    private BigDecimal itemPrice;

    /**
     * 已支付金额
     */
    private BigDecimal paidAmount;

    /**
     * 支付成功时间
     */
    private Date paySucceedTime;

    /**
     * 下单确认时间
     */
    private Date orderConfirmedTime;

    /**
     * 下单确认时间
     */
    private Date orderFinishedTime;

    /**
     * 订单关闭时间
     */
    private Date orderClosedTime;

    /**
     * 商品Id
     */
    private String goodsId;

    /**
     * 商品类型
     */
    private GoodsType goodsType;

    /**
     * 商品名称
     */
    private String goodsName;

    /**
     * 商品图片
     */
    private String goodsPicUrl;

    /**
     * 支付方式
     */
    private PayChannel payChannel;

    /**
     * 支付流水号
     */
    private String payStreamId;

    /**
     * 订单状态
     */
    private TradeOrderState orderState;

    /**
     * 关单类型
     */
    private String closeType;

    /**
     * 快照版本
     */
    private Integer snapshotVersion;

    @JSONField(serialize = false) //通过注解来指定该字段不参与序列化
    public Boolean isPaid() {
        return orderState == TradeOrderState.FINISH || orderState == TradeOrderState.PAID;
    }

    @JSONField(serialize = false)
    public Boolean isConfirmed() {
        return orderState == TradeOrderState.CONFIRM;
    }

    @JSONField(serialize = false)
    public Boolean isTimeout() {
        //订单已关闭 (订单未支付且未关闭 并且 订单已经达到了超时时间)
        //判断当前的订单是否超时
        return (orderState == TradeOrderState.CLOSED && closeType == TradeOrderEvent.TIME_OUT.name())
                || (orderState == TradeOrderState.CONFIRM && this.getGmtCreate().compareTo(DateUtils.addMinutes(new Date(), -TradeOrder.DEFAULT_TIME_OUT_MINUTES)) < 0);
    }

    @JSONField(serialize = false)
    public Boolean isClosed() {
        return orderState == TradeOrderState.CLOSED;
    }

    @JSONField(serialize = false)
    public Date getPayExpireTime() {
        return DateUtils.addMinutes(this.getGmtCreate(), TradeOrder.DEFAULT_TIME_OUT_MINUTES);
    }

    public static TradeOrder createOrder(OrderCreateRequest request) {
        TradeOrder tradeOrder = TradeOrderConvertor.INSTANCE.mapToEntity(request);
        tradeOrder.setReverseBuyerId(StringUtils.reverse(request.getBuyerId()));
        //订单状态为创建，该状态的订单用户是不可见的
        tradeOrder.setOrderState(TradeOrderState.CREATE);
        tradeOrder.setPaidAmount(BigDecimal.ZERO);
        String orderId = request.getOrderId();
        tradeOrder.setOrderId(orderId);
        return tradeOrder;
    }

    /**
     * 处理订单确认的逻辑。
     * 该方法接收一个订单确认请求对象，用于更新订单的确认时间并推进订单状态，将。
     *
     * @param request 订单确认请求对象，包含操作时间和订单事件等信息。
     * @return 处理后的 TradeOrder 对象，包含更新后的订单信息。
     */
    public TradeOrder confirm(OrderConfirmRequest request) {
        this.setOrderConfirmedTime(request.getOperateTime());
        TradeOrderState orderState = OrderStateMachine.INSTANCE.transition(this.getOrderState(), request.getOrderEvent());
        this.setOrderState(orderState);
        return this;
    }

    /**
     * 处理订单支付成功的逻辑。
     * 该方法接收一个订单支付请求对象，用于更新订单的支付相关信息并推进订单状态。
     *
     * @param request 订单支付请求对象，包含支付流水号、支付金额、支付渠道、操作时间和订单事件等信息。
     * @return 处理后的 TradeOrder 对象，包含更新后的订单信息。
     */
    public TradeOrder paySuccess(OrderPayRequest request) {
        this.setPayStreamId(request.getPayStreamId());
        //上次支付成功的时间
        this.setPaySucceedTime(request.getOperateTime());
        //设置支付渠道，是为了避免一笔订单被多个渠道重复支付
        this.setPayChannel(request.getPayChannel());
        this.setPaidAmount(request.getAmount());
        //推进订单状态的更新
        //如果当前订单是支付成功，则推进订单状态时，会报错，后续就会进行退款流程
        TradeOrderState orderState = OrderStateMachine.INSTANCE.transition(this.getOrderState(), request.getOrderEvent());
        this.setOrderState(orderState);
        return this;
    }

    public TradeOrder close(BaseOrderUpdateRequest request) {
        this.setOrderClosedTime(request.getOperateTime());
        //根据订单的当前状态和接收到的订单事件，通过订单状态机来转换订单的状态。
        TradeOrderState orderState = OrderStateMachine.INSTANCE.transition(this.getOrderState(), request.getOrderEvent());
        this.setOrderState(orderState);
        //设置订单关单类型
        this.setCloseType(request.getOrderEvent().name());
        return this;
    }

    public TradeOrder discard(BaseOrderUpdateRequest request) {
        this.setOrderClosedTime(request.getOperateTime());
        TradeOrderState orderState = OrderStateMachine.INSTANCE.transition(this.getOrderState(), request.getOrderEvent());
        this.setOrderState(orderState);
        this.setCloseType(request.getOrderEvent().name());
        return this;
    }

    public TradeOrder finish(OrderFinishRequest request) {
        this.setOrderFinishedTime(request.getOperateTime());
        TradeOrderState orderState = OrderStateMachine.INSTANCE.transition(this.getOrderState(), request.getOrderEvent());
        this.setOrderState(orderState);
        return this;
    }
}

```

下面这些方法都是定义在TradeOrder类中的，这就是充血模型，而不是在业务逻辑中自己set



## 事件驱动


领域事件，是DDD中比较常见的一个概念，他一般是领域内的模型发生了一些状态或行为时，向外部发送的一个通知。被定义为领域事件。



他和我们常说的MQ中的事件不一样，领域事件不是分布式的，他只能在自己的服务中传递。

他起到的最大的好处和MQ一样，就是解耦，通过事件的方式，接触领域之间的耦合，通过发布事件的方式，进行一个松耦合度的通信，而不用依赖具体的实现。

![[bcce2227-a705-4a66-9089-f37064b9e3df.png]]

这个spring发送异步事件就是一个事件



![[30ae08af-6a47-4050-8066-d0a9a5e64bb6.png]]

会有一个监听器来完成相应的操作，默认是同步的，我们可以用@Async接口让线程池异步的执行任务。

