在我们的项目中，经常会需要查询用户信息，比如前端页面需要展示，用户下单需要查询，交易成功后上链时也需要查询。

而这些用户信息，很多内容都是不变的，比如用户的昵称、手机号、链地址，或者说是变化不频发的。所以这些数据我们就可以做缓存。

在缓存选择上，我们不仅用了分布式缓存，同时为了提升性能，还做了本地缓存。也就是说我们的一次查询，会先查询本地缓存，如果本地缓存查不到，再查询分布式缓存。并且在分布式缓存中查询到之后保存到本地缓存中一份。

![[Pasted image 20250816150322.png]]

代码逻辑如下：

```java
public String query(String key) {
	String localResult = localCache.get(key);
	if (localResult == null) {
		String remoteResult = remoteCache.get(key);
		if (remoteResult != null) {
			localCache.put(remoteResult);
		}
	}
	return localResult;
}
```

但是为了方便，减少这段代码，我们选择用通用的二级缓存框架，阿里开源的 JetCache。

JetCache是一个基于java的缓存系统封装，提供统一的API和注解简化缓存的使用。 JetCache提供了比SpringCache更强大的注解，可以原生的支持TTL、两级缓存、分布式自动
刷新，提供了Cache接口用于手工缓存操作。

接入后，我们在我们的用户查询接口中增加注解：

@Cached
@CacheRefresh
@CacheInvalidate

逐一介绍一下：
- @Cached：为一个方法添加缓存，创建对应的缓存实例，注解可以添加在接口或者类的方法上面，该类必须是spring bean 
	- name：指定缓存实例名称，如果没有指定，会根据类名+方法名自动生成。
	- expire：超时时间。如果注解上没有定义，会使用全局配置，如果此时全局配置也没有定义，则为无穷大。
	- cacheType：缓存的类型，支持：REMOTE、LOCAL、BOTH，如果定义为BOTH，会使用LOCAL和REMOTE组合成两级缓存。
	- key：使用SpEL指定缓存key，如果没有指定会根据入参自动生成。
	- cacheNullValue：当方法返回值为null的时候是否要缓存。
- @CacheRefresh：用于标识这个缓存需要自动刷新
	- refresh：刷新的时间间隔
	- timeUnit：时间单位
- CacheInvalidate：用于标识这个方法被调用时需要移除缓存
	- name：指定缓存的唯一名称，一般指向对应的@Cached定义的name。
	- key：使用SpEL指定key，如果没有指定会根据入参自动生成。

除此之外，JetCache 还提供了@CreateCache、@CacheUpdate等常用注解，我们这里并没有使用。我们基于缓存的自动更新+失效，来保证缓存的一致性。