---
layout    : post
title     : "《深入理解Java虚拟机_JVM高级特性与最佳实践 第2版》读书笔记"
date      : 2019-10-03
lastupdate: 2019-10-03
categories: java
---

----

* TOC
{:toc}

----

# 第2章 内存管理机制
## java内存模型

| -      | -                                              |
|:-------|:-----------------------------------------------|
| 堆      | 共享，给实例、数组分配内存。新生代和老年代，分代垃圾回收算法                 |
| 方法区    | 共享，存一些类信息、常量、静态变量                              |
| 运行时常量池 | 编译期生成的各类字面量和符号引用                               |
| 栈      | 虚拟机栈 本地方法栈                                     |
| 直接内存   | JDK1.4 NIO 用通道和缓冲区直接操作内存，避免在java堆和本地方法堆中来回复制数据 |

*定义*:
**线程私有**：各个线程独立的数据，互不影响。如PC
**java虚拟机栈**：包含1.局部变量表（存放了编译时可以确定的各种数据类型）


## 对象的创建
1. 去常量池查看类信息，有没有加载 解析 初始化
2. 已经加载过在堆中的，去Eden代去分配内存
1 如果内存是规整的直接移动指针 （指针碰撞）
2 如果内存有碎片 查看分配表  找到一块足够的区域分配给他（空闲列表） 
3. 由于创建对象是一个非常频繁的事，可能A的指针还没来的及修改B由来修改指针，为了保证线程安全采用CAS和TLAB
4. 初始化0值
5. 设置对象头：虚拟机对对象设置，如哪个类实例，hash码，对象GC，元数据
6. 执行程序员的INIT方法

## 对象的访问定位
1. 句柄池  R-池-地址
2. 直接指针 R-地址

# 第3章 垃圾回收
## 引用判断方法
1. 引用计数法：循环引用问题
2. 可达性分析法：栈中对象、方法区静态和常量

## 4种引用

| -      | -                                              |
|:-------|:----------------------|
| 强   | O o=newO（）的 永远不会回收 |
| 软   | 在内存不够的时候才回收      |
| 弱   | 只要有垃圾回收就回收        |
| 虚   | 引用等于没有                |

## 垃圾回收算法

- 标记-整理：标记然后清除，有碎片
- 复制：把内存分成一半，每次回收把存活对象直接复制到另一半，然后把自己这半清除，用在新生代，因为统计98%的新生代对象都会被回收。80%Eden代，10% 10%surviver空间，回收时把Eden代的内容复制到10%空间中，如果空间不够使用老年代空间做担保。
- 标记-整理：让存活对象向一边移动，用在老年代，因为老年代对象的存活率很高。
- 分代收集法：分成新生代和老年代，分别用复制和标记整理算法。

发起GC：在安全点（主动中断，指令复用的地方如循环）或安全区（一段引用不会发生变化的区域）

## 垃圾收集器
- serial （OLD）：单线程 简单速度快 ，会stop the world
- parnew：serial的多线程版本

只有上面两种可以与CMS配合使用
 - parallel（+OLD 1.6）：可控吞吐量

### CMS（重点）concurrent Mark-sweep
1. 初始标记： 标记GCROOTS，标记根节点 stop theworld
2. 并发标记concurrent：从根节点遍历标记 耗时最长
3. 重新标记：重新标记有变动的
4. 并发清除：清除
- 缺点：
  * 很吃CPU资源，尤其CPU只有1-2个核心时，要吃一半
  * 清的过程中还产生垃圾，所以阈值要设68%，如果设置92%，在CMS运行期间 内存无法满足主程序运行需要（垃圾清的太慢了）就是清除失败CMF，只能采用   serial old这样停顿时间就很长了
  * 标记清除有碎片

### G1（重点）garbage-first1：
- 停顿时间可控，回收步骤类似CMS，宏观标记-整理，region之间标记-复制。
- 把所有堆分成多个region，维护个优先列表，回收价值最大的。
- region不是独立的，一个region的可能引用另一个region的对象，使用remembered set记录对象是否处于不同region中。

## 内存分配机制：
1. 先在新生代分配，如果不够就执行一次minorGC把对象转移到survivor区，如果还不够，就通过担保机制转移到老年代。
2. 大对象直接进入老年代。
3. 长期存活的对象进入老年代：移动到survivor后年龄设置为1，每经过一次minorGC+1，默认15岁移动到老年代。or相同年龄所有对象的总和超过survivor的一半也移动，无需等到15。

## 类加载过程：
加载-连接（验证-准备-解析）-初始化

1. 加载字节码文件，可以从各个地方加载（本地、网络等）
2.  验证 字节码文件是否合法
    准备 静态变量==的分配内存初始化值
   解析  将符号引用替换为直接引用（字段、方法、接口）
6. 初始化 《clinit》由父到子执行程序员定义的静态方法

类加载器：放在JVM外部，可以由程序决定怎么加载字节码文件。即使是同一个类，用不同加载器加载，比较起来也不相等，因为比较只有在同一个加载器下才有意义。

## 双亲委派模型：
 如果一个类加载器收到了加载请求，不会自己尝试去加载，委派给父加载器，一直到顶层，只有顶层反馈不能加载时，子加载器才会尝试去加载。

要求自定义类加载器要有自己的父加载器（启动类RT.JAR-扩展类EXT）

目的：保证java最基础的行为，如Object类是同一个类

# 并发
线程在操作系统中的实现
- 轻量级进程和用户线程混合
- 使用多个轻量管理多个用户线程 或 1个轻量管理1个或多个

好处：可以开很多线程，不受到内核数影响，不易使整个进程阻塞；切换消耗低，不要频繁系统调用。.

## 线程安全的实现方法：
1. 互斥同步（阻塞同步）悲观
信号量，lock临界区unlock。
java中synchronized、reentrantlock实现同步。阻塞和唤醒一个线程需要操作系统来帮忙，由用户态转换到核心态中的时间有可能比用户执行代码的时间还长，是重量级的操作。
2. 非阻塞同步
先操作，如果没有其他线程争用共享数据就操作成功；如果有那就不断地重试（冲突检测）。
指令集原子操作CAS（UNSAFE类）：V原值，A预期值，B新值。如果V=A，那么用B更新V，返回旧V值，否则不执行更新。
3. 无同步方案
线程本地存储，threadlocal，线程独享变量

## 锁优化
1. 自旋锁和自适应自旋
互斥同步中阻塞引起线程挂起和恢复是要转入内核态完成，对性能影响很大。而自旋就是让等待的线程不放弃自己的执行时间，执行一个忙循环等待别人释放锁。适应性自旋就是如果上次等待成功可以就让他多等几次，如果失败就少等待几次，避免浪费资源。
2. 锁消除
虚拟机检测到变量不会被别的线程访问到，就把他当做栈上私有的，消除锁。
3. 锁粗化
同一个对象即使没有竞争也连续加锁解锁（如加锁在循环体内），很浪费性能，虚拟机自动扩大他的加锁范围（自动把锁粗话到循环外）。
4. 轻量级锁
CAS操作在对象头标记，代表获得锁
先使用cas获取锁对象；如果存在两个以上线程争用同一个锁，则升级为重量级锁，阻塞线程。绝大数锁在同步周期内不存在竞争。
5. 偏向锁
如果在第一个获取锁的线程在执行过程中不存在竞争，那就不同步了；否则转为轻量锁





# 实战：下面是一段 jvm 运行参数，对其进行解释

```
ENV JAVA_OPT_MEMORY="-Xms512m -Xmx512m -Xmn386m -XX:MetaspaceSize=386m -XX:MaxMetaspaceSize=256m"
ENV JAVA_OPT_JVM="-XX:ActiveProcessorCount=1 \
      -XX:ParallelGCThreads=4 -XX:+PrintFlagsFinal -XX:SurvivorRatio=4 \
      -XX:TargetSurvivorRatio=70 -XX:+UseConcMarkSweepGC \
      -XX:+UseParNewGC -XX:+CMSClassUnloadingEnabled -XX:+DisableExplicitGC \
      -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 \
      -XX:+CMSScavengeBeforeRemark -XX:+PrintGC -XX:+PrintGCDetails \
      -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCApplicationStoppedTime \
      -XX:+PrintFlagsFinal -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow \
      -Xloggc:/app/${APP_NAME}/logs/gc.%t.log \
      -XX:HeapDumpPath=/app/${APP_NAME}/logs/dump_core_pid.%t.hprof \
      -XX:ErrorFile=/app/${APP_NAME}/logs/hs_err_pid.%t.log"
```

这些是Java的环境变量设置，主要用于配置Java虚拟机(JVM)的参数。下面是每个参数的解释：

- `JAVA_OPT_MEMORY`：这个环境变量设置了Java堆内存的最小值(`-Xms512m`)和最大值(`-Xmx512m`)，以及新生代(`-Xmn386m`)和元空间(`-XX:MetaspaceSize=386m`和`-XX:MaxMetaspaceSize=256m`)的大小。
- `JAVA_OPT_JVM`：这个环境变量包含了一系列的JVM参数，包括：
  - `-XX:ActiveProcessorCount=1`：设置JVM可以使用的处理器数量。
  - `-XX:ParallelGCThreads=4`：设置垃圾收集器的线程数量。
  - `-XX:SurvivorRatio=4`和`-XX:TargetSurvivorRatio=70`：设置新生代中Eden区与Survivor区的比例，以及Survivor区的目标使用率。
  - `-XX:+UseConcMarkSweepGC`和`-XX:+UseParNewGC`：启用CMS垃圾收集器和ParNew垃圾收集器。
  - `-XX:+CMSClassUnloadingEnabled`：允许CMS垃圾收集器回收类加载器加载的类。
  - `-XX:+UseCMSInitiatingOccupancyOnly`和`-XX:CMSInitiatingOccupancyFraction=70`：设置CMS垃圾收集器的启动阈值。
  - `-XX:+PrintGC`，`-XX:+PrintGCDetails`，`-XX:+PrintGCDateStamps`，`-XX:+PrintGCTimeStamps`，`-XX:+PrintGCApplicationStoppedTime`：打印垃圾收集的相关信息。
  - `-XX:+HeapDumpOnOutOfMemoryError`，`-Xloggc:/app/${APP_NAME}/logs/gc.%t.log`，`-XX:HeapDumpPath=/app/${APP_NAME}/logs/dump_core_pid.%t.hprof`，`-XX:ErrorFile=/app/${APP_NAME}/logs/hs_err_pid.%t.log`：设置在内存溢出时生成堆转储文件，以及设置垃圾收集日志，堆转储文件和错误文件的路径。

这些参数可以帮助你更好地控制和监控Java应用程序的运行。但是，需要注意的是，这些参数的设置需要根据你的应用程序的实际需求来调整。不恰当的设置可能会对应用程序的性能产生负面影响。



#### 如果是在docker 容器中部署，应该怎么设置

- 在Docker容器中部署Java应用程序时，JVM参数的设置基本上与在物理机或虚拟机上的设置相同。但是，有一些特定的考虑因素需要注意：

  1. **内存限制**：Docker容器的内存通常会被限制。因此，您需要确保JVM的最大堆大小（`-Xmx`）和元空间大小（`-XX:MaxMetaspaceSize`）的总和不超过Docker容器的内存限制。否则，可能会出现OutOfMemoryError。
  2. **CPU限制**：如果Docker容器的CPU被限制，那么JVM可能无法准确地检测到可用的处理器数量。在这种情况下，您可能需要手动设置`-XX:ActiveProcessorCount`参数。
  3. **内存和CPU的可见性**：在某些情况下，JVM可能无法准确地看到Docker容器的内存和CPU限制。为了解决这个问题，Java 10引入了一个新的参数`-XX:+UseContainerSupport`，它可以让JVM更好地理解容器的限制。在Java 8的更新版本中，这个参数也被加入了。
  4. **垃圾收集器的选择**：在容器环境中，选择合适的垃圾收集器也很重要。例如，G1垃圾收集器在处理有限的内存资源时通常表现得比并发标记-清除（CMS）垃圾收集器更好。

  以下是一个示例，展示了如何在Docker容器中设置JVM参数：

  ```bash
  ENV JAVA_OPT_MEMORY="-Xms1g -Xmx1g -Xmn512m -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m"
  ENV JAVA_OPT_JVM="-XX:ActiveProcessorCount=2 \
        -XX:ParallelGCThreads=2 -XX:+PrintFlagsFinal -XX:SurvivorRatio=8 \
        -XX:TargetSurvivorRatio=90 -XX:+UseG1GC \
        -XX:+UseContainerSupport -XX:+DisableExplicitGC \
        -XX:+UseCMSInitiatingOccupancyOnly -XX:CMSInitiatingOccupancyFraction=70 \
        -XX:+PrintGC -XX:+PrintGCDetails \
        -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCApplicationStoppedTime \
        -XX:+PrintFlagsFinal -XX:+HeapDumpOnOutOfMemoryError -XX:-OmitStackTraceInFastThrow \
        -Xloggc:/app/${APP_NAME}/logs/gc.%t.log \
        -XX:HeapDumpPath=/app/${APP_NAME}/logs/dump_core_pid.%t.hprof \
        -XX:ErrorFile=/app/${APP_NAME}/logs/hs_err_pid.%t.log"
  ```

  在这个示例中，我们假设Docker容器的内存限制为2GB，CPU限制为2核。我们将堆内存的最小值和最大值都设置为1GB，新生代的大小设置为512MB，元空间的初始大小设置为256MB，最大大小设置为512MB。我们将JVM可以使用的处理器数量和垃圾收集器的线程数量都设置为2，启用了G1垃圾收集器和容器支持，并禁用了显式垃圾收集。

  

#### 为什么说“减少老年代的大小，从而减少Full GC的频率”？解释下原因

在Java的内存管理中，堆内存被分为新生代和老年代两部分。新生代又被分为Eden区和两个Survivor区。*大部分新创建的对象都会被分配到**Eden**区，当Eden区满时，就会触发Minor GC，清理掉无用的对象，并将还在使用的对象移动到**Survivor**区，如果Survivor区也满了，就会将对象移动到**老年代**。当老年代满了，就会触发Full GC，这是一种全局的垃圾回收，会暂停所有的应用线程*，所以通常会比Minor GC耗时更长。

因此，*如果我们减少老年代的大小，相应地增大新生代的大小，那么更多的对象会在新生代被清理掉，从而减少了对象进入老年代的数量，这样就可以减少老年代满的频率，从而减少Full GC的频率*。但是，这种做法也有可能导致新生代过早地满，从而增加Minor GC的频率，所以需要根据实际的应用情况来调整这两者的比例。这就是为什么我们说“减少老年代的大小，从而减少Full GC的频率”的原因。希望这个解释对你有所帮助！

#### 元空间的作用

元空间（Metaspace）是Java 8中引入的一个新的内存空间，用于替代之前版本中的永久代（PermGen）。元空间主要用于存储类的元数据，包括类的**名称、字段、方法和其他信息**。这些信息在类被加载到JVM时被创建，并在类被卸载时被删除。

元空间的一个重要特性是，它并不在Java堆内存中，而是在本地内存（也就是操作系统的内存）中。这意味着元空间的大小不再受到Java堆大小的限制，而是**受到本地内存大小**的限制。这可以避免因为永久代空间不足而导致的OutOfMemoryError。

然而，虽然元空间的大小理论上可以非常大，但在实际使用中，我们通常还是需要对其大小进行一定的限制，以避免过多的本地内存被占用。这就是为什么在JVM参数中，我们会看到`-XX:MetaspaceSize`和`-XX:MaxMetaspaceSize`这两个参数，它们分别用于设置元空间的初始大小和最大大小。

#### “-XX:SurvivorRatio=4和-XX:TargetSurvivorRatio=70：设置新生代中Eden区与Survivor区的比例，以及Survivor区的目标使用率。” 其中具体的比例和使用率是多少

`-XX:SurvivorRatio=4` 这个参数设置了新生代中Eden区与Survivor区的比例。具体来说，如果设置为4，那么Eden区与一个Survivor区的比例就是4:1。也就是说，新生代的80%（4/5）会被分配给Eden区，剩下的20%（1/5）会被分配给两个Survivor区，每个Survivor区占10%（1/10）。

`-XX:TargetSurvivorRatio=70` 这个参数设置了Survivor区的目标使用率。具体来说，如果设置为70，那么在每次Minor GC后，Survivor区的使用率目标值就是70%，也就是说，Survivor区的70%的空间会被存活的对象占用，剩下的30%的空间会被空闲。

