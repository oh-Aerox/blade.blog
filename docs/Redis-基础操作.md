# String
二进制安全，意思就是string类型可以包含任何数据，比如jpg图片或者序列化的对象，最大能存储512MB
## 命令
### 1. 不存在设置 setnx/set key value nx
在指定的 key 不存在时，为 key 设置指定的值。
**例：**分布式锁时，多个连接发起同一个删除操作，可以使用此命令，哪一个返回成功就相当于抢到了锁，其他会返回失败
### 2. 存在时设置 set key value xx
键已经存在时， 才对键进行设置操作
### 3. 加 incrby key value
将 key 所储存的值加上给定的增量值
#### 补充object命令
通过命令set n 99保存值，通过type查看，返回string类型。那么incr为什么可以增加数值？
object命令：允许从内部察看给定 key 的 Redis 对象，有多个子命令：
```
OBJECT REFCOUNT <key> 返回给定 key 引用所储存的值的次数。此命令主要用于除错。
OBJECT ENCODING <key> 返回给定 key 锁储存的值所使用的内部表示(representation)。
OBJECT IDLETIME <key> 返回给定 key 自储存以来的空转时间(idle， 没有被读取也没有被写入)，以秒为单位。
```
如果通过encoding查看n就会返回int。这里就涉及到对象的编码方式。redis对象可以有多种编码：
```
字符串可以被编码为 raw (一般字符串)或 int (用字符串表示64位数字是为了节约空间)。
列表可以被编码为 ziplist 或 linkedlist 。 ziplist 是为节约大小较小的列表空间而作的特殊表示。
集合可以被编码为 intset 或者 hashtable 。 intset 是只储存数字的小集合的特殊表示。
哈希表可以编码为 zipmap 或者 hashtable 。 zipmap 是小哈希表的特殊表示。
有序集合可以被编码为 ziplist 或者 skiplist 格式。 ziplist 用于表示小的有序集合，而 skiplist 则用于表示任何大小的有序集合。
```
这个编码并不是一成不变的。有些命令会触发修改这个编码。先执行命令set n2 9 然后看一下编码类型 object encoding n2，现在是int类型，然后执行append n2 999，这个时候在执行object encoding n2，可以看到编码变成了raw，最后再执行incr n2，再看下编码变为int。这其中有一步更新编码的过程
命令执行成功，进行更新，方便下次操作。
### 4. Setbit KEY_NAME OFFSET
用于对 key 所储存的字符串值，设置或清除指定偏移量上的位(bit)。
offset是二进制位而非字节数组。一个字节八个二进制位，redis中说索引是面向字节的，在使用位命令时，面向二进制位也有索引，他的索引从左向右0-7就对应索引的第一个位置，8-16就是第二个位置，虽然字节是分开的，但是redis里面二进制位就是一个长串。
**例** setbit k 1
这个1是8个二进制位中的1，也就是说他把从左向右第二位变成了1，也就是01000000，可以查看下长度为1，在ASCII码中这一个字节正好是@

**使用场景**：
需求：统计一下某段时间所有用户的登录天数
关系型数据库存储：至少需要两个字段，用户ID和登录日期，每天登录新增数据。这会导致这张表的数据量非常大。
使用redis：一年365天或者366天，我们可以取400个二进制位，每个二进制位对应一天，这样一个用户一年只需要50个字节（400➗8）
比如setbit zhangsan 1 1 代表zhangsan第二天登录了
setbit zhangsan  7 1代表zhangsan第八天登录了
最后通过bitcount获取指定日期范围之内的登录天数
bitcount zhangsan -2 -1 ，-2和-1就是最后两个字节也就是最后16天的登录数量

## 长度
执行set k1 999，用strlen看下长度为1，执行incr k1，再看长度为4；
执行set k2 a，strlen看下长度为1，append k2 中，再看长度为4。
这是因为客户端的编码是UTF8，汉字在UTF8中是三个字节，如果客户端切换成GBK，客户端会先通过GBK编码变成字节数据组再发给redis，再此设置一个汉字，获取一下长度可以看到是2。
 

[其他命令](https://www.redis.net.cn/order/3544.html)

# List
按照插入顺序排序，可以添加元素到头部或尾部。
同向命令可以用作栈，反向命令是用作队列。
**比如**：
Lpush将一个或多个值插入到列表头部

同向命令，先进后出，用作栈
Lpop移出并获取列表的第一个元素

反向命令，先进先出，用作队列
Rpop移除并获取列表最后一个元素

## 命令
[参考](https://www.redis.net.cn/order/3577.html)


# Hash
string类型的field和value的映射表，适用于存储对象
## 命令
[参考](https://www.redis.net.cn/order/3564.html)

# Set
1. String类型的无序集合
2. 通过哈希表实现，添加删除查找的复杂度O(1)
3. 不允许重复
## 命令
### 1.  SRANDMEMBER KEY [count]
返回集合中一个或多个随机数
```
redis 127.0.0.1:6379> SADD myset1 "hello"
(integer) 1
redis 127.0.0.1:6379> SADD myset1 "world"
(integer) 1
redis 127.0.0.1:6379> SADD myset1 "bar"
(integer) 1
redis 127.0.0.1:6379> SRANDMEMBER myset1
"bar"
redis 127.0.0.1:6379> SRANDMEMBER myset1 2
1) "Hello"
2) "world"
```
如果 count 为正数，且小于集合基数，返回一个包含 count 个元素的数组，数组中的元素各不相同。如果 count 大于等于集合基数，那么返回整个集合。
如果 count 为负数，那么命令返回一个数组，数组中的元素可能会重复出现多次，而数组的长度为 count 的绝对值。

**场景**：不可重复抽奖count大于0，可重复抽奖count小于0

[其他命令](https://www.redis.net.cn/order/3594.html)


# Sorted set
有序集合和集合一样也是 string 类型元素的集合,且不允许重复的成员。

不同的是每个元素都会关联一个 double 类型的分数。redis 正是通过分数来为集合中的成员进行从小到大的排序。

有序集合的成员是唯一的,但分数(score)却可以重复。

集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。 

## 命令
### 1. 并集 Zunionstore 
完整语法：
```
redis 127.0.0.1:6379> ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]
```
计算给定的一个或多个有序集的并集，其中给定 key 的数量必须以 numkeys 参数指定，并将该并集(结果集)储存到 destination 。

默认情况下，结果集中某个成员的分数值是所有给定集下该成员分数值之和，
除此之外可以通过weights指定权重，max取大，或者min取小

[其他命令](https://www.redis.net.cn/order/3609.html)

场景：歌曲排行榜
