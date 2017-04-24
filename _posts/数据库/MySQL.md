* 存储引擎是基于表的，而不是数据库
* InnoDB -> OLTP, MyISAM -> OLAP， Memory -> 临时表
* Thread
  * Master Thread: 主循环，后台循环，刷新循环，暂停循环
  * IO Thread
  * Purge Thread
  * Page Cleaner Thread
* 缓冲池实例

* InsertBuffer
  * InsertBuffer：优化 ___非唯一___ 的 ___辅助索引___ 的离散写入
  * InsertBuffer在物理存储上也是B+树，所有的表共用存储空间，用BitMap页追踪所有的辅助索引页是否有内容在InsertBuffer中
  * ChangeBuffer: InsertBuffer的升级，InsertBuffer, DeleteBuffer, PurgeBuffer
  * SSD
* DoubleWrite
* 自适应哈希索引(AHI): 优化B+树，直接利用hash结构来提高性能
* 异步IO: IO异步，IO Merge
* 刷新临接页：对于固态硬盘可以关闭该特性

* CheckPoint技术
  * Sharp Checkpoint
  * Fuzzy Checkpoint: Master Thread; FLUSH_LRU_LIST; Async/Sync Flush; Dirty Page too much

# 文件
* 参数文件
* 日志文件：错误日志文件，二进制日志文件(缓冲，与事务关系，sync_binlog=1立即同步磁盘，ROW,STATEMENT,MIXED)，慢查询，查询日志文件
* socket文件
* pid文件
* MySQL表结构文件,.frm
* 存储引擎文件
  * 表空间文件：共享表空间，表单独的表空间文件(数据，索引，插入缓冲的BITMAP)
  * 重做日志文件: 按组划分，每个组内至少2个重做日志文件，循环写入。重做日志缓冲，512字节，不需要Double-write。三种时机:只写缓冲等待master线程调用；fsync;不fsync

# 表
* 段：数据段，索引段，rollback段
* 区，页

# 索引
* B+树 VS B树；B+树的分裂
* 聚集索引，辅助索引
* Cardinality
* 联合索引，覆盖索引，Multi-Range Read优化（MRR优化)
* 全文检索

# 锁
* InnoDB中的锁，行锁，意向锁？锁的算法：行，间隙，next_key_lock
* 二阶段加锁协议
* 一致性非锁定读(快照读)，一致性锁定读(当前读)
* MVCC？
  * http://blog.csdn.net/chen77716/article/details/6742128
  * 

# 事务
* 事务的隔离性:不同隔离级别的效果
* Redo：RedoLog Buffer，RedoLog File, fsync的设置, 与BinLog的区别
* LSN：重做日志的总量，RedoLogFile的量(CheckPoint)，页的LSN
* Undo: UndoLog存放于共享表空间的Undo段(可调整为独立表空间)，逻辑日志，写undolog的过程也需要记入redoLog, MVCC的实现，UndoLog链表和PurgeThread(离散读取)
  * insert的undoLog可以在事务之后直接删除
  * update undo log
* PurgeThread:真正的delete被update会被“延迟”执行， undo history list
* group commit, binlog与redolog的两阶段事务

# 复制备份
* 备份相关
  * 冷备，热备，温备
  * mysqldump --single-transaction 与  --lock-tables互斥， --where, source
  * select into outfile xx from table VS insert into table select xx from
* 复制：binlog, 异步实时, 3个线程，主从延迟， point-in-time，延时复制，多线程区分库

# 其他
* 占CPU过高，show processeslist，慢查询
* explain格式
```
+----+-------------+---------------+-------+----------------+---------+---------+-----+------+-------------+
| id | select_type | table         | type  | possible_keys  | key     | key_len | ref | rows | Extra       |
+----+-------------+---------------+-------+----------------+---------+---------+-----+------+-------------+
```
* 大字段拆成子表
* innodb与MyISAM区别
* InnoDB引擎的行锁是通过加在索引值实现的，因为innodb表数据是索引组织表形式存放?
* datetime和timestamp的区别？
* char与varchar区别
* int(5), float(5,3)中数字的含义
* 数据库中间件？
* 范式的概念？


上述更新前建立undo log，根据各种策略读取时非阻塞就是MVCC，undo log中的行就是MVCC中的多版本，这个可能与我们所理解的MVCC有较大的出入，一般我们认为MVCC有下面几个特点：
每行数据都存在一个版本，每次数据更新时都更新该版本
修改时Copy出当前版本随意修改，个事务之间无干扰
保存时比较版本号，如果成功（commit），则覆盖原记录；失败则放弃copy（rollback）
就是每行都有版本号，保存时根据版本号决定是否成功，听起来含有乐观锁的味道。。。，而Innodb的实现方式是：
事务以排他锁的形式修改原始数据
把修改前的数据存放于undo log，通过回滚指针与主数据关联
修改成功（commit）啥都不做，失败则恢复undo log中的数据（rollback）
二者最本质的区别是，当修改数据时是否要排他锁定，如果锁定了还算不算是MVCC？ 

Innodb的实现真算不上MVCC，因为并没有实现核心的多版本共存，undo log中的内容只是串行化的结果，记录了多个事务的过程，不属于多版本共存。但理想的MVCC是难以实现的，当事务仅修改一行记录使用理想的MVCC模式是没有问题的，可以通过比较版本号进行回滚；但当事务影响到多行数据时，理想的MVCC据无能为力了