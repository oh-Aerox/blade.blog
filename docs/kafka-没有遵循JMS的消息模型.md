Java消息服务（Java Message Service，JMS）目前支持两种消息模型：点对点与发布订阅。这两种模式主要区别或解决的问题就是发送到队列的消息能否重复消费(多订阅)，但是kafka并没有遵循这个规范。首先介绍一下这两种消息模型。

# 点对点模型
基于队列
* 消息的生产者和消费者之间没有时间上的相关性
* 生产者把消息发送到队列中，可以有多个发送者，但只能被一个消费者消费。一条消息只能被一个消费者消费一次。
* 消费者无需订阅，当消费者未消费到消息时就会处于阻塞状态

# 发布订阅模型
基于主题
* 生产者和消费者之间有时间上的相关性，订阅一个主题的消费者只能消费自它订阅之后发布的消息。
* 生产者将消息发送到主题(Topic)上
* 消费者必须先订阅，JMS规范允许提供客户端创建持久订阅
> 持久化订阅者：特殊的消费者，告诉主题，我一直订阅着，即使网络断开，消息服务器也记住所有持久化订阅者，如果有新消息，也会知道必定有人回来消费。

# kafka 中 Push vs. Pull
kafka中采用 producer 把数据 push 到 broker，然后 consumer 从 broker 中 pull 数据。 

原因：
* push-based的缺点
让 broker 控制数据传输速率主要是为了让 consumer 能够以可能的最大速率消费；不幸的是，这导致着在 push-based 的系统中，当消费速率低于生产速率时，consumer 往往会不堪重负（本质上类似于拒绝服务攻击）。
当大批量数据要发送给consumer时，push-based 系统必须选择立即发送请求或者积累更多的数据，然后在不知道下游的 consumer 能否立即处理它的情况下发送这些数据。如果系统调整为低延迟状态，这就会导致一次只发送一条消息，以至于传输的数据不再被缓冲，这种方式是极度浪费的

* pull-based的优点
当 consumer 速率落后于 producer 时，可以在适当的时间赶上来。还可以通过使用某种 backoff 协议来减少这种现象：即 consumer 可以通过 backoff 表示它已经不堪重负了，然而通过获得负载情况来充分使用 consumer（但永远不超载）。
pull-based 的设计中， consumer 总是将所有可用的（或者达到配置的最大长度）消息 pull 到 log 当前位置的后面，从而使得数据能够得到最佳的处理而不会引入不必要的延迟。

## 对pull-based的改进
简单的 pull-based 系统的不足之处在于：如果 broker 中没有数据，consumer 可能会在一个紧密的循环中结束轮询，实际上 busy-waiting 直到数据到来。为了避免 busy-waiting，我们在 pull 请求中加入参数，使得 consumer 在一个“long pull”中阻塞等待，直到数据到来（还可以选择等待给定字节长度的数据来确保传输长度）。

# Kafka中消费模型的实现方式
Kafka 通过消费组实现消费的两种模式：队列模式（Queuing）和发布/订阅模式（Publish-Subscribe）。
* 只有一个消费组，而这个消费组有多个消费者，一条消息只能被这个消费组中的一个消费者所消费，类似队列模式；
* 有多个消费组，每个消费组只有一个消费者，同一条消息可被多个消费组消费，类似发布/订阅模式。

Kafka 中的 Producer 和 Consumer 采用的是 push（推送）、pull（拉取）的模式，即 Producer 只是向 Broker push 消息，Consumer 只是从 Broker pull 消息，push 和 pull 对于消息的生产和消费是异步进行的。pull 模式的一个好处是 Consumer 可自主控制消费消息的速率，同时 Consumer 还可以自己控制消费消息的方式是批量地从 Broker 拉取数据还是逐条消费数据。Kafka 通过消费组实现消费的两种模式：队列模式（Queuing）和发布/订阅模式（Publish-Subscribe）。
只有一个消费组，而这个消费组有多个消费者，一条消息只能被这个消费组中的一个消费者所消费，类似队列模式；
有多个消费组，每个消费组只有一个消费者，同一条消息可被多个消费组消费，类似发布/订阅模式。
Kafka 中的 Producer 和 Consumer 采用的是 push（推送）、pull（拉取）的模式，即 Producer 只是向 Broker push 消息，Consumer 只是从 Broker pull 消息，push 和 pull 对于消息的生产和消费是异步进行的。pull 模式的一个好处是 Consumer 可自主控制消费消息的速率，同时 Consumer 还可以自己控制消费消息的方式是批量地从 Broker 拉取数据还是逐条消费数据。

[详](https://kafka.apachecn.org/documentation.html#design_pull)

