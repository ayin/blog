---
categories:
  - Technology
tags:
  - Java
---

# 零
## 0:1 简介
* 相对于c，c++来说，JVM内存管理这道“墙”为java提供了一种“安全”的环境
* 墙里面的人总需要翻墙看看的，java提供了一些翻墙神器
 * JNI：.c, .cpp, .dll, .so ...；限定于特定平台
 * com.sun.jna：基于JNI，只能实现java访问c函数，而不能反过来
 * sun.misc.Unsafe：方便那些想站在“围墙”边上往外瞅瞅，而不需要跑太远的同学使用，但是就这已经足够瞅到很多东西了~

```
|-----------|            |           |------------|
|   Safe    |<---sun.misc.Unsafe--->|   Unsafe   |
|-----------|            |           |------------|
                        JVM
```

## 0:2 同类
* C#: Common Language Runtime (CLR)中可以执行指针操作等Unsafe代码
* Go: Go语言中unsafe.Pointer其实就是类似C的void*, 在golang中是用于各种指针相互转换的桥梁

# 壹

## 1:1 万事开头难
Unsafe不是一个愿意让人new来new去的家伙，要想得到一个Unsafe对象，得进入她的内心
```java
static {
    registerNatives();
    sun.reflect.Reflection.registerMethodsToFilter(Unsafe.class, "getUnsafe");
}

private Unsafe() {}

private static final Unsafe theUnsafe = new Unsafe();

public static Unsafe getUnsafe() {
    Class cc = sun.reflect.Reflection.getCallerClass(2);
    if (cc.getClassLoader() != null)
        throw new SecurityException("Unsafe");
    return theUnsafe;
}
```
上面的sun.reflect.Reflection.getCallerClass(2)返回的就是你自己的调用getUnsafe的类。而一般BootStrap Class Loader才会为null，自己的类不会通过BootStrap ClassLoader加载(__–Xbootclasspath强制除外，这是获取Unsafe对象的一种方法__)。复习一下：
* BootStrap ClassLoader：称为启动类加载器，是Java类加载层次中最顶层的类加载器，负责加载JDK中的核心类库，如：rt.jar、resources.jar、charsets.jar等
* Extension ClassLoader：称为扩展类加载器，负责加载Java的扩展类库，默认加载JAVA_HOME/jre/lib/ext/目下的所有jar
* App ClassLoader：称为系统类加载器，负责加载应用程序classpath目录下的所有jar和class文件
* 自定义加载器

在HotSpot中Unsafe以单例的形式存在，field名字叫theUnsafe，因为Reflection不可能被禁用，因此可以通过下面方式获取Unsafe类：
```java
public static Unsafe getUnsafe() throws Exception{
    Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
    theUnsafe.setAccessible(true);
    return  (Unsafe) theUnsafe.get(null);
}
```
上面方式在很多共用的jar包中被使用，但是却不是很通用的，比如Android平台下单例的名字叫'THE_ONE',下面是另外一种获取Unsafe的方法：
```java
public static Unsafe getUnsafe2() throws Exception{
    Constructor<Unsafe> unsafeConstructor = Unsafe.class.getDeclaredConstructor();
    unsafeConstructor.setAccessible(true);
    return unsafeConstructor.newInstance();
}
```
思考
* 为什么不能用Unsafe.class.newInstance()获取Unsafe对象? newInstance的本质是调用构造函数
* 如果自己需要一个这样的类该如何实现？


## 1:2 对象操作能力
得到Unsafe对象不容易，但是却可以通过它很容易的例化其他对象

### 1:2:1 定义对象defineClass
利用defineClass可以从字节码中动态得到一个class

```java
public void testClass() throws Exception {
    Unsafe unsafe = UnsafeTest.getUnsafe1();
    File f = new File("/helloworld/unsafe/A.class");
    FileInputStream input = new FileInputStream(f);
    byte[] content = new byte[(int)f.length()];
    input.read(content);
    input.close();

    Class c = unsafe.defineClass(null, content, 0, content.length);
    System.out.print(c.getMethod("getValue").invoke(c.newInstance(), null));
}
```

相关函数
```java
public native Class defineAnonymousClass(Class hostClass, byte[] data, Object[] cpPatches);
```
如何使用？

### 1:2:2 例化对象allocateInstance
通过allocateInstance()方法，可以创建一个类的实例，但是却不需要调用它的构造函数、初使化代码、各种JVM安全检查以及其它的一些底层的东西。即使构造函数是私有，也可以通过这个方法创建它的实例。
```java
public class A {
    private int value;

    public A() {
        this.value = 1;
    }

    public int getValue() {
        return value;
    }
}
public void testAllocateInstance() throws Exception {
    A a1 = new A();
    System.out.println(a1.getValue()); //输出：1
    A a2 = A.class.newInstance();
    System.out.println(a2.getValue()); //输出：1
    A a3 = (A) getUnsafe1().allocateInstance(A.class);
    System.out.println(a3.getValue()); //输出：0
}
```
通过allocateInstance得到的对象同样可以被垃圾回收器回收。

### 1:2:3 操作普通对象
通过unsafe可以方便的操作对象中的各个field，无论public, private, final, static

```java
public void testReadWrite() throws Exception {
    Unsafe unsafe = UnsafeTest.getUnsafe1();
    A a = (A)unsafe.allocateInstance(A.class);
    System.out.println(a.getValue());
    long offset = unsafe.objectFieldOffset(a.getClass().getDeclaredField("value"));
    unsafe.putInt(a, offset, 123);
    System.out.println(a.getValue());
    a.setValue(234);
    System.out.println(unsafe.getInt(a, offset));
}
```
相关函数
```java
public int fieldOffset(Field f);
public native long staticFieldOffset(Field f);
public native long objectFieldOffset(Field f);
public native Object staticFieldBase(Field f);
//Deprecated, 有些JVM不会把同一个class的static存到一个地方？
public Object staticFieldBase(Class c);
/**
 * Ensure the given class has been initialized. This is often
 * needed in conjunction with obtaining the static field base of a
 * class.
 */
public native void ensureClassInitialized(Class c);

public native int getInt(Object o, long offset);
public native void putInt(Object o, long offset, int x);
public native Object getObject(Object o, long offset);
public native void putObject(Object o, long offset, Object x);
//......
```
思考：  
1. 使用Unsafe提供的put和get操作相比直接使用赋值操作的性能提升是什么？
2. final对象呢？


### 1:2:4 操作数组对象
```java
public native int arrayBaseOffset(Class arrayClass);
public static final int ARRAY_INT_BASE_OFFSET = theUnsafe.arrayBaseOffset(int[].class);
public native int arrayIndexScale(Class arrayClass);
public static final int ARRAY_BOOLEAN_INDEX_SCALE = theUnsafe.arrayIndexScale(boolean[].class);
```

### 1:2:5 应用场景
#### 1:2:5:1 获取对象大小
* java.lang.instrument，[In Java, what is the best way to determine the size of an object?](http://stackoverflow.com/questions/52353/in-java-what-is-the-best-way-to-determine-the-size-of-an-object)
* [Code Tools: jol](http://openjdk.java.net/projects/code-tools/jol/)

使用Unsafe大略获取：
```java
public long sizeOf(Class<?> clazz) throws Exception{
    Unsafe unsafe = UnsafeTest.getUnsafe1();
    long maximumOffset = 0;
    do {
        for (Field f : clazz.getDeclaredFields()) {
            if (!Modifier.isStatic(f.getModifiers())) {
                maximumOffset = Math.max(maximumOffset, unsafe.objectFieldOffset(f));
            }
        }
    } while ((clazz = clazz.getSuperclass()) != null);
    return maximumOffset + 8;
}
```

![meta数据](http://1.bp.blogspot.com/-UoqPzdwGqCE/Uq4eVIGH-bI/AAAAAAAAARk/ERaXISTnZCM/s320/object_layout.png)

#### 1:2:5:2 序列化
很多的序列化框架都会使用Unsafe::allocateInstance，它在初始化对象的时候，能够避免调用构造器方法，在反序列化的时候，这是很有用的。这样做会节省很多时间并且能够保证安全性，因为对象的状态是通过反序列化过程重建的
Serialization:
* Build schema for object using reflection. It can be done once for class.
* Use Unsafe methods getLong, getInt, getObject, etc. to retrieve actual field values.
* Add class identifier to have capability restore this object.
* Write them to the file or any output.
Deserialization:
* Create instance of serialized class. allocateInstance helps, because does not require any constructor.
* Build schema. The same as 1 step in serialization.
* Read all fields from file or any input.
* Use Unsafe methods putLong, putInt, putObject, etc. to fill the object.


#### 1:2:5:3 未成功的例子(32位，64位)
多继承（Multiple Inheritance）
```java
public void testMultipleInheritance() throws Exception{
    Unsafe unsafe = UnsafeTest.getUnsafe1();
    long intClassAddress = normalize(unsafe.getInt(new Integer(0), 4L));
    long strClassAddress = normalize(unsafe.getInt("", 4L));
    unsafe.putAddress(intClassAddress + 36, strClassAddress);
    System.out.print((String) (Object) (new Integer(666)));
}

private static long normalize(int value) {
    if(value >= 0) return value;
    return (~0L >>> 32) & value;
}
```

## 1:3 内存操作
如同java的强大离不开JMM一样，Unsafe的强大离不开其内存操作的能力

### 1:3:1 堆栈内存
上面小节中的对对象的操作能力就是Unsafe的堆栈操作能力的体现
### 1:3:2 直接内存
```java
public void testDirectMemory() throws Exception {
    Unsafe unsafe = UnsafeTest.getUnsafe1();

    System.out.println(Runtime.getRuntime().totalMemory());
    Thread.sleep(20000);
    long size = 1000*1024*1024L;
    long start = unsafe.allocateMemory(size);
    unsafe.setMemory(start, size, (byte) 0);

    System.out.println(Runtime.getRuntime().totalMemory());
    Thread.sleep(20000);
    System.out.println(start);
}
```
这个时候使用jmap -heap不会看到什么变化，但是如果使用top查看可以看到内存占用的变化情况，在MAC系统上分配1000M内存前后
__分配前：__
```bash
top -pid `jps | grep "AppMain" | cut -f 1 -d ' '`

PID    COMMAND      %CPU TIME     #TH  #WQ  #POR MEM  PURG CMPR PGRP  PPID  STATE    BOOSTS     %CPU_ME %CPU_OTHRS UID  FAULT COW
33979  java         0.0  00:00.23 21   0    79   16M  0B   0B   20154 20154 sleeping *0[1]      0.00000 0.00000    501  8598  339
```
__分配后：__
```bash
PID    COMMAND      %CPU TIME     #TH  #WQ  #POR MEM   PURG CMPRS PGRP  PPID  STATE    BOOSTS     %CPU_ME %CPU_OTHRS UID  FAULTS
33931  java         0.1  00:00.87 21   0    79   705M  0B   311M  20154 20154 sleeping *0[1]      0.00000 0.00000    501  264845
```
关于MAC上的CMPRS Memory,可参看[Understanding Compressed Memory in OS X](http://macs.about.com/od/macoperatingsystems/fl/Understanding-Compressed-Memory-in-OS-X.htm)

在Linux下如果使用top命令查看同样能看到前后的变化，如果pmap命令查看更能找到对应的分配的地址，与程序中分配的地址一致。
```bash
00007f61803ca000  10244K rw---   [ anon ]
```

__注意allocateMemory和setMemory, 如果只是allocateMemory有时候内存占用大小并没有感觉到变化，setMemory之后才会变化，感觉造成这样的原因可能是allocateMemory只负责分配了地址空间，而setMemory之后是分配了真正的内存__

其他相关函数
```java
public native long reallocateMemory(long address, long bytes);
public native void setMemory(Object o, long offset, long bytes, byte value);
public native void copyMemory(Object srcBase, long srcOffset, Object destBase, long destOffset, long bytes);
public void copyMemory(long srcAddress, long destAddress, long bytes);
public native void freeMemory(long address);

//指针或者引用大小，返回4或者8
public native int addressSize();
public native int pageSize();
```
* 通过allocateMemory分配的内存需要主动free掉
* DirectByteBuffer中释放内存的方法：冰山对象，System.gc(),full GC......

### 1:3:3 应用场景
#### 1:3:3:1 分配超大数组
java中数组大小的限制是int型的，因此不能分配比较大的数组

举例：略

#### 1:3:3:2 零拷贝
* netty
* DirectByteBuffer
* ......

#### 1:3:3:3 浅拷贝(Shallow Copy)

## 1:4 原子访问和多线程相关
和内存操作一样，原子访问能力是Unsafe的又一把利器

### 1:4:1 putOrderedXXX
例子一：AtomicXX中的lazySet
```java
public final void lazySet(int newValue) {
    unsafe.putOrderedInt(this, valueOffset, newValue);
}
```
[A Faster Volatile](http://robsjava.blogspot.com/2013/06/a-faster-volatile.html)
对volatile和putOrderXXX的性能进行了对比


例子二：ForkJoinPool中的addSubmission
```java
private void addSubmission(ForkJoinTask<?> t) {
    final ReentrantLock lock = this.submissionLock;
    lock.lock();
    try {
        ForkJoinTask<?>[] q; int s, m;
        if ((q = submissionQueue) != null) {    // ignore if queue removed
            long u = (((s = queueTop) & (m = q.length-1)) << ASHIFT)+ABASE;
            UNSAFE.putOrderedObject(q, u, t);
            queueTop = s + 1;
            if (s - queueBase == m)
                growSubmissionQueue();
        }
    } finally {
        lock.unlock();
    }
    signalWork();
}
```
例子三：
[Java内存访问重排序的研究](http://tech.meituan.com/java-memory-reordering.html)
美团的一篇不错的文章，介绍了在特定硬件体系结构下使用putOrderXXX代替volatile带来的性能提升。


例子四：ConcurrentLinkedQueue
```java
void lazySetNext(Node<E> val) {
    UNSAFE.putOrderedObject(this, nextOffset, val);
}
```
ConcurrentHashMap中也有类似使用

### 1:4:2 putXXXVolatile和getXXXVolatile
平常已经很多地方介绍volatile了，对比这order/lazySet来理解volatile
putXXXVolatile(Object target, long offset, XXX value): Will place value at target's address at the specified offset and not hit any thread local caches.

putOrderedXXX(Object target, long offset, XXX value): Will place value at target's address at the specified offet and might not hit all thread local caches.

- volatile内存屏障
![volatile内存屏障](http://o9l56z0kf.bkt.clouddn.com/image/blog/volatile-memory-barriers.png)

- volatile内存屏障代码示例
![volatile内存屏障](http://o9l56z0kf.bkt.clouddn.com/image/blog/volatile-memory-barrier-instructions.png)


### 1:4:3 compareAndSwapXXX
java中各种CAS已经用的很广泛了，其底层实现就是调用的Unsafe包中的compareAndSwapXXX

### 1:4:4 park&unpark
LockSupport.java中
```java
public static void unpark(Thread thread) {
    if (thread != null)
        unsafe.unpark(thread);
}
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    unsafe.park(false, 0L);
    setBlocker(t, null);
}
```
* unpark和park都是不可以叠加的，多次unpark的会被一次park消耗掉的
* unpark可以先于park执行，两个线程分别调用park和unpark时不用担心时序问题(蛋疼的wait/notify/notifyAll)


### 1:4:5 monitor
 * monitorEnter
 * tryMonitorEnter
 * monitorExit
这三个方法实际没人用，也许会从Unsafe中移除，放在这个类中的原因仅仅是因为他们是Unsafe的，而且据说他们的性能也没用synchronized好

## 1:5 其他

### 1:5:1 throwException
```
/** Throw the exception without telling the verifier. */
public native void throwException(Throwable ee);
```
可以用来抛出一个undeclared checked exception，应用场景是什么？实际中我们通常显示声明并抛出checked exception或者直接unchecked exception
```java
public static void testThrowException()throws IOException{
    Unsafe unsafe1;
    try {
        Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
        theUnsafe.setAccessible(true);
        unsafe1 = (Unsafe) theUnsafe.get(null);
    }catch (IllegalAccessException e) {
        throw new RuntimeException("");
    } catch (NoSuchFieldException e) {
        throw new RuntimeException("");
    }
    unsafe1.throwException(new SQLException());
    throw new IOException();
}
```

### 1:5:2 getLoadAverage
```
//获取过去1min,5min,15min的系统负载
public native int getLoadAverage(double[] loadavg, int nelems);
```

# 贰 替代方案

## 2:1 对象操作
### 2:1:1 AtomicXX
```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
      try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
      } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
    .......
}
```
* 不足：引入了Atomic实例和实例记录的数据本身两个对象，导致额外的内存占用以及Atomic实例的解引用操作。

### 2:1:2 AtomicXXFieldUpdater
AtomixXFieldUpdater是正常Atomic类的内存优化版本，和AtomicXX一样也利用Unsafe来实现，它牺牲了API的简洁性来换取内存占用的优化，通过该组件的单个实例就能支持某个类的多个实例
### 2:1:3 变量句柄
[参考](http://openjdk.java.net/jeps/193)


## 2:2 内存管理:Unsafe::allocateMemory
基于Unsafe实现的java.nio.ByteBuffer, ByteBuffer只能用于大约2GB的数据



# 叁 使用Unsafe的一些常用包

* 前面提到的java.util.concurrent
* JAVA NIO中的direct byte buffers
* Log4j2中用到的disruptor
 * 缓存行失效
* Netty
* Hazelcast
* Cassandra
* Mockito / EasyMock / JMock / PowerMock
* Scala Specs
* Spock
* Robolectric
* Grails
* Neo4j
* Spring Framework
* Akka
* Apache Kafka
* Apache Wink
* Apache Storm
* Apache Hadoop
* Apache Continuum


# 肆 附录
## 4:1 参考

* volatile: [java和cpp中volatile的区别](http://hedengcheng.com/?p=725#_Toc373765727)
* Unsafe的生死：[Removal of sun.misc.Unsafe in Java 9 - A disaster in the making](http://blog.dripstat.com/removal-of-sun-misc-unsafe-a-disaster-in-the-making/)
* [Understanding sun.misc.Unsafe](https://dzone.com/articles/understanding-sunmiscunsafe)
* [sun.misc.Unsafe的后启示录](http://www.infoq.com/cn/articles/A-Post-Apocalyptic-sun.misc.Unsafe-World)
* [Java内存访问重排序的研究](http://tech.meituan.com/java-memory-reordering.html)
* [Java的LockSupport.park()实现分析](http://blog.csdn.net/hengyunabc/article/details/28126139)
* [sun.misc.Unsafe](http://rongmayisheng.com/post/sun-misc-unsafe)
* [http://ifeve.com/sun-misc-unsafe/](http://ifeve.com/sun-misc-unsafe/)

## 4:2 部分实现
[源码](https://github.com/aeste/gcc/blob/master/libjava/sun/misc/natUnsafe.cc)
```cpp
void
sun::misc::Unsafe::putLong (jobject obj, jlong offset, jlong value)
{
  jlong *addr = (jlong *) ((char *) obj + offset);
  spinlock lock;
  *addr = value;
}
void
sun::misc::Unsafe::putOrderedLong (jobject obj, jlong offset, jlong value)
{
  volatile jlong *addr = (jlong *) ((char *) obj + offset);
  spinlock lock;
  *addr = value;
}
void
sun::misc::Unsafe::putLongVolatile (jobject obj, jlong offset, jlong value)
{
  volatile jlong *addr = (jlong *) ((char *) obj + offset);
  spinlock lock;
  *addr = value;
}
void
sun::misc::Unsafe::putInt (jobject obj, jlong offset, jint value)
{
  jint *addr = (jint *) ((char *) obj + offset);
  *addr = value;
}
void
sun::misc::Unsafe::putOrderedInt (jobject obj, jlong offset, jint value)
{
  volatile jint *addr = (jint *) ((char *) obj + offset);
  *addr = value;
}
void
sun::misc::Unsafe::putIntVolatile (jobject obj, jlong offset, jint value)
{
  write_barrier ();
  volatile jint *addr = (jint *) ((char *) obj + offset);
  *addr = value;
}
jint
sun::misc::Unsafe::getIntVolatile (jobject obj, jlong offset)
{
  volatile jint *addr = (jint *) ((char *) obj + offset);
  jint result = *addr;
  read_barrier ();
  return result;
}
```
