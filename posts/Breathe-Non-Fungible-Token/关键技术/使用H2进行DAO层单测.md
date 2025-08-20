在我们的业务逻辑代码中，很重要的一部分是数据库增删改查的逻辑，那么如果对这部分的代码进行单元测试呢？



如果直连数据库的话，不仅操作比较慢，还会造成大量的脏数据，而且数据库已有的一些历史数据也可能会影响单元测试的结果，那么有什么好的办法吗？



那就是可以通过内存数据库来帮助我们解决上面的问题，内存数据库主要依靠内存存储数据的数据库管理系统，在单元测试时使用内存数据库就可以在单测运行时初始化数据库及表结构及数据，在运行结束后直接回收掉。特别的方便。



**H2 数据库**是一个比较好用的内存数据库，它可以随着程序启动去创建数据表和数据，随着程序关闭而销毁，非常方便，可以作为单元测试数据库的替代品；



我们项目中就使用的这个，只需要配置一个sql和一个测试源就可以了。



如我们的OrderManageServiceTest，虽然是 Servie 的单测，但是不会写到我们真正的MySQL数据库中，而是从 H2中进行 CRUD。



```java
public class OrderManageServiceTest extends OrderBaseTest {
    @Autowired
    OrderManageService orderService;

    @Autowired
    OrderMapper orderMapper;

    @Autowired
    OrderReadService orderReadService;

    @MockBean
    public CollectionFacadeService collectionFacadeService;

    @Before
    public void init() {
        CollectionSaleResponse response = new CollectionSaleResponse();
        response.setSuccess(true);
        when(collectionFacadeService.trySale(any())).thenReturn(response);
    }

    @Test
    public void create() {
        OrderCreateRequest orderCreateRequest = orderCreateRequest();

        String orderId = orderService.create(orderCreateRequest).getOrderId();
        Assert.assertNotNull(orderId);
    }

    @Test
    public void createAndQuery() {
        OrderCreateRequest orderCreateRequest = orderCreateRequest();

        String orderId = orderService.create(orderCreateRequest).getOrderId();
        System.out.println(orderId);

        QueryWrapper<TradeOrder> queryWrapper = new QueryWrapper();
        queryWrapper.eq("order_id", orderId);
        TradeOrder tradeOrder = orderService.getOne(queryWrapper);
        Assert.assertNotNull(tradeOrder);

        queryWrapper = new QueryWrapper();
        queryWrapper.eq("buyer_id", orderCreateRequest.getBuyerId());
        tradeOrder = orderService.getOne(queryWrapper);
        Assert.assertNotNull(tradeOrder);

        queryWrapper = new QueryWrapper();
        queryWrapper.eq("seller_id", orderCreateRequest.getSellerId());
        tradeOrder = orderService.getOne(queryWrapper);
        Assert.assertNotNull(tradeOrder);
    }

}
```

