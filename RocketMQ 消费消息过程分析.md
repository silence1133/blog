title: RocketMQ 消费消息过程分析
author: Silence
tags:
  - RocketMQ
categories:
  - 消息中间件
  - ''
date: 2019-03-03 23:16:00
---
# 开篇

## 先抛几个问题

为了能更深入学习consumer端消费消息的源码之前，这里先抛几个问题，我们在分析源码过程中来解答对应问题。
> 1、为什么说DefaultMQPushConsumer本质还是pull？既然是pull，那rocketmq是怎么保证消息消费的实时性？

> 2、消费消息是否存在超时问题?超时了会重试吗?

> 3、什么情况下代表消费消息失败？怎么样又代表消费消息成功？

> 4、为什么说consumer端消费消息要保证幂等？什么情况下会重复消费？

> 5、消费消息失败了是怎么实现重试的？

## 源码学习引用实例

[源码引用实例](https://github.com/apache/rocketmq/tree/master/example/src/main/java/org/apache/rocketmq/example/quickstart)

```
      DefaultMQPushConsumer consumer=new    
      DefaultMQPushConsumer("testConsumer1")；
      consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
      consumer.setNamesrvAddr("localhost:9876");
      consumer.subscribe("TopicTest", "*");

      consumer.registerMessageListener(new MessageListenerConcurrently() {
                  @Override
                  public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs,
                      ConsumeConcurrentlyContext context) {
                      System.out.printf("%s Receive New Messages: %s %n", Thread.currentThread().getName(), msgs);
                      return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                  }
              });
      consumer.start();
      System.out.printf("Consumer Started.%n");
```

这里我们基本最想关心的是：consumer.start到底做了哪些事情？

# consumer启动过程解析


```
//代码有精简
    public synchronized void start() throws MQClientException {
        switch (this.serviceState) {
            case CREATE_JUST:
                //1、校验consumer的配置
                this.checkConfig();

                //2、实例化mQClientFactory
                this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

                //3、设置reblance相关属性   
                this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
                this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
                this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
                this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

                //4、设置pullAPIWrapper的消息过滤钩子
                this.pullAPIWrapper = new PullAPIWrapper(
                    mQClientFactory,
                    this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
                this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

                //5、设置consumer的offsetStore参数
                if (this.defaultMQPushConsumer.getOffsetStore() != null) {
                    this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
                } else {
                    switch (this.defaultMQPushConsumer.getMessageModel()) {
                        case BROADCASTING:
                            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        case CLUSTERING:
                            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
                            break;
                        default:
                            break;
                    }
                    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
                }

                //6、根据consumer设置的messageListner不同子类实例化不同的consumeMessageService,然后启动该类代表的线程
                if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
                    this.consumeOrderly = true;
                    this.consumeMessageService =
                        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
                } else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
                    this.consumeOrderly = false;
                    this.consumeMessageService =
                        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
                }
                this.consumeMessageService.start();
                //7、注册当前的consumer
                boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
                //8、启动各种线程任务（这里还启动了netty客户端）
                mQClientFactory.start();
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
            case START_FAILED:
            case SHUTDOWN_ALREADY:
            default:
                break;
        }
        this.updateTopicSubscribeInfoWhenSubscriptionChanged();
        this.mQClientFactory.checkClientInBroker();
        this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
        //9、直接执行reblance逻辑(也就是决定consumer的负载均衡)
        this.mQClientFactory.rebalanceImmediately();
    }
```

## 1、校验consumer的配置

其实就是校验consumer设置的值是否正确，这里顺便解释下consumer重要的相关参数：

```
> messageModel:消费消息的模式(广播模式和集群模式）
> consumeFromWhere:选择起始消费位置的方式
> allocateMessageQueueStrategy:分配具体messageQuene的策略子类。（负载均衡逻辑实现的关键类）
> consumeThreadMin：消费消息线程池的最小核心线程数(默认20)
> consumeThreadMax：最大线程数（默认64）
> pullInterval：拉取消息的间隔，默认是0
> consumeMessageBatchMaxSize：每批次消费消息的条数，默认为1
> pullBatchSize：每批次拉取消息的条数，默认32

```

## 2、实例化mQClientFactory

我们从实例化mQClientFactory代码可以看出：一个consumer客户端只会对应一个mQClientFactory（因为factoryTable存放的mQClientFactory是以客户端作为key存放的），也就是说一个应用节点只会有一个mQClientFactory实例。

```
    public MQClientInstance getAndCreateMQClientInstance(final ClientConfig clientConfig, RPCHook rpcHook) {
        String clientId = clientConfig.buildMQClientId();
        //factoryTable存放的就是client的实例，key为clientid。
        MQClientInstance instance = this.factoryTable.get(clientId);
        if (null == instance) {
            instance =new MQClientInstance(clientConfig.cloneClientConfig(),this.factoryIndexGenerator.getAndIncrement(), clientId, rpcHook);
            MQClientInstance prev = this.factoryTable.putIfAbsent(clientId, instance);
            if (prev != null) {
                instance = prev;
            } else {
            }
        }
        return instance;
    }
```


## 3、设置reblance相关属性 
也就是设置该consumer对应的负载均衡策略需要的相关参数，例如messageModel、allocateMessageQueueStrategy、实例化mQClientFactory等。

## 4、设置pullAPIWrapper的消息过滤钩子

此步作用在于可以由用户自己指定consumer过滤消息的策略，只需要调用consumer的registerFilterMessageHook，将自己实现的过滤消息的FilterMessageHook设置给consumer即可。

## 5、设置consumer的offsetStore

也就是设置consumer使用哪种处理消息消费位置offset的类。

如果是广播消费模式，则选择LocalFileOffsetStore；

如果是集群消费模式，则选择RemoteBrokerOffsetStore；

## 6、设置consumer的consumeMessageService

根据consumer设置的MessageListener来决定使用具体ConsumeMessageService。

如果是MessageListenerOrderly，则使用代表顺序消息消费的service：ConsumeMessageOrderlyService；

如果是MessageListenerConcurrently，则使用非顺序消息service：ConsumeMessageConcurrentlyService。

***注意*：此步还调用了consumeMessageService的start方法，这里只是启动了一个定时线程去做cleanExpireMsg的操作，并没有启动消费消息的线程。**

## 7、注册当前的consumer
这里只是将当前consumer放到了一个缓存map中，key为consumerGroup的名称。

## 8、mQClientFactory.start

![image](http://static.silence.work/20190304-1.png)

此步启动了netty的客户端，同时启动了很多任务，这里分两类：

一类是使用了同一定时线程池的各种任务（线程名称为MQClientFactoryScheduledThread）包括：

```
//获取namaserver地址，2分钟执行一次
MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
//从nameserver获取最新的路由信息更新到缓存，30s执行一次
MQClientInstance.this.updateTopicRouteInfoFromNameServer();
//清除下线的broker列表，30s
MQClientInstance.this.cleanOfflineBroker();
//发送心跳给broker，30s
MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
//持久化消费进度，5s执行一次
MQClientInstance.this.persistAllConsumerOffset();
MQClientInstance.this.adjustThreadPool();
```


一类是独立的线程任务，包括拉取消息pullMessageService和做reblance的rebalanceService。

```
 //拉取消息线程
 this.pullMessageService.start();
 //reblance线程
 this.rebalanceService.start();
```
注意：根据前面第2步实例化mQClientFactory的逻辑，我们知道一个consumer客户端即使有无数的consumer实例，由于mQClientFactory是单例的，因此最终只会有有一个pullMessageService和一个rebalanceService，也就是说一个拉取线程会负责一个consumer客户端所有消费者的订阅消息的拉取。**那么这里会成为性能瓶颈吗？？？**

## 9、mQClientFactory.rebalanceImmediately()；

实际只是rebalanceService.wakeup()，将第8步启动的rebalance线程唤醒。（第8步线程实际启动了，但有一个stopped标识不会让rebalanceService进入doReblance的逻辑）


## 总结

consumer.start()其实主要就是根据相关的属性准备好相关的类，然后启动几个线程和一系列的定时任务。

接下来我们重点关注pullMessageService代表的拉取消息线程类。


# 消息拉取过程分析

PullMessageService负责从broker拉取消息，该类本质上是线程的任务类，因此我们看最终任务执行逻辑是在run方法内。

```
//ServiceThread继承了Runnable接口
public class PullMessageService extends ServiceThread {
        @Override
    public void run() {
        while (!this.isStopped()) {
            //从阻塞队列pullRequestQueue中拿pullRequest。
            PullRequest pullRequest = this.pullRequestQueue.take();
            this.pullMessage(pullRequest);
        }
    }
}
```
run的逻辑看起来很简单：一个while循环不停第从阻塞队列中获取pullRequest，然后执行pullMessage逻辑。

这里我们分三步分析PullMessageService：

> 1、pullRequestQueue存放的是什么？

> 2、pullRequestQueue阻塞队列什么时候put的？

> 3、this.pullMessage(pullRequest);

## 问题一：pullRequestQueue存放的是什么

![image](http://static.silence.work/20190304-3.png)

```
public class PullRequest {
    //消费组
    private String consumerGroup;
    //要消费的队列
    private MessageQueue messageQueue;
    //消息处理队列，从broker拉取到的消息先存入到processQueue，然后再提交到消费线程中进行消费
    private ProcessQueue processQueue;
    //待拉取的consumequeue的offset
    private long nextOffset;
    private boolean lockedFirst = false;
}
```
从PullRequest类结构，这里基本可以看出存放的pullRequest封装的是每一个consumergroup以及负责消费的消费队列messageQuene。

## 问题二：pullRequestQueue的put逻辑

怎么定位到pullRequestQueue在哪里触发put的？我们这里可以在本类中找下put的方法，对应是executePullRequestImmediately方法，在put处打打断点，最终看执行线程栈，可以发现put调用的入口主要分两类地方。

### 入口一：RebalanceService.run

RebalanceService其实就是针对consumer端消费哪些messageQuene做一个负载均衡的策略。当consumer某个节点挂了，则要考虑重新做rebalance，将messagequene重新按照存活consumer节点进行分配，这里不做深入研究。这里只贴一张关键图解释该入口：

![image](http://static.silence.work/20190304-4.png)
![image](http://static.silence.work/20190304-5.png)

> 1.当需要重新分配messagequene的时候，会将需要重新分配的结果存放到pullRequestList中
> 
> 2.pullRequestList封装了重新分配的messageQuene信息，从而将重新分配的messagequene放到pullRequestQueue中。

注意：前面consumer.start过程我们有讲到RebalanceService线程会启动执行，从而可以理解当consumer一启动，相应的pullRequestQueue就会存放有对象了。

### 入口二：DefaultMQPushConsumerImpl.pullMessage方法里面定义的PullCallback

关键入口逻辑提炼：

也就是说其实就是PullCallback的onSuccess和onException中调用了pullRequestQueue的put逻辑。
而PullCallback实际就是每次拉取消息之后的回调类。也就是保证了拉完消息之后都会调用pullRequestQueue的put逻辑
![image](http://static.silence.work/20190304-6.png)
![image](http://static.silence.work/20190304-7.png)

### 总结：

根据入口一和入口二我们得出结论：

当consumer启动时，RebalanceService使得了pullRequestQueue有值，PullMessageService的线程不停地从pullRequestQueue中take messagequene拉取消息处理，处理完之后继续往pullRequestQueue存放messagequene，从而使得pullRequestQueue不会因为没有值而阻塞。这里也可以基本解释了前面抛出的问题1（pullRequestQueue每次take完因此，都会再继续put messagequene，而拉取消息实际又是一个while不停地循环去拉取消息，这样就保证了消费消息的及时性）。


## 问题三：this.pullMessage(pullRequest)

这里源码链路较长，但是一步步断点跟下去还是很清晰，我们总结下这个链路过程。

### 1 构建好PullCallback
![image](http://static.silence.work/20190304-8.png)

#### 2 取得要从哪台broker拉取消息的broker地址
![image](http://static.silence.work/20190304-9.png)

### 3 构建要拉取消息的网络请求头
![image](http://static.silence.work/20190304-10.png)

### 4 执行网络层请求broker的代码，根据结果执行对应的回调处理
这里深入到网络的调用过程，我们基本可以发现本质是交给了netty的work线程去broker拉取消息，拉取到消息之后异步回调拉取的结果，这里只贴入口代码。
![image](http://static.silence.work/20190304-11.png)

### 5 执行第一步构建的PullCallback的onSuccess逻辑

![image](http://static.silence.work/20190304-12.png)

### 6 根据broker响应的不同结果做不同的逻辑处理

```
PullCallback pullCallback = new PullCallback() {
            @Override
            public void onSuccess(PullResult pullResult) {
                if (pullResult != null) {
                    switch (pullResult.getPullStatus()) {
                        case FOUND:
                            //本拉取消息的offset
                            long prevRequestOffset = pullRequest.getNextOffset();
                            //下一次拉取消息的offset    
                            pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                            //将拉取的消息存放到processQueue
                            boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                            //将processQueue丢给consumeMessageService（从而让拉取的消息进行消费）
                           DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(pullResult.getMsgFoundList(),processQueue,pullRequest
                            getMessageQueue(),dispatchToConsume);
                            //是否有设置拉取消息间隔，有则走间隔拉取消息的逻辑
                            if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                            } else {
                                //将pullRequest放到内存队列pullRequestQueue中（PullMessageService的一个变量） DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                            }
                            break;
                        case NO_NEW_MSG:
                            //....
                            break;
                        case NO_MATCHED_MSG:
                            //...
                            break;
                        case OFFSET_ILLEGAL:
                            //...
                            break;
                        default:
                            break;
                    }
                }
            }

            @Override
            public void onException(Throwable e) {
                DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
            }
        };
```

### 7 将拉取到的消息交给consumeMessageService

其实也就是交给consumeMessageService代表的消费消息线程池处理

```
ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
this.consumeExecutor.submit(consumeRequest);
```
也就是说拉取到消息之后，接下来就是消费消息了，

那么接下来我们将会分析consumeMessageService对拉取到的消息是如何消费的。

### 8 将下一次需要拉取的pullRequest再次放到pullRequestQuene中

![image](http://static.silence.work/20190304-13.png)

### 拉消息过程总结

一个consumer客户端会分配一个拉取消息线程（PullMessageService），不停地从存放了messageQuene的阻塞队列中take需要拉取消息的messagequene，最后通过调用通知网络层发起拉取消息拉取的网络请求（实际就是交给netty的worker线程拉消息），netty的worker线程拉取到消息后调用处理PullCallback处理拉取的结果。

由于从broker拉取消息的网络请求交给了netty的worker线程处理，并且work线程处理完之后再异步通知拉取结果处理，我们可以知道pullmessage本身并没有太重的操作，同时每次请求broker拉取消息是批量拉取（默认值是每批32条），因此即使一个consuemr客户端只会有一个线程负责所有consumerGroup，也不会有太慢以及太大的性能瓶颈。



# 消息消费过程分析

ConsumeMessageService负责消息的消费，它本身只是是一个接口，提供了两个子类，分别代表了顺序消费的消息处理和普通消息的消费处理。
![image](http://static.silence.work/20190304-14.png)

消费消息的逻辑实际在代表任务的ConsumeRequest中，ConsumeRequest任务交给了ConsumeMessageThread_名称的线程池，该线程池定义如下：
![image](http://static.silence.work/20190304-15.png)

我们观察该线程池的corePoolSize和maximumPoolSize实际就是前面讲的consumer的两个自定义变量，用户来控制消费消息的线程池大小。

揭晓我们重点看消费消息的逻辑，也就是ConsumeRequest的run方法。

```
class ConsumeRequest implements Runnable {
        private final List<MessageExt> msgs;
        private final ProcessQueue processQueue;
        private final MessageQueue messageQueue;

        @Override
        public void run() {
            //消息是否已经删除
            if (this.processQueue.isDropped()) {
                return;
            }
            //messageListner（也就是消息消息的业务实现类）
            MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
            //准备messageListener需要的context
            ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
            ConsumeConcurrentlyStatus status = null;
            ConsumeMessageContext consumeMessageContext = null;
            long beginTimestamp = System.currentTimeMillis();
            boolean hasException = false;
            ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
            try {
                //与消费重试消息有关
                ConsumeMessageConcurrentlyService.this.resetRetryTopic(msgs);
                if (msgs != null && !msgs.isEmpty()) {
                    for (MessageExt msg : msgs) {
                        MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
                    }
                }
                //调用消费消息的业务逻辑
                status=listener.consumeMessage(Collections.unmodifiableList(msgs), context);
            } catch (Throwable e) {
                hasException = true;
            }
            //业务消费代码处理时长
            long consumeRT = System.currentTimeMillis() - beginTimestamp;
            
            if (null == status) {
                if (hasException) {
                    returnType = ConsumeReturnType.EXCEPTION;
                } else {
                    returnType = ConsumeReturnType.RETURNNULL;
                }
            //超时消费
            } else if (consumeRT >= defaultMQPushConsumer.getConsumeTimeout() * 60 * 1000) {
                returnType = ConsumeReturnType.TIME_OUT;
            } else if (ConsumeConcurrentlyStatus.RECONSUME_LATER == status) {
                returnType = ConsumeReturnType.FAILED;
            } else if (ConsumeConcurrentlyStatus.CONSUME_SUCCESS == status) {
                returnType = ConsumeReturnType.SUCCESS;
            }
            //将returnType放到consumeMessageContext中。（这个到consumeMessageContext实际好像给hook用）
            if (ConsumeMessageConcurrentlyService.this.defaultMQPushConsumerImpl.hasHook()) {
                consumeMessageContext.getProps().put(MixAll.CONSUME_CONTEXT_TYPE, returnType.name());
            }
            if (null == status) {
                status = ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
            //处理消费消息的结果
            ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);
        }
    }
```

逻辑比较简单：取得业务方实现的messageListner，调用该逻辑，得到处理结果。

我们继续看messageListner得到的消费结果之后做了什么处理。
处理消费消息结果,入口： ConsumeMessageConcurrentlyService.this.processConsumeResult(status, context, this);


```
    public void processConsumeResult(
        final ConsumeConcurrentlyStatus status,
        final ConsumeConcurrentlyContext context,
        final ConsumeRequest consumeRequest
    ) {
        int ackIndex = context.getAckIndex();
        //此部分switch就是根据消费消息结果做点数据记录
        switch (status) {
            case CONSUME_SUCCESS:
                if (ackIndex >= consumeRequest.getMsgs().size()) {
                    ackIndex = consumeRequest.getMsgs().size() - 1;
                }
                int ok = ackIndex + 1;
                int failed = consumeRequest.getMsgs().size() - ok;
                this.getConsumerStatsManager().incConsumeOKTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), ok);
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(), failed);
                break;
            case RECONSUME_LATER:
                ackIndex = -1;
                this.getConsumerStatsManager().incConsumeFailedTPS(consumerGroup, consumeRequest.getMessageQueue().getTopic(),
                    consumeRequest.getMsgs().size());
                break;
            default:
                break;
        }

        //此部分switch根据根据消费模式不同对消费失败的消息做不同处理
        switch (this.defaultMQPushConsumer.getMessageModel()) {
            case BROADCASTING:
                for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                    MessageExt msg = consumeRequest.getMsgs().get(i);
                    log.warn("BROADCASTING, the message consume failed, drop it, {}", msg.toString());
                }
                break;
            case CLUSTERING:
                List<MessageExt> msgBackFailed = new ArrayList<MessageExt>(consumeRequest.getMsgs().size());
                //处理失败的消息，没有失败消息则不会进该for循环
                for (int i = ackIndex + 1; i < consumeRequest.getMsgs().size(); i++) {
                    MessageExt msg = consumeRequest.getMsgs().get(i);
                    //失败的消息直接将失败的消息重新发送至broker
                    boolean result = this.sendMessageBack(msg, context);
                    if (!result) {
                        msg.setReconsumeTimes(msg.getReconsumeTimes() + 1);
                        msgBackFailed.add(msg);
                    }
                }
                //msgBackFailed有值，也就是上一步sendMessageBack依然失败了，则走另外的方式处理消息失败的消息（submitConsumeRequestLater）
                if (!msgBackFailed.isEmpty()) {
                    consumeRequest.getMsgs().removeAll(msgBackFailed);
                    this.submitConsumeRequestLater(msgBackFailed, consumeRequest.getProcessQueue(), consumeRequest.getMessageQueue());
                }
                break;
            default:
                break;
        }
        //更新offset
        long offset = consumeRequest.getProcessQueue().removeMessage(consumeRequest.getMsgs());
        if (offset >= 0 && !consumeRequest.getProcessQueue().isDropped()) {
            //这里实际只是更新RemoteBrokerOffsetStore.offsetTable存储的offset值（实际会由定时线程提交到broker，入口：MQClientInstance.this.persistAllConsumerOffset();）
            this.defaultMQPushConsumerImpl.getOffsetStore().updateOffset(consumeRequest.getMessageQueue(), offset, true);
        }
    }
```


## 总结

关于消费消息的源码，我们得到以下关键信息：

> 1、调用业务实现的消费消息逻辑，得到消费消息结果（即使消费超时了，也最终会根据messageListner执行返回的结果来决定是否重新消费消息，也就回答了我们开篇提出的问题2以及问题3）；
> 
> 2、根据消费模式不同对消费失败的结果做不同处理。对于广播模式，失败了消息会直接丢弃；集群模式会重新消费消息。
> 
> 3、需要重新消费的消息会将消息重新发送至broker。
> 
> 4、不管消息消费是否成功，都会更新consumerGroup消费到的offset。

分析到这里，我们基本找到了重新消费消息的入口，以及大体思路；以及offset更新的入口，接下来我们看更新offset的过程。

# offset的更新过程


```
    @Override
    public void updateOffset(MessageQueue mq, long offset, boolean increaseOnly) {
        if (mq != null) {
            //实际每个messageQuene消费到的offset存放在offsetTable缓存map中。
            AtomicLong offsetOld = this.offsetTable.get(mq);
            if (null == offsetOld) {
                offsetOld = this.offsetTable.putIfAbsent(mq, new AtomicLong(offset));
            }

            if (null != offsetOld) {
                if (increaseOnly) {
                    MixAll.compareAndIncreaseOnly(offsetOld, offset);
                } else {
                    offsetOld.set(offset);
                }
            }
        }
    }
```
消费完消息之后会将消费之后的messageQuene对应的offset存放在缓存map中。

这里我们可以猜测，offset的更新应该会有定时线程提交给broker，根据前面分析的consumer启动过程可以定位到：


```

        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.persistAllConsumerOffset();
                } catch (Exception e) {
                    log.error("ScheduledTask persistAllConsumerOffset exception", e);
                }
            }
        }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);
```

此定时器最终会委托OffsetStore的实现类，将缓存的offsetTable提交给broker。

```
    @Override
    public void persistAll(Set<MessageQueue> mqs) {
        if (null == mqs || mqs.isEmpty())
            return;

        final HashSet<MessageQueue> unusedMQ = new HashSet<MessageQueue>();
        if (!mqs.isEmpty()) {
            for (Map.Entry<MessageQueue, AtomicLong> entry : this.offsetTable.entrySet()) {
                MessageQueue mq = entry.getKey();
                AtomicLong offset = entry.getValue();
                if (offset != null) {
                    if (mqs.contains(mq)) {
                        try {
                            this.updateConsumeOffsetToBroker(mq, offset.get());
                            log.info("[persistAll] Group: {} ClientId: {} updateConsumeOffsetToBroker {} {}",
                                this.groupName,
                                this.mQClientFactory.getClientId(),
                                mq,
                                offset.get());
                        } catch (Exception e) {
                            log.error("updateConsumeOffsetToBroker exception, " + mq.toString(), e);
                        }
                    } else {
                        unusedMQ.add(mq);
                    }
                }
            }
        }

        if (!unusedMQ.isEmpty()) {
            for (MessageQueue mq : unusedMQ) {
                this.offsetTable.remove(mq);
                log.info("remove unused mq, {}, {}", mq, this.groupName);
            }
        }
    }
```

分析到最后我们基本可以得出结论：

> 1、由于是先消费消息，再提交offset，因此可能存在消费完消息之后，提交offset失败；当然这种可能性极低（因为消费完之后提交offset只是做了内存操作）
> 2、由于offset是先存在内存中，定时器间隔几秒提交给broker，消费之后的offset是完全存在可能丢失的风险（例如consumer端突然宕机），从而会导致没有提交offset到broker，再次启动consumer客户端时，会重复消费。

基于2点，这里也就回答了前面的问题4，所以consumer的业务消费代码一定要保证幂等。

这里可能有疑问，如果offset都是依赖定时器提交broker，那应用正常关闭的时候，是不是offset丢失的概率很大？其实不然，我们观察consumer的shutdown方法会主动触发一次持久化offset到broker的方法。

![image](http://static.silence.work/20190307-1.png)

**因此，当关闭应用的时候，consumer实例一定要记得关闭，否则offset不会主动触发提交给broker。**

# 消费消息重试原理

我们从消费消息部分已经了解到消费失败的消息，会重新发送给broker。同时，如果重新发送给broker也依然失败了，则会将原来失败的消息交给定时任务重新尝试消费。

那我们这里重点看broker如何处理消费失败的消息。

consumer端将失败消息重新发送给broker会使用RequestCode.CONSUMER_SEND_MSG_BACK类型的RemotingCommand。

![image](http://static.silence.work/20190307-2.png)

我们可以在broker根据处理该类型的请求找到broker处理消费消息失败消息的入口：SendMessageProcessor

```
public RemotingCommand processRequest(ChannelHandlerContext ctx,
                                          RemotingCommand request) throws RemotingCommandException {
        SendMessageContext mqtraceContext;
        switch (request.getCode()) {
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                //处理重试的消息
                return this.consumerSendMsgBack(ctx, request);
            default:
                //...
        }
    }
```


```
  private RemotingCommand consumerSendMsgBack(final ChannelHandlerContext ctx, final RemotingCommand request)
        throws RemotingCommandException {
        final RemotingCommand response = RemotingCommand.createResponseCommand(null);
        final ConsumerSendMsgBackRequestHeader requestHeader =
            (ConsumerSendMsgBackRequestHeader)request.decodeCommandCustomHeader(ConsumerSendMsgBackRequestHeader.class);

        SubscriptionGroupConfig subscriptionGroupConfig =
            this.brokerController.getSubscriptionGroupManager().findSubscriptionGroupConfig(requestHeader.getGroup());
        //将原消息的topic名称定义为为：%RETRY%+groupName
        String newTopic = MixAll.getRetryTopic(requestHeader.getGroup());
        //计算重试消息存放的queneid
        int queueIdInt = Math.abs(this.random.nextInt() % 99999999) % subscriptionGroupConfig.getRetryQueueNums();

        TopicConfig topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
            newTopic,
            subscriptionGroupConfig.getRetryQueueNums(),
            PermName.PERM_WRITE | PermName.PERM_READ, topicSysFlag);

        MessageExt msgExt =this.brokerController.getMessageStore().lookMessageByOffset(requestHeader.getOffset());

        int delayLevel = requestHeader.getDelayLevel();
        //如果超过了消费重试的次数
        if (msgExt.getReconsumeTimes() >= maxReconsumeTimes
            || delayLevel < 0) {
            newTopic = MixAll.getDLQTopic(requestHeader.getGroup());
            queueIdInt = Math.abs(this.random.nextInt() % 99999999) % DLQ_NUMS_PER_GROUP;

            topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(newTopic,
                DLQ_NUMS_PER_GROUP,
                PermName.PERM_WRITE, 0
            );
            if (null == topicConfig) {
                response.setCode(ResponseCode.SYSTEM_ERROR);
                response.setRemark("topic[" + newTopic + "] not exist");
                return response;
            }
        } else {
            if (0 == delayLevel) {
                delayLevel = 3 + msgExt.getReconsumeTimes();
            }
            //根据重试次数设置消息的延迟时长（这里基本可以猜测重试消息的本质利用了延迟消息）
            msgExt.setDelayTimeLevel(delayLevel);
        }

        MessageExtBrokerInner msgInner = new MessageExtBrokerInner();
        msgInner.setTopic(newTopic);
        msgInner.setBody(msgExt.getBody());
        msgInner.setFlag(msgExt.getFlag());
        MessageAccessor.setProperties(msgInner, msgExt.getProperties());
        msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgExt.getProperties()));
        msgInner.setTagsCode(MessageExtBrokerInner.tagsString2tagsCode(null, msgExt.getTags()));

        msgInner.setQueueId(queueIdInt);
        msgInner.setSysFlag(msgExt.getSysFlag());
        msgInner.setBornTimestamp(msgExt.getBornTimestamp());
        msgInner.setBornHost(msgExt.getBornHost());
        msgInner.setStoreHost(this.getStoreHost());
        msgInner.setReconsumeTimes(msgExt.getReconsumeTimes() + 1);

        String originMsgId = MessageAccessor.getOriginMessageId(msgExt);
        MessageAccessor.setOriginMessageId(msgInner, UtilAll.isBlank(originMsgId) ? msgExt.getMsgId() : originMsgId);

        //存储消息（封装重试消息的相关信息作为延迟消息存储）
        PutMessageResult putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
        if (putMessageResult != null) {
            switch (putMessageResult.getPutMessageStatus()) {
                case PUT_OK:
                    String backTopic = msgExt.getTopic();
                    String correctTopic = msgExt.getProperty(MessageConst.PROPERTY_RETRY_TOPIC);
                    if (correctTopic != null) {
                        backTopic = correctTopic;
                    }

                    this.brokerController.getBrokerStatsManager().incSendBackNums(requestHeader.getGroup(), backTopic);

                    response.setCode(ResponseCode.SUCCESS);
                    response.setRemark(null);

                    return response;
                default:
                    break;
            }

            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark(putMessageResult.getPutMessageStatus().name());
            return response;
        }

        response.setCode(ResponseCode.SYSTEM_ERROR);
        response.setRemark("putMessageResult is null");
        return response;
    }
```

根据处理失败消息过程可以得到两个非常关键的信息：
> 1、失败的消息会将topic转换为%RETRY%+groupName的topic名称；

> 2、此失败的消息会设置延迟级别（转化成了延迟消息）；

如果了解延迟消息的原理（不明白可以移步至[RocketMQ-延迟消息的使用与原理分析](http://silence.work/2018/12/16/RocketMQ-%E5%BB%B6%E8%BF%9F%E6%B6%88%E6%81%AF%E7%9A%84%E4%BD%BF%E7%94%A8%E4%B8%8E%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)），我们基本可以回答了前面提到的问题5：

**消息消费失败，消费端conusmer会将该消息重新再发给broker，broker接收到该消息，会作为延迟消息存放起来（因为重试消息是有时间间隔），利用延迟消息的功能，broker端到了延迟的时间点，再将该延迟消息转换为重试消息（%RETRY%+consumerGroup），此时consumer端可见，从而会拉取到该重试消息，从而达到延迟重复消费的目的。**