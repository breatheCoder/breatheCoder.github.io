我们的数藏业务，每一个藏品都有固定的数量，那么需要在藏品购买成功后，根据支付顺序，生成对应的藏品编号，从1开始，依次累加上去。

那么，如何在支付成功这一刻，针对同一个商品，能实现高并发的编号生成呢？即不重复，也不丢呢？

我们项目中采用的是依赖 Redis 的自增 id 实现，因为 redis是单线程的，所以可以保证高并发情况下的线程安全，每一个支付成功处理时，去 redis 中获取一下自增 id，就作为本商品的编号即可。

在HeldCollectionService中，有两个方法，分别是 create 和 batchCreate，其中都包含了编号生成的逻辑：

```java
HeldCollection heldCollection = new HeldCollection();
Long serialNo = 
redissonClient.getAtomicLong(HELD_COLLECTION_BIND_BOX_PREFIX + request.getGoodsType() + CacheConstant.CACHE_KEY_SEPARATOR + request.getSerialNoBaseId()).incrementAndGet();heldCollection.init(request, serialNo.toString());
```

这里，就借助了 Redisson 的getAtomicLong来获取一个原子的 Long，然后调用他的incrementAndGet方法，就能先增加1，然后再返回这个值。

至于 AtomicLong，如何让不同的藏品的 ID 互相不干扰呢，那就要传入不同的参数，我们的实现是传入：

```java
HELD_COLLECTION_BIND_BOX_PREFIX + request.getGoodsType() + CacheConstant.CACHE_KEY_SEPARATOR + request.getSerialNoBaseId()
```

其中HELD_COLLECTION_BIND_BOX_PREFIX是一个固定的前缀：`HC:SALES:`
然后拼上GoodsType。如 COLLECTION、或者 BLIND_BOX，让藏品和盲盒区分开。
然后拼上一个间隔符，即`:`
然后，拼上一个SerialNoBaseId，这是个啥呢，其实就是你要根据啥生成编号，如果是同一个藏品，那么这里就是藏品 ID，如果是同一个盲盒，那么这里就是盲盒 id。

最终得到的 key 就是这样的：

![[Pasted image 20250816181154.png]]

然后其中存的值就是一个数字，其实代表的就是这个商品的销量。

为啥这个销量不用数据库来实现的，因为数据库的库存是在下单的时候扣减的，支付的时候获取到的不准了。我们之前有一套 try-confirm-cancel 的方案来扣减库存的，那个方案单独冗余了一个 occupied_inventory，这个倒是可以用来做这个编号，但是因为这个方案后来被我们废弃了，主要是他有性能和并发问题，所以用 redis 实现是一个简单且有效的方案。

另外，这个 key 不需要提前初始化，第一次用的时候，如果没有，默认会初始化成0，0进行 increase 之后返回的就是1了。