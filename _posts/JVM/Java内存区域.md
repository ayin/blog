# 运行时数据区域

## 程序计数器(PC)

* 线程私有，各线程独立存储
* 执行Native方法时该值为空(Undefined)
* 唯一一个没有规定任何OOM的区域

## Java虚拟机栈

* 栈帧：存储局部变量表、操作数栈、动态链接、方法出口等信息
* 平常所说的栈主要指局部变量表，局部变量表的大小是在编译期间确定的(@Contened呢？)
* 如果超出所允许的栈深度抛出StackOverflowError异常
* 如果无法申请到足够的内存抛出OutOfMemoryError异常

## 本地方法栈

* Sun HotSpot把本地方法栈和Java虚拟机栈合二为一了
* 同样会StackOverflowError 和 OOM

## Java堆

* 对象实例和数组一般在堆上分配(JIT和逃逸分析可以进行一些栈上分配，标量替换等优化)
* GC的主要区域，分代划分
* 线程私有的分配缓冲区(TLAB, Thread Local Allocation Buffer)
* 逻辑连续，使用-Xmx和-Xms控制大小，可抛出OOM异常

## 方法区

* 存储类信息、常量、静态变量、即时编译后的代码等
* 在hotspot中方法区属于永久代管理，但是这样并不太好，JAVA8中用NativeMemory来实现了，Java7中字符串常量池也移出了永久代
* CGLIB,JSP，动态语言Groovy等对方法区的影响
* 会抛出OOM异常

## 运行时常量池

* 属于方法区的一部分
* String的intern方法：Java6中intern方法会把首次遇到的字符串实例复制到永久代并返回永久代中的地址，而Java7中不会再复制实例，只在常量池中记录首次出现的实例引用

## 直接内存

* NIO中的DirectByteBuffer等
* 可以产生OOM异常，-XX:MaxDirectMemorySize来指定，默认与Java堆最大值(-Xmx)一样
* 这部分OOM时HeapDump的文件中不会包含直接内存的内容

# Hotspot虚拟机对象

## 对象创建

1. new指令，类加载、解析、初始化
2. 内存分配：规整的->指针碰撞(Serial, ParNew)，不规整的->空闲列表(CMS)
3. 内存分配的线程安全：一种方案是CAS配上失败重试保证原子性；另一种方案是使用TLAB，只在分配新的TLAB时进行同步锁定，-XX:+/-UseTLAB参数来设定
4. 初始化为零值(TLAB可以提前至分配TLAB时)，对象头除外(How?)
5. 设置对象头，包括元数据，hash值，GC分代年龄，是否偏向锁等
6. 执行init方法，invokespecial指令

## 对象的内存布局

* 对象头：包括hash,GC分代年龄,锁状态标志，线程持有的锁，偏向线程ID，偏向时间戳等；有些虚拟机包括类型指针；数组长度信息
* 实例数据：受分配策略和字段在源码中的顺序影响，HotSpot默认分配策略是相同宽度的字段分配到一起；CompactFields参数为true时窄变量插入到空隙中
* 对齐填充

## 对象的访问定位

* 句柄 VS 直接指针
* Hotspot使用直接指针的方式