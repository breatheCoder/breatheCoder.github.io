- [x] like导致索引失效性能优化  [completion:: 2025-08-16]

本次优化的commit记录:

[阿里云登录页](https://codeup.aliyun.com/66263f57d833774c93cc0418/NFTurbo/NFTurbo_Server/commit/ea24377ff1653273d62646fe929e07c6200d4f99?branch=master)

在我们的项目中, 有定时任务分片扫表, 为了让不同的机器扫描到的数据不重复所以我们采用以下方式来扫表: 通过在xxl-job配置的分片数量和分片序号来取模获取锁扫描的表的后缀, 

```java
int shardIndex = XxlJobHelper.getShardIndex();  
int shardTotal = XxlJobHelper.getShardTotal();  
  
LOG.info("orderTimeOutExecute start to execute , shardIndex is {} , shardTotal is {}", shardIndex, shardTotal);  
  
List<String> buyerIdTailNumberList = new ArrayList<>();  
for (int i = 0; i <= MAX_TAIL_NUMBER; i++) {  
    if (i % shardTotal == shardIndex) {  
        buyerIdTailNumberList.add(StringUtils.leftPad(String.valueOf(i), 2, "0"));  
    }  
}  
  
buyerIdTailNumberList.forEach(buyerIdTailNumber -> {  
    try {  
        List<TradeOrder> tradeOrders = orderReadService.pageQueryTimeoutOrders(PAGE_SIZE, buyerIdTailNumber, null);  
        //其实这里用put更好一点，可以避免因为队列满了而导致异常而提前结束。  
        orderTimeoutBlockingQueue.addAll(tradeOrders);  
        forkJoinPool.execute(this::executeTimeout);  
  
        while (CollectionUtils.isNotEmpty(tradeOrders)) {  
            long maxId = tradeOrders.stream().mapToLong(TradeOrder::getId).max().orElse(Long.MAX_VALUE);  
            tradeOrders = orderReadService.pageQueryTimeoutOrders(PAGE_SIZE, buyerIdTailNumber, maxId + 1);  
            orderTimeoutBlockingQueue.addAll(tradeOrders);  
        }  
    } finally {  
        orderTimeoutBlockingQueue.add(POISON);  
        LOG.debug("POISON added to blocking queue ，buyerIdTailNumber is {}", buyerIdTailNumber);  
    }  
});
```

这里的查询方式如下:

```java
public List<TradeOrder> pageQueryTimeoutOrders(int pageSize, @Nullable String buyerIdTailNumber, Long minId) {  
    QueryWrapper<TradeOrder> wrapper = new QueryWrapper<>();  
    wrapper.in("order_state", TradeOrderState.CONFIRM.name(), TradeOrderState.CREATE.name());  
    wrapper.lt("gmt_create", DateUtils.addMinutes(new Date(), -TradeOrder.DEFAULT_TIME_OUT_MINUTES));  
    if (buyerIdTailNumber != null) {  
        wrapper.likeRight("reverse_buyer_id", buyerIdTailNumber);  
    }  
    if (minId != null) {  
        wrapper.ge("id", minId);  
    }  
    wrapper.orderBy(true, true, "gmt_create");  
    wrapper.last("limit " + pageSize);  
  
    return this.list(wrapper);  
}
```

其实就是如下SQL:

```sql
select * from trade_order where gmt_create < "2024-09-07 00:00:00" and order_state = "CREATE" and buyer_id like "%12";
```

这里相当于用用户的尾号分片了, 每台机器只扫描不同尾号的用户, 这样就能避免重复.

> 为啥要用尾号呢？因为尾号更均匀，如果用用户 iD的前两位， 会不均匀

但是这个SQL存在一个问题, 就是他无法走buyer_id的索引, 因为不符合(最左优先匹配原则), 最多只能用到gmt_create和order_state的索引, 而gmt_create还是一个范围查询, 效率并不高.

当表中数据量很大的时候, 这个SQL就是一个慢SQL, 超过1s很正常, 甚至可能会达到几秒钟甚至10秒钟都有可能.

那么, 这就是一个典型的模糊查询的优化方案, 我们是怎么优化的呢?

总结下来就是一句话: 把用户id冗余在订单表中, 然后按照这个逆序的用户ID查询.

```sql
ALTER TABLE `trade_order` 
ADD COLUMN `reverse_buyer_id` varchar(32) NULL COMMENT '逆序的买家ID' AFTER `buyer_id`, 
ADD KEY `idx_rvbuyer_state`(`reverse_buyer_id`,`order_state`,`gmt_create`) USING BTREE;
```

增加一个reverse_buyer_id的字段, 表示用户ID的逆序, 如果用户ID是1234, 那么这个字段存储的就是4321.

同时创建一个联合索引, 把(`reverse_buyer_id`, `order_state`, `gmt_create`)放到一起作为联合索引.

> 历史数据初始化: update trade_order set (`reverse_buyer_id` = REVERSE(`buyer_id`));

这样就可以用索引了:

select * from `trade_order_0003` where gmt_create < "2024-09-07 00:00:00" and order_state = "CREATE" and `reverse_buyer_id` like "12%";

explain 如下:

![[Pasted image 20250816100250.png]]

把likeLeft改成了likeRight, 这样就符合最左前缀匹配了, 就可以更好的用这个联合索引.