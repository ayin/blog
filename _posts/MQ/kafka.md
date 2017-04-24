# 作用

参考：[Top 10 Uses For A Message Queue](https://www.iron.io/top-10-uses-for-message-queue/)等
1. 解耦
2. 平滑
3. 异步
4. 冗余
5. 伸缩

# 结构

## 整体

* Broker + ZK
* Producer push
* Consumer pull

思考：push vs pull
* push模式很难适应消费速率不同的消费者
* pull多次回放数据

## 名词

* broker : 集群中的服务器
* topic : 区分不同业务数据
* consumer group : 对consumer分组，用于单播和多播。同一个consumer group中一个消息只被一个consumer消费，不同consumer group可以同时消费相同的消息
* partition : 同一个topic内分成多个partition分布到不同的broker上进行并行处理；提供有序性支持,如果需要整个topic有序，则设一个partition
* replica : topic可以设置副本，以partition维度区分主从，存储到不同的broker上，提高broker宕机时的可靠性
  * leader : leader处理读写，follower从leader同步数据
  * isr : replica中能够跟上leader的实例称为isr(in sync replica)

## 特点

* producer
  * 支持同步复制和异步复制
  * push模式
  * 批量发送


# 存储

* partition对应磁盘上的文件夹
* 每个partition划分为多个segment，每个文件夹下包括多个segment，每个segment由日志文件(.log)和索引文件(.index)组成, log文件和index文件一一对应，均以offset命名。log文件由log entry(len+magic+crc+payload)组成；索引文件表明中是msg offset->physical addr
* 每个message加入partition中是顺序写磁盘，效率比较高
* 删除旧数据有两种策略：基于时间；基于partition文件大小
* 消息无状态，不需要标记是否已消费，consumer控制消费消息的offset，不需要锁，提高了吞吐率；修改offset即可以重新消费


# partition

* kafka.producer.Partitioner接口根据消息的key(可能为null)和划分的partition数量来计算映射到哪一个partition上
* 数据倾斜问题：80%的用户只使用20%的功能 , 20%的用户贡献了80%的访问量，数据也类似
* key为null时partition的选择
  * 使用legacy Scala Producer API时，如果key为null，则随机计算一个partition index，然后缓存起来，一段时间(默认10mins)内就使用这个partition。为了减少服务端socket连接数量？
  * 使用new Java Producer API时，如果key为null，使用轮询策略
* partition与consumer对应关系：一个consumer实例只会消费某一个或多个特定partition的数据，而某个partition的数据只会被某一个特定的consumer实例所消费
  * 优势：consumer不用跟大量broker通信，降低通信开销，降低分配难道，实现简单
  * 劣势：无法让同一个consumer group中的consumer均匀消费


//todo:
```
几个简单说明：
consumer thread nums > partition nums 时，一部分 consumer thread 永远不会消费到 msg 
consumer thread nums < partition nums 时，将 partition 均分为 consumer thread nums 份，每个 consumer thread 得到一份
疑问：consumer thread 正在处理某个 partition 时，如何转去处理 另一个 partition（补充详细操作）
Note：consumer、broker 的变化，会触发 reblance，重新绑定 partition 与 consumer 之间的消费关系。

```

# replication&leader election

* 方便进行failover
* leader会track "in sync"的node list,如果follower宕机或者落后太多，leader会将其从isr列表中删除
* 一个消息只有被所有follower复制后才被认为commit，但是consumer可以配置是否等待消息commit
* follower从leader进行批量复制数据
* 支持同步复制和异步复制
* leader选举没有采用“majority vote”方式,利用了ISR队列
  * majority vote, 优点：系统的latency只取决于最快的几台Server； 缺点：为了容忍n个失败，需要2n+1个replica
* 如果partition所有的replica都挂了，就无法保证数据不丢失了，有两种方案，kafka目前选择第二种，应该可以配置
  * 等待ISR中的任一个replica“活”过来，并且选它作为leader
  * 选择第一个“活”过来的replica（不一定是ISR中的）作为leader
* 尽量均匀分布leader partition，以避免broker故障时触发大量的leader election,kafka选出一个broker作为controller，监控所有的broker，从而能够在出现故障时批量处理leader

# broker

* zerocopy : sendfile不经过用户空间

# consumer

* zerocopy : sendfile不经过用户空间
* pull模式
* broker无状态，consumer自己保持offset
* high level API VS low level API
  * High Level API消息的offset存在于ZK中(也支持存于Kafka专用的topic中)，而Low Level API消息的offset由consumer自己控制
* 数据重复消费问题
```
Consumer 重复消费 msg，一般 2 种原因：
producer 向 Kafka 集群重复发送数据：
Producer 设置同步发送，可能重复发送数据（补充详细场景）
Producer 设置异步发送，可能丢失数据，也可能重复发送数据
consumer 从 Kafka 读取数据，并且 offset 由 consumer 控制：
consumer 消费完 msg 之后，未将 offset 更新至 zookeeper，然后 consumer 宕机
重启之后，consumer 会重复消费 msg
Note：向 zookeeper 提交 offset 的时间间隔：auto.commit.interval.ms，默认，60 000
补充几点知识：
Producer 设置同步发送、异步发送：
参数：producer.type
Kafka 0.8.2 及以下版本：默认 sync，Producer 同步发送 msg
Kafka 0.9.0+ 版本：不再使用 producer.type 参数，换作其他参数，并且默认 异步发送
producer.type 设置为 async 时，Producer 异步发送 msg，即，在本地合并小的 msg，批量发送，提升系统的整体有效吞吐量
Producer 触发Broker 同步复制、异步复制：
参数：request.required.acks，默认 0，即，不要求 broker 进行 ack
其余取值：1，要求 leader 返回 ack；-1，要求所有的 follower 完成复制之后，再返回 ack；
```
* consumer rebalance过程
  * herd effect: 任何broker或者consumer的增减都会触发所有的consumer的rebalance
  * Split Brain: 
  * Coordinator: 0.9.x


* 消息投递，kafka默认是保证At Least Once
  * At Most Once
  * At Least Once
  * Exactly Once


# 参考
1. [Kafka深度解析](http://www.jasongj.com/2015/01/02/Kafka%E6%B7%B1%E5%BA%A6%E8%A7%A3%E6%9E%90/)

问题
msg offset这些metadata保存的位置?
一般情况下partition的数量大于等于broker的数量？
nagle?
producer 和consumer和broker的网络连接情况？

主从之间的复制都是如何进行的？redis, zk, kafka, innodb等

为什么操作磁盘有的是线性的，有的是随机的？kafka提前分配了磁盘空间？kafka写磁盘时是否都调用了fsync?