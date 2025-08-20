我们的项目中，在做库存扣减的时候，先基于Lua脚本和redis实现库存的预扣减的，这样可以在秒杀扣减的时候确保操作的原子性和高效性。



库存扣减部分的lua脚本如下：

```lua
String luaScript = """
        if redis.call('hexists', KEYS[2], ARGV[2]) == 1 then
            return redis.error_reply('OPERATION_ALREADY_EXECUTED')
        end
                        
        local current = redis.call('get', KEYS[1])
        if current == false then
            return redis.error_reply('KEY_NOT_FOUND')
        end
        if tonumber(current) == nil then
            return redis.error_reply('current value is not a number')
        end
        if tonumber(current) < tonumber(ARGV[1]) then
            return redis.error_reply('INVENTORY_NOT_ENOUGH')
        end
                        
        local new = tonumber(current) - tonumber(ARGV[1])
        redis.call('set', KEYS[1], tostring(new))
                        
        local time = redis.call("time")
        local currentTimeMillis = (time[1] * 1000) + math.floor(time[2] / 1000)
                        
        redis.call('hset', KEYS[2], ARGV[2], cjson.encode({
            action = "increase",
            from = current,
            to = new,
            change = ARGV[1],
            by = ARGV[2],
            timestamp = currentTimeMillis
        }))
                        
        return new
        """;
```

其实很简单，就是先做了一些前置判断，如果判断都通过的话，就扣减库存，记录流水，然后返回新的库存值。



在这个 lua 执行成功的时候，就会保证库存能够被扣减成功，并且记录了一条流水。如果运行失败，那么会根据他的返回的错误异常信息，我们对应的返回给调用方不同的错误信息。比如是库存不足、还是库存不存在等等。



## 为什么要记录流水，作用是什么？
[[redis的流水有什么用，什么情况会用到它，什么时候删？]]
