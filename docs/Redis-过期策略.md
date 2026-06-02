# 1. 设置过期时间的四种方式
```
# 将 key 的过期时间设置为 ttl 秒
expire <key> <ttl> 
# 将 key 的过期时间设置为 ttl 毫秒
pexpire <key> <ttl>
# 将 key 的过期时间设置为 timestamp 指定的秒数时间戳
expire <key> <timestamp>
# 将 key 的过期时间设置为 timestamp 指定的毫秒数时间戳
pexpire <key> <timestamp>
```
# 2. 过期删除策略

## 2.1定期删除

定期策略是每隔一段时间执行一次删除过期键的操作，并通过限制删除操作执行的时长和频率来减少删除操作对CPU 时间的影响，同时也减少了内存浪费

Redis 默认会每秒进行 10 次（redis.conf 中通过 hz 配置）过期扫描，扫描并不是遍历过期字典中的所有键，而是采用了如下方法：
> 从过期字典中随机取出 20 个键
删除这 20 个键中过期的键
如果过期键的比例超过 25% ，重复步骤 1 和 2
为了保证扫描不会出现循环过度，导致线程卡死现象，还增加了扫描时间的上限，默认是 25 毫秒（即默认在慢模式下，如果是快模式，扫描上限是 1 毫秒）

因此，如果只采用定期删除策略，会导致很多key到时间没有删除。于是，惰性删除派上用场。
## 2.2 惰性删除
也就是说在你获取某个key的时候，redis会检查一下，这个key如果设置了过期时间，那么是否过期了？如果过期了此时就会删除。

## 2.3 定时删除
用一个定时器来负责监视key,过期则自动删除。虽然内存及时释放，但是十分消耗CPU资源。在大并发请求下，CPU要将时间应用在处理请求，而不是删除key,因此没有采用这一策略。

## 2.4 redis采用的定期删除+惰性删除策略
Redis 服务器采用惰性删除和定期删除这两种策略配合来实现，这样可以平衡使用 CPU 时间和避免内存浪费

**采用定期删除+惰性删除就没其他问题了么?**
不是的，如果定期删除没删除key。然后你也没及时去请求key，也就是说惰性删除也没生效。这样redis的内存会越来越高。那么就应该采用内存淘汰机制。

# 3.淘汰策略
在redis.conf中有一行配置 
**maxmemory-policy**
该配置就是配内存淘汰策略的，可以六选一
* **volatile-lru** -> remove the key with an expire set using an LRU algorithm 在设置了过期时间的键空间中，移除最近最少使用的Key
* **allkeys-lru** -> remove any key according to the LRU algorithm 在整个键空间中，移除最近最少使用的key
* **volatile-random** -> remove a random key with an expire set 在设置了过期时间的键中随机移除key
* **allkeys-random** -> remove a random key, any key 在整个键空间中随机移除某个Key
* **volatile-ttl** -> remove the key with the nearest expire time (minor TTL) 在设置了过期时间的键空间中，有更早过期时间的key优先移除
* **noeviction** -> don't expire at all, just return an error on write operations 如果都不配置那就抛出错误

ps：如果没有设置 expire 的key, 不满足先决条件(prerequisites); 那么 volatile-lru, volatile-random 和 volatile-ttl 策略的行为, 和 noeviction(不删除) 基本上一致。

参考：
https://zhuanlan.zhihu.com/p/86531660
