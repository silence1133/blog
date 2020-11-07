title: jvm学习一：java内存区域与内存溢出
tags:
  - java
  - jvm
categories:
  - jvm
author: Silence
date: 2018-06-07 17:28:00
---
**1、运行时数据区域**
=============

1.1 结构图：
--------

![Markdown](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/56e6l35ori7.png)
-------------------------------------------------------------------------------

1.2 各区域详细介绍
-----------

### 1.2.1 程序计数器

较小的内存空间，它可以看做是当前线程所执行的字节码的行号指示器。线程私有的内存区域。唯一一个区域没有任何OutOfMemoryError的区域。

### 1.2.2 java虚拟机栈

线程私有，生命周期与线程相同。描述的是java方法执行的内存模型，每个方法在执行的同时都会创建一个栈帧，存储了局部变量表、操作数栈、动态链接、方法出口等信息。一个方法的调用直至执行完，就相当于是一个栈帧在虚拟机中入栈到出栈。  
局部变量表：存放了编译期可知的各种基本数据类型、对象引用和returnAddress类型。局部变量表所需的内存空间在编译期间完成分配。如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverFlowError。如果虚拟机栈可以动态扩展，扩展时无法申请到足够内存，就抛出OutOfMemoryError异常。

### 1.2.3 本地方法栈

和java虚拟机栈类似，唯一不同是本地方法栈是用于使用native方法的服务。

### 1.2.4 java堆

线程共享，虚拟机启动时创建，唯一目的就是存放对象实例，从内存回收的角度，细分为新生代（Eden空间、From Survivor、To Survivor），老年代。java堆可以处于物理上不连续的空间。如果堆中没有内存完成实例分配，并且堆也无法扩展，将抛出OutOfMemoryError。

### 1.2.5 方法区

线程共享，用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。java虚拟机规范也把方法区作为堆的一个逻辑部分。垃圾收集行为在这个内存区域是较少出现的，这个区域内存回收的目的主要是针对常量池的回收和对类型的卸载。

> #### 运行时常量池
> 
> 方法区的一部分，用于存放编译器生成的各种字面量和符号引用。这部分内容将在类加载后进入方法区的运行时常量池中存放。运行期间也可能将新的常量放入池中。如String的intern()方法。

### 疑难点：方法区与永久代的理解，以及在jdk1.7、jdk1.8中的变化。

> **    方法区（method area）**只是**JVM规范**中定义的一个概念，用于存储类信息、常量池、静态变量、JIT编译后的代码等数据，具体放在哪里，不同的实现可以放在不同的地方。而**永久代**是**Hotspot**虚拟机特有的概念，是方法区的一种实现，别的JVM都没有这个东西。
> 
>     移除永久代的工作从JDK1.7就开始了。JDK1.7中，存储在永久代的部分数据就已经转移到了Java Heap或者是 Native Heap。但永久代仍存在于JDK1.7中，并没完全移除，譬如符号引用(Symbols)转移到了native heap；字面量(interned strings)转移到了java heap；类的静态变量(class statics)转移到了java heap。jdk1.8已经将永久代彻底废除，JDK 1.8中 PermSize 和 MaxPermGen 已经无效。jdk1.8引入了**Metaspace（元空间）。**
> 
>     元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制。
> 
> 关于这块可以参考：[http://www.cnblogs.com/paddix/p/5309550.html](http://www.cnblogs.com/paddix/p/5309550.html)

### 1.2.6 直接内存

**不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域。**

例如在jdk1.4之后引入的nio，引入了一个基于管道与缓冲区的I/O方式，它可以使用Native函数库直接分配堆外内存，然后通过一个存储在java堆中的DirectByteBuffer对象作为这块内存的引用进行操作。避免了java堆和native堆中来回复制数据。

不受java堆大小的限制，但会受到本机总内存影响。异常也会出现OOM异常。

**2、HotSpot虚拟机对象揭秘**
====================

2.1 对象的创建
---------

![](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/7r8vkvfjdrn.png)

1、虚拟机遇到new指令，检测这个类能都定位到类的符号引用，并检查类是否已经被加载了。

2、类加载检测通过后，接下来虚拟机为新生对象分配内存（对象所需的内存在类加载完后就已经确定了），也就是从java堆中分配一块对空间出来。

> 划分堆内存有两种方式：
> 
> 1.  **指针碰撞**：在java堆中内存绝对规整的情况下，移动指针，划分出需要的空间。
> 2.  **空闲列表**：如果内存是不规整的（已使用的内存和未使用的交错分布），虚拟机必须维护一个列表，记录哪些内存是可用的，在分配的时候从列表中找到一块足够大的内存空间，然后更新为已用。
> 
> 选择哪种分配方式由java堆是否规整觉得。java堆是否规整由垃圾收集器是否带有压缩功能决定。

3、内存分配完后，虚拟机将分配的内存空间都初始化为零值，这一步保证了对象的实例在不赋初始值直接使用。

4、对对象进行必要的设置

5、执行init方法，对对象进行初始化。经过这步一个真正的对象才算产生。

2.2 对象的内存布局
-----------

### ![](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/kurmx14t47o.png)

### 2.2.1 对象头

对象头包括两部分内容：

    **1、对象自身的运行时数据**。包括哈希码、GC分代年龄、锁状态标志。。。。这部分的长度在32位虚拟机和64位分别对应了32位和64位。

  **  2、类型指针**。即对象指向它的类元数据的指针，虚拟机通过这个指针确定是哪个类的实例。注意不是所有的虚拟机都有这部分内容。

### 2.2.2 实例数据

对象真正存储的有效信息，无论是父类继承下来还是子类中定义的都需要记录下来，这部分存储顺序受虚拟机分配策略参数和字段在java源码中定义的影响。

### 2.2.3 对齐填充

并不是必然存在的，也没有特别含义，起着占位符的作用。HotSpot VM的自动内存管理要求对象起始地址必须是8直接的整数倍。

2.3 对象的访问定位
-----------

对象访问方式由虚拟机决定，目前主流的访问方式分为以下两种：

**   1、 句柄访问：**java堆中划分出一块内存来作为句柄池，栈中存放的reference存储的就是对象的句柄地址，句柄中包含了对象实例数据的地址信息以及类型数据的地址信息。优点：在对象被移动的时候，只需要改变句柄池，无需改变reference。

![](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/api03176ebo.png)

**  2、 直接指针访问**：栈中的reference存储的就是对象实例数据的地址以及对象类型数据的指针。优点：最大好处就是访问速度快。Hotspot使用的是直接指针访问方式。

![](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/8xntvxrlum.png)

3、实战：OutOfMemoryError异常
=======================

3.1 java堆溢出
-----------

### 3.1.1 示例代码

    public class HeapOOM {
    
        public static void main(String[] args) {
            List<OOMObject> list = new ArrayList<OOMObject>();
           while(true){
               list.add(new OOMObject());
           }
        }
    
        static class OOMObject{
        }
    }

虚拟机参数设置：

> -Xms20M -Xmx20M ：设置最小堆内存20M(也叫初始堆内存)，最大堆内存20M
> 
> -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=F:\\JvmTemp\\dump：
> 
> 发生内存溢出时Dump出当前内存堆转储快照到指定的路径下。
> 
> -Xms20M -Xmx20M -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=F:\\JvmTemp\\dump

运行结果：

    java.lang.OutOfMemoryError: Java heap space
    Dumping heap to F:\JvmTemp\dump\java_pid1472.hprof ...

分析Dump文件：

可以使用jdk自带的jvisualvm打开dump文件进行分析。

![](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/d2epq5c1nra.png)

3.2 栈溢出
-------

![](http://fcmall-test-1251779293.image.myqcloud.com/fcapp/mall/gzw506lli4b.png)

### 3.2.1 示例一（StackOverflowError）：

    public class JavaVMStackSOF {
        private int length = 1;
        public void stackLeak(){
            length ++ ;
            stackLeak();
        }
        public static void main(String[] args) throws Throwable {
            JavaVMStackSOF javaVMStackSOF = new JavaVMStackSOF();
            try{
                javaVMStackSOF.stackLeak();
            }catch (Throwable e){
                System.out.println("stack length:"+javaVMStackSOF.length);
                throw e;
            }
        }
    }

参数设置：

> -Xss128k
> 
> 设置每个线程的栈内存为128k

运行结果：

    stack length:968
    Exception in thread "main" java.lang.StackOverflowError
    ...

### 3.2.2 示例二（OutOfMemoryError）

通过不断建立线程的方式可以产生内存溢出异常

    public class JavaVMStackOOM {
        public void thread(){
            while(true){
                Thread t = new Thread(new Runnable() {
                    public void run() {
                        dontStop();
                    }
                });
                t.start();
            }
        }
    
        private void dontStop() {
            while (true){}
        }
    
        public static void main(String[] args) {
            JavaVMStackOOM javaVMStackOOM = new JavaVMStackOOM();
            javaVMStackOOM.thread();
        }
    }

参数设置：-Xss2M

运行结果：OOM（此段代码不宜在windows下执行，会导致系统假死，看不到结果）。

### 3.2.3 结论:

1.  在当个线程下，无论是由于栈帧太大还是虚拟机栈容量太小，虚拟机抛出的都是StackOverflowError。
2.  每个线程分配的栈容量越大，可以建立的线程数量越小。
3.  如果建立过多线程会导致内存溢出，在不减少线程数或者更换64位虚拟机的情况下，就只能通过减少最大堆和减少栈容量来换取更多的线程。

3.3 方法区和运行时常量池溢出
----------------

### 3.3.1 示例（方法区）：

方法区用于存放Class的相关信息，则可以通过产生大量的类来填满方法区。

使用cglib产生大量的代理类。

    public class JavaMethodAreaOOM {
        public static void main(String[] args) {
            while(true){
                Enhancer enhancer = new Enhancer();
                enhancer.setSuperclass(OOMObject.class);
                enhancer.setUseCache(false);
                enhancer.setCallback(new MethodInterceptor() {
                    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                        return methodProxy.invokeSuper(o,objects);
                    }
                });
                enhancer.create();
            }
        }
        static class OOMObject{}
    }

参数设置：

> -XX:PermSize=10M -XX:MaxPermSize=10M
> 
> -XX:PermSize：设置方法区（永久代）最小内存为10M，最大内存为10M

结果：

    java.lang.OutOfMemoryError: PermGen space
    ...

**注意：**这里使用java 8不会出现OOM，是因为没有了永久代，类的信息已经移到Metaspace了，这个参数已经没有任何意义了

### 3.3.2 总结：

方法区溢出也是一种常见的内存溢出异常，一个类要被垃圾回收掉，判定条件是比价苛刻的。在经常生成大量Class的应用中，需要特别注意类的回收状况。

3.4 本机直接内存溢出
------------

        DirectMemory容量可通过-XX：MaxDirectMemorySize指定，如果不指定，则默认与java堆最大值一样。由于DirectMemory导致的内存溢出，一个明显的特征是在Heap Dump文件中不会看见明显的异常，如果发现OOM之后Dump文件很小，而程序又直接或间接使用了nio，那就应该注意是不是这方面的原因。