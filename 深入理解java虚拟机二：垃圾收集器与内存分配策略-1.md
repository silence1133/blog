title: jvm学习二：垃圾收集器与内存分配策略
author: Silence
tags:
  - java
  - jvm
categories:
  - jvm
date: 2018-06-09 15:34:00
---
# **1、垃圾回收的时机**

## 1.1 判定对象是否存活

### 1.1.1 引用计数法

        给对象添加一个引用计数器，每当有一个对象引用它时，计数器值加1；引用失效，计数减一。计数为0的对象则是没有被使用对象。这种方式很难解决相互循环引用的问题。

### 1.1.2 可达性分析算法

        通过一系列成为GC Roots的对象作为起始点，从这些节点开始向下搜索，搜索所走过的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（从GC Roots到这个对象不可达）时，则证明对象是不可用的。

> 关于什么是GC Roots的理解：
> 
> 所有活跃着的被引用的对象。例如方法运行时，方法中引用的对象，类的静态引用的对象，类中常量引用的对象；Native方法中引用的对象等。

## 1.2 对象的引用

![](http://pbl9wvifs.bkt.clouddn.com/2-1.png)

## 1.3 对象回收的过程

![](http://pbl9wvifs.bkt.clouddn.com/2-2.png)

## 1.4 方法区的回收

在方法区垃圾回收效率非常低。垃圾回收70~95%是在堆中。此区域垃圾回收主要两个部分：废弃 的常量和无用的类。

判断一个常量是否是废弃常量与堆中的对象基本一致。

判断一个类是否是无用的类条件比较苛刻，需要满足以下3个条件：

1.  java堆中不存在该类的任何实例；
2.  加载该类的ClassLoader已经被回收；
3.  类对应的Class对象没有在任何地方被引用。

当一个类被判断为无用类之后，是否会被回收还由虚拟机参数决定。

> -Xnoclassgc：不就行类的回收。
> 
> -verbose:class：是否查看类加载的信息。

# **2、垃圾回收算法**

![](http://pbl9wvifs.bkt.clouddn.com/2-3.png)

## **2.1 标记清除**

        标记-清除算法是现代垃圾回收算法的思想基础。标记-清除算法将垃圾回收分为两个阶段：标记阶段和清除阶段。一种可行的实现是，在标记阶段，首先通过根节点，标记所有从根节点开始的可达对象。因此，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。

![](http://pbl9wvifs.bkt.clouddn.com/2-4.png)

从图中可以很容易看出标记-清除算法实现起来比较容易，但是有一个比较严重的问题就是容易产生内存碎片。

## **2.2 复制算法**

-   与标记-清除算法相比，复制算法是一种相对高效的回收方法
-   在对象存活率较高的时就要进行较多的复制操作，效率会非常低，因此不适用于存活对象较多的场合，如老年代。
-   将原有的内存空间分为两块，每次只使用其中一块，在垃圾回收时，将正在使用的内存中的存活对象复制到未使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收
-   实际情况不会按照1:1去划分内存空间，而是将对内存划分为一块较大的Eden空间和两块较小的Survior空间，每次使用Eden和其中一块Survior。例如HotSpot细腻机默认Eden和Survivor的大小比例是8:1，也就是说实际只有10%的内存会被“浪费”。

![](http://pbl9wvifs.bkt.clouddn.com/2-5.png)

## **2.3 标记整理**

    标记-压缩算法适合用于存活对象较多的场合，如老年代。它在标记-清除算法的基础上做了一些优化。和标记-清除算法一样，标记-压缩算法也首先需要从根节点开始，对所有可达对象做一次标记。但之后，它并不简单的清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。

![](http://pbl9wvifs.bkt.clouddn.com/2-6.png)

## **2.4 分代收集算法**

　　分代收集算法是目前大部分JVM的垃圾收集器采用的算法。它的核心思想是根据对象存活的生命周期将内存划分为若干个不同的区域。一般情况下将堆区划分为老年代（Tenured Generation）和新生代（Young Generation），老年代的特点是每次垃圾收集时只有少量对象需要被回收，而新生代的特点是每次垃圾回收时都有大量的对象需要被回收，那么就可以根据不同代的特点采取最适合的收集算法。

　　目前大部分垃圾收集器对于新生代都采取复制算法，因为新生代中每次垃圾回收都要回收大部分对象，也就是说需要复制的操作次数较少，但是实际中并不是按照1：1的比例来划分新生代的空间的，一般来说是将新生代划分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden空间和其中的一块Survivor空间，当进行回收时，将Eden和Survivor中还存活的对象复制到另一块Survivor空间中，然后清理掉Eden和刚才使用过的Survivor空间。

![](http://pbl9wvifs.bkt.clouddn.com/2-7.jpg)

　　而由于老年代的特点是每次回收都只回收少量对象，一般使用的是标记整理算法。

![](http://pbl9wvifs.bkt.clouddn.com/2-7.png)

图说明：三块区域（Eden Space、From Space、To Space），对于较大的对象会直接移动到老年代。

# **3、HotSpot的算法实现**

![](http://pbl9wvifs.bkt.clouddn.com/2-8.png)

# **4、垃圾收集器**

![](http://pbl9wvifs.bkt.clouddn.com/2-9.png)


![](http://pbl9wvifs.bkt.clouddn.com/2-10.png)

## 4.1 Serial收集器

单线程，单CPU下收集效率高。运行在Client模式下的虚拟机来说是一个很好的选择。

![](http://pbl9wvifs.bkt.clouddn.com/2-11.png)

## 4.2 ParNew收集器

Serial收集器的多线程版本，除了使用多线程进行垃圾收集之外，其余特性和Serial都一致，除了Serial收集器外，唯一可以与CMS收集器配合工作的。

默认开启的收集线程数与cpu的数量相同，可以使用-XX ParallelGCThreads参数来限制垃圾收集的线程数

Serial收集器 VS ParNew收集器：单cpu下由于存在线程交互的开销，性能上Serial收集器更优，多核情况下ParNew性能上才有优势。

![](http://pbl9wvifs.bkt.clouddn.com/2-12.png)

## 4.3 Parallel Scavenge收集器

新生代收集器，与ParNew一样都是并行的多线程收集器。设计目标是达到一个可控制的吞吐量。吞吐量=运行用户代码的时间/（运行用户代码的时间+垃圾收集时间）。

**相比ParNew收集器它的目标是高吞吐量，ParNew收集器的目标是低停顿时间**

-XX MaxGCPauseMillis：设置垃圾收集停顿的最大时间

-XX GCTimeRatio：设置吞吐量

-XX +UseAdaptiveSizePolicy：GC自适应的调节策略开关。不需要手工指定新生代大小、Eden区与Suivivor区的比例等参数

## 4.4 Serial Old收集器

Serial收集器的老年代版本，使用标记-整理算法，两大用途：

1、JDK1.5之前的版本与Parallel Scavenge搭配使用；

2、作为CMS收集器的后背预案。

![](http://pbl9wvifs.bkt.clouddn.com/2-13.png)

## 4.5 Parallel Old收集器

Parallel Scavenge的老年代版本，使用多线程，标记-整理算法，jdk1.6以后才出现，主要用途是与Parallel Scavenge搭配使用（解决了1.6之前Parallel Scavenge只能与单线程的Serial Old使用）。

![](http://pbl9wvifs.bkt.clouddn.com/2-14.png)

## 4.6 CMS收集器

![](http://pbl9wvifs.bkt.clouddn.com/2-15.png)


以获取最短回收垃圾停顿时间为目标的收集器，在服务响应的速度上有较好的优势。

使用的是标记-清除算法。

耗时最长的两个过程是并发标记和并发清理，这两个过程都可以与用户的线程并发工作，这是能缩短停顿时间的关键所在。

缺点：

1、CMS收集器对cpu资源非常敏感。CMS默认启动的回收线程数量是（cpu数量+3）/4，特别是当cpu数量少于4个的时候， 可能会分出一半左右的线程去做垃圾收集，这样也就导致性能降低了50%，性能会很差。

2、无法处理浮动垃圾。由于在并发清理的过程中用户线程也在执行，清理的同时又有垃圾产生，这种垃圾就是浮动垃圾。因此不能像其他收集器在老年代接近满的时候才进行垃圾回收。1.5的默认阈值是老年代使用了68%，1.6之后达到了92%。当CMS运行期间内存无法满足程序使用时，会出现Concurrent Mode Failure失败，虚拟机会启动后备预案：使用Seria old收集器来重新进行老年代垃圾收集，这样会导致停顿时间变长，因此-XX CMSInitiatingOccupancyFraction参数不能设置太高。

-XX CMSInitiatingOccupancyFraction：提高促发百分比（也就是前面说的阈值）

3、会产生大量的垃圾碎片。会导致频繁触发full gc。

-XX +UseCMSCompactAtFullCollection：默认开启，Full GC同时是否需要开启内存碎片的合并整理过程。这个解决了碎片问题，但是停顿时间会变长。

-XX CMSFullGCsBeforeCompaction：多少次full gc后进行一次合并整理，默认为0。

## 4.7 G1收集器

![](http://pbl9wvifs.bkt.clouddn.com/2-16-0.png)

# **5、GC日志**

-XX:+PrintGC:只是简单地打印出整个堆垃圾回收的情况。

![](http://pbl9wvifs.bkt.clouddn.com/2-16-1.png)

-XX:+PrintGCDetails:打印出新生代、老年代的堆空间回收情况。

![](http://pbl9wvifs.bkt.clouddn.com/2-16-2.png)

-XX:+PrintGCTimeStamps、-XX:+PrintGCDateStamps：需要打印出时间则加上这个时间参数。

-Xloggc:/Users/a000996/gc.log：GC输出到日志文件

# **6、内存分配与回收策略**

![](http://pbl9wvifs.bkt.clouddn.com/2-16.png)



# **7、虚拟机性能监控与故障处理工具**

![](http://pbl9wvifs.bkt.clouddn.com/2-17.png)

相关命令行工具的具体使用参见：[http://docs.oracle.com/javase/8/docs/technotes/tools/](http://docs.oracle.com/javase/8/docs/technotes/tools/)