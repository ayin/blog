---
categories:
  - Technology
tags:
  - Java
---

# 一 遇到的问题
最近遇到了一个关于多线程下hashMap不安全的情况，对hashMap有了更深入的了解，下面先直接抛出出问题的代码示例并给出解释，然后再对hashMap做一些总结。

## 1.1 出问题的伪代码
```java
synchronized (this) {
  调用返回一个map的函数，该函数在多线程下生产一个普通的hashMap map;
  LOGGER.info(map.size());
  LOGGER.info(JSON.toJSONString(map.keySet()));
  ConcurrentSkipListMap putAll(map);
}
```
## 1.2 遇到的现象一
系统报类似如下异常
```
Exception in thread "***" java.lang.OutOfMemoryError: Java heap space
	at com.alibaba.fastjson.serializer.SerializeWriter.expandCapacity(SerializeWriter.java:204)
	at com.alibaba.fastjson.serializer.SerializeWriter.writeLong(SerializeWriter.java:583)
	at com.alibaba.fastjson.serializer.CollectionSerializer.write(CollectionSerializer.java:87)
	at com.alibaba.fastjson.serializer.JSONSerializer.write(JSONSerializer.java:369)
	at com.alibaba.fastjson.JSON.toJSONString(JSON.java:418)
	at com.alibaba.fastjson.JSON.toJSONString(JSON.java:406)
```
产生上面异常的原因是多线程环境下使用普通的hashMap产生的结果中bucket可能包含了一个循环链(后面解释原因)，导致JSON进行序列化是一个无限循环get，然后造成线程OOM。这个线程是一个jetty线程，报这个异常并没有影响到整个系统的正常运行(服务是由daemontools进行管理的，从进程的启动时间可以判断不是daemontools重启了进程，服务整体上正常)。

刚开始没有意思到map的错误，日志中map.size打印结果"看着也正常"，所以只是简单的把JSON.toJSONString(map.keySet())这行给注释掉了，于是同样的根源又导致了下面的现象
## 1.3 遇到的现象二
> jetty线程被大量block

```
"jetty-worker-79" prio=10 tid=0x00007ff160d41800 nid=0x4e93 waiting for monitor entry [0x00007ff1ac484000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.XXX(XXX.java:125)
        - waiting to lock <0x00000007433e57f8> (a com.XXXX)
        at com.XXXX(XXXX.java:129)

"jetty-worker-78" prio=10 tid=0x00007ff160d3f800 nid=0x4e92 runnable [0x00007ff1ac505000]
   java.lang.Thread.State: RUNNABLE
        at java.util.concurrent.ConcurrentSkipListMap.put(ConcurrentSkipListMap.java:1645)
        at java.util.AbstractMap.putAll(AbstractMap.java:273)
        at com.XXX(XXX.java:87)
        at com.XXX(XXX.java:128)
        - locked <0x00000007433e57f8> (a com.XXX)
```
>  jetty线程配置的是有限个数

```
<Set name="ThreadPool">
    <New class="org.eclipse.jetty.util.thread.QueuedThreadPool">
        <Set name="name">jetty-worker</Set>
        <Set name="minThreads">20</Set>
        <Set name="maxThreads">200</Set>
        <Set name="maxQueued">200</Set>
    </New>
</Set>
```
> jetty线程耗完，jetty疯狂打印如下日志(jetty 8下面，其他版本不清楚)：

```
[2016-06-12 22:01:40.469][jetty-worker-65 Selector0][WARN][org.eclipse.jetty.io.nio.SelectChannelEndPoint][225]:Dispatched Failed! SCEP@40dc2ba1{l(/10.4.32.28:56447)<->r(/10.32.70.61:8082),d=false,open=true,ishut=false,oshut=false,rb=false,wb=false,w=true,i=0}-{AsyncHttpConnection@36b4784b,g=HttpGenerator{s=0,h=-1,b=-1,c=-1},p=HttpParser{s=-14,l=0,c=0},r=0} to org.eclipse.jetty.server.nio.SelectChannelConnector$ConnectorSelectorManager@2459ffc
[2016-06-12 22:01:40.469][jetty-worker-65 Selector0][WARN][org.eclipse.jetty.io.nio.SelectChannelEndPoint][225]:Dispatched Failed! SCEP@40dc2ba1{l(/10.4.32.28:56447)<->r(/10.32.70.61:8082),d=false,open=true,ishut=false,oshut=false,rb=false,wb=false,w=true,i=1r}-{AsyncHttpConnection@36b4784b,g=HttpGenerator{s=0,h=-1,b=-1,c=-1},p=HttpParser{s=-14,l=0,c=0},r=0} to org.eclipse.jetty.server.nio.SelectChannelConnector$ConnectorSelectorManager@2459ffc
```
> jetty疯狂打印日志，系统进一步恶化，cpu也开始出现报警

因此上面流程就是：

多线程产生包含循环链接的hashMap -> putAll遇到环形链接，进入死循环 -> synchronized阻塞其他线程 -> jetty线程耗尽，无法dispatch请求，疯狂打印日志 -> 系统CPU耗尽，产生报警

关于现象二中有另外两点附近说明：
* jetty疯狂打印的dispatch failed日志可以关掉：[Jetty 8.1 flooding the log file with “Dispatched Failed” messages](http://stackoverflow.com/questions/16592537/jetty-8-1-flooding-the-log-file-with-dispatched-failed-messages)，原理参考：http://m.blog.csdn.net/article/details?id=50537293
* 由于使用的synchronized关键字，这里报的是线程block，后面的例子中使用的是CountLatchDown，报的是线程parked，效果是一样的

# 二 hashMap原理简介
关于hashMap的原理资料相当多，就不再多介绍，下面只是简单列一下：
* hashMap内部大体结构图示
![hashMap内部结构](https://jf-bucket-public.oss-cn-qingdao.aliyuncs.com/jfperiodical/attached/image/20160613/1787394254.jpg)
如上图，hashMap索引定位到的是一个bucket数组，对于hash冲突的Entry由链表链接起来

* hashMap如果不指定大小，默认bucket数组初始大小是16，然后随着put数据，自动以翻一番的速度调整bucket数组大小，这样防止每个bucket下的链表过长
* hashMap进行重新调整大小，也就是resize时原来的结构会被破坏，数据会重新进行hash


# 三 hashMap不安全的一些现象和原因

以下面一段程序进行演示
```java
public class Test {

    public static void main(String args[]) throws Exception {
        ExecutorService BUILD_SECOND_BOX_EXECUTOR = Executors.newFixedThreadPool(60);
        int i = 1000000;
        while (--i > 0) {
            final Map<Long, Long> map = Maps.newHashMap();
            final LocalDateTime time = LocalDateTime.now();
            final CountDownLatch latch = new CountDownLatch(100);
            for (int k = 0; k < 100; k++) {
                final int l = k;
                BUILD_SECOND_BOX_EXECUTOR.execute(new Runnable() {
                    public void run() {
                        try {
                            map.put(time.plusSeconds(l).withMillisOfSecond(0).toDate().getTime(), 1L);
                        } catch (Exception e) {
                            System.out.println(e.getMessage());
                        } finally {
                            latch.countDown();
                        }
                    }
                });
            }
            latch.await();
            System.out.print(map.keySet().size() + ":");  //---------------------- a
            //Maps.newHashMap().putAll(map);              //---------------------- b
            try {
                JSON.toJSONString(map.keySet());          //---------------------- c
            } catch (OutOfMemoryError e) {
                //printMap(map);
                Iterator<Long> iterator = map.keySet().iterator();
                while (iterator.hasNext()) {
                    Thread.sleep(1000);
                    System.out.println(iterator.next());
                }
                System.out.println(e.getMessage());
            }

            System.out.println(i);
        }
    }

    /**
    * 打印hashMap的结构
    */
    public static void printMap(Map<Long, Long> map) throws Exception{
        System.out.print("\n");
        Field field = map.getClass().getDeclaredField("table");
        field.setAccessible(true);
        Map.Entry<Long, Long>[] table = (Map.Entry[])field.get(map);
        int j = 0;
        Set<Long> set = Sets.newHashSet();
        for(int i = 0;i<table.length;i++){
            System.out.print(String.format("%3s :", i));
            Map.Entry<Long, Long> e = table[i];
            while (e != null){
                if(set.contains(e.getKey())){
                    System.out.print(String.format("(%s)here infinite", e.getKey()));
                    break;
                }
                j++;
                Long key = e.getKey();
                set.add(key);
                System.out.print(key + "  ->  ");
                Field nextField = e.getClass().getDeclaredField("next");
                nextField.setAccessible(true);
                e = (Map.Entry<Long, Long>)nextField.get(e);
            }
            System.out.println();
        }
        System.out.println("realSize:" + j);
    }
}
```

## 3.1 为什么size不是100？
运行上面的程序，a行处基本得不到100
```java
transient int size;
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```
普通的size, 普通的自增，没有任何线程安全措施，因此如果两个线程同时取到了size值，同时进行加操作，这样总的结果就漏掉了一些加操作，因此size值只会小于等于期望值，而不会出现大于的情况。

## 3.2 hashMap put进入无限循环
执行上面程序，会有一定概率输出下面内容后再也不输出任何东西，这个时候程序进入了死循环，循环点是hashMap的put地方
```
97:999969
81:999968
89:999967
90:999966
90:999965
91:999964
91:999963
88:999962

```
下面链接中给出了非常详细的解释，强烈建议仔细读一遍：[http://mailinator.blogspot.hk/2009/06/beautiful-race-condition.html](http://mailinator.blogspot.hk/2009/06/beautiful-race-condition.html)
下面的图是从上面链接中解释的最后的结果：
[hashMap put无限循环](http://3.bp.blogspot.com/_wdOYAcPCMJE/Si5skScRmnI/AAAAAAAAB4s/k7djpnVbcqU/s400/thread1-4.gif)
核心内容是：
* 两个线程thread1和thread2同时进行resize，resize需要对原有的数据进行rehash，链表结构会被打破
* thread1 的entry和next指针如图指向了A, B然后暂停，thread2开始执行
* thread2执行完毕，会把链表进行反转，此时B指向了A
* thread1继续执行，这个时候就会使得A,B进入环形的链表形状，put进入死循环


## 3.3 hashMap get进入无限循环
执行上面程序，会有一定概率输出下面内容，说明程序出现了OOM异常，出现异常的根本原因就是和本文开始描述的现象一一样，程序进入了hashMap引起的无限循环中
```
86:999978
89:999977
93:999976
85:1465914173000
1465914112000
......
1465914178000
1465914172000
1465914156000
1465914153000
1465914131000
1465914153000
1465914131000
1465914153000
1465914131000
```
如果打印出OOM的异常如下：
```
96:999916
91:999915
92:999914
89:999913
97:999912
92:999911
91:999910
90:Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at com.alibaba.fastjson.serializer.SerializeWriter.expandCapacity(SerializeWriter.java:204)
	at com.alibaba.fastjson.serializer.SerializeWriter.writeLong(SerializeWriter.java:583)
	at com.alibaba.fastjson.serializer.CollectionSerializer.write(CollectionSerializer.java:87)
	at com.alibaba.fastjson.serializer.JSONSerializer.write(JSONSerializer.java:369)
	at com.alibaba.fastjson.JSON.toJSONString(JSON.java:418)
	at com.alibaba.fastjson.JSON.toJSONString(JSON.java:406)
```
hashMap有幸逃脱了put的死循环，但是又一不小心落入了get的死循环。同样用链接中[http://mailinator.blogspot.hk/2009/06/beautiful-race-condition.html](http://mailinator.blogspot.hk/2009/06/beautiful-race-condition.html)进行举例子。

* thread1 的entry指针如图指向了A, next指向了null, 然后暂停，thread2开始执行
* thread2执行完毕，会把链表进行反转，此时B指向了A
* thread1继续执行，这个时候就会使得A,B进入环形的链表形状,但是put不会死循环，外部使用这个map时get到这个地方会进入死循环

如果想观察这个现象，可以用上面的printMap,打印出hashMap各个链表的结构

另外，在死循环发生时如果dump出java的内存，通过MAT进行分析，可以找到类似下图中的结果：
![MAT中显示的hashMap死循环](/Users/Desktop/1.png)

## 3.4 无论put还是get死循环，为什么hashMap或者ketSet的size能够一直获取？
```java
private final class KeySet extends AbstractSet<K> {
    public Iterator<K> iterator() {
        return newKeyIterator();
    }
    public int size() { 
        return size;
    }
    public boolean contains(Object o) {
        return containsKey(o);
    }
    public boolean remove(Object o) {
        return HashMap.this.removeEntryForKey(o) != null;
    }
    public void clear() {
        HashMap.this.clear();
    }
}
```
如上代码中所示，size的取值不会遍历entry数组和链表结构

## 3.5 hashMap的序列化，反序列化，为什么代码实现中用transient关键字？

* transient 是表明该数据不参与序列化。因为 HashMap 中的存储数据的数组数据成员中，数组还有很多的空间没有被使用，没有被使用到的空间被序列化没有意义。所以需要手动使用 writeObject() 方法，只序列化实际存储元素的数组。
* 由于不同的虚拟机对于相同 hashCode 产生的 Code 值可能是不一样的，如果你使用默认的序列化，那么反序列化后，元素的位置和之前的是保持一致的，可是由于 hashCode 的值不一样了，那么定位函数 indexOf（）返回的元素下标就会不同，这样不是我们所想要的结果。


## 3.6 如果使用hashMap并且能预先知道大小尽量传入大小参数
hashMap默认初始值是16，如果事先设置好大小，能避免动态的扩大和结构调整，提高效率
## 3.7 hashTable 和 concurrenthashmap
这两个都是线程安全的，不过hashTable是比较笨的版本，在get，put时都通过synchronized进行保护了，而concurrenthashmap比较聪明，主要在entry数组结构发生变化时进行同步保护
## 3.8 尽量使用简单的不可变对象作为key值
如果作为key的对象发生变化，hash计算结果不一样，hashMap中不能感知到key的变化的

# 四 java8中的HashMap
参考：[Java HashMap原理探究](http://www.jointforce.com/jfperiodical/article/2037?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)

java8中hashmap的实现主要变化的是数组中存储的不是entry了，而是node，Node就像Entry一样，存储了相同的信息，所以仍然是一个链表，不过Node可以继承自treeNode,TreeNode是一个红黑树，通过继承，内部表可以同时包含Node（链表）和TreeNode（红黑树）。Oracle决定按照下面的规则使用两个数据结构：
1. 对于内部表中给定索引位置的桶超过8个节点，则链表转换成红黑树
2. 对于内部表中给定索引位置的桶低于6个节点，则树会被转换链表
带来的主要好处是，如果key的hash值分配不均匀，仍能保证链表这一部分查询O(lg(n))的效率
