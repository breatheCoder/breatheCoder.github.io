针对一些热点商品，我们提供了预约功能，就像京东卖茅台一样，需要提前预约才能购买，不约不能买。

同一个商品我们需要记录哪些用户预约过，同时需要快速的查询，并且可以尽可能的减少存储空间，避免浪费，我们使用了bitset

为了快速的存取，我们使用redis，redis整好也支持bitset，同时为了避免redis宕机，我们也要存储一张预约表，在数据库中做持久化。


Redis 的 bitset 是一种特殊的字符串类型，它允许你对字符串中的每一位（bit）进行操作。每个位可以是 0 或 1，因此你可以将 bitset 视为一个非常高效的布尔数组。使用 Redis bitset 的好处

+ 节省内存
+ 每个位只占用 1 位（bit），相比于布尔数组或整数数组，bitset 可以显著减少内存占用。这对于存储大量布尔值（如用户是否预约了某个商品）非常有用。
+ 高效的操作
+ Redis 提供了丰富的命令来操作 bitset，例如 SETBIT、GETBIT、BITCOUNT 等。这些命令可以高效地设置、获取和统计位的状态。
+ 原子性操作
+ Redis 的 bitset 操作是原子性的，这意味着多个客户端可以同时对同一个 bitset 进行操作而不会产生竞争条件。这在高并发场景下非常重要。



因为我们的用户 ID 是一定不重复的，并且可以转换成integer，所以每一个用户可以映射到一个唯一的 bit 上面，这样预约过的用户对应的 bit 设置为1 ，没预约过的默认为0，就能实现存储预约信息了。



redis中的key就是商品的ID，存储一个bitset，bitset存储的就是一个预约过的用户ID列表。



具体实现如下

```java
// 因为用户id都是不重复的，并且可以转换成integer，所以这里可以使用BitSet来存储预约信息，减少存储量
RBitSet bookedUsers = redissonClient.getBitSet(BOOK_KEY + request.getGoodsType() + CacheConstant.CACHE_KEY_SEPARATOR + request.getGoodsId());
// 不报错则成功
bookedUsers.set(Integer.parseInt(request.getBuyerId()));
```



就这样，就可以把一个预约的信息保存下来了。

![[f39364b5-f160-4582-89f2-f95b52ad68bb.png]]

当29这个用户 ID 预约过之后：

![[c4618ba1-1ec9-4c62-a413-8deb4e4c94d5.png]]

想要查询某个用户是否预约过的时候，可以：

```java
RBitSet bookedUsers = redissonClient.getBitSet(BOOK_KEY + goodsType + CacheConstant.CACHE_KEY_SEPARATOR + goodsId);
return bookedUsers.get(Integer.parseInt(buyerId));
```

这样就能查看某个用户是否在 bitset 中，即是否预约过。

以上操作其实就相当于执行了命令：

```java
GETBIT "goods:book:COLLECTION:10085" "39"
```

