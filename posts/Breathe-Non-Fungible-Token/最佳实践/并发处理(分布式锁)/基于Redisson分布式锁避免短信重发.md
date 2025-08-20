- [x] 基于Redisson分布式锁避免短信重发  [completion:: 2025-08-16]
短信发送在我们的项目中，主要是登录和注册的时候会做，虽然我们在前端做了控制，短信发送后按钮会置灰，但是后端接口还是要做防控的。

一方面是避免用户在页面快速的重复点击。因为发短信是需要花钱的，所以需要做好并发控制。类似的接口还有像实名认证等其他付费接口。

还有就是短信服务作为一个底层的公共服务，理论上（虽然实际上并不一定能发生）是可能被多个场景同时调用的，比如注册，登录，实名，包括后续的修改密码等功能，以及一些营销的场景。

以下是我们的短信服务：

```java
@Slf4j  
@Setter  
public class SmsServiceImpl implements SmsService{  
  
    private static Logger logger = LoggerFactory.getLogger(SmsServiceImpl.class);  
  
    private String host;  
  
    private String path;  
  
    private String appcode;  
  
    private String smsSignId;  
  
    private String templateId;  
  
    @DistributeLock(scene = "SEND_SMS", keyExpression = "#phoneNumber")  
    @Override  
    public SmsSendResponse sendMsg(String phoneNumber, String code) {  
  
        SmsSendResponse smsSendResponse = new SmsSendResponse();  
  
        String method = "POST";  
        Map<String, String> headers = Maps.newHashMapWithExpectedSize(1);  
        //最后在header中的格式(中间是英文空格)为Authorization:APPCODE 83359fd73fe94948385f570e3c139105  
        headers.put("Authorization", "APPCODE " + appcode);  
        Map<String, String> querys = Maps.newHashMapWithExpectedSize(4);  
        querys.put("mobile", phoneNumber);  
        querys.put("param", "**code**:" + code + ",**minute**:5");  
  
        //smsSignId（短信前缀）和templateId（短信模板），可登录国阳云控制台自助申请。参考文档：http://help.guoyangyun.com/Problem/Qm.html  
  
        querys.put("smsSignId", smsSignId);  
        querys.put("templateId", templateId);  
        Map<String, String> bodys = Maps.newHashMapWithExpectedSize(2);  
  
        try {  
            ResponseEntity response = RestClientUtils.doPost(host, path, headers, querys, bodys);  
            if (response.getStatusCode().is2xxSuccessful()) {  
                smsSendResponse.setSuccess(true);  
            }  
        } catch (Exception e) {  
            logger.error("sendMsg error", e);  
            smsSendResponse.setSuccess(false);  
            smsSendResponse.setResponseCode(SYSTEM_ERROR.name());  
            smsSendResponse.setResponseMessage(StringUtils.substring(e.toString(), 0, 1000));  
        }  
        return smsSendResponse;  
    }  
}
```

这里用到了我们定义的分布式锁组件，用了@DistributeLock注解来实现分布式锁的添加。

[[通用分布式锁注解实现]]

这里的 key 我们选择的是`phoneNumber`也就是手机号，这样就可以避免同一个手机号在同一时刻被发送多次。

当然，如果想要做更加细粒度的控制，比如注册和登录等多个场景之间互相不影响的话，也可以在 key 中拼接上具体的场景。

有人会问，我们不是做了限流了么，为啥还需要加锁？

[[基于Redisson滑动窗口实现验证码发送限流]]

这个前面其实提到了，就是短信作为一个基础服务，会有多个调用方，每个调用方自己可能会做限流，但是到公共的短信服务这里可能还是会发生并发的。

其实，这里之所以要防控的这个严格，主要两个原因：

1. 短信的发送是需要付费的
2. 短信发送需要给用户做防打扰，如果有发多了，会导致用户投诉（尤其是营销类、通知类短信），一旦投诉到工信部，封号非常严重，即使是有理由发短息也没用，直接封。