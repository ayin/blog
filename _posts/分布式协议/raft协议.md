# 角色

* Leader, 类似Master
* Follower，类似Slave
* Candidate，候选者，leader选举过程中的中间状态，Leader和Follower在新一轮选举时首先变为Candidate状态，选举结束后转换为Leader状态或者Follower状态

# Term

* 类似于其他系统中的epoch等
* 每一轮选举都是一个Term，每一个Term内只能选出一个Leader，可以有Term选不出Leader
* 当Leader或者Candidate发现自己的Term比别的Follower小时会转为Follower，Term递增
* 当Follower的Term比别的Term小时Follower也将更新Term保持与其他Follower一致


# Leader选举

* 开始时所有服务都为Follower，选举由定时器触发，每个服务的定时器都不一样，一般在150ms~300ms，超时发送Request Vote Message给其他服务，包含本身的一票
* 收到过半的回复的服务器变身为Leader
* Leader可以开始给其他Follower发送Log Append Entry信息，随着heartbeat 心跳包一块发送
* 该过程中某个Follower心跳包超时的话会变身为Candidate，重新选举Leader
* 如果两个Candidate得到了相同的Votes，会等待超时进入下一轮的选举

Election Timeout: Follower超时变身为Candidate的时间
Heartbeat Timeout：Follower和Leader保持连通的时间

# Log Repulation

* 事务请求都经由Leader
* 发送Log用的Append Entries Message和HeartBeat是同样的Message
* Leader把数据同步给其他Follower，不需要等待所有Follower都返回，只需要大多数返回就行，就代表Commit成功，给客户端返回Reply，同时在下个HearBeat内通知所有的Follower进行Commit

# 安全性

* 选举安全性
* Leader完整性：保证Leader日志的完整性，根据Term和和Log Index判断是否投票还是拒绝？类似于Zookeeper?

# Raft VS ZK
FROM sb
* Raft:目前只是follower在检测，如果在election timeout内未收到leader的心跳，则转换为candidate，并自增term发起新一轮的投票，leader遇到新的term自动转为follower状态
* Zookeeper:leader和follower都会进行检测，follower是检测是否收到leader心跳，leader是检测过半follower未返回心跳回复。如果leader检测失败，则进入election状态，其他follower因收不到leader心跳也会进入election状态，从而开启新一轮的选举。如果follower检测失败，而leader和其他follower正常，则该leader会去学习leader，而不触发新一轮选举。
* raft可能遇到投票冲突，会在下一个term内重新进行一轮投票
* zookeeper会在一个epoch内进行多次投票，有利于找出包含更多更新日志的server，理论上可能出现活锁


# Raft的优化
参考 [TiKV 源码解析系列 - Raft 的优化](http://mp.weixin.qq.com/s/MgyUE3pG2SkyZT0m3CjcXQ)
* Batch and Pipeline：Leader和Follower之间一般是稳定的，在Follower返回响应之前就把下一个request发送给Follower
* Append Log Rarallelly：Leader本身进行log append的同时把log发送给Follower，如果log被大多数Follower成功append了，即使本Leader本身的log appender失败了，只要满足了N/1+1个Follower成功，仍然可以认为这个log是被Committed成功了
* Asynchronous Apply，这样整个流程变为了如下：
>1. Leader 接受一个 client 发送的 request。
>2. Leader 将对应的 log 发送给其他 follower 并本地 append。
>3. Leader 继续接受其他 client 的 requests，持续进行步骤 2。
>4. Leader 发现 log 已经被 committed，在另一个线程 apply。
>5. Leader 异步 apply log 之后，返回结果给对应的 client

# 参考

* [Raft](http://thesecretlivesofdata.com/raft/)

raft与ZAB区别
http://m635674608.iteye.com/blog/2337085