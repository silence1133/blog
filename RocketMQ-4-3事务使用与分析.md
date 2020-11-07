title: RocketMQ 事务消息的使用与原理分析
author: Silence
tags:
  - 消息中间件
  - RocketMQ
categories:
  - 消息中间件
date: 2018-08-22 17:52:00
---
# 事务消息的引出

以购物场景为例，张三购买物品，账户扣款 100 元的同时，需要保证在下游的会员服务中给该账户增加 100 积分。而扣款的业务和增加积分的业务是在两个不同的应用，正常处理逻辑一般是先扣除100元，然后网络通知积分服务增加100积分。如下图：
![image](http://static.silence.work/mq.png)

以上过程会存在3个问题：

1. 账号服务在扣款的时候宕机了，这时候可能扣款成功，也可能扣款失败；

2. 由于网络稳定性无法保证，通知扣积分服务可能失败，但是扣款成功了；

3. 扣款成功，并且通知成功，但是增加积分的时候失败了。

实际上，rocketmq的事务消息解决的是问题1和问题2这种场景，也就是**解决本地事务执行与消息发送的原子性问题**。即解决Producer执行业务逻辑成功之后投递消息可能失败的场景。

而对于问题3这种场景，rocketmq提供了消费失败重试的机制。但是如果消费重试依然失败怎么办？rocketmq本身并没有提供解决这种问题的办法，例如如果加积分失败了，则需要回滚事务，实际上增加了业务复杂度，而官方给予的建议是：人工解决。RocketMQ目前暂时没有解决这个问题的原因是：在设计实现消息系统时，我们需要衡量是否值得花这么大的代价来解决这样一个出现概率非常小的问题。

# 事务消息的实现思路和过程

RocketMQ 事务消息的设计流程同样借鉴了两阶段提交理论，通过在执行本地事务前后发送两条消息来保证本地事务与发送消息的原子性，过程如下图：
![image](http://static.silence.work/mq_transcation.png)

## 事务消息详细过程说明

1. Producer发送Half(prepare)消息到broker；
2. half消息发送成功之后执行本地事务；
3. （由用户实现）本地事务执行如果成功则返回commit，如果执行失败则返回roll_back。
4. Producer发送确认消息到broker（也就是将步骤3执行的结果发送给broker），这里可能broker未收到确认消息，下面分两种情况分析：


- **如果broker收到了确认消息：**

> 如果收到的结果是commit，则broker视为整个事务过程执行成功，将消息下发给Conusmer端消费；

> 如果收到的结果是rollback，则broker视为本地事务执行失败，broker删除Half消息，不下发给consumer。

- **如果broker未收到了确认消息：**

> broker定时回查本地事务的执行结果；

>（由用户实现）如果本地事务已经执行则返回commit；如果未执行，则返回rollback；

> Producer端回查的结果发送给broker；

> broker接收到的如果是commit，则broker视为整个事务过程执行成功，将消息下发给Conusmer端消费；如果是rollback，则broker视为本地事务执行失败，broker删除Half消息，不下发给consumer。如果broker未接收到回查的结果（或者查到的是unknow），则broker会定时进行重复回查，以确保查到最终的事务结果。


***补充***：对于过程3，如果执行本地事务突然宕机了（相当本地事务执行结果返回unknow），则和broker未收到确认消息的情况一样处理。


# 事务消息的使用

关于rocketmq事务消息如何使用，最好的学习思路是从github上下载下源码，参考[demo示例](https://github.com/apache/rocketmq/tree/master/example/src/main/java/org/apache/rocketmq/example/transaction)。这里也以官方的demo讲解如何使用（在demo基础上做了一点修改）。


## 代码示例

为了模拟事务执行的异常场景，这里会模拟发送5条事务消息，前三条（msg-1、msg-2、msg-3）对应的本地事务执行结果为unknow（模拟本地事务执行未知的情况）;

第4条消息（msg-4）返回COMMIT_MESSAGE（模拟本地事务执行成功的情况），第5条消息（msg-5）返回ROLLBACK_MESSAGE（模拟本地事务执行失败的情况）;

对于前三条消息，模拟回查到的本地事务处理结果分别为UNKNOW，COMMIT_MESSAGE，ROLLBACK_MESSAGE。

- **发送事务的逻辑：**

```
public class TransactionProducer {
    public static void main(String[] args) throws MQClientException, InterruptedException {
        //事务执行的listener，由用户实现及接口，提供本地事务执行的代码，以及回查本地事务处理结果的逻辑。
        TransactionListener transactionListener = new TransactionListenerImpl();

        TransactionMQProducer producer = new TransactionMQProducer("TransactionProducer");
        producer.setNamesrvAddr("localhost:9876");
        producer.setTransactionListener(transactionListener);
        producer.start();

        //模拟发送5条消息
        for (int i = 1; i < 6; i++) {
            try {
                Message msg = new Message("TransactionTopicTest", null, "msg-" + i, ("Hello RocketMQ " + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
                producer.sendMessageInTransaction(msg, null);
                Thread.sleep(10);
            } catch (MQClientException | UnsupportedEncodingException e) {
                e.printStackTrace();
            }
        }

        Thread.sleep(Integer.MAX_VALUE);
        producer.shutdown();
    }
}
```
- **提供本地事务执行以及回查本地事务的逻辑：**

```
public class TransactionListenerImpl implements TransactionListener {
    private AtomicInteger transactionIndex = new AtomicInteger(0);
    private AtomicInteger checkTimes = new AtomicInteger(0);

    private ConcurrentHashMap<String, Integer> localTrans = new ConcurrentHashMap<>();

    /**
     * 本地事务的执行逻辑实现
     * 模拟5条消息本地事务的处理结果
     * @param msg Half(prepare) message
     * @param arg Custom business parameter
     * @return
     */
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {

        LocalTransactionState state = null;
        //msg-4返回COMMIT_MESSAGE
        if(msg.getKeys().equals("msg-4")){
            state = LocalTransactionState.COMMIT_MESSAGE;
        }
        //msg-5返回ROLLBACK_MESSAGE
        else if(msg.getKeys().equals("msg-5")){
            state = LocalTransactionState.ROLLBACK_MESSAGE;
        }else{
            //这里返回unknown的目的是模拟执行本地事务突然宕机的情况（或者本地执行成功发送确认消息失败的场景）
            state = LocalTransactionState.UNKNOW;
            //假设3条消息的本地事务结果分别为1，2，3
            localTrans.put(msg.getKeys(), transactionIndex.incrementAndGet());
        }
        System.out.println("executeLocalTransaction:" + msg.getKeys() + ",excute state:" + state +",current time：" + new Date());
        return state;
    }

    /**
     * 回查本地事务的代码实现
     * 第1条消息模拟unknow（例如回查的时候网络依然有问题的情况）。
     * 第2条消息模拟本地事务处理成功结果COMMIT_MESSAGE。
     * 第3条消息模拟本地事务处理失败结果需要回滚ROLLBACK_MESSAGE。
     *
     * @param msg Check message
     * @return
     */
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        System.out.print("checkLocalTransaction message key："+msg.getKeys()+",current time：" + new Date());
        //根据key获取到3条消息本地事务的处理结果(实际业务场景一般是通过获取msg中的消息体数据来确定某条消息的本地事务是否执行成功)
        Integer status = localTrans.get(msg.getKeys());
        if (null != status) {
            switch (status) {
                case 1:
                    System.out.println(" check result：unknow ，回查次数："+checkTimes.incrementAndGet());
                    //依然无法确定本地事务的执行结果，返回unknow，下次会继续回查结果
                    return LocalTransactionState.UNKNOW;
                case 2:
                    //查到本地事务执行成功，返回COMMIT_MESSAGE，producer继续发送确认消息（此逻辑无需自己写，mq本身提供）
                    //或者查到本地事务执行成功了，但是想回滚掉，则这里需要返回ROLLBACK_MESSAGE，同时写回滚的逻辑，实际如何处理根据业务场景而定
                    System.out.println(" check result：commit message");
                    return LocalTransactionState.COMMIT_MESSAGE;
                case 3:
                    //查询到本地事务执行失败，需要回滚消息。
                    System.out.println(" check result：rollback message");
                    return LocalTransactionState.ROLLBACK_MESSAGE;
            }
        }
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}

```
## 运行结果分析

![image](http://static.silence.work/maConsole.png)

![image](http://static.silence.work/msg.png)

**仔细观察日志输出和romcketmq的控制台，我们可以得出如下结论**：

- msg-4、msg-5消息没有执行回查事务消息的逻辑，是因为msg-4、msg-5在本地执行事务的时候已经返回了确定的事务执行结果，因此msg-4、msg-5不会回查；
- msg-1、msg-2、msg-3在执行完本地事务10s后，都回查了本地事务的结果；
- msg-2、msg-3只回查了一次，因为这两条消息在回查的时候已经返回了确切的事务执行结果；
- msg-1回查了5次，并且间隔为1分钟，因为msg-1在回查的事务状态依然为unknow，因此会反复回查，直到超过了回查的默认次数不再回查。
- 对比msg-2和msg-4的消息存储时间，msg-4的存储时间恰好是执行本地事务返回的时间，而msg-2的存储时间则恰好是第一次回查事务结果返回的时间。

# 源码分析

## 发送事务消息：

提炼出的关键过程代码如下：
```

 public TransactionSendResult sendMessageInTransaction(final Message msg,final TransactionListener tranExecuter, final Object arg){
        //1.发送prepare消息
        SendResult sendResult = this.send(msg);
        
        LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
        Throwable localException = null;
        switch (sendResult.getSendStatus()) {
            case SEND_OK: {
                try {
                    //2.如果prepare消息发送成功，执行TransactionListener的executeLocalTransaction实现，也就是本地事务方法
                    localTransactionState = tranExecuter.executeLocalTransaction(msg, arg);
                } catch (Throwable e) {
                    localException = e;
                }
            }
            break;
            case FLUSH_DISK_TIMEOUT:
            case FLUSH_SLAVE_TIMEOUT:
            case SLAVE_NOT_AVAILABLE:
                localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
                break;
            default:
                break;
        }
        //3.结束事务，其实就是针对前面发送的prepare消息再发送一条确认消息（这条确认消息包含了本地事务执行的结果，这里可以猜测broker接收到该确认消息和之前的prepare消息必然有比较大的关联）
        this.endTransaction(sendResult, localTransactionState, localException);
    }
```
大致思路是：

1. 发送prepare消息；
2. 执行实现了TransactionListener的executeLocalTransaction方法，也就是执行本地事务的逻辑；
3. 结束事务，将过程2得到的本地事务结果通过发送另外一条确认消息告诉broker；

因此我们这里可以推测：broker必然会根据前后两条消息来确定如何处理该事务消息。

## broker端的处理事务消息回查逻辑

关键代码提炼如下：

```
public class TransactionalMessageCheckService extends ServiceThread {
    @Override
    public void run() {
        //检查间隔，默认一分钟，可配置
        long checkInterval = brokerController.getBrokerConfig().getTransactionCheckInterval();
        while (!this.isStopped()) {
            try {
                //等待一分钟，以实现每一分钟回查需要的事务消息结果
                waitPoint.await(interval, TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                log.error("Interrupted", e);
            } finally {
                //处理事务消息回查的核心逻辑方法
                brokerController.getTransactionalMessageService().check(timeout, checkMax,this.brokerController.getTransactionalMessageCheckListener());
            }
        }
    }
}

public class TransactionalMessageServiceImpl implements TransactionalMessageService {

    public void check(long transactionTimeout, int transactionCheckMax,AbstractTransactionalMessageCheckListener listener) {
            //获取到所有的RMQ_SYS_TRANS_HALF_TOPIC消息队列（prepare消息）
            Set<MessageQueue> msgQueues = transactionalMessageBridge.fetchMessageQueues("RMQ_SYS_TRANS_HALF_TOPIC");
            for (MessageQueue messageQueue : msgQueues) {
                //从RMQ_SYS_TRANS_OP_HALF_TOPIC消息队列中获取到prepare消息对应的op消息（确认消息）
                MessageQueue opQueue = getOpQueue(messageQueue);
                //prepare消息的offset
                long halfOffset = transactionalMessageBridge.fetchConsumeOffset(messageQueue);
                //prepare消息
                MessageExt msgExt = getHalfMsg(messageQueue, i);
                //中间会有一堆的逻辑判断用于是否需要回查事务状态。
                //例如：是否超过了回查的次数（默认五次）、消息是否已经失效了、对应的op消息是否已经处理了等。
                if (isNeedCheck) {
                    //交给线程池异步处理回调查询事务的状态。
                    listener.resolveHalfMsg(msgExt);
                }
            }
    }
}
```

大概的处理思路是：

broker维护一个死循环，每一分钟执行一次，broker通过使用两个内部队列：
RMQ_SYS_TRANS_HALF_TOPIC、RMQ_SYS_TRANS_OP_HALF_TOPIC来存储事务消息推进状态，服务端通过比对两个队列的差值来找到尚未提交的超时事务，调用Producer端，用来查询事务处理结果。

## Producer端接收broker回查的逻辑

关键代码提炼如下：

```
    //接收broker的回调，回查本地事务情况，进行相应处理
    @Override
    public void checkTransactionState(final String addr, final MessageExt msg,final CheckTransactionStateRequestHeader header) {
        //处理broker检查本地事务处理情况的回调任务
        Runnable request = new Runnable() {
            @Override
            public void run() {
                    //执行TransactionListener实现的checkLocalTransaction方法，检查本地事务处理情况。
                    LocalTransactionState localTransactionState = transactionCheckListener.checkLocalTransaction(message);
                    //将检查本地事务处理情况再次发送给broker。
                    this.processTransactionState(localTransactionState,group,exception);
            }

            //处理本地事务处理的结果反馈
            private void processTransactionState(final LocalTransactionState localTransactionState,final String producerGroup,final Throwable exception) {
                final EndTransactionRequestHeader thisHeader = new EndTransactionRequestHeader();
                ...
                根据检查到的本地事务执行的不同结果封装成不同的处理类型发送给broker
                switch (localTransactionState) {
                    case COMMIT_MESSAGE:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_COMMIT_TYPE);
                        break;
                    case ROLLBACK_MESSAGE:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_ROLLBACK_TYPE);
                        break;
                    case UNKNOW:
                        thisHeader.setCommitOrRollback(MessageSysFlag.TRANSACTION_NOT_TYPE);
                        break;
                    default:
                        break;
                }
                //结果反馈给broker
                DefaultMQProducerImpl.this.mQClientFactory.getMQClientAPIImpl().endTransactionOneway(brokerAddr,thisHeader,remark,3000);
            }
        };
        //提交任务到线程池
        this.checkExecutor.submit(request);
    }
```

大致的处理思路是：

Producer端一个线程池维护执行TransactionListener的executeLocalTransaction实现，也就是本地事务方法的任务。将查询到的本地事务结果反馈给broker端，broker来决定对事务消息如何处理。


参考资料：

[《RocketMQ 4.3正式发布，支持分布式事务》](http://www.infoq.com/cn/news/2018/08/rocketmq-4.3-release)

[《分布式开放消息系统(RocketMQ)的原理与实践》](https://www.jianshu.com/p/453c6e7ff81c)