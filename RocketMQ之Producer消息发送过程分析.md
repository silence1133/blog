title: RocketMQ之Producer消息发送过程分析
author: Silence
date: 2019-12-30 17:07:14
tags:
---
# 启动过程

Producer的启动过程本质就是围绕MQClientInstance的构建过程。

**入口：DefaultMQProducerImpl.start**

## 检查配置

检查producer是否有指定producerGroup，没有会报错

## 创建MQClientInstance实例

**MQClientInstance的理解：**

MQClientInstance封装了RocketMQ网络处理的API，是Producer、Consumer与NameServer、broker打交道的网络通道。

同一个jvm中不同consumer和producer获取的MQClientInstance实例是同一个。

**创建过程：**

1、先设置clientId，clientId组成有ip@pid@unitname，unitname可选。例如：10.204.246.26@27066。

2、根据clientID从factoryTable中获取，也就是缓存map中获取。没有则new一个MQClientInstance。

## 注册当前producer

也就是将当前producer缓存到MQClientInstance实例的producerTable成员变量中。

key为producer的名称，value为当前Producer的实例

```
    private final ConcurrentMap<String/* group */, MQProducerInner> producerTable = new ConcurrentHashMap<String, MQProducerInner>();
```

## 启动MQClientInstance
入口：MQClientInstance.start()

启动过程会处理很多逻辑，主要是启动各种定时线程，由于MQClientInstance同时承载了Producer和Cosnumer与Broker的打交道的职能，因此这里启动的很多定时任务都是consumer端使用的，我们这里重点关注this.startScheduledTask()，代码如下：


```
   private void startScheduledTask() {
        //如果没有namesrvAddr，则启动定时器获取namesrvAddr地址（2分钟执行1次）
        if (null == this.clientConfig.getNamesrvAddr()) {
            this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

                @Override
                public void run() {
                    try {
                        MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
                    } catch (Exception e) {
                        log.error("ScheduledTask fetchNameServerAddr exception", e);
                    }
                }
            }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
        }
        //每30秒更新一次所有的topic的路由信息，延迟10毫秒执行
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.updateTopicRouteInfoFromNameServer();
                } catch (Exception e) {
                    log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
                }
            }
        }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);

        // 每30秒对下线的broker进行移除
        // 每30秒发送一次心跳
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.cleanOfflineBroker();
                    MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
                } catch (Exception e) {
                    log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
                }
            }
        }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);

        // 持久化消费端offSet，每5s执行一次
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
        //定时调整消费者端线程池大小
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.adjustThreadPool();
                } catch (Exception e) {
                    log.error("ScheduledTask adjustThreadPool exception", e);
                }
            }
        }, 1, 1, TimeUnit.MINUTES);
    }
```

**这里我们可以得出以下重要结论**：

1、Producer感知topic的路由信息变化是通过定时线程，每30s去nameserver拉取，然后对本地缓存的路由信息更新。

2、每30s向Broker发送一次心跳。

3、每30s更新本地缓存的存活broker。


# 发送消息过程

入口：DefaultMQProducerImpl.sendDefaultImpl

## 消息校验

我们简单看下消息Message类封装了哪些字段：

```
public class Message implements Serializable {
    //指定的topic
    private String topic;
    //RocketMQ特有的消息tag，用于服务端过滤消息
    private int flag;
    //map存放其他参数，方便拓展
    private Map<String, String> properties;
    //消息体
    private byte[] body;
    //事务消息用
    private String transactionId;
}
```
消息的校验主要就是校验topic以及消息体的长度，发送消息的最大长度不能超过4M。

## 从nameserver查找路由信息

```
TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
```
先从缓存里面取，缓存没有则从nameserver取，取到之后会放到缓存。

```
    //存放topic路由信息的map，key为topic
    private final ConcurrentMap<String, TopicPublishInfo> topicPublishInfoTable =new ConcurrentHashMap<String, TopicPublishInfo>();
```

这里我们重点认识下缓存在Producer端的路由信息类TopicPublishInfo

### TopicPublishInfo

```
public class TopicPublishInfo {
    //是否是顺序消息
    private boolean orderTopic = false;
    //是否存在路由信息
    private boolean haveTopicRouterInfo = false;
    //改topic对应的逻辑队列，每一个逻辑队列就对应一个MessageQueue
    private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
    //用于选择消息队列的值，每选择一次消息队列，该值会自增1。
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    // topic路由元数据，如borker的元信息和队列的元信息
    private TopicRouteData topicRouteData;
    //....
    //....
}

//数据结构如下：
TopicPublishInfo [
orderTopic=false, 
messageQueueList=[MessageQueue [topic=TopicTest, brokerName=Silence.local, queueId=0], MessageQueue [topic=TopicTest, brokerName=Silence.local, queueId=1], MessageQueue [topic=TopicTest, brokerName=Silence.local, queueId=2], MessageQueue [topic=TopicTest, brokerName=Silence.local, queueId=3]], 
sendWhichQueue=ThreadLocalIndex{threadLocalIndex=null},
haveTopicRouterInfo=true]

```
### TopicRouteData

TopicRouteData是存放在Nameserver端缓存的路由信息类，在分析Nameserver的时候有提过。

```
public class TopicRouteData extends RemotingSerializable {
    private String orderTopicConf;
    //队列元信息，如readQueueNums、writeQueueNums、perm等
    private List<QueueData> queueDatas;
    //broker的原信息brokerName、brokerAddrs
    private List<BrokerData> brokerDatas;
    //broker上过滤服务器地址列表
    private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}
```

## 选择具体的队列

其实就是从上一步获取的topicPublishInfo的messageQueueList中选择具体的MessageQuene。

```
MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
```

前面我们说过客户端是每30s才去Nameserver更新最新的路由信息（并且Nameserver维护的存活broker也不是实时的，120s。。），结合Nameserver更新存活broker列表的策略，**客户端感知有故障的broker至少需要3分钟**。

意味着producer在发送消息时，并不能立刻感知到有故障的broker，那么如何规避有故障的broker变得尤为重要，
接下来我们重点看选择队列的时候时是如何规避掉有故障的broker。

在选择具体队列时，会根据sendLatencyFaultEnable参数不同，决定是否选择启用broker故障延迟机制，默认不启用。

```
 public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
        //启用broker故障延迟机制
        if (this.sendLatencyFaultEnable) {
           return .....
        }
        //不启用，默认走这里
        return tpInfo.selectOneMessageQueue(lastBrokerName);
    }
```

### 不启用broker故障延迟

Producer的默认策略，sendLatencyFaultEnable=false

TopicPublishInfo.selectOneMessageQueue方法

```
    public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
        if (lastBrokerName == null) {
            return selectOneMessageQueue();
        } else {
            int index = this.sendWhichQueue.getAndIncrement();
            for (int i = 0; i < this.messageQueueList.size(); i++) {
                int pos = Math.abs(index++) % this.messageQueueList.size();
                if (pos < 0)
                    pos = 0;
                MessageQueue mq = this.messageQueueList.get(pos);
                //过滤掉上次发送消息失败的队列
                if (!mq.getBrokerName().equals(lastBrokerName)) {
                    return mq;
                }
            }
            return selectOneMessageQueue();
        }
    }
```


取哪个messageQueue的逻辑是：当前线程有个ThreadLocal变量，存放了一个随机数，然后根据队列长度取模。

```
   public MessageQueue selectOneMessageQueue() {
        int index = this.sendWhichQueue.getAndIncrement();
        int pos = Math.abs(index) % this.messageQueueList.size();
        if (pos < 0)
            pos = 0;
        return this.messageQueueList.get(pos);
    }
```

当消息第一次发送失败时，lastBrokerName会存放当前选择失败的broker，通过重试，此时lastBrokerName有值，代表上次选择的boker发送失败，则重新对sendWhichQueue本地线程变量+1，遍历选择消息队列，直到不是上次的broker，也就是为了规避上次发送失败的broker的逻辑所在。

这里在规避发送到有问题的broker上的策略逻辑虽然比较简单，但其实是有弊端的，接下来我们来分析为什么存在弊端，以及Producer通过启用broker故障延迟是如何解决这里存在的弊端的。

### 启用broker故障延迟

sendLatencyFaultEnable=true；通过设置Producer的sendLatencyFaultEnable值指定。

**什么是故障转移机制？**

保证在一定的时间范围内不选择有故障的broker。当不开启故障转移机制，意味着每一次发送新的消息都有可能选择到故障的broker，**从而引发消息发送的重试机制**，导致不必要的性能损耗，因此故障转移就是为了避免每次发送消息通过重试来规避故障的broker，而是直接保证在一定范围内不再选择到有故障的队列。

当发送消息失败时，都会调用updateFaultItem方法，目的就是封装有故障的broker。

**DefaultMQProducerImpl#updateFaultItem**

逻辑主要就是封装发生故障的brokerName以及恢复故障的时间。关于故障的恢复时间如何计算，这里不做说明，是一个预估时间。

```
    public void updateFaultItem(final String brokerName, final long currentLatency, boolean isolation) {
        if (this.sendLatencyFaultEnable) {
            //估算故障不可用的时长
            long duration = computeNotAvailableDuration(isolation ? 30000 : currentLatency);
            //根据brokerName，currentLatency，故障不可用时长封装FaultItem对象。
            this.latencyFaultTolerance.updateFaultItem(brokerName, currentLatency, duration);
        }
    }
    
    class FaultItem implements Comparable<FaultItem> {
        //有故障的brokerName
        private final String name;
        //当前发生故障时发生消息耗时
        private volatile long currentLatency;
        //预估的故障恢复时间
        //=duration+当前时间
        private volatile long startTimestamp;
        //...
        //...
    }
```

**我们再看开启故障转移获取队列的方法**：

相比未开启故障转移时获取队列，主要多了latencyFaultTolerance.isAvailable(mq.getBrokerName())的逻辑，该方法其实就是判断当前实际是否已经过了前面设置的故障恢复时间。

```
    public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
        if (this.sendLatencyFaultEnable) {
            try {
                //遍历队列，获取没有故障的broker队列或者故障已经恢复的队列
                int index = tpInfo.getSendWhichQueue().getAndIncrement();
                for (int i = 0; i < tpInfo.getMessageQueueList().size(); i++) {
                    int pos = Math.abs(index++) % tpInfo.getMessageQueueList().size();
                    if (pos < 0)
                        pos = 0;
                    MessageQueue mq = tpInfo.getMessageQueueList().get(pos);
                    if (latencyFaultTolerance.isAvailable(mq.getBrokerName())) {
                        if (null == lastBrokerName || mq.getBrokerName().equals(lastBrokerName))
                            return mq;
                    }
                }
                //如果所有队列都不可用，随机选择一个队列
                final String notBestBroker = latencyFaultTolerance.pickOneAtLeast();
                int writeQueueNums = tpInfo.getQueueIdByBroker(notBestBroker);
                if (writeQueueNums > 0) {
                    final MessageQueue mq = tpInfo.selectOneMessageQueue();
                    if (notBestBroker != null) {
                        mq.setBrokerName(notBestBroker);
                        mq.setQueueId(tpInfo.getSendWhichQueue().getAndIncrement() % writeQueueNums);
                    }
                    return mq;
                } else {
                    latencyFaultTolerance.remove(notBestBroker);
                }
            } catch (Exception e) {
                log.error("Error occurred when selecting message queue", e);
            }

            return tpInfo.selectOneMessageQueue();
        }

        return tpInfo.selectOneMessageQueue(lastBrokerName);
    }

```



## 发送消息

```
 //for循环来重试
 for (; times < timesTotal; times++) {
 				//保存上一次选择的队列，重试规避故障broker需要使用到
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                    try {
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        //不同的发送消息方式
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            //同步刷屏的情况下，只有isRetryAnotherBrokerWhenNotStoreOK才重试
                            case SYNC:
                                if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
                                    if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
                                        continue;
                                    }
                                }
                                return sendResult;
                            default:
                                break;
                        }
                    } catch (RemotingException e) {
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        continue;
                    } catch (MQClientException e) {
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        continue;
                    } catch (MQBrokerException e) {
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, true);
                        exception = e;
                        switch (e.getResponseCode()) {
                            case ResponseCode.TOPIC_NOT_EXIST:
                            case ResponseCode.SERVICE_NOT_AVAILABLE:
                            case ResponseCode.SYSTEM_ERROR:
                            case ResponseCode.NO_PERMISSION:
                            case ResponseCode.NO_BUYER_ID:
                            case ResponseCode.NOT_IN_CURRENT_UNIT:
                                continue;
                            default:
                                if (sendResult != null) {
                                    return sendResult;
                                }

                                throw e;
                        }
                    } catch (InterruptedException e) {
                        this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
                        throw e;
                    }
                } 
            }
```

消息发送的核心逻辑在sendKernelImpl方法，这里简单归纳下，主要做了以下几件事：

> 1、根据对应的messageQuene获取broker网络地址。
> 
> 2、为消息分配全局的唯一id
> 
> 3、压缩消息，如果消息体大小超过compressMsgBodyOverHowmuch配置的值（默认4K），则进行压缩。
> 
> 4、是否存在发送消息钩子sendMessageHookList，存在则执行钩子。
> 
> 5、构建消息发送的请求头SendMessageRequestHeader
> 
> 6、根据不同发送方式进行消息的发送。如果失败进入循环重试。
> 
> - **同步发送（SYNC）**：同步阻塞等待broker处理完消息后返回结果。
> 
> - **异步发送（ASYNC）**：不阻塞等待broker处理消息的结果，通过提供回调方法，响应消息发送结果。这种方式的发送，RocketMQ做了并发控制，通过clientAsyncSemaphoreValue参数控制，默认值是65535。异步发送消息的消息重试次数是通过retryTimesWhenSendAsyncFailed控制的，但如果网络出现异常是无法发生重试的。
> 
> - **单向发送（ONEWAY）**：不关心消息发送是否成功，只管发送。
> 
> 7、继续判断发送消息钩子，有则执行钩子的after逻辑。