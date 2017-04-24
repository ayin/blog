
Queue包括BlockingQueue中常用的方法，其中后面的阻塞和超时退出是BlockingQueue提供的
```
|-------------|-----------|----------|----------|-------------------|
| 方法\处理方式 |	 抛出异常  | 返回特殊值 | 一直阻塞  |     超时退出        |
|   插入方法   |   add(e)  | offer(e) |  put(e)  | offer(e,time,unit)|
|   移除方法   |  remove() | poll()   |  take()  | poll(time,unit)   |
|   检查方法   | element() | peek()   |  不可用   |    不可用          |
|-------------|-----------|----------|----------|-------------------|
```

BlockingQueue不接受NULL值

# ArrayBlockingQueue

* 单锁双条件
* 可支持公平和非公平可重入锁

# LinkedBlockingQueue

* 双锁双条件


>双锁的好处：put和take操作避免相互影响，一定意义上看，降低了操作的锁粒度，提高了并发性

使用双锁时要避免死锁，LinkedBlockingQueue中添加fullyLock和fullyUnlock来避免死锁：
```java
    /**
     * Locks to prevent both puts and takes.
     */
    void fullyLock() {
        putLock.lock();
        takeLock.lock();
    }

    /**
     * Unlocks to allow both puts and takes.
     */
    void fullyUnlock() {
        takeLock.unlock();
        putLock.unlock();
    }
```

# DelayQueue
* 单锁单条件

# SynchronousQueue




* HashMap
  * 实现结构：Bucket数组+Entry链表
  * 不安全的原因：size普通的自增；resize导致循环链表；get,put死循环
  * 使用transient：无用的bucket空间不需要序列号；不同机器产生的hashCode可能不一致
  * hashMap默认初始值是16，如果事先设置好大小，能避免动态的扩大和结构调整，提高效率
  * 使用简单的key，hashMap不会感知复杂key内部的变化
  * HashTable线程安全，在get，put时都通过synchronized进行保护了，效率比较低
  * Collections.synchronizedMap的效率类似HashTable，使用sychronized(mutex)
  * JAVA8中的entry变为node,node继承自TreeNode，桶超过8个节点，则链表转换成红黑树，桶低于6个节点，则树会被转换链表。因此Hash值分配不均匀时也能有很高的查询效率
  * 强一致性
  * HashMap的key和value都可以为null, HashTable的key和value都不可以为null
* ConcurrentHashMap
  * JAVA7版本
    * segment + table + bucket + entry
    * 用 HashEntery 对象的不变性来降低读操作对加锁的需求：HashEntry 中的 key，hash，next 都声明为 final 型, value是volatile类型的
    * put操作在链表头部添加，clear操作直接清空桶，remove通过clone来处理要变动的节点，三个操作都不影响原链表结构，因此都不影响并发读
    * get方法什么情况下需要锁？为什么不能put一个null值？key可以为null, value不可以为null
    * size操作通过两遍统计判断各segment的和是否变化，如果变化则加锁重新统计，否则就返回。判断是否变化使用modCount
    * 对hashCode会重新计算hash，目的是减少冲突
    * 迭代器弱一致性
  * JAVA8版本