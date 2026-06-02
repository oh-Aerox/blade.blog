## 相关命令
### MULTI
用于标记事务块的开始。Redis会将后续的命令逐个放入队列中，然后使用EXEC命令原子化地执行这个命令序列。

### EXEC
在一个事务中执行所有先前放入队列的命令，然后恢复正常的连接状态

### DISCARD
清除所有先前在一个事务中放入队列的命令，然后恢复正常的连接状态。

### WATCH
监视一个(或多个) key ，如果在事务执行之前这个(或这些) key 被其他命令所改动，那么事务将被打断。
语法：watch key [key…]

### UNWATCH
清除所有先前为一个事务监控的键。

## 特征
1. Redis的事务是通过MULTI，EXEC，DISCARD和WATCH这四个命令来完成的。
2. Redis的单个命令都是原子性的，所以这里确保事务性的对象是命令集合。
3. Redis将命令集合序列化并确保处于同一事务的命令集合连续且不被打断的执行。如果在一个事务中的命令出现错误，那么所有的命令都不会执行；如果在一个事务中出现运行错误，那么正确的命令会被执行。
4. Redis不支持回滚操作，在事务失败时不进行回滚，而是继续执行余下的命令”。

**redis事务带有以下三个重要的保证：**
* 批量操作在发送 EXEC 命令前被放入队列缓存。
* 收到 EXEC 命令后进入事务执行，事务中任意命令执行失败，其余的命令依然被执行。
* 在事务执行过程，其他客户端提交的命令请求不会插入到事务执行命令序列中。

> 注：如果多个客户端开启事务，那么谁的Exec命令先到达，则先执行谁的命令集
```shell script
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set a aaa
QUEUED
127.0.0.1:6379> set b bbb
QUEUED
127.0.0.1:6379> set c ccc
QUEUED
127.0.0.1:6379> exec
1) OK
2) OK
3) OK
127.0.0.1:6379> get a
"aaa"
```
如果命令错误，该操作集是全都不执行
```shell script
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set aa aaaa
QUEUED
127.0.0.1:6379> set bb
(error) ERR wrong number of arguments for 'set' command
127.0.0.1:6379> set bb ssss
QUEUED
127.0.0.1:6379> sef cc cccc
(error) ERR unknown command 'sef'
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
127.0.0.1:6379> get aa
(nil)
```

### 为什么redis不支持事务回滚
大多数事务失败是因为语法错误或者类型错误，这两种错误，在开发阶段都是可以预见的，redis为了性能方面就忽略了事务回滚。核心的一句话就是，使用redis是因为速度快，作者在开发这个功能的时候就是按照这个思路来的。

