title: RocketMQ保证消息不丢失的手段
author: Silence
tags:
  - Rocket
  - 消息中间件
categories:
  - 消息中间件
date: 2019-05-04 01:12:00
---
# 消息整体处理过程

![image](http://static.silence.work/2019050401.png)

这里我们将消息的整体处理阶段分为3个阶段进行分析：

- Producer发送消息阶段。

- Broker处理消息阶段。

- Consumer消费消息阶段。

# Producer发送消息阶段

发送消息阶段涉及到Producer到broker的网络通信，因此丢失消息的几率一定会有，那RocketMQ在此阶段用了哪些手段保证消息不丢失了（或者说降低丢失的可能性）。

#### 手段一：提供SYNC的发送消息方式，等待broker处理结果。

RocketMQ提供了3种发送消息方式，分别是：


```
    //同步发送
    SendResult send(final Message msg) throws MQClientException, RemotingException, MQBrokerException,
        InterruptedException;

    //异步发送
    void send(final Message msg, final SendCallback sendCallback) throws MQClientException,
        RemotingException, InterruptedException;

    //oneway发送（不等结果）
    void sendOneway(final Message msg) throws MQClientException, RemotingException,
        InterruptedException;
```


- **同步发送**：Producer 向 broker 发送消息，阻塞当前线程等待 broker 响应 发送结果。

- **异步发送**：Producer 首先构建一个向 broker 发送消息的任务，把该任务提交给线程池，等执行完该任务时，回调用户自定义的回调函数，执行处理结果。

- **Oneway发送**：Oneway 方式只负责发送请求，不等待应答，Producer 只负责把请求发出去，而不处理响应结果。

我们在调用producer.send方法时，不指定回调方法，则默认采用同步发送消息的方式，这也是丢失几率最小的一种发送方式。

#### 手段二：发送消息如果失败或者超时，则重新发送。

发送重试源码如下，本质其实就是一个for循环，当发送消息发生异常的时候重新循环发送。默认重试3次，重试次数可以通过producer指定。

```
//只有同步发送的方式才会重试
int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
 //for循环来重试
 for (; times < timesTotal; times++) {
                MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
                    try {
                        sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, timeout - costTime);
                        //不同的发送消息方式
                        switch (communicationMode) {
                            case ASYNC:
                                return null;
                            case ONEWAY:
                                return null;
                            
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
                        continue;
                    } catch (MQClientException e) {
                        continue;
                    } catch (MQBrokerException e) {
                        exception = e;
                    }
                } 
            }
```

#### 手段三：broker提供多master模式，即使某台broker宕机了，保证消息可以投递到另外一台正常的broker上。

如果broker只有一个节点，则broker宕机了，即使producer有重试机制，也没用，因此利用多主模式，当某台broker宕机了，换一台broker进行投递。

### 总结

producer消息发送方式虽然有3种，但为了减小丢失消息的可能性尽量采用同步的发送方式，同步等待发送结果，利用**同步发送+重试机制+多个master节点**，尽可能减小消息丢失的可能性。


# Broker处理消息阶段

#### 手段四：提供同步刷盘的策略

```
public enum FlushDiskType {
    SYNC_FLUSH, //同步刷盘
    ASYNC_FLUSH//异步刷盘（默认）
}
```


我们知道，当消息投递到broker之后，会先存到page cache，然后根据broker设置的刷盘策略是否立即刷盘，也就是如果刷盘策略为异步，broker并不会等待消息落盘就会返回producer成功，也就是说当broker所在的服务器突然宕机，则会丢失部分页的消息。

关于broker的处理过程可以参考我之前的文章：[RocketMQ Broker端处理消息过程分析](http://silence.work/2019/05/03/RocketMQ-Broker%E7%AB%AF%E5%A4%84%E7%90%86%E6%B6%88%E6%81%AF%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/)

#### 手段五：提供主从模式，同时主从支持同步双写

即使broker设置了同步刷盘，如果主broker磁盘损坏，也是会导致消息丢失。
因此可以给broker指定slave，同时设置master为SYNC_MASTER，然后将slave设置为同步刷盘策略。

此模式下，producer每发送一条消息，都会等消息投递到master和slave都落盘成功了，broker才会当作消息投递成功，保证休息不丢失。

#### 总结

在broker端，消息丢失的可能性主要在于刷盘策略和同步机制。
RocketMQ默认broker的刷盘策略为异步刷盘，如果有主从，同步策略也默认的是异步同步，这样子可以提高broker处理消息的效率，但是会有丢失的可能性。因此可以通过**同步刷盘策略+同步slave策略+主从的方式**解决丢失消息的可能。

# Consumer消费消息阶段

#### 手段六：consumer默认提供的是At least Once机制

从producer投递消息到broker，即使前面这些过程保证了消息正常持久化，但如果consumer消费消息没有消费到也不能理解为消息绝对的可靠。因此RockerMQ默认提供了At least Once机制保证消息可靠消费。

**何为At least Once？**

Consumer先pull 消息到本地，消费完成后，才向服务器返回ack。

通常消费消息的ack机制一般分为两种思路：

> 1、先提交后消费；
> 
> 2、先消费，消费成功后再提交；

思路一可以解决重复消费的问题但是会丢失消息，因此Rocket默认实现的是思路二，由各自consumer业务方保证幂等来解决重复消费问题。


#### 手段七：消费消息重试机制

当消费消息失败了，如果不提供重试消息的能力，则也不能算完全的可靠消费，因此RocketMQ本身提供了重新消费消息的能力。

#### 总结

consumer端要保证消费消息的可靠性，主要通过**At least Once+消费重试机制**保证。


# 参考资料

[RocketMQ 生产者 Producer 发送消息三种方式分析](https://www.jianshu.com/p/1cf2efc2a89d)

[RocketMQ 开发挃南](http://www.sxt.cn/ueditor/php/upload/file/20170901/1504248466272324.pdf)
