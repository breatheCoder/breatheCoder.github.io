在电商场景中，商品查询是非常高频的，而且也是非常重要的。而且对于商品信息来说，很多东西都是变化不频繁的，除了库存。

所以，在商品搜索这里，我们其实是做了缓存的。包括库存。

上面不是说库存频繁变化么？频繁变化也能走缓存吗？因为我们的库存这里是先在缓存做的预扣减，所以所有变化 redis 是都可以感知到的，所以可以走缓存查询。

在缓存的实现上，我们是基于 Redis 做分布式缓存，并且本地缓存使用 Caffeine，二者组成了二级缓存，并且使用JetCache 做二级缓存的管理。

以下是我们的商品详情的查询：

```java
@Override  
public SingleResponse<CollectionVO> queryById(Long collectionId) {  
    Collection collection = collectionService.queryById(collectionId);  
  
    InventoryRequest request = new InventoryRequest();  
    request.setGoodsId(collectionId.toString());  
    request.setGoodsType(GoodsType.COLLECTION);  
    SingleResponse<Integer> response = inventoryFacadeService.queryInventory(request);  
  
    //没查到的情况下，默认用数据库里面的库存做兜底  
    Integer inventory = collection.getSaleableInventory().intValue();  
    if (response.getSuccess()) {  
        inventory = response.getData();  
    }  
  
    CollectionVO collectionVO = CollectionConvertor.INSTANCE.mapToVo(collection);  
    collectionVO.setInventory(inventory.longValue());  
    collectionVO.setState(collection.getState(), collection.getSaleTime(), inventory.longValue());  
   
    return SingleResponse.of(collectionVO);  
}
```

这里主要是先调用collectionService查询商品信息，然后再查询Redis 中的库存信息，二者结合后返回给前端。

而collectionService的查询，我们是用了二级缓存的，把商品id 当作缓存的 key，并且在本地存储10分钟，分布式缓存中存60分钟，主要是本地内存资源有限，不太热的数据就可以让他淘汰掉。

并且我们为了提升缓存的命中了，设置了50分钟更新一次，也就意味着，Redis 中的数据是不太会过期的，因为还没等到60分钟的过期时间，就自动刷新了

并且我们为了提升缓存的命中了，设置了50分钟更新一次，也就意味着，Redis 中的数据是不太会过期的，因为还没等到60分钟的过期时间，就自动刷新了

因为商品信息其实变化根本就不太频繁，所以设置个50分钟更新一次即可。

那为啥 Redis 还要设置60分钟的过期时间呢？

其实是为了避免万一定时任务执行的时候应用重启了，导致数据没更新。

```java
@Override  
@Cached(name = ":collection:cache:id:", expire = 60, localExpire = 10, timeUnit = TimeUnit.MINUTES, cacheType = CacheType.BOTH, key = "#collectionId", cacheNullValue = true)  
@CacheRefresh(refresh = 50, timeUnit = TimeUnit.MINUTES)  
public Collection queryById(Long collectionId) {  
    return getById(collectionId);  
}
```

后续，等我们的后台功能上了之后，商品信息修改的时候，会让缓存失效。这样下次查询的时候就可以把新值更新进去了.