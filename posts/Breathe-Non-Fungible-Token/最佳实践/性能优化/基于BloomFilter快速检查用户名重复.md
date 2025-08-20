在我们的项目中, 用户名是不能重复的, 因为在数藏项目中, 用户名的唯一性很重要, 因为藏品要做溯源, 要确保整个链路上的参与者的唯一性, 虽然用户ID也是我唯一的, 但是页面上展示的时候不能用ID啊

有些网站, 比如淘宝网, 也是要求用户名唯一的.

所以为了实现这个用户在注册，改名的时候的用户名的唯一性，我们一般是先从数据库查询是否存在，不存在则让用户注册。

但是为了考虑到性能，我们会把用户名存储到缓存中，一般使用 redis 缓存。

那既然都用了缓存了，那还不如干脆直接就用布隆过滤器来存储，既可以做重复校验，又能节省空间。所以，我们在用户名重复检验这里就用到了布隆过滤器。

布隆过滤器是一种数据结构, 用于快速检索一个元素是否可能存在于一个bit数组中.

它的基本原理是利用多个哈希函数，将一个元素映射成多个位，然后将这些位设置为 1。当查询一个元素时，如果这些位都被设置为 1，则认为元素**可能**存在于集合中，否则**肯定**不存在。

所以，布隆过滤器可以准确的判断一个元素是否一定不存在，但是因为哈希冲突的存在，所以他没办法判断一个元素一定存在。只能判断可能存在。

所以，我们定义了两个方法，nickNameExist用于判断是否存在，addNickName用于向布隆过滤器中添加已注册的用户名。

```java
public boolean nickNameExist(String nickName) {  
    //如果布隆过滤器中存在，再进行数据库二次判断  
    if (this.nickNameBloomFilter != null && this.nickNameBloomFilter.contains(nickName)) {  
        return userMapper.findByNickname(nickName) != null;  
    }  
  
    return false;  
}
```

这里nickNameExist方法，先从布隆过滤器中查询，如果如果查到了，再去数据库中查了一下，为啥呢？

因为布隆过滤器有误判的，存在一定的误判率，他会把不存在的用户判断为存在，所以当检查结果是存在的时候，需要再次判断一次。

因为用户注册的时候，大多数情况下都是不重复的，所以我们可以快速的用布隆过滤器进行不存在的判断。

这里的bloomFilter是这样被初始化出来的：

```java
/**  
 * 用户名布隆过滤器  
 */  
private RBloomFilter<String> nickNameBloomFilter;  
  
/**  
 * 邀请码布隆过滤器  
 */  
private RBloomFilter<String> inviteCodeBloomFilter;

@Override  
public void afterPropertiesSet() throws Exception {  
    this.nickNameBloomFilter = redissonClient.getBloomFilter("nickName");  
    if (nickNameBloomFilter != null && !nickNameBloomFilter.isExists()) {  
        this.nickNameBloomFilter.tryInit(100000L, 0.01);  
    }  
  
    this.inviteCodeBloomFilter = redissonClient.getBloomFilter("inviteCode");  
    if (inviteCodeBloomFilter != null && !inviteCodeBloomFilter.isExists()) {  
        this.inviteCodeBloomFilter.tryInit(100000L, 0.01);  
    }  
  
    this.inviteRank = redissonClient.getScoredSortedSet("inviteRank");  
}
```

我们设置了10000000的容量，误判率是0.01。

然后在修改用户名这里，用这样的方式进行调用的。

![[Pasted image 20250816180719.png]]

(图中用黄色线框起来的)

## 如果用户名修改了怎么办

如果原来用户名是Hollis，被放到布隆过滤器了，但是后面我改成 Hollis666了，那么意味着 Hollis已经没有了，那么如何从布隆过滤器删除呢？

很遗憾，不支持！(但是可以用布谷鸟过滤器)

那怎么办呢？

其实问题也不大，因为布隆过滤器本身就存在误判率，但我们检查布隆过滤器发现存在的时候，还是会去数据库再确认一遍的。

只要我们定期的重建一下布隆过滤器就行了。重建就是都删了，然后重新构建。