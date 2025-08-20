在我们的项目中，为了避免订单的重复下单，我们用了token的机制，这个token是用户访问页面的时候获取到的。



然后在真正下单的时候把这个token删除掉，确保他只能用一次。



![[250fb9c3-9b63-429e-957e-e5fc0137d7bf.png]]



同时，这个token我们也会当作订单创建时候的幂等号来用，一方面他是唯一的，可以帮我们做幂等判断，另外借助它也能实现redis库存和订单的一致性核对。



所以，我们需要把这个token传递下来，但是token的传递是在header中的，在tokenFilter中校验完成以后，就被删了，那么后续的订单我们是如何拿到这个token呢？



我们当然可以在controller中解析header，从header中取，但是其实有一个更好的方案，那就是把他存储在threadlocal中，这样在当前线程中，任意时刻都可以获取到这个token来使用。



我们在tokenFilter中定义了一个tokenThreadLocal。



```java
public class TokenFilter implements Filter {

    public static final ThreadLocal<String> tokenThreadLocal = new ThreadLocal<>();
}
```



然后再做 token 校验的时候把 token 放到这个 tl 中，如以下代码的第16行。



```java
private boolean checkTokenValidity(String token) {
    String luaScript = """
            local value = redis.call('GET', KEYS[1])
            redis.call('DEL', KEYS[1])
            return value""";

    // 6.2.3以上可以直接使用GETDEL命令
    // String value = (String) redisTemplate.opsForValue().getAndDelete(token);

    String result = (String) redissonClient.getScript().eval(RScript.Mode.READ_WRITE,
            luaScript,
            RScript.ReturnType.STATUS,
            Arrays.asList(token));

    tokenThreadLocal.set(result);
    return result != null;
}
```



然后再使用的时候，及后续的过程中，如tradeController 的 buy 方法中就可以直接用这个 tl 来获取了：



```java
@PostMapping("/buy")
public Result<String> buy(@Valid @RequestBody BuyParam buyParam) {
    String userId = (String) StpUtil.getLoginId();
    //创建订单
    OrderCreateRequest orderCreateRequest = new OrderCreateRequest();
    orderCreateRequest.setIdentifier(tokenThreadLocal.get());
}
```



如上面的第6行，就是从tokenThreadLocal中获取到这个 token 当作幂等号使用。

