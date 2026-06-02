Redis主从复制的缺点：没有办法对master进行动态选举，需要使用Sentinel机制完成动态选举
# 1、哨兵模式
   Sentinel(哨兵)进程是用于监控redis集群中Master主服务器工作的状态，在Master主服务器发生故障的时候，可以实现Master和Slave服务器的切换，保证系统的高可用（HA）
# 2、Sentinel配置
```
port 26379 
dir /opt/soft/redis/data
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 30000 
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 180000
#sentinel auth-pass <master-name> <password>
#sentinel notification script <master-name> <script-ath>
#sentinel client-reconfig-script <master-name> <script-path>
```
port和dir分别代表Sentinel节点的端口和工作目录。

## 2.1 sentinel monitor
用来告诉Redis监控一个叫做mymaster的主节点，地址是 127.0.0.1 端口号是6379，并且有2个仲裁机器。所有的意思都很明显，但是除了这个quorum 参数：quorum参数用于故障发现和判定，例如uorum被设置为2，代表至少有2个Sentinel节点认为主节点不可达，那么这个不可达的判定才是客观的。对于quorum设置的越小，那么达到下线的条件越宽松，反之越严格。一般建议将其设置为Sentinel节点的一半加1。

## 2.2 sentinel down-after-milliseconds 
每个Sentinel节点都要通过定期发送 ping命令来判断 Redis数据节点和其余 Sentnel节点是否可达，如果超过了down-after-milliseconds配置的时间且没有有效的回复,则判定节点不可达，<times>(单位为毫秒)就是超时时间。这个配置是对节点失败判定的重要依据。

**优化说明**:down-after-milliseconds越大，代表 Sentinel 节点对于节点不可达的条件越宽松，反之越严格。条件宽松有可能带来的问题是节点确实不可达了，那么应用方需要等待故障转移的时间越长，也就意味着应用方故障时间可能越长。条件严格虽然可以及时发现故障完成故障转移，但是也存在一定的误判率。

> down-after-milliseconds虽然<以master-name>为参数，但实际上对
 Sentinel节点、主节点、从节点的失败判定同时有效。

## 2.3 sentinel parallel-syncs
当Sentnel节点集合对主节点障定达成一致时，Sentine领导者节点会做故障转移操作，选出新的主节点，原来的从节点会向新的主节点发起复制操作，paralle1-syncs就是用来限制在一次故障转移之后，每次向新的主节点发起复制操作的从节点个数。如果这个参数配置的比较大，那么多个从节点会向新的主节点同时发起复制操作，尽管复制操作通常不会阻塞主节点，但是同时向主节点发起复制，必然会对主节点所在的机器造成一定的网络和磁盘I0开销。

## 2.4 sentinel failover-timeout 
 failover-timeout通常被解释成故障转移超时时间，但实际上它作用于故障转移的各个阶段：
1. 选出合适从节点。
2. 晋升选出的从节点为主节点。
3. 命令其余从节点复制新的主节点。
4. 等待原主节点恢复后命令它去复制新的主节点。 

failover-timeout 的作用具体体现在四个方面:
1. 如果Redis Sentinel 对一个主节点故障转移失败，那么下次再对该主节点做故障转移的起始时间是 failover-timeout 的 2 倍。
2. 在阶段二时，如果 Sentinel节点向阶段一选出来的从节点执行slaveof no one 一直失败(例如该从节点此时出现故障)，当此过程超过failover-timeout 时，则故障转移失败。
3. 在阶段二如果执行成功，Sentinel节点还会执行info命令来确认阶段一选出来的节点确实晋升为主节点，如果此过程执行时间超过failover-timeout 时，则故障转移失败。
4. 如果阶段三执行时间超过了failover-timeout (不包含复制时间)，则故障转移失败。注意即使过了这个时间，Sentine节点也会最终配置从节点去同步最新的主节点。
## 2.5 其他
略


# 3、哨兵进程
## 3.1 作用

1. 监控(Monitoring): 哨兵(sentinel) 会不断地检查你的Master和Slave是否运作正常。
2. 提醒(Notification)： 当被监控的某个Redis节点出现问题时, 哨兵(sentinel) 可以通过 API 向管理员或者其他应用程序发送通知。
3. 自动故障迁移(Automatic failover)：当一个Master不能正常工作时，哨兵(sentinel) 会开始一次自动故障迁移操作。
* 它会将失效Master的其中一个Slave升级为新的Master, 并让失效Master的其他Slave改为复制新的Master；
* 当客户端试图连接失效的Master时，集群也会向客户端返回新Master的地址，使得集群可以使用现在的Master替换失效Master。
* Master和Slave服务器切换后，Master的redis.conf、Slave的redis.conf和sentinel.conf的配置文件的内容都会发生相应的改变，即，Master主服务器的redis.conf配置文件中会多一行slaveof的配置，sentinel.conf的监控目标会随之调换。

## 3.2 工作方式
1. 每个Sentinel进程以每秒钟一次的频率向整个集群中的Master主服务器，Slave从服务器以及其他Sentinel进程发送一个 PING 命令。
2. 如果一个实例（instance）距离最后一次有效回复 PING 命令的时间超过 down-after-milliseconds 选项所指定的值， 则这个实例会被 Sentinel进程标记为主观下线（SDOWN）。
3. 如果一个Master主服务器被标记为主观下线（SDOWN），则正在监视这个Master主服务器的所有 Sentinel（哨兵）进程要以每秒一次的频率确认Master主服务器的确进入了主观下线状态。
4. 当有足够数量的 Sentinel（哨兵）进程（大于等于配置文件指定的值）在指定的时间范围内确认Master主服务器进入了主观下线状态（SDOWN）， 则Master主服务器会被标记为客观下线（ODOWN）。
5. 在一般情况下， 每个 Sentinel（哨兵）进程会以每 10 秒一次的频率向集群中的所有Master主服务器、Slave从服务器发送 INFO 命令。
6. 当Master主服务器被 Sentinel（哨兵）进程标记为客观下线（ODOWN）时，Sentinel（哨兵）进程向下线的 Master主服务器的所有 Slave从服务器发送 INFO 命令的频率会从 10 秒一次改为每秒一次。
7. 若没有足够数量的 Sentinel（哨兵）进程同意 Master主服务器下线， Master主服务器的客观下线状态就会被移除。若 Sentinel（哨兵）进程重新向 Master主服务器发送 PING 命令返回有效回复，Master主服务器的主观下线状态就会被移除。

# 3.3 实例演示
修改从机的sentinel.conf
```
$ cd /usr/local/redis-5.0.7/
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
sentinel monitor mymaster 192.168.10.133 6379 1

#其他配置项说明
# 哨兵sentinel实例运行的端口 默认26379
port 26379

# 哨兵sentinel的工作目录
dir /tmp
 
# 哨兵sentinel监控的redis主节点的 ip port
# master-name  可以自己命名的主节点名字 只能由字母A-z、数字0-9 、这三个字符".-_"组成。
# quorum 当这些quorum个数sentinel哨兵认为master主节点失联 那么这时 客观上认为主节点失联了
# sentinel monitor <master-name> <ip> <redis-port> <quorum>
  sentinel monitor mymaster 127.0.0.1 6379 2
 
# 当在Redis实例中开启了requirepass foobared 授权密码 这样所有连接Redis实例的客户端都要提供密码
# 设置哨兵sentinel 连接主从的密码 注意必须为主从设置一样的验证密码
# sentinel auth-pass <master-name> <password>
sentinel auth-pass mymaster MySUPER--secret-0123passw0rd
 
# 指定多少毫秒之后 主节点没有应答哨兵sentinel 此时 哨兵主观上认为主节点下线 默认30秒
# sentinel down-after-milliseconds <master-name> <milliseconds>
sentinel down-after-milliseconds mymaster 30000
 
# 这个配置项指定了在发生failover主备切换时最多可以有多少个slave同时对新的master进行 同步，
这个数字越小，完成failover所需的时间就越长，
但是如果这个数字越大，就意味着越 多的slave因为replication而不可用。
可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。
# sentinel parallel-syncs <master-name> <numslaves>
sentinel parallel-syncs mymaster 1
 
# 故障转移的超时时间 failover-timeout 可以用在以下这些方面：
#1. 同一个sentinel对同一个master两次failover之间的间隔时间。
#2. 当一个slave从一个错误的master那里同步数据开始计算时间。直到slave被纠正为向正确的master那里同步数据时。
#3.当想要取消一个正在进行的failover所需要的时间。  
#4.当进行failover时，配置所有slaves指向新的master所需的最大时间。不过，即使过了这个超时，slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来了
# 默认三分钟
# sentinel failover-timeout <master-name> <milliseconds>
sentinel failover-timeout mymaster 180000
 
# SCRIPTS EXECUTION
 
#配置当某一事件发生时所需要执行的脚本，可以通过脚本来通知管理员，例如当系统运行不正常时发邮件通知相关人员。
#对于脚本的运行结果有以下规则：
#若脚本执行后返回1，那么该脚本稍后将会被再次执行，重复次数目前默认为10
#若脚本执行后返回2，或者比2更高的一个返回值，脚本将不会重复执行。
#如果脚本在执行过程中由于收到系统中断信号被终止了，则同返回值为1时的行为相同。
#一个脚本的最大执行时间为60s，如果超过这个时间，脚本将会被一个SIGKILL信号终止，之后重新执行。
 
#通知型脚本:当sentinel有任何警告级别的事件发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本，
这时这个脚本应该通过邮件，SMS等方式去通知系统管理员关于系统不正常运行的信息。调用该脚本时，将传给脚本两个参数，
一个是事件的类型，
一个是事件的描述。
如果sentinel.conf配置文件中配置了这个脚本路径，那么必须保证这个脚本存在于这个路径，并且是可执行的，否则sentinel无法正常启动成功。
#通知脚本
# sentinel notification-script <master-name> <script-path>
  sentinel notification-script mymaster /var/redis/notify.sh
 
# 客户端重新配置主节点参数脚本
# 当一个master由于failover而发生改变时，这个脚本将会被调用，通知相关的客户端关于master地址已经发生改变的信息。
# 以下参数将会在调用脚本时传给脚本:
# <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
# 目前<state>总是“failover”,
# <role>是“leader”或者“observer”中的一个。
# 参数 from-ip, from-port, to-ip, to-port是用来和旧的master和新的master(即旧的slave)通信的
# 这个脚本应该是通用的，能被多次调用，不是针对性的。
# sentinel client-reconfig-script <master-name> <script-path>
 sentinel client-reconfig-script mymaster /var/redis/reconfig.sh
```


参考：
https://www.redis.com.cn/topics/sentinel.html
