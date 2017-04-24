---
categories:
  - Technology
tags:
  - Java
---

本文首先简单介绍ThreadLocal的用处，然后主要分析了下ThreadLocal的实现，下文所有的代码均以Java8为参考。

# 基本介绍

__作用__: 线程作用域内的全局变量

```
                   多线程不安全
全局变量(共享资源) ---------------> 线程作用域内
```

* 典型场景：webapp线程上下文中的user信息，request信息等
* __线程池__ 中使用时注意清理掉ThreadLocal变量
* initialValue只在get时如果之前没有进行过set才会被触发
* 通常声明为static但不是必须


# 实现
ThreadLocal底层实现用的是一个 __ThreadLocalMap__ ，每个线程有唯一的ThreadLocalMap，在Thread.java中：
```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```
ThreadLocal的get和set操作都最终会调用ThreadLocalMap的get和set操作

## ThreadLocalMap

### Entry
ThreadLocalMap的 __Entry结构是继承WeakReference实现的__
```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
### Get操作
```java
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }

        private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }
```
* ThreadLocalMap的Hash __冲突采用开放地址法中的线性探测再散列__ ，因此get操作需要有getEntryAfterMiss进行查找
* expungeStaleEntry用于判断WeakReference的key为nul的话去清理value值, expungeStaleEntry的代码如下

```java
        /**
         * Expunge a stale entry by rehashing any possibly colliding entries
         * lying between staleSlot and the next null slot.  This also expunges
         * any other stale entries encountered before the trailing null.  See
         * Knuth, Section 6.4
         *
         * @param staleSlot index of slot known to have null key
         * @return the index of the next null slot after staleSlot
         * (all between staleSlot and this slot will have been checked
         * for expunging).
         */
        private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;

            // expunge entry at staleSlot
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            // Rehash until we encounter null
            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;

                        // Unlike Knuth 6.4 Algorithm R, we must scan until
                        // null because multiple entries could have been stale.
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            return i;
        }
```
正如注释所说，这个方法主要清理无用的Entry，同时需要进行Rehash，也会清理其他遇到的无用的Entry

### Set操作
```java
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
整体思路：在冲突范围内(下一个null之前)如果找到了合适的位置(相同的key或者staleEntry)就替换掉，否则就放到后一个位置。这里注意两个调用replaceStaleEntry和cleanSomeSlots

```java
        /**
         * Replace a stale entry encountered during a set operation
         * with an entry for the specified key.  The value passed in
         * the value parameter is stored in the entry, whether or not
         * an entry already exists for the specified key.
         *
         * As a side effect, this method expunges all stale entries in the
         * "run" containing the stale entry.  (A run is a sequence of entries
         * between two null slots.)
         *
         * @param  key the key
         * @param  value the value to be associated with key
         * @param  staleSlot index of the first stale entry encountered while
         *         searching for key.
         */
        private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            Entry e;

            // Back up to check for prior stale entry in current run.
            // We clean out whole runs at a time to avoid continual
            // incremental rehashing due to garbage collector freeing
            // up refs in bunches (i.e., whenever the collector runs).
            int slotToExpunge = staleSlot;
            for (int i = prevIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = prevIndex(i, len))
                if (e.get() == null)
                    slotToExpunge = i;

            // Find either the key or trailing null slot of run, whichever
            // occurs first
            for (int i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();

                // If we find key, then we need to swap it
                // with the stale entry to maintain hash table order.
                // The newly stale slot, or any other stale slot
                // encountered above it, can then be sent to expungeStaleEntry
                // to remove or rehash all of the other entries in run.
                if (k == key) {
                    e.value = value;

                    tab[i] = tab[staleSlot];
                    tab[staleSlot] = e;

                    // Start expunge at preceding stale entry if it exists
                    if (slotToExpunge == staleSlot)
                        slotToExpunge = i;
                    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
                    return;
                }

                // If we didn't find stale entry on backward scan, the
                // first stale entry seen while scanning for key is the
                // first still present in the run.
                if (k == null && slotToExpunge == staleSlot)
                    slotToExpunge = i;
            }

            // If key not found, put new entry in stale slot
            tab[staleSlot].value = null;
            tab[staleSlot] = new Entry(key, value);

            // If there are any other stale entries in run, expunge them
            if (slotToExpunge != staleSlot)
                cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
        }
```
主要注意点，在把staleSlot位置替换为key-value时，要检查后面是否有相同的key，有的话需要替换为staleEntry，然后调用expungeStaleEntry进行清理无用的Entry。
```java
        /**
         * Heuristically scan some cells looking for stale entries.
         * This is invoked when either a new element is added, or
         * another stale one has been expunged. It performs a
         * logarithmic number of scans, as a balance between no
         * scanning (fast but retains garbage) and a number of scans
         * proportional to number of elements, that would find all
         * garbage but would cause some insertions to take O(n) time.
         *
         * @param i a position known NOT to hold a stale entry. The
         * scan starts at the element after i.
         *
         * @param n scan control: {@code log2(n)} cells are scanned,
         * unless a stale entry is found, in which case
         * {@code log2(table.length)-1} additional cells are scanned.
         * When called from insertions, this parameter is the number
         * of elements, but when from replaceStaleEntry, it is the
         * table length. (Note: all this could be changed to be either
         * more or less aggressive by weighting n instead of just
         * using straight log n. But this version is simple, fast, and
         * seems to work well.)
         *
         * @return true if any stale entries have been removed.
         */
        private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```
cleanSomeSlots以log2(n)的复杂度尝试做清理工作

### Remove操作
```java
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```
remove操作比较简单，主要就是找到散列的位置后把key设置为null，然后调用expungeStaleEntry来实现

### Rehash操作
```java
        /**
         * Re-pack and/or re-size the table. First scan the entire
         * table removing stale entries. If this doesn't sufficiently
         * shrink the size of the table, double the table size.
         */
        private void rehash() {
            expungeStaleEntries();

            // Use lower threshold for doubling to avoid hysteresis
            if (size >= threshold - threshold / 4)
                resize();
        }

        /**
         * Double the capacity of the table.
         */
        private void resize() {
            Entry[] oldTab = table;
            int oldLen = oldTab.length;
            int newLen = oldLen * 2;
            Entry[] newTab = new Entry[newLen];
            int count = 0;

            for (int j = 0; j < oldLen; ++j) {
                Entry e = oldTab[j];
                if (e != null) {
                    ThreadLocal<?> k = e.get();
                    if (k == null) {
                        e.value = null; // Help the GC
                    } else {
                        int h = k.threadLocalHashCode & (newLen - 1);
                        while (newTab[h] != null)
                            h = nextIndex(h, newLen);
                        newTab[h] = e;
                        count++;
                    }
                }
            }

            setThreshold(newLen);
            size = count;
            table = newTab;
        }

        /**
         * Expunge all stale entries in the table.
         */
        private void expungeStaleEntries() {
            Entry[] tab = table;
            int len = tab.length;
            for (int j = 0; j < len; j++) {
                Entry e = tab[j];
                if (e != null && e.get() == null)
                    expungeStaleEntry(j);
            }
        }
```

## 弱引用

```
To help deal with very large and long-lived usages, the hash table entries use WeakReferences for keys. However, since reference queues are not used, stale entries are guaranteed to be removed only when the table starts running out of space.
```
* 如果线程时间比较长，一些ThreadLocal如果比较大就比较占用内存，为了及时清理这些内容，Entry的key本质上是WeakReference
* 使用了WeakReference但并没有使用WeakReference的ReferenceQueue，因此要注意清理工作，代码中的expungeStaleEntry，cleanSomeSlots，replaceStaleEntry都是为了达到这样的目的，当然这几个再处理WeakReference的同时也负责维护Hash冲突的处理
* 清理工作是在get和set操作过程中触发的，不是自动进行的


## Hash冲突
解决Hash冲突的主要方法：__开放地址法__ 和 __链地址法__ , ThreadLocalMap使用的是 __开放地址法__ ,而且用的是 __线性探测(linear-probe)__

## 神奇数字0x61c88647

用来计算hash值的数字
```java
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```
>从上面的计算过程可见，__一个ThreadLocal变量的hash值的计算与ThreadLocal变量本身数值并与直接的关系__,为什么可以这样？因为它的生命周期仅限于本线程内

有人做了个实验：
```python
>>> HASH_INCREMENT = 0x61c88647
>>> def magic_hash(n):
...     for i in range(n):
...         nextHashCode = i * HASH_INCREMENT + HASH_INCREMENT
...         print nextHashCode & (n - 1),
...     print
... 
>>> magic_hash(16)
7 14 5 12 3 10 1 8 15 6 13 4 11 2 9 0
>>> magic_hash(32)
7 14 21 28 3 10 17 24 31 6 13 20 27 2 9 16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25 0
```

* ((sqrt(5)-1) * (2^31))
* 黄金分割
* 斐波那契散列

# InheritableThreadLocal
__作用__：可以继承父线程的ThreadLocal中的值，但父线程并不能获取子线程中设置的值。如果需要修改父线程中的ThreadLocal值的话可以覆盖childValue方法。
```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * Get the map associated with a ThreadLocal.
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

在Thread.java的init方法中，有下面一段代码:

```java
if (parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
```

* 线程初始化的时候就对inheritableThreadLocals进行拷贝
* 子线程运行后如果父线程又设置了新的值，子线程是得不到的
* 父线程也不会获得子线程中设置的值

# 补充1 —— Hash冲突
hash冲突时解决方案：
* 开放地址法：容易产生堆积问题；不适于大规模的数据存储
  * Hi(key)=(H(key)+di) mod m ( i = 1,2,…… ， k ( k ≤ m – 1)),其中： H ( key ) 为关键字 key 的直接哈希地址,m 为哈希表的长度， di 为每次再探测时的地址增量
  * 根据di取法的不同
    * d i ＝ 1 ， 2 ， 3 ， …… 线性探测再散列
    * d i ＝ 1^2 ，－ 1^2 ， 2^2 ，－ 2^2 ， k^2， -k^2…… 二次探测再散列
    * d i ＝ 伪随机序列 伪随机再散列
* 链地址法：平均查找长度短，增删方便

# 补充2 —— 引用
1. __强引用__：我们平时使用的引用，Object obj = new Object(); 只要引用存在，GC 时，就不会回收对象；
2. __软引用__：还有一些用，但非必需的对象，系统发生内存溢出之前，会回收软引用指向的对象；
3. __弱引用__：非必需的对象，每次 GC 时，都会回收弱引用指向的对象；
4. __虚引用__：不影响对象的生命周期，每次 GC 时，都会回收，虚引用用于在 GC 回收对象时，获取一个系统通知。