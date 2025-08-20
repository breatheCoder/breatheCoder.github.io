在我们的项目中, 有用户的邀请功能, 每一次邀请别人注册, 会有一定的积分, 然后我们提供了一个排行榜功能, 可以基于这个进行排名.

排名比较简单, 就是基于积分去排序就行了, 这里面我们利用了redis的ZSET的数据结构实现快速排序.

[[✅如何实现百万级排行榜功能？]]

因为ZSET是一个天然有序的数据结构，我们可以把积分当做score，用户id当做member，放到zset中，zset会默认按照SCORE进行排序的。

```java
@DistributeLock(keyExpression = "#telephone", scene = "USER_REGISTER")  
@Transactional(rollbackFor = Exception.class)  
public UserOperatorResponse register(String telephone, String inviteCode) {  
    String defaultNickName;  
    String randomString;  
    do {  
        randomString = RandomUtil.randomString(6).toUpperCase();  
        //前缀 + 6位随机数 + 手机号后四位  
        defaultNickName = DEFAULT_NICK_NAME_PREFIX + randomString + telephone.substring(7, 11);  
    } while (nickNameExist(defaultNickName) || inviteCodeExist(randomString));  
  
    String inviterId = null;  
    if (StringUtils.isNotBlank(inviteCode)) {  
        User inviter = userMapper.findByInviteCode(inviteCode);  
        if (inviter != null) {  
            inviterId = inviter.getId().toString();  
        }  
    }  
    User user = register(telephone, defaultNickName, telephone, randomString, inviterId);  
    Assert.notNull(user, UserErrorCode.USER_OPERATE_FAILED.getCode());  
  
    addNickName(defaultNickName);  
    addInviteCode(randomString);  
    //重点是这行, 更新邀请排名
    updateInviteRank(inviterId);  
    updateUserCache(user.getId().toString(), user);  
  
    //加入流水  
    long streamResult = userOperateStreamService.insertStream(user, UserOperateTypeEnum.REGISTER);  
    Assert.notNull(streamResult, () -> new BizException(RepoErrorCode.UPDATE_FAILED));  
  
    UserOperatorResponse userOperatorResponse = new UserOperatorResponse();  
    userOperatorResponse.setSuccess(true);  
  
    return userOperatorResponse;  
}
```

updateInviteRank的代码逻辑如下:

```java
/**  
 * 更新排名，排名规则：  
 * <pre>  
 *     1、优先按照分数排，分数越大的，排名越靠前  
 *     2、分数相同，则按照上榜时间排，上榜越早的排名越靠前  
 * </pre>  
 *  
 * @param inviterId  
 */  
private void updateInviteRank(String inviterId) {  
    if (inviterId == null) {  
        return;  
    }  
    //1、这里因为是一个私有方法，无法通过注解方式实现分布式锁。  
    //2、register方法已经加了锁，这里需要二次加锁的原因是register锁的是注册人，这里锁的是邀请人  
    RLock rLock = redissonClient.getLock(inviterId);  
    rLock.lock();  
    try {  
        //获取当前用户的积分  
        Double score = inviteRank.getScore(inviterId);  
        if (score == null) {  
            score = 0.0;  
        }  
  
        //获取最近一次上榜时间  
        long currentTimeStamp = System.currentTimeMillis();  
        //把上榜时间转成小数(时间戳13位，所以除以10000000000000能转成小数)，并且倒序排列（用1减），即上榜时间越早，分数越大（时间越晚，时间戳越大，用1减一下，就反过来了）  
        double timePartScore = 1 - (double) currentTimeStamp / 10000000000000L;  
  
        //1、当前积分保留整数，即移除上一次的小数位  
        //2、当前积分加100，表示新邀请了一个用户  
        //3、加上“最近一次上榜时间的倒序小数位“作为score  
        inviteRank.add(score.intValue() + 100.0 + timePartScore, inviterId);  
    } finally {  
        rLock.unlock();  
    }  
}
```

这里主要是用到了redisson的RLock进行了加锁, 并且用的是多个lock方法, 在加锁失败的时候阻塞一直尝试. 主要就是避免多个用户被同时邀请时, 更新分数会出现并发而导致累加错误

这里面的排行榜inviteRank, 其实是:

private RScoredSortedSet<`String`> inviteRank;

在这里面初始化和实例化的, 其实他是一个RScoredSortedSet, 是一个支持排序的SET, 他提供很多方法可以方便的实现排名的功能.

- getScore: 获取指定成员的分数.
- add: 向一个有序集合中添加一个成员, 指定该成员的分数.
- rank: 获取指定成员在有序集合中的排名(从小到大排序, 排名从0开始)
- revRank: 获取指定成员在有序集合中的排名(从大到小排序, 排序从0开始)
- entryRank: 获取分数在指定范围内的成员及其分数的集合.

比如我们提供了以下几个和排名有关的方法，其实就是对上述方法的一些封装：

```java
//获取指定用户的排名, 按照分数从高到低
public Integer getInviteRank(String userId) {  
    Integer rank = inviteRank.revRank(userId);  
    if (rank != null) {  
        return rank + 1;  
    }  
    return null;  
}
```

```java
//按照分数从高到低, 获取前n个用户的排名信息
public List<InviteRankInfo> getTopN(Integer topN) {  
    Collection<ScoredEntry<String>> rankInfos = inviteRank.entryRangeReversed(0, topN - 1);  
  
    List<InviteRankInfo> inviteRankInfos = new ArrayList<>();  
  
    if (rankInfos != null) {  
        for (ScoredEntry<String> rankInfo : rankInfos) {  
            InviteRankInfo inviteRankInfo = new InviteRankInfo();  
            String userId = rankInfo.getValue();  
            if (StringUtils.isNotBlank(userId)) {  
                User user = findById(Long.valueOf(userId));  
                if (user != null) {  
                    inviteRankInfo.setNickName(user.getNickName());  
                    inviteRankInfo.setInviteCode(user.getInviteCode());  
                    inviteRankInfo.setInviteScore(rankInfo.getScore().intValue());  
                    inviteRankInfos.add(inviteRankInfo);  
                }  
            }        }    }  
    return inviteRankInfos;  
}
```

分数相同时，按照上榜时间排序

[[基于ZSET的多维度排行榜实现]]

