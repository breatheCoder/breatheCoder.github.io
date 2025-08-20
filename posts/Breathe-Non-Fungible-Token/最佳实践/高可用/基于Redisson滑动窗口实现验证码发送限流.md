- [x] 基于Redisson滑动窗口实现验证码发送限流  [completion:: 2025-08-17]

短信发送在我们的项目中，主要是登录和注册的时候会做，虽然我们在前端做了控制，短信发送后按钮会置灰，但是后端接口还是要做防控的。

因为有可能被灰黑产抓包后拿到我们的发送接口，不断的调用。比如一些呼死你的软件，就是到处抓取这种短信接口来实现批量发骚扰短信的目的。如果我们的短信接口没有做好避免重复发送和限流的话，一方面可能会出现资金浪费（因为发短信是需要花钱的），还可能会导致因为投诉而被封。

以下是我们项目中生成并发送验证码的方法：

```java
/**  
 * 生成并发送短信验证码  
 *  
 * @param telephone  
 * @return  
 */  
@Facade  
@Override  
public NoticeResponse generateAndSendSmsCaptcha(String telephone) {  
    Boolean access = slidingWindowRateLimiter.tryAcquire(telephone, 1, 60);  
  
    if (!access) {  
        throw new SystemException(SEND_NOTICE_DUPLICATED);  
    }  
  
    // 生成验证码  
    String captcha = RandomUtil.randomNumbers(4);  
  
    // 验证码存入Redis  
    redisTemplate.opsForValue().set(CAPTCHA_KEY_PREFIX + telephone, captcha, 5, TimeUnit.MINUTES);  
  
    Notice notice = noticeService.saveCaptcha(telephone, captcha);  
  
    Thread.ofVirtual().start(() -> {  
        SmsSendResponse result = smsService.sendMsg(notice.getTargetAddress(), notice.getNoticeContent());  
        if (result.getSuccess()) {  
            notice.setState(NoticeState.SUCCESS);  
            notice.setSendSuccessTime(new Date());  
            noticeService.updateById(notice);  
        } else {  
            notice.setState(NoticeState.FAILED);  
            notice.addExtendInfo("executeResult", JSON.toJSONString(result));  
            noticeService.updateById(notice);  
        }  
    });  
  
    return new NoticeResponse.Builder().setSuccess(true).build();  
}
```

这里，调用我们的SlidingWindowRateLimiter 进行了一个限流，限流的 key 是手机号，时间是限制的1分钟内只能发送1条。

[[限流组件：nft-turbo-limiter]]

通过这样的方式，就能保证，同一个手机号，一分钟内只可以发送一次验证码。并且我们这里会把验证码放到 Redis 中存5分钟，让存储时长更长一点。时间太短的话，可能因为网络延迟用户收到短信的时候就过期了，另外也能给用户多点时间，万一他临时接了个电话啥的，也不至于很快就过期了。

但是，在我们的项目中用到的其实是 Redisson 的 RRateLimiter，但是他底层其实是令牌桶的实现，但是为啥我们说他是滑动窗口的限流方式呢？

其实，主要是因为我们用的限流器有一个特殊的前提条件，那就是在一定的时间范围内，我们允许的请求量为1，当请求量为1的时候，RRateLimiter的实现其实就是一个滑动窗口。

[[✅Redisson分布式限流器RRateLimiter原理解析]]

