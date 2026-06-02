节选自：[高性能MySQL实战](https://t3.lagounews.com/5RE3R9R3cF5B6)
------开始----------

MySQL InnoDB 存储引擎，实现的是基于多版本的并发控制协议——MVCC，而不是基于锁的并发控制。

MVCC 最大的好处是读不加锁，读写不冲突。在读多写少的 OLTP（On-Line Transaction Processing）应用中，读写不冲突是非常重要的，极大的提高了系统的并发性能，这也是为什么现阶段几乎所有的 RDBMS（Relational Database Management System），都支持 MVCC 的原因。 

# 快照读与当前读

在 MVCC 并发控制中，读操作可以分为两类: 快照读（Snapshot Read）与当前读 （Current Read）。
* 快照读：读取的是记录的可见版本（有可能是历史版本），不用加锁。
* 当前读：读取的是记录的最新版本，并且当前读返回的记录，都会加锁，保证其他事务不会再并发修改这条记录。 

> 注意：MVCC 只在 Read Commited 和 Repeatable Read 两种隔离级别下工作。

如何区分快照读和当前读呢？ 可以简单的理解为：

快照读：简单的 select 操作，属于快照读，不需要加锁。 

当前读：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。 

# MVCC 多版本实现

为了让大家更直观地理解 MVCC 的实现原理，这里举一个“事务对某行记录更新的过程”的案例来讲解 MVCC 中多版本的实现。

假设 F1～F6 是表中字段的名字，1～6 是其对应的数据。后面三个隐含字段分别对应该行的隐含ID、事务号和回滚指针，如下图所示。
![image.png](https://www.hounk.world/upload/2021/02/image-414b322a2b6940dc9a7fbd07f1beae3c.png)
* 隐含 ID（DB_ROW_ID），6 个字节，当由 InnoDB 自动产生聚集索引时，聚集索引包括这个 DB_ROW_ID 的值。
* 事务号（DB_TRX_ID），6 个字节，标记了最新更新这条行记录的 Transaction ID，每处理一个事务，其值自动 +1。
* 回滚指针（DB_ROLL_PT），7 个字节，指向当前记录项的 Rollback Segment 的 Undo log记录，通过这个指针才能查找之前版本的数据。

具体的更新过程，简单描述如下:
1. 首先，假如这条数据是刚 INSERT 的，可以认为 ID 为 1，其他两个字段为空。
2. 然后，当事务 1 更改该行的数据值时，会进行如下操作，如下图所示。
![image.png](https://www.hounk.world/upload/2021/02/image-6371919b34f649b2b84bb25e2c543768.png)
* 用排他锁锁定该行；记录 Redo log；
* 把该行修改前的值复制到 Undo log，即图中下面的行；
* 修改当前行的值，填写事务编号，使回滚指针指向 Undo log 中修改前的行。

接下来，与事务 1 相同，此时 Undo log 中有两行记录，并且通过回滚指针连在一起。因此，如果 Undo log 一直不删除，则会通过当前记录的回滚指针回溯到该行创建时的初始内容，所幸的是在 InnoDB 中存在 purge 线程，它会查询那些比现在最老的活动事务还早的 Undo log，并删除它们，从而保证 Undo log 文件不会无限增长，如下图所示。
![image.png](https://www.hounk.world/upload/2021/02/image-182e72f3533941c8a80ddb6157d3c11c.png)
--------------结束-----------

更多参考：[<全网最全的一篇数据库MVCC详解，不全我负责>](https://www.php.cn/mysql-tutorials-460111.html)
