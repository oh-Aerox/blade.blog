# 1 什么是复制
[《复制》](https://www.redis.com.cn/topics/replication.html)

# 2 怎么使用
配从不配主
使用命令或修改redis.conf配置文件
> slaveof < masterip > < masterport >
5.0之后的新版版使用replicaof

saleof no one 断开复制

# 3 Redis 复制功能是如何工作的
参考[原理](https://www.redis.com.cn/topics/replication.html)中的 <Redis 复制功能是如何工作的>
![image.png](https://www.hounk.world/upload/2021/02/image-2003937ae6284a10a9d18bf57f2a1495.png)

# 4 作用与问题
**作用：**

主从模式可以将主节点的数据改变同步给从节点，这样从节点就可以起到两个作用：
* 作为主节点的一个备份，主节点出现故障不可达，从节点可以作为后备，并且保证数据尽量不丢失
* 从节点可以扩展主节点的读能力，主节点不能支撑大并发读操作，从节点可以帮助分担读压力。

简单概括就是：读写分离、容灾恢复

**问题：**
1. 主节点出现问题，需要手动将一个从节点晋升为主节点，同时需要修改应用方的主节点地址，还需要命令其他从节点去复制新的主节点，整个过程都需要人工干预。
2. 主节点的写能力受到单机的限制
3. 主节点的存储能力受到单机的限制
