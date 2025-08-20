我们都知道，ThreadLocal 是无法在多个线程之间传递参数的，而InheritableThreadLocal 倒是可以在父子线程之间传递参数，但是如果有线程池的情况下，InheritableThreadLocal也无能为力。

所以当我们需要在存在线程池的场景中使用ThreadLocal 进行参数的传递的时候，就需要借助阿里开源的TTL：
https://github.com/alibaba/transmittable-thread-local

在我们的项目中也用到了TTL，场景是这样的：

有一个 MockPayChannelServiceImpl，用于在不对接第三方支付的时候，直接把支付渠道给他 mock 掉。

在他实现的接口PayChannelService中定义了两个方案，分别是 pay 方法和 notify 方法，一个是调外部渠道发起支付的，一个是接收外部渠道的回调消息的

比如微信支付，支付宝支付，我们都是先调他的支付方法，然后再等他的回调通知。

但是，我们的 MockPay 服务的话，是没有人帮我们做回调的，我们就需要自己模拟这个回调。

于是我们就在 pay 方法中做了延迟任务，三秒钟之后调用自己的notify方法。

```java
@Override  
public PayChannelResponse pay(PayChannelRequest payChannelRequest) {  
    PayChannelResponse payChannelResponse = new PayChannelResponse();  
    payChannelResponse.setSuccess(true);  
    payChannelResponse.setPayUrl("http://www.nfturbo.com");  
    Map<String, Serializable> params = new HashMap<>(12);  
    params.put("payOrderId", payChannelRequest.getOrderId());  
    params.put("paidAmount", payChannelRequest.getAmount());  
    context.set(params);  
  
    //异步线程延迟3秒钟之后调用 notify 方法  
    scheduler.schedule(() -> {  
        this.notify(null, null);  
    }, 3, TimeUnit.SECONDS);  
  
    return payChannelResponse;  
}
```

但是，这里面scheduler的定义是一个线程池：

而我们的回调方法需要知道支付的一些参数，要不然他没办法做回调的，比如我要知道支付单号吧，要不然我回调的时候怎么知道哪笔成功了呢？这就需要在 pay和 notify 之间做参数传递。

而我们的 notify 定义的时候，入参是HttpServletRequest，想要把它构造出来那可太麻烦了，于是我就想了个办法，用 TTL 来传递参数。

TransmittableThreadLocal<`Map`> context = new TransmittableThreadLocal<>();

TTL 的TransmittableThreadLocal定义如上，我们在 pay 方法中把重要参数放到 TTL 中：

然后再 notify方法中把他们取出来：

```java
Map<String, Serializable> params = (Map<String, Serializable>) context.get();
```

这样就实现了在线程池中的共享参数的传递。