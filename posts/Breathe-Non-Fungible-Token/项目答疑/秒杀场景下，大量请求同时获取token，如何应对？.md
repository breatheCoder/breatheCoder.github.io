[[基于Token校验避免订单重复提交]]


上面介绍了token 防重复扣减的方案

但是大家有没有考虑过，如果秒杀场景，用户进入详情页就会获取一次token，那么会不会有大量的请求去获取token，如何防的住呢？



其实，这个我们可以做个优化，你比如淘宝，他并不是在详情页的时候就获取token，而是在订单确认页做的token获取。



订单确认页就是这个页面：



![[4549fc98-2d86-444a-bd50-77d057fba998.jpeg]]



在这个页面，用户需要确认收货地址、快递方式、优惠等等信息。



那么我们可以在这个页面唤起的时候获取token，那么久可以过滤掉很多之访问详情页，但是不下单的用户的token获取的请求，能大大降低这个数据量。



另外还有一个方案，那就是我们通过限制，让每个用户针对每一个商品只能生成一个token，来避免他对同一个商品提前获取token来下单，形成恶意请求。



![[393aa8c4-0112-42f3-ba5e-6be9a371a60d.svg]]



因为tokenKey是基于scene、userId和key拼接而成的，所以他和用户+商品强相关，同一个key向redis进行set的时候会覆盖，这样就能确保，如果有新的Token生成了， 老的就会失效了。这样就保证了，同一个用户在同一商品下，同时只能有一个有效的token。



TokenController的代码如下：

```java
@GetMapping("/get")
public Result<String> get(@NotBlank String scene, @NotBlank String key) {
    if (StpUtil.isLogin()) {
        String userId = (String) StpUtil.getLoginId();
        //token:buy:29:10085
        String tokenKey = TOKEN_PREFIX + scene + CACHE_KEY_SEPARATOR + userId + CACHE_KEY_SEPARATOR + key;
        String tokenValue = TokenUtil.getTokenValueByKey(tokenKey);
        //key：token:buy:29:10085
        //value：YZdkYfQ8fy7biSTsS5oZrbsB8eN7dHPgtCV0dw/36AHSfDQzWOj+ULNEcMluHvep/txjP+BqVRH3JlprS8tWrQ==
        stringRedisTemplate.opsForValue().set(tokenKey, tokenValue, 30, TimeUnit.MINUTES);
        return Result.success(tokenValue);
    }
    throw new AuthException(AuthErrorCode.USER_NOT_LOGIN);
}
```



TokenUtil的代码如下：

```java
import cn.hutool.crypto.SecureUtil;

import java.nio.charset.StandardCharsets;
import java.util.UUID;

import static cn.hollis.nft.turbo.cache.constant.CacheConstant.CACHE_KEY_SEPARATOR;

public class TokenUtil {

    private static final String TOEKN_AES_KEY = "xxx保密，详见代码";

    public static final String TOKEN_PREFIX = "token:";

    public static String getTokenValueByKey(String tokenKey) {
        if (tokenKey == null) {
            return null;
        }
        String uuid = UUID.randomUUID().toString();
        //token:buy:29:10085:5ac6542b-64b1-4d41-91b9-e6c55849bb7f
        String tokenValue = tokenKey + CACHE_KEY_SEPARATOR + uuid;

        //YZdkYfQ8fy7biSTsS5oZrbsB8eN7dHPgtCV0dw/36AHSfDQzWOj+ULNEcMluHvep/txjP+BqVRH3JlprS8tWrQ==
        return SecureUtil.aes(TOEKN_AES_KEY.getBytes(StandardCharsets.UTF_8)).encryptBase64(tokenValue);
    }
}
```

这里面针对tokenValue进行了加密，防止用户找到规律进行拼接。



需要注意的是AES的key需要是16位的，否则加解密会报错。

