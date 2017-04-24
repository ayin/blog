---
categories:
  - Technology
tags:
  - Java
excerpt: 概述：由Java8中的Contended注解引出对False Sharing的介绍及其解决办法
---

# 背景
Java8中除了新增Stream等内容之外，还有一个不太引人注目的@Contended注解，本文简单介绍该注解的用处

# False Sharing

概述：Cache中的存储单位是Cache Line, Cache Line的大小一般为64字节，如果多线程中修改相互独立的变量，而这独立的变量又刚好处于同一个Cache Line上，就会相互影响彼此的性能，这就是False Sharing(伪共享)

```
      Core1         Core2
        |             |
 L1 Cache(X,Y)   L1 Cache(X,Y)
        \             /
        L2 Cache(X,Y)
              |
       Main Memory(X,Y)
```
比如上图中(只简单实用L1 Cache,L2 Cache示意，实际中一般都有L3 Cache)，如果X和Y刚好位于同一个CacheLine上，Core1不停的更新X，Core2不停的更新Y，无论更新X还是Y都会使对方的缓存行失效，这样就需要来来回回经过L2 Cache,大大影响性能。

>获取Cache Line的大小，MAC: sysctl machdep.cpu.cache.linesize,  LINUX: getconf LEVEL1_DCACHE_LINESIZE

# 如何办：JAVA8之前

Java8之前一般使用 ___缓存行填充___来消除False Sharing,大概内容如下。具体到Java6和Java7会稍有不同，因为Java7中会对无用的变量进行消除，因此一般用继承子类进行封装，避免被优化掉。下面类似的代码大量见于java Concurrent包、Disruptor中RingBuffer的实现等处。

```java
public long p1, p2, p3, p4, p5, p6, p7; // cache line padding
private volatile long cursor = INITIAL_CURSOR_VALUE;
public long p8, p9, p10, p11, p12, p13, p14; // cache line padding
```

类似的为了避免多个CPU对一个变量进行频繁操作而导致缓存不停失效的也见于其他一些地方，比如自旋锁的CLH实现也有类似的思想。

# 如何办：JAVA8的@Contended

@Contended的效果实际上就是指示JVM通过添加padding把变量分配到单独的缓存行上，主要有三种用法：
1. 注解添加到class对象上
2. 注解添加到field对象上
3. @Contended可以进行分组，如：@Contended("group1")，但只支持字段级别(Field-Level)

下面对比不使用@Contended和使用@Contended的运行速度：

Unpadded的版本：SomeUnPaddedClass

```java
public class SomeUnPaddedClass {
    public volatile long valueA;
    public volatile long valueB;
}
```

Padded的版本：SomePaddedClass:

```java
import sun.misc.Contended;

public class SomePaddedClass {
    public volatile long valueA;
    @Contended
    public volatile long valueB;
}
```

测试，注意运行时添加参数 ___-XX:-RestrictContended___：

```java
public class TestContended {

    private static final SomeUnPaddedClass unPadded = new SomeUnPaddedClass();
    private static final SomePaddedClass padded = new SomePaddedClass();
    private static final int ITERATIONS = 1_00_000_000;

    public static long test(IntConsumer fun1, IntConsumer fun2) throws InterruptedException {
        Thread thread1 = new Thread(() -> IntStream.range(0, ITERATIONS).forEach(fun1));
        Thread thread2 = new Thread(() -> IntStream.range(0, ITERATIONS).forEach(fun2));
        long start = System.nanoTime();
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        return System.nanoTime() - start;
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("UnPadded Version:" + TestContended.test(it -> unPadded.valueA++, it -> unPadded.valueB++) / 1E9);
        System.out.println("Padded Version:" + TestContended.test(it -> padded.valueA++, it -> padded.valueB++) / 1E9);
    }
}
```

某次运行结果如下：

```
UnPadded Version:5.028771889
Padded Version:0.986878633
```

# 内存分布

在我机器上运行JVM分配了2倍CacheLine大小的padding。如果使用[JOL](http://openjdk.java.net/projects/code-tools/jol/) 查看内存分布，运行命令：

```bash
java -XX:-RestrictContended -cp classes:jol-cli-0.7.1-full.jar org.openjdk.jol.Main internals com.test.SomePaddedClass
```

## 情况1：不使用@Contended
```java
public class SomePaddedClass {
    public volatile long valueA;
    public volatile long valueB;
}
```
输出如下：
```
Instantiated the sample instance via default constructor.

com.test.SomePaddedClass object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a3 11 01 f8 (10100011 00010001 00000001 11111000) (-134147677)
     12     4        (alignment/padding gap)                  
     16     8   long SomePaddedClass.valueA                    0
     24     8   long SomePaddedClass.valueB                    0
Instance size: 32 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

## 情况2：使用一个@Contended
```java
public class SomePaddedClass {
    public volatile long valueA;
    @Contended
    public volatile long valueB;
}
```
输出如下：
```
Instantiated the sample instance via default constructor.

com.test.SomePaddedClass object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a3 11 01 f8 (10100011 00010001 00000001 11111000) (-134147677)
     12     4        (alignment/padding gap)                  
     16     8   long SomePaddedClass.valueA                    0
     24   128        (alignment/padding gap)                  
    152     8   long SomePaddedClass.valueB                    0
    160   128        (loss due to the next object alignment)
Instance size: 288 bytes
Space losses: 132 bytes internal + 128 bytes external = 260 bytes total
```

## 情况3：使用两个@Contended
```java
public class SomePaddedClass {
    @Contended
    public volatile long valueA;
    @Contended
    public volatile long valueB;
}
```
输出如下：
```
Instantiated the sample instance via default constructor.

com.test.SomePaddedClass object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           a3 11 01 f8 (10100011 00010001 00000001 11111000) (-134147677)
     12   132        (alignment/padding gap)                  
    144     8   long SomePaddedClass.valueA                    0
    152   128        (alignment/padding gap)                  
    280     8   long SomePaddedClass.valueB                    0
    288   128        (loss due to the next object alignment)
Instance size: 416 bytes
Space losses: 260 bytes internal + 128 bytes external = 388 bytes total
```

## 情况4：使用两个@Contended，相同分组内

读者自己测试，我本地测试两个变量是分配到了一起，但是运行上面的测试程序就要具体情况具体分析了

# 参考
1. [False Sharing, Cache Coherence, and the @Contended Annotation on the Java 8 VM](https://dzone.com/articles/false-sharing-cache-coherence-and-the-contended-an)
2. [伪共享](http://ifeve.com/falsesharing/)
3. [Java8的伪共享和缓存行填充](http://www.cnblogs.com/Binhua-Liu/p/5623089.html)
4. [伪共享和缓存行填充，从Java 6, Java 7 到Java 8](http://www.cnblogs.com/Binhua-Liu/p/5620339.html)