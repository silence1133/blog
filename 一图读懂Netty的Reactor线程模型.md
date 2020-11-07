title: 一图读懂Netty的Reactor线程模型
author: Silence
tags:
  - Netty
  - NIO
categories:
  - Netty
date: 2020-08-26 13:48:00
---

# 开门见山
![image](http://static.silence.work/netty-reactor-2.png)

## 1. 图中组件的解释

1. netty的io线程分两类，分别由Boss NioEventLoopGroup和Work NioEventLoopGroup来管理线程，这里说的线程可以理解为NioEventLoop。


2. NioEventLoop持有一个thread，一个任务队列，以及一个Selector。


3. NioEventLoop不等同于线程，只是持有了一个thread，从类名称看它是一个while true循环loop，不停地检测处理io事件和异步任务（图中圆环处理的三步逻辑）。


4. Boss NioEventLoopGroup持有的NioEventLoop主要用于服务端处理客户端的连接接入，即accept事件（图中Boss NioEventLoop的圆环）。


5. Boss NioEventLoop处理accpet事件的逻辑由其持有的ChannelPipepline中的Handler提供。


6. 每一个客户端的连接对应一个NioSocketChannel，多个NioSocketChannel对应一个Work NioEventLoop。


7. Work NioEventLoopGroup持有的NioEventLoop主要处理客户端channel上的读写事件，即read/write 事件（图中Work NioEventLoop的圆环）。


8. Work NioEventLoopGroup的NioEventLoop轮训到读事件后，按序执行ChannelPipeline中的handler逻辑。


9. 每一个ChannelHandler会包装成一个ChannelHandlerContext，ChannelHandlerContext持有前后节点的引用，组成双向链表结构。


## 2. 服务端接收连接并处理读事件的过程

1. 服务端启动后 Boss NioEventLoop会一直处于轮训过程中，当客户端请求服务端，其持有的Selector会检测到accept事件（图中Boss NioEventLoop圆环中的第一个过程select）


2. Boss NioEventLoop检测到accept事件后，接着进入第二个过程：process selected keys。


3. Boss NioEventLoop执行的process selected key逻辑会经过服务端channelPipeline中的handler进行处理，沿着pipeline链从头节点开始执行。


4. 服务端channelPipeline处理连接接入主要在ServerBootstrapAcceptor这个Handler完成，会构造客户端channel：NioSocketChannel，代表本条连接的socket通信通道。


5. 构造好的NioSocketChannel绑定到某个Work NioEventLoop上，该连接通道上的读写都由这个Work NioEventLoop处理，同时注册Selector。客户端channel上的io事件检测由其持有的Selector来完成。


6. 客户端与服务端建立好连接后，发送数据给服务端，Work NioEventLoop轮训到读事件（图中Work NioEventLoop圆环中的第一个过程select），接着进入process selected keys过程（图中Work NioEventLoop圆环中的第二个过程process selected keys）。


7. process selected keys的过程同样也是经过pipline链进行处理，由客户端channelPipeline从头节点开始进行相应的逻辑处理。


---

# 一探究竟

为了探究Netty的Reactor背后实现原理，本部分按照4个部分从源码进行分析。

**1. NioEventLoop的创建与启动过程。**

**2. NioEventLoop的循环三步曲。**

**3. 服务端处理连接接入**

**4. 服务端处理读事件**


**源码分析Demo：**

```
public final class ServerD {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childOption(ChannelOption.TCP_NODELAY, true)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) {
                            ch.pipeline().addLast(new BizHandler());
                        }
                    });
            ChannelFuture f = b.bind(8888).sync();

            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```
## 1. NioEventLoop的创建与启动过程

### 1.1 NioEventLoop的创建

入口：NioEventLoopGroup构造函数。

我们重点看MultithreadEventExecutorGroup的构造函数逻辑。


```
   protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (executor == null) {
            //1. 创建ThreadPerTaskExecutor
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }
        
        children = new EventExecutor[nThreads];
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                //2. 创建eventloop
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                    //。。。。
                }
            }
        }
        
        //3.构造选择器
        chooser = chooserFactory.newChooser(children);

        //。。。
    }
```


#### 1.1.1 创建ThreadPerTaskExecutor

该类的作用是在执行线程任务的时候会根据设置的ThreadFactory来创建线程执行。

```
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}
```

- NioEventLoop的启动就是调用ThreadPerTaskExecutor的execute启动的
- 每次执行任务都会创建一个线程
- NioEventLoop线程命名规则nioEventLoop-index-xxx 

#### 1.1.2 创建NioEventLoop

根据指定的线程数，for循环进行创建。这里的EventExecutor数组就是用来存放NioEventLoop的。

```
    children = new EventExecutor[nThreads];

    for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                //。。。
            }
    }
```

newChild方法实际就是调用NioEventLoop的构造函数。

NioEventLoop构造方法做了如下逻辑：

**1. 保存上一步的线程执行器ThreadPerTaskExecutor**


**2. 创建taskQueue任务队列（MpscQueue）**

MpscQueue：MpscLinkedQueue是netty自己实现的线程安全的队列，netty的无锁化串行设计。允许有多个生产者，只有一个消费者的队列

**3. RejectedExecutionHandler线程拒绝策略**

**4. 创建一个selector**

Netty重新实现了jdk的selector，SelectedSelectionKeySetSelector，对keySet的数据结构进行了优化。


#### 1.1.3. 构造选择器

**选择器的作用**：对新的链接选择NioEventLoop
，也就是从上一步生成好的NioEventLoop数组中选择一个来处理后续这个连接通道上的所有io事件。

根据NioEventLoop的数组长度是否是2的n次幂，来构造具体的选择器。


```
  chooser = chooserFactory.newChooser(children);
  
  public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTwoEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }
```


- **是2的n次幂：** PowerOfTwoEventExecutorChooser，和hashmap取下标算法一样。


```
executors[idx.getAndIncrement() & executors.length - 1]
```

- **不是：** 自增根据长度取模。


```
return executors[Math.abs(idx.getAndIncrement() % executors.length)];
```


#### 创建过程总结

![image](http://static.silence.work/20200826-1.png)

创建过程特别要注意的一点是NioEventLoop构建好后，对应持有的thread是null，thread的创建和启动是在接下来要分析的启动过程中完成的。

### 1.2 启动过程

- 入口：SingleThreadEventExecutor.excute（NioEventLoop.excute调用，excute方法在父类）


**执行链路：**

```
SingleThreadEventExecutor.excute
    ->inEventLoop()
    ->SingleThreadEventExecutor.startThread
        ->SingleThreadEventExecutor.doStartThread
            ->ThreadPerTaskExecutor.execute      
                ->thread = Thread.currentThread()
                ->SingleThreadEventExecutor.this.run()
```


```
## SingleThreadEventExecutor.excute

   public void execute(Runnable task) {
        //1. 当前线程是否是eventLoop持有的线程
        boolean inEventLoop = inEventLoop();
        //2. 将任务添加到队列，保证串行化执行
        addTask(task);
        if (!inEventLoop) {
            //3. 启动一个线程，通过持有的ThreadPerTaskExecutor。
            startThread();
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }

...
```

**1、判断当前线程是否是eventLoop持有的线程；**

```
boolean inEventLoop = inEventLoop();
```
**2、将当前任务放到taskQueue中；**


**3、第一步判断的结果不在，则启动一个线程**。

进入SingleThreadEventExecutor.startThread方法，然后进入doStartThread()方法，接下来看下doStartThread()的逻辑

#### doStartThread()

最外层通过NioEventLoop的ThreadPerTaskExecutor.excute方法，也就是new一个线程并启动执行任务。

```

#ThreadPerTaskExecutor.excute

public void execute(Runnable command) {
    threadFactory.newThread(command).start();
}

 private void doStartThread() {
        assert thread == null;
        //executor是ThreadPerTaskExecutor
        executor.execute(new Runnable() {
            @Override
            public void run() {
                //1.将当前线程赋值给NioEventLoop的thread变量
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                   
                   //2. 执行run方法
                   SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                  
                //...
                }
            }
        });
    }
    
```

在任务的run方法中做了两件事情：

**1、将当前线程赋值给NioEventLoop的thread变量**。

**2、执行NioEventLoop的run方法。**

 这里是NioEventLoop循环三部曲的逻辑所在，我们放到第二个部分分析：NioEventLoop的循环三步曲。


#### 启动过程总结

启动过程就是将NioEventLoop持有的线程实例化并启动的，以及将任务通过添加到任务队列来保证任务的无锁化线程安全操作。

## 2. NioEventLoop的循环三步曲

即NioEventLoop的执行过程

- 入口：NioEventLoop.run();

![image](http://static.silence.work/20200826-2.png)

- **step 1：select（轮训io事件）**


- **step 2：process selected keys（处理轮训到的io事件）**


- **step 3：run tasks（执行异步任务）**

```
#NioEventLoop.run

    protected void run() {
        for (;;) {
            try {
                switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                    case SelectStrategy.CONTINUE:
                        continue;
                    case SelectStrategy.SELECT:
                        //1. 轮训io事件
                        select(wakenUp.getAndSet(false)); 
                        if (wakenUp.get()) {
                            selector.wakeup();
                        }
                        // fall through
                    default:
                }
                //处理io事件与异步任务的时间分配
                cancelledKeys = 0;
                needsToSelectAgain = false;
                final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        //2. 处理io事件
                        processSelectedKeys();
                    } finally {
                        // 3. 执行异步任务
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        //2. 处理io事件
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        //3. 执行异步任务
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
            // Always handle shutdown even if the loop processing threw an exception.
            try {
                if (isShuttingDown()) {
                    closeAll();
                    if (confirmShutdown()) {
                        return;
                    }
                }
            } catch (Throwable t) {
                handleLoopException(t);
            }
        }
    }
```


### 2.1 select

- 入口：NioEventLoop.select()


```
 private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            //当前select操作不能超过这个时间：当前时间+第一个定时任务的执行时间
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

            for (;;) {
                //超时时间
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                //是否超时
                if (timeoutMillis <= 0) {
                    if (selectCnt == 0) {
                        //非阻塞的select方法
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                //如果异步队列有任务
                if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                    //非阻塞式select
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }
                
                //以上if都不进，则指定超时时间阻塞select
                int selectedKeys = selector.select(timeoutMillis);
                selectCnt ++;
                //轮训到事件
                if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                    // - Selected something,
                    // - waken up by user, or
                    // - the task queue has a pending task.
                    // - a scheduled task is ready for processing
                    break;
                }
                //...
                //处理jdk空轮训bug
                //轮训时间没有超过阻塞select的时间，则是一次空轮训，进入else逻辑
                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                //存在空轮训，并且空轮训域值达到阈值
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    // The selector returned prematurely many times in a row.
                    // Rebuild the selector to work around the problem.
                    logger.warn(
                            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                            selectCnt, selector);
                    //重新构建一个selector，将原select注册的事件注册到新的selector上
                    rebuildSelector();
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;
            }
            ///
        } catch (CancelledKeyException e) {
            ///
        }
    }
```


**主要逻辑：**

**1、deadline以及任务穿插逻辑处理。**

这里主要是计算执行select操作允许的截止时间，用于控制select操作与执行任务队列任务的时间分配。

这个截止时间计算=当前时间+第一个定时任务的执行时间。

**2、阻塞式select。**

当任务队列为空时，则会进行阻塞式select操作，同时指定允许的阻塞时间。


**3、避免jdk空轮训的bug**

正常情况当经过阻塞selector操作后，没有轮训到事件，则当前时间减开始时间一定是超过的阻塞select的时间。

如果未超过，则一定是一次空轮训，利用selectCnt来计数，当selectCnt超过阈值(SELECTOR_AUTO_REBUILD_THRESHOLD)了，则调用rebuildSelector重新注册一个selector。



<font color='red'>问题点：什么时候终止轮训？</font>

答案：

```
    if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
            // - Selected something,
            // - waken up by user, or
            // - the task queue has a pending task.
            // - a scheduled task is ready for processing
            break;
    }
```



### 2.2 process selected keys

- 入口：NioEventLoop.processSelectedKeys()

遍历selectedKeys数组，执行processSelectedKey方法。也就是判断SelectionKey上绑定事件，通过NioUnsafe执行对应的逻辑。

***说明：关于NioUnsafe执行read的逻辑我们在服务端处理连接接入与读事件过程进行分析。***

```
1. NioEventLoop#processSelectedKeysOptimized:

    private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // null out entry in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.keys[i] = null;
            //获取到绑定的channel
            final Object a = k.attachment();

            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
            //...
        }
    }

2. NioEventLoop#processSelectedKey(SelectionKey k, AbstractNioChannel ch)

  private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        //...
        try {
            int readyOps = k.readyOps();
            //链接事件
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT;
                k.interestOps(ops);

                unsafe.finishConnect();
            }

            // 写事件
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }

            // 读或者链接接入事件
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }

```

### 2.3 run tasks


- 入口：SingleThreadEventExecutor.runAllTasks()

循环从队列中取出任务，并执行，任务执行没有太复杂的逻辑，主要逻辑用于控制异步任务执行的时长。

当前方法会传入一个截止时间，也就是执行队列中的任务的截止时间，如果超过这个截止时间了，则退出执行，重新进入轮训io事件的for循环。

```
  protected boolean runAllTasks(long timeoutNanos) {
        //将定时任务队列聚合放到普通任务队列内
        fetchFromScheduledTaskQueue();
        //获取任务
        Runnable task = pollTask();
        if (task == null) {
            afterRunningAllTasks();
            return false;
        }
        //计算执行任务的截至时间
        final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
        long runTasks = 0;
        long lastExecutionTime;
        for (;;) {
            //执行，其实就是Runnable的run方法。
            safeExecute(task);

            runTasks ++;

            // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            //执行64个任务之后计算是否已经过了截至时间
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }
            //获取下一个任务
            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }
        
        afterRunningAllTasks();
        this.lastExecutionTime = lastExecutionTime;
        return true;
    }

```

#### 任务聚合

- 入口：SingleThreadEventExecutor.fetchFromScheduledTaskQueue()

从定时任务队列中获取到当前时间已经大于等于了定时任务截止时间的任务，将其添加到普通任务队列中。


```
    private boolean fetchFromScheduledTaskQueue() {
        long nanoTime = AbstractScheduledEventExecutor.nanoTime();
        Runnable scheduledTask  = pollScheduledTask(nanoTime);
        while (scheduledTask != null) {
            if (!taskQueue.offer(scheduledTask)) {
                // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
                scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
                return false;
            }
            scheduledTask  = pollScheduledTask(nanoTime);
        }
        return true;
    }
```

#### 任务的添加

##### 1. 普通任务

- 入口：SingleThreadEventExecutor.addTask()

对前面分析内容还有印象的话，我们知道任务是在NioEventLoop启动过程中添加的。


```
    protected void addTask(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }
        if (!offerTask(task)) {
            reject(task);
        }
    }
```

<font color='red'>问题：那么到底什么样的任务会添加的队列呢？</font>

我们可以带着这个问题在**服务端处理连接接入与读事件过程中**去寻找。

##### 2. 定时任务

入口：AbstractScheduledEventExecutor.schedule

向scheduledTaskQueue中添加任务

```
    <V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
        if (inEventLoop()) {
            scheduledTaskQueue().add(task);
        } else {
            execute(new Runnable() {
                @Override
                public void run() {
                    scheduledTaskQueue().add(task);
                }
            });
        }

        return task;
    }
```




###  执行过程总结

执行过程主要是两层for (;;)循环，通过轮训io事件，以及轮训到io事件后进行io事件的处理。在轮训处理io事件的过程还要考虑异步任务执行的时间分配，通过ioRatio控制io事件轮训和任务队列执行的时间分配。

**伪代码如下**：

```
for (;;) {
    for (;;) {
        //任务队列有任务
        if (hasTasks() && wakenUp.compareAndSet(false, true)) {
            //非阻塞轮训
            selector.selectNow();
            selectCnt = 1;
            break;
        }
        //允许阻塞的时间
        long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
        //阻塞轮训
        int selectedKeys = selector.select(timeoutMillis);
        //轮训到事件或者任务队列有任务了
        if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
            break;
        }
    }
    
    //处理io事件
    processSelectedKeys();
    
    for (;;) {
        task = pollTask();
        task.run();
        if (lastExecutionTime >= deadline) {
            break;
        }
        task = pollTask();
     }
}
```

##  3. 服务端处理连接接入

通过分析如何服务端如何处理连接与读事件，能加深我们对netty的线程模型认识，这里分成4个部分进行分析：

**1. 检测新链接**

**2. 创建NioSocketChannel**

**3. 分配线程并注册Selector（通过服务端channel的pipeline执行完成）**

**4. 向Selector注册读事件**

![image](http://static.silence.work/20200826-3.png)

### 3.1 检测新链接

在分析NioEventLoop启动后执行过程我们知道，
服务端channel绑定的NioEventLoop在不停地做select操作，当轮训到io事件后则会进入processSelectedKeys方法。

- 入口：NioEventLoop.processSelectedKeys()


```
NioEventLoop.processSelectedKey()
    ->AbstractNioMessageChannel.NioMessageUnsafe.read()
        ->NioServerSocketChannel.doReadMessages
            ->SocketChannel ch = SocketUtils.accept(javaChannel())
```

**伪代码：**


```
for (int i = 0; i < selectedKeys.size; ++i) {
      SelectionKey k=selectedKeys.keys[i];
      int readyOps = k.readyOps();
      //accept事件
      if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) 
      {
            //用于接入客户端链接速率控制
            final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
            do {
                //获取客户端NioSocketChannel
                SocketChannel ch = serverSocketChannel.accept();
                //没有链接break循环
                if(ch == null){
                    break;
                }
                buf.add(new NioSocketChannel(this, ch)); allocHandle.incMessagesRead(localRead);
                //接入连接计数+1
                allocHandle.incMessagesRead();
            } 
            while (allocHandle.continueReading());
      }
}
```


### 3.2 创建NioSocketChannel

- 入口：NioServerSocketChannel#doReadMessages


```
   protected int doReadMessages(List<Object> buf) throws Exception {
        SocketChannel ch = SocketUtils.accept(javaChannel());
        try {
            if (ch != null) {
                //构造NioSocketChannel并缓存起来
                buf.add(new NioSocketChannel(this, ch));
                return 1;
            }
        } catch (Throwable t) {

        }
        return 0;
    }
```

也就是在检测新链接过程中通过serverSocketChannel.accept()获取到jdk原生的SocketChannel，再将其包装成netty的NioSocketChannel过程。

整个构造过程做了如下处理：

**1、设置3个参数：id、unsafe、pipeline；**

**2、保存读感兴趣的事件（后面在向Selector注册读事件会有用）**

**3、配置NioSocketChannel非阻塞；**

**4、setTcpNoDelay(true)：禁用Nagle算法，允许小包的发送**

### 3.3 新链接NioEventLoop分配与Selector注册


- 入口：AbstractNioMessageChannel.NioMessageUnsafe#read

也就是给客户端NioSocketChannel非配NioEventLoop，同时绑定Selector。

```
AbstractNioMessageChannel.NioMessageUnsafe#read
    //检测到新链接缓存到readBuf，也就是前面检测新链接分析的代码
    //...
    //遍历缓存的客户端连接，触发服务端channel的channelRead方法
    for (int i = 0; i < size; i ++) {
        readPending = false;
        pipeline.fireChannelRead(readBuf.get(i));
    }
```


当检测到客户端连接后，会使用服务端channel的pipeline调用fireChannelRead方法，也就是在服务端Channel的pipeline链中进行传播。

服务端channel的pipeline构成如下，我们沿着这些绑定的Handler的channelRead方法进行分析。

![image](http://static.silence.work/20200826-4.png)

其中HeadContext的channelRead方法并未做任何处理，关键逻辑处理都在**ServerBootstrapAcceptor**。

我们看ServerBootstrapAcceptor.channelRead的方法


```
 public void channelRead(ChannelHandlerContext ctx, Object msg) {
            //获得客户端连接的channel
            final Channel child = (Channel) msg;
            //在客户端的pipeline中添加设置自定义的childHandler，在注册读事件的时候会触发对应的initChannel方法将自定义的handler添加到pipeline
            child.pipeline().addLast(childHandler);
            //设置Options
            setChannelOptions(child, childOptions, logger);
            //设置attrs
            for (Entry<AttributeKey<?>, Object> e: childAttrs) {
                child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
            }
            try {
                //这里的childGroup是workerGroup，通过拿到一个NioEventLoop进行注册
                //注册selector的入口
                childGroup.register(child).addListener(new ChannelFutureListener() {
                    @Override
                    public void operationComplete(ChannelFuture future) throws Exception {
                        if (!future.isSuccess()) {
                            forceClose(child, future.cause());
                        }
                    }
                });
            } catch (Throwable t) {
                forceClose(child, t);
            }
        }
```

ServerBootstrapAcceptor.channelRead()做了以下几个事件

#### 3.3.1 添加childHandler

#### 3.3.2 设置options和attrs

#### 3.3.3 选择NioEventLoop

利用NioEventLoop的选择器来获取一个用于客户端channel的NioEventLoop。

通过调用childGroup（也就是work NioEventLoopGroup）的next方法，最终通过chooser从eventExcutor[]数组中获取一个NioEventLoop。这里选择work NioEventLoop是为了后面注册selector和读事件。


```
MultithreadEventLoopGroup#register()

chooser.next()
```



#### 3.3.4 注册selector

**1、AbstractUnsafe#register**

将上一步选择的NioEventLoop保存到客户端channel中，完成了channel绑定NioEventLoop过程，接着**将后续的执行逻辑交给这个客户端channel的NioEventLoop去异步执行**(这一点也别要注意，并不是用boss ioEventLoop处理的)。


```
     public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            //...
            //保存当前客户端channel对应的NioEventLoop
            AbstractChannel.this.eventLoop = eventLoop;
            //这里的eventLoop不是服务端EventLoop，走else逻辑
            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            //注册Selector
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```

对于这里异步的逻辑，参考NioEventLoop的启动过程，实际回答了前面的问题什么任务会添加到任务队列。

这里也就是会将客户端channel注册到Selector的逻辑封装成一个Runnbale，将其添加到任务队列中，由work NioEventLoop的run task过程完成。

```
    eventLoop.execute(new Runnable() {
            @Override
            public void run() {
                //注册Selector
                register0(promise);
            }
    });
    
    SingleThreadEventExecutor#execute:
    
    public void execute(Runnable task) {
        boolean inEventLoop = inEventLoop();
        addTask(task);
        if (!inEventLoop) {
            startThread();
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }
        //...
    }
    
    
    
```



**2、AbstractChannel.register0**

register0由客户端channel分配的NioEventLoop来执行，然后调用doRegister()方法

```
  private void register0(ChannelPromise promise) {
            try {
                boolean firstRegistration = neverRegistered;
                //给客户端channel注册Selector
                doRegister();
                neverRegistered = false;
                registered = true;
                //触发自定义的childHandler的initChannel方法，将自定义Handler添加到客户端channel的pipline中
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise);
                //触发channelRegistered方法
                pipeline.fireChannelRegistered();
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {
                    if (firstRegistration) {
                       //执行handler链的channelActive，接着会执行读事件的注册。最终逻辑实在AbstractNioChannel的doBeginRead方法
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                ///
            }
        }
```

**3、AbstractNioChannel#doRegister**

在NioEventLoop构造过程我们知道每一个NioEventLoop都对应有一个Selector，这里就是调用jdk底层的channel的register方法，同时拿到当前NioEventLoop持有的Selector完成注册。

```
 protected void doRegister() throws Exception {
        boolean selected = false;
        for (;;) {
            //调用jdk底层Channel的register方法
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        }
    }
```


### 3.4 注册读事件

在AbstractChannel.register0方法中完成Selector注册后，接着触发客户端channel的pipline链中handler的channelActive方法。

同样客户端channel pipline链中也会绑定HeadContext和TailContext，注册读事件也就是在HeadContext的channelActive方法中。

注册读事件链路比较长，耐心跟下去：

```
HeadContext.channelActive
readIfIsAutoRead()
    ->AbstractChannel.read()
        ...
        ->HeadContext.read()
            ->AbstractChannel.AbstractUnsafe#beginRead
                ->AbstractNioChannel#doBeginRead
```

看AbstractNioChannel#doBeginRead

```
    protected void doBeginRead() throws Exception {
        // Channel.read() or ChannelHandlerContext.read() was called
        final SelectionKey selectionKey = this.selectionKey;
        if (!selectionKey.isValid()) {
            return;
        }

        readPending = true;

        final int interestOps = selectionKey.interestOps();
        if ((interestOps & readInterestOp) == 0) {
            selectionKey.interestOps(interestOps | readInterestOp);
        }
    }

```


### 总结

服务端处理连接接入，会构造客户端的NioSocketChannel，接着会分配一个NioEventLoop给当前NioSocketChannel，然后利用分配的NioEventLoop异步完成绑定Selector注册读事件。

同服务端channel构建对应的pipline一样，客户端channel的pipline也会绑定HeadContext和TailContext。客户端channel的pipline链如下：

![image](http://static.silence.work/20200826-5.png)

在绑定Selecotr和注册读事件过程中会依次触发
1. handlerAdded：代表自定义handler已经添加到pipline了。
2. channelRegistered：代表Selector已经绑定了
3. channelActive：代表监听事件注册了，代表通道建立完成了

## 4. 服务端处理读事件

服务端处理读事件逻辑比较简单，这里不做深入分析，这里简单看下处理过程即可

同服务端处理连接事件一样，也从NioEventLoop.processSelectedKeys这个入口进入读事件的处理，区别在于这个是在work NioEventLoop线程中执行。

![image](http://static.silence.work/20200826-6.png)

### 4.1. 检测读事件

同服务端检测连接接入事件一样，也会执行到如下这行代码，唯一区别是这里用的read方法是NioByteUnsafe提供（服务端用的NioMessageUnsafe）

```
NioEventLoop#processSelectedKey()

    private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
            int readyOps = k.readyOps();
            //...
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                unsafe.read();
            }
    }
    
AbstractNioByteChannel.NioByteUnsafe#read  

    public final void read() {
                  do {
                    //分配读byteBuf
                    byteBuf = allocHandle.allocate(allocator);
                    allocHandle.lastBytesRead(doReadBytes(byteBuf));
                    if (allocHandle.lastBytesRead() <= 0) {
                        // nothing was read. release the buffer.
                        byteBuf.release();
                        byteBuf = null;
                        close = allocHandle.lastBytesRead() < 0;
                        if (close) {
                            // There is nothing left to read as we received an EOF.
                            readPending = false;
                        }
                        break;
                    }

                    allocHandle.incMessagesRead(1);
                    readPending = false;
                    //执行读事件的pipeline链 pipeline.fireChannelRead(byteBuf);
                    byteBuf = null;
                } while (allocHandle.continueReading());

                allocHandle.readComplete();
                pipeline.fireChannelReadComplete();
        }
```


### 4.2 执行读事件的pipeline链

按照客户端channel的handler链依次支持channelRead方法，这块逻辑比较简单，不做深入分析。

Head头节点未做任何逻辑，直接将读事件向下传递，经过自定义指定的handler，最后在tail节点完成内存的释放。