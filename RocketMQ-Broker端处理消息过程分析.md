title: RocketMQ Broker端处理消息过程分析
author: Silence
tags:
  - 消息中间件
  - RocketMQ
categories:
  - 消息中间件
date: 2019-05-03 21:27:00
---
# 整体过程


消息投递到broker之后，会先存到broker的堆内存，同时再写到堆外内存，最后根据刷盘策略是否立即将堆外内存的消息刷到磁盘。

同步刷盘：写入page cache之后，会同步等待消息落盘之后才返回消息处理成功；

异步刷盘：写入page cache之后，唤醒后台刷盘线程，不等刷盘结果，直接返回消息处理成功；

![image](http://static.silence.work/20190503-1.png)


# 详细过程分析

```

SendMessageProcessor.processRequest
  -->buildMsgContext(ctx, requestHeader);
  -->this.executeSendMessageHookBefore(ctx, request, mqtraceContext);
  -->sendMessage
    (检查消息)
    -->super.msgCheck(ctx, requestHeader, response);
    (查询topic的配置信息)
    -->this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
    (处理重试消息)
    -->handleRetryAndDLQ(requestHeader, response, request, msgInner, topicConfig)
    -->this.brokerController.getMessageStore().putMessage(msgInner);
        -->this.commitLog.putMessage(msg);
            -->this.mappedFileQueue.getLastMappedFile();
            -->mappedFile.appendMessage(msg, this.appendMessageCallback);
            (如果是同步刷盘则立即调用该方法刷盘)
            -->handleDiskFlush(result, putMessageResult, msg);
            -->异步刷盘的情况下通过唤起FlushRealTimeService的线程刷盘
            (处理主从同步)
            -->handleHA(result, putMessageResult, messageExtBatch);
            
  
```

以上是broker接收消息的源码处理流程，我们这里提炼出几个关键的过程：

## 消息的前置处理

此部分不做介绍

## 获取MappedFile

### 怎么理解MappedFile？

MappedFile：代表了消息存储文件的实体，MappedFile这个类非常关键，基本看几个关键的成员变量就能推测消息存储用到的技术思路是：Mmap 内存映射（将一个程序指定的文件映射进虚拟内存，对文件的读写就变成了对内存的读写）。

![image](http://static.silence.work/20190503-2.png)

![image](http://static.silence.work/20190503-3.png)

FileChannel，MappedByteBuffer：
这两个类代表的是Mmap 这样的内存映射技术，Mmap 能够将文件直接映射到用户态的内存地址，使得对文件的操作不再是 write/read,而转化为直接对内存地址的操作。

Mmap技术本身也有局限性，也就是操作的文件大小不能太大，因此RocketMQ 中限制了单文件大小来避免这个问题。也就是那个filesize定为1G的原因。

### 获取MappedFile的过程

![image](http://static.silence.work/20190504-4.png)

在Broker启动的时候，其会将位于存储目录下的所有消息文件加载到一个列表中，当有消息需要存储的时候，会默认选择列表中的最后一个文件来进行消息的保存。

```
    public MappedFile getLastMappedFile() {
        MappedFile mappedFileLast = null;
        while (!this.mappedFiles.isEmpty()) {
            try {
                //从集合mappedFiles的末尾取一个MappedFile
                mappedFileLast = this.mappedFiles.get(this.mappedFiles.size() - 1);
                break;
            } catch (IndexOutOfBoundsException e) {
                //continue;
            } catch (Exception e) {
                log.error("getLastMappedFile has exception.", e);
                break;
            }
        }
        return mappedFileLast;
    }
```

如果文件不存在或者文件满了（每个文件的大小为1G），则创建一个新的MappedFile

```
if (null == mappedFile || mappedFile.isFull()) {
    mappedFile = this.mappedFileQueue.getLastMappedFile(0); // Mark: NewFile may be cause noise
}
```



## 将消息从堆内内存写入到MappedByteBuffer。

![image](http://static.silence.work/20190505-5.png)

利用Mmap技术，将堆内的消息写入page cache。

第二步操作的byteBuffer就是我们前面说的MappedByteBuffer，mappedByteBuffer的put行为实际是对内存进行的操作，因此这里的写入基本等同于内存的操作。


## 根据刷盘策略处理消息持久化

消息写入page cache之后，接着根据broker设置的策略来决定是否立即将page cache内的消息刷到磁盘。

```
public void handleDiskFlush(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        //  同步刷盘
        if (FlushDiskType.SYNC_FLUSH == this.defaultMessageStore.getMessageStoreConfig().getFlushDiskType()) {
            final GroupCommitService service = (GroupCommitService) this.flushCommitLogService;
            if (messageExt.isWaitStoreMsgOK()) {
                GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                //唤醒GroupCommitServic刷盘线程
                service.putRequest(request);
                //等待GroupCommitServic刷盘线程处理完，这里设置了默认等待的超时时间是5s
                boolean flushOK = request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                if (!flushOK) {
                    log.error("do groupcommit, wait for flush failed, topic: " + messageExt.getTopic() + " tags: " + messageExt.getTags()
                        + " client address: " + messageExt.getBornHostString());
                    putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_DISK_TIMEOUT);
                }
            } else {
                service.wakeup();
            }
        }
        // 异步刷盘
        else {
            if (!this.defaultMessageStore.getMessageStoreConfig().isTransientStorePoolEnable()) {
                //唤醒FlushRealTimeService刷盘线程 flushCommitLogService.wakeup();
            } else {
                commitLogService.wakeup();
            }
        }
    }
```

如果是同步刷盘，则唤醒GroupCommitService线程去刷盘，但是主线程会等待，等待GroupCommitService线程刷盘完后通知主线程，如果主线程等待的时间超过了超时时间，则会返回PutMessageStatus.FLUSH_DISK_TIMEOUT。

如果是异步刷盘，则唤醒FlushRealTimeService线程，不等待。

不管是同步还是异步，本质都是调用了
mappedByteBuffer.force()，也就是强制操作系统将page cache的内容刷到磁盘。

其中异步刷盘还有一个策略：就是当要刷新的页数已经超过配置的MessageStoreConfig.flushCommitLogLeastPages才会刷盘(默认是4)。（而同步刷盘的线程只要被唤起，就会刷盘，不需要满足这个条件）

![image](http://static.silence.work/20190506-6.png)


## 主从同步

如果存在slave，将消息同步给slave。

只有当broker的角色是同步master的时候才会主动将消息同步给slave。如果同步失败，将失败结果返回给出去。broker为其他角色不做任何处理。

关于主备消息的同步过程这里不做具体分析。

```
public void handleHA(AppendMessageResult result, PutMessageResult putMessageResult, MessageExt messageExt) {
        //如果broker的角色是同步master（broker有3种角色：同步master、异步master、slave）
        if (BrokerRole.SYNC_MASTER == this.defaultMessageStore.getMessageStoreConfig().getBrokerRole()) {
            //同步消息到slave的service
            HAService service = this.defaultMessageStore.getHaService();
            if (messageExt.isWaitStoreMsgOK()) {
                // 检测 Slave 和 Master 落下的同步进度是否太大
                if (service.isSlaveOK(result.getWroteOffset() + result.getWroteBytes())) {
                    GroupCommitRequest request = new GroupCommitRequest(result.getWroteOffset() + result.getWroteBytes());
                    service.putRequest(request);
                    service.getWaitNotifyObject().wakeupAll();
                    //等待同步slave的处理完成
                    boolean flushOK =
                        request.waitForFlush(this.defaultMessageStore.getMessageStoreConfig().getSyncFlushTimeout());
                    if (!flushOK) {
                        log.error("do sync transfer other node, wait return, but failed, topic: " + messageExt.getTopic() + " tags: "
                            + messageExt.getTags() + " client address: " + messageExt.getBornHostNameString());
                        putMessageResult.setPutMessageStatus(PutMessageStatus.FLUSH_SLAVE_TIMEOUT);
                    }
                }
                // Slave problem
                else {
                    // Tell the producer, slave not available
                    putMessageResult.setPutMessageStatus(PutMessageStatus.SLAVE_NOT_AVAILABLE);
                }
            }
        }

    }
```


## 返回处理结果

根据以上过程的处理结果封装到PutMessageResult，最后构建response返回给Producer。

其中有三种异常结果我们要注意返回给Producer，Producer会当成消息发送成功处理。分别是：

> FLUSH_DISK_TIMEOUT：刷盘超时

> FLUSH_SLAVE_TIMEOUT：同步到slave超时

> SLAVE_NOT_AVAILABLE：slave不可用


# 参考资料

[RocketMQ 消息存储流程](https://www.kunzhao.org/blog/2018/03/12/rocketmq-message-store-flow/)

[RocketMQ开发指南
](http://www.sxt.cn/ueditor/php/upload/file/20170901/1504248466272324.pdf)

[RocketMQ高性能之底层存储设计
](https://juejin.im/post/5ba798995188255c806663c0/)