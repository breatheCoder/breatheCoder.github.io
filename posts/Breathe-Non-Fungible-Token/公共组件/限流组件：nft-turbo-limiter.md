- [x] 限流组件：nft-turbo-limiter  [completion:: 2025-08-19]

![[Pasted image 20250819155728.png]]

在高并发的场景中，我们需要对单个时间内的同一类请求进行限制，以防止过多的请求在短时间内对系统造成太大压力。

这个组件中，我们封装了 Sentinel、Redis 以及Redisson等，其中用Redisson实现的限流器，这种方法利用Redis进行分布式限流，很适合高并发和分布式环境。

定义了一个通用的滑动窗口限流器：

### 限流方法逻辑解析

1.获取限流器实例：

2.初始化限流器配置:

判断限流器是否已经存在，如果不存在则进行配置。

使用RateType.OVERALL表示集群限流策略。

设置限流速率，即在windowSize秒内最多允许limit个请求。

3.尝试获取令牌：

尝试从限流器中获取令牌，如果成功则返回true，否则返回false。

### 限流实例配置

为了方便使用，我们自定义了 bean——slidingWindowRateLimiter：

### RRateLimiter 的实现原理

[[✅Redisson分布式限流器RRateLimiter原理解析]]

