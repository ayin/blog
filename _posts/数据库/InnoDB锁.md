* Lock VS Latch

* 行锁
  * 共享锁 S
  * 排他锁 X

* 表锁
  * 普通表锁：整个表
  * 意向表锁：表中的几行，包括：意向共享锁（IS）和意向排他锁（IX）

下表代表四种锁直接的兼容关系，Y表示兼容，N表示不兼容：

|   |IS |IX | S | X |
|---|---|---|---|---|
|IS | Y | Y | Y | N |
|IX | Y | Y | N | N |
| S | Y | N | Y | N |
| X | N | N | N | N |

* 二阶段加锁协议：事务中只加锁不释放，事务结束一起释放
>思考：为什么不直接一阶段？事务开始时无法知道需要加锁的数据

* AUTO-INC锁：___不遵守两阶段加锁协议___，为了提高并发效率，在语句执行结束后事务完成前就会释放，如果事务失败，可能会出现自增ID不连续的情况。新版本的提供有轻量级锁(Mutex)机制
  * Simple inserts：通过分析insert语句可以确定插入数量的insert语句
  * Bulk inserts：通过分析insert语句无法知道插入数量的insert语句
  * Mixed-mode inserts：不确定是否需要分配auto_increment id

* 一致性非锁定读(快照读)：利用undo页实现的MVCC机制。
  * 对于Read Committed，读取的总是最新的快照
  * 对于Read Repeatable，读取的总是事务开始时的快照
* 一致性锁定读(当前读)：SELECT ... FOR UPDATE(X锁);SELECT ... LOCK IN SHARE MODE(S锁); SERIALIZABLE隔离级别下的读；插入/更新/删除操作


# 锁的算法

* 行锁
  * Record Lock
  * Gap Lock：间隙锁，锁定一个范围，但不包含记录本身
  * Next Key Lock: Record Lock + Gap Lock 锁定一个范围（一条记录和它之前的间隙），并锁定记录本身。当查询的索引含有唯一属性时可以优化为Record Lock
> Next Key Lock如何防止 ___幻读___ ？

# 锁问题

* 脏读
* 不可重复读


* 丢失更新：第一类丢失更新(抹去)；第二类丢失更新(覆盖)