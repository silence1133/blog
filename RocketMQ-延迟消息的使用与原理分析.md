title: RocketMQ 延迟消息的使用与原理分析
author: Silence
tags:
  - RocketMQ
  - 消息中间件
categories:
  - 消息中间件
date: 2018-12-16 19:09:00
---
# 延迟消息的使用

使用比较简单，指定message的DelayTimeLevel即可。示例代码如下：


```
    Message msg = new Message("DelayTopicTest","TagA",("Hello RocketMQ ").getBytes(RemotingHelper.DEFAULT_CHARSET) );
    //设置延迟级别，注意这里的3不是代表延迟3s
    msg.setDelayTimeLevel(3);
    SendResult sendResult = producer.send(msg);
```
目前rockatmq支持的延迟时间有：
> 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h

以上支持的延迟时间在msg.setDelayTimeLevel对应的级别依次是1，2，3。。。。

# 实现原理


延迟队列的核心思路是：所有的延迟消息由producer发出之后，都会存放到同一个topic（SCHEDULE_TOPIC_XXXX）下，不同的延迟级别会对应不同的队列序号，当延迟时间到之后，由定时线程读取转换为普通的消息存的真实指定的topic下，此时对于consumer端此消息才可见，从而被consumer消费。

延迟消息存放的结构如下图所示：

```
consumequeue
├── SCHEDULE_TOPIC_XXXX
│   ├── 0
│   │   └── 00000000000000000000
│   ├── 1
│   │   └── 00000000000000000000
│   ├── 2
│   │   └── 00000000000000000000
│   ├── 3
│   │   └── 00000000000000000000
│   ├── 4
│   │   └── 00000000000000000000
    .....
    .....
├── DelayTopicTest
│   ├── 0
│   │   └── 00000000000000000000
│   ├── 1
│   │   └── 00000000000000000000
│   ├── 2
│   │   └── 00000000000000000000
│   └── 3
│       └── 00000000000000000000
```


其中不同的延迟级别放在不同的队列序号下（**queueId=delayLevel-1**）。每一个延迟级别对应的延迟消息转换为普通消息的位置标识存放在~/store/config/delayOffset.json文件内。

key为对应的延迟级别，value对应不同延迟级别转换为普通消息的offset值。

```
{
	"offsetTable":{1:1,2:1,3:11,4:1,5:1,6:1,7:1,8:1,9:1,10:1,11:1,12:1,13:1,14:1,15:0,16:0,17:0,18:0}
}
```
# 源码分析

## 入口：ScheduleMessageService.start

broker启动的时候会调用此方法。

```
    public void start() {
        //1. 根据支持的各种延迟级别，添加不同延迟时间的TimeTask
        for (Map.Entry<Integer, Long> entry : this.delayLevelTable.entrySet()) {
            Integer level = entry.getKey();
            Long timeDelay = entry.getValue();
            //每一个延迟级别对应已经读取为普通消息的offset值
            Long offset = this.offsetTable.get(level);
            if (null == offset) {
                offset = 0L;
            }
            if (timeDelay != null) {
                this.timer.schedule(new DeliverDelayedMessageTimerTask(level, offset), 1000L);
            }
        }
        //2. 添加一个10s执行一次的TimeTask
        this.timer.scheduleAtFixedRate(new TimerTask() {
            @Override
            public void run() {
                try {
                    ScheduleMessageService.this.persist();
                } catch (Throwable e) {
                    log.error("scheduleAtFixedRate flush exception", e);
                }
            }
        }, 10000, this.defaultMessageStore.getMessageStoreConfig().getFlushDelayOffsetInterval());
    }
```
**从以上源码基本可以得出几个结论：**

> 1、通过jdk自带的Timer类开启一个timer定时器，在这个timer类添加了多个TimeTask。其中不同的延迟级别都对应DeliverDelayedMessageTimerTask的不同实例。

> 2、TimeTask分为两类：DeliverDelayedMessageTimerTask（每秒执行1次）和 ScheduleMessageService.this.persist()（每10秒是执行一次）

> 3、每一个延迟级别对应一个offset，这个offset是干嘛的呢？（先抛结论：这个offset的值代表每个级别的延迟队列已经转换为普通消息的位置）


## 两类TimeTask的作用

### 1、DeliverDelayedMessageTimerTask

作用：扫描延迟消息队列（SCHEDULE_TOPIC_XXXX）的消息，将该延迟消息转换为指定的topic的消息。

**核心代码：ScheduleMessageService.executeOnTimeup**

```
  public void executeOnTimeup() {
	    //读取队列SCHEDULE_TOPIC_XXXX，其中不同的延迟级别对应不同的队列id（queueId=delayLevel-1）
            ConsumeQueue cq =ScheduleMessageService.this.defaultMessageStore.findConsumeQueue(“SCHEDULE_TOPIC_XXXX”,delayLevel2QueueId(delayLevel));
            long failScheduleOffset = offset;
            if (cq != null) {
                SelectMappedBufferResult bufferCQ = cq.getIndexBuffer(this.offset);
                if (bufferCQ != null) {
                    try {
                        long nextOffset = offset;
                        int i = 0;
                        ConsumeQueueExt.CqExtUnit cqExtUnit = new ConsumeQueueExt.CqExtUnit();
			            //循环读取延迟消息
                        for (; i < bufferCQ.getSize(); i += ConsumeQueue.CQ_STORE_UNIT_SIZE) {
                            long offsetPy = bufferCQ.getByteBuffer().getLong();
                            nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                            long countdown = this.correctDeliverTimestamp(now, tagsCode) - now;
			                //只有当延迟消息发送的时间在当前时间之前才处理，否则此消息应该延迟后再处理
                            if (countdown <= 0) {
				                //根据offset值读取SCHEDULE_TOPIC_XXXX队列的消息
                                MessageExt msgExt = ScheduleMessageService.this.defaultMessageStore.lookMessageByOffset(offsetPy, sizePy);
                                if (msgExt != null) {
                                    try {
					                  //将读取的消息转换为真实topic的消息（也就是普通消息）
                                        MessageExtBrokerInner msgInner = this.messageTimeup(msgExt);
					                    //存放此消息
                                        PutMessageResult putMessageResult = ScheduleMessageService.this.defaultMessageStore.putMessage(msgInner);

                                    } catch (Exception e) {
                                    }
                                }
                            } else {
                                ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset),countdown);
                                ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                                return;
                            }
                        } 
			            //计算下一次读取延迟队列的offset，是定时任务下一次从该位置读取延时消息
                        nextOffset = offset + (i / ConsumeQueue.CQ_STORE_UNIT_SIZE);
                        ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel, nextOffset), DELAY_FOR_A_WHILE); 
			            //将下一次读取延迟队列的offset存放到一个缓存map中 
			            ScheduleMessageService.this.updateOffset(this.delayLevel, nextOffset);
                        return;
                    }
                } 
                else {
                    long cqMinOffset = cq.getMinOffsetInQueue();
                    if (offset < cqMinOffset) {
                        failScheduleOffset = cqMinOffset;
                    }
                }
            } 
            ScheduleMessageService.this.timer.schedule(new DeliverDelayedMessageTimerTask(this.delayLevel,failScheduleOffset), DELAY_FOR_A_WHILE);
	}
             
```

以上贴的代码较长，我做了一点精简，这里我梳理下大概思路：

> 1、读取不同延迟级别对应的延迟消息；
> 
> 2、取得对应延迟级别读取的开始位置offset；
> 
> 3、将延迟消息转换为指定topic的普通消息并存放起来。
> 
> 4、修改下一次读取的offset值（修改的只是缓存），并指定下一次转换延迟消息的timetask。
> 

### 2、ScheduleMessageService.this.persist()

将延迟队列扫描处理的进度offset持久化到delayOffset.json文件中。

```
    public synchronized void persist() {
        //读取offsetTable缓存的延迟队列的值
        String jsonString = this.encode(true);
        if (jsonString != null) {
            //读取delayOffset.json的文件地址
            String fileName = this.configFilePath();
            try {
                //持久化到delayOffset.json文件中
                MixAll.string2File(jsonString, fileName);
            } catch (IOException e) {
                log.error("persist file " + fileName + " exception", e);
            }
        }
    }
```




# 总结


通过源码分析我们其实明白了，延迟消息相比普通消息只不过是在broker多了一层消息topic的转换，对于消息的发送和消费和普通消息没有什么差异。

但这里有一点要注意：RocketMQ的延迟消息本身有一个很大的缺点，熟悉java自带的Timer类的小伙伴应该知道一个timer对应只有一个线程，然后来处理不同的timeTask，而RockerMQ本身也确实只new了一个Timer，**也就是说当同时发送的延迟消息过多的时候一个线程处理速度一定是有瓶颈的，因此在实际项目中使用延迟消息一定不要过多依赖，只能作为一个辅助手段**。