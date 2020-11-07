title: 关于spring-data操作redis报invalid bulk length的错误分析
author: Silence
tags:
  - BUG
  - Redis
  - ''
categories:
  - Redis
date: 2020-02-16 23:07:00
---
# 背景
应用每隔一段时间频繁会收到invalid bulk length错误告警，并且每次报错应用操作的key都不一样，并且本身命令没有任何问题，错误堆栈如下：

```
2020-01-28 16:10:03.198 [ DubboServerHandler-172.20.60.18:20880-thread-497 ] AddressBookFacadeImpl - queryAddressBookByUserId failure:parameter:332223798256357376: org.springframework.dao.InvalidDataAccessApiUsageException: ERR Protocol error: invalid bulk length; nested exception is redis.clients.jedis.exceptions.JedisDataException: ERR Protocol error: invalid bulk length
    at org.springframework.data.redis.connection.jedis.JedisExceptionConverter.convert(JedisExceptionConverter.java:64)
    at org.springframework.data.redis.connection.jedis.JedisExceptionConverter.convert(JedisExceptionConverter.java:41)
    at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:37)
    at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:37)
    at org.springframework.data.redis.connection.jedis.JedisClusterConnection.convertJedisAccessException(JedisClusterConnection.java:3985)
    at org.springframework.data.redis.connection.jedis.JedisClusterConnection.get(JedisClusterConnection.java:570)
    at org.springframework.data.redis.connection.DefaultStringRedisConnection.get(DefaultStringRedisConnection.java:296)
    at org.springframework.data.redis.core.DefaultValueOperations$1.inRedis(DefaultValueOperations.java:46)
    at org.springframework.data.redis.core.AbstractOperations$ValueDeserializingRedisCallback.doInRedis(AbstractOperations.java:57)
    at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:207)
    at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:169)
    at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:91)
    at org.springframework.data.redis.core.DefaultValueOperations.get(DefaultValueOperations.java:43)
    at com.fcbox.mall.item.repository.redis.impl.RedisCacheDAOImpl.getForValue(RedisCacheDAOImpl.java:67)
    at com.fcbox.mall.item.repository.redis.impl.RedisCacheDAOImpl$$FastClassBySpringCGLIB$$e19d7be6.invoke(<generated>)
    at org.springframework.cglib.proxy.MethodProxy.invoke(MethodProxy.java:204)
    at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.invokeJoinpoint(CglibAopProxy.java:736)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:157)
    at org.springframework.dao.support.PersistenceExceptionTranslationInterceptor.invoke(PersistenceExceptionTranslationInterceptor.java:136)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179)
    at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:671)
    at com.fcbox.mall.item.repository.redis.impl.RedisCacheDAOImpl$$EnhancerBySpringCGLIB$$2356a474.getForValue(<generated>)
    at com.fcbox.mall.item.biz.addressbook.front.impl.AddressBookServiceImpl.queryAddressBookByUserIdAndSource(AddressBookServiceImpl.java:198)
    at com.fcbox.mall.item.endpoint.provider.front.AddressBookFacadeImpl.queryAddressBookByUserId(AddressBookFacadeImpl.java:115)
    at com.alibaba.dubbo.common.bytecode.Wrapper102.invokeMethod(Wrapper102.java)
    at com.alibaba.dubbo.rpc.proxy.javassist.JavassistProxyFactory$1.doInvoke(JavassistProxyFactory.java:46)
    at com.alibaba.dubbo.rpc.proxy.AbstractProxyInvoker.invoke(AbstractProxyInvoker.java:72)
    at com.alibaba.dubbo.rpc.protocol.InvokerWrapper.invoke(InvokerWrapper.java:53)
    at com.alibaba.extension.filter.AccessInvokeLogFilter.invoke(AccessInvokeLogFilter.java:29)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.filter.ExceptionFilter.invoke(ExceptionFilter.java:64)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.monitor.support.MonitorFilter.invoke(MonitorFilter.java:75)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.filter.TimeoutFilter.invoke(TimeoutFilter.java:42)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.protocol.dubbo.filter.TraceFilter.invoke(TraceFilter.java:78)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.fcbox.mall.bizmode.spread.BizModeDubboProviderFilter.invoke(BizModeDubboProviderFilter.java:33)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.filter.ContextFilter.invoke(ContextFilter.java:60)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.filter.GenericFilter.invoke(GenericFilter.java:112)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.filter.ClassLoaderFilter.invoke(ClassLoaderFilter.java:38)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.filter.EchoFilter.invoke(EchoFilter.java:38)
    at com.alibaba.dubbo.rpc.protocol.ProtocolFilterWrapper$1.invoke(ProtocolFilterWrapper.java:91)
    at com.alibaba.dubbo.rpc.protocol.dubbo.DubboProtocol$1.reply(DubboProtocol.java:108)
    at com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.handleRequest(HeaderExchangeHandler.java:84)
    at com.alibaba.dubbo.remoting.exchange.support.header.HeaderExchangeHandler.received(HeaderExchangeHandler.java:170)
    at com.alibaba.dubbo.remoting.transport.DecodeHandler.received(DecodeHandler.java:52)
    at com.alibaba.dubbo.remoting.transport.dispatcher.ChannelEventRunnable.run(ChannelEventRunnable.java:82)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:745)
Caused by: redis.clients.jedis.exceptions.JedisDataException: ERR Protocol error: invalid bulk length
    at redis.clients.jedis.Protocol.processError(Protocol.java:130)
    at redis.clients.jedis.Protocol.process(Protocol.java:164)
    at redis.clients.jedis.Protocol.read(Protocol.java:218)
    at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:341)
    at redis.clients.jedis.Connection.getBinaryBulkReply(Connection.java:260)
    at redis.clients.jedis.BinaryJedis.get(BinaryJedis.java:246)
    at redis.clients.jedis.BinaryJedisCluster$3.execute(BinaryJedisCluster.java:101)
    at redis.clients.jedis.BinaryJedisCluster$3.execute(BinaryJedisCluster.java:98)
    at redis.clients.jedis.JedisClusterCommand.runWithRetries(JedisClusterCommand.java:117)
    at redis.clients.jedis.JedisClusterCommand.runBinary(JedisClusterCommand.java:58)
    at redis.clients.jedis.BinaryJedisCluster.get(BinaryJedisCluster.java:103)
    at org.springframework.data.redis.connection.jedis.JedisClusterConnection.get(JedisClusterConnection.java:568)
    ... 50 common frames omitted
```

**环境信息：**

Spring-boot：1.5.20.RELEASE

spring-data-redis：1.8.20.RELEASE

redis集群

# 分析过程

1、观察错误堆栈，推测到该错误本身是redis服务器直接返回的。
查看redis的通信协议，发现redis服务器响应错误
时会通过-ERR xxxx给客户端。

```
ERR Protocol error: invalid bulk length
```

2、下载redis源码搜索关键字：Protocol error: invalid bulk length。

搜索到到源码文件networking.c，如下：
![image](http://static.silence.work/20200216-1.png)
该段代码对应的方法是processMultibulkBuffer，该方法是redis根据自身的RESP协议用来解析客户端发送过来的请求报文。

进一步分析发现只有当以下if条件成立才会返回异常：

```
!ok || ll < 0 || ll > server.proto_max_bulk_len
```

其中ll > server.proto_max_bulk_len是指当前解析的命令超过了redis配置的最大长度，根据实际报错命令非常小不太可能。

那么可以猜测是string2ll返回的ok为false或者ll<0，这里了解到string2ll方法是解析对应的bulk长度。
那么这里可以初步猜测是解析客户端发送过来命令长度发生了异常。然而实际报错的命令却非常正常，不可能出现解析命令长度失败错误。


3、再次重新审视错误日志后发现一个很明显的特征，每隔5分钟就可能会报一次错误，同时又发现了一个可疑的定时任务，每隔5分钟会做一次redis的set操作，并且set的时候由于没有对value值判空，导致set的value值每隔5分钟会set null。

进一步发现了当set 的value是null的时候，会抛出错误堆栈，指明在Protocol.sendCommand会发生空指针异常：
![image](http://static.silence.work/20200216-2.png)
![image](http://static.silence.work/20200216-3.png)


分析到这里问题开始浮出水面了，jedis上层并没有在网络协议层之上拦住value为null的情况。

而是直接导致了在网络输出流中写字节写到一半异常中断，从而会导致不正常的序列化结果，当下一个正常的命令过来时如果又拿到了上一个链接，则会在同一个输出流中继续往后写入字节码。

结合前面分析的过程以及redis 的通信协议，上一个不正确的序列化内容+本次正确的序列化内容可能会导致redis服务器无法正确解析本次正确的内容，从而引发redis的invalid bulk length错误。

# 初步结论

上一次set value为null的操作+本次正确的命令操作会导致redis无法正确解析本次正确的命令。

例如：
上一次value为null序列化后的结果为：

```
*3
$3
SET
$5
mykey
$
```
本次正确的命令序列化结果：
```
*2
$3
GET
$5
mykey
```
当两次写入到同一个输出流中会得到：

```
*3
$3
SET
$5
mykey
$*2
$3
GET
$5
mykey
```
这里通过telnet 到redis服务器，写入该段序列化后的内容，会得到redis服务器的响应结果：-ERR Protocol error: invalid bulk length

![image](http://static.silence.work/20200216-4.png)

# 验证结论

## 错误重现

```
        new Thread(() -> {
            try {
                stringRedisTemplate.opsForValue().set("mall:shoppingcart:100004923218866176", null);
            } catch (Exception e) {
                log.error("线程1 set null操作报错",e);
            }
        }).run();

        new Thread(() -> {
            try {
                String result = stringRedisTemplate.opsForValue().get("mall:shoppingcart:100004923218866176");
                System.out.println(result);
            }catch (Exception e){
                log.error("线程2 执行正常的key操作报错",e);
            }
        }).run();

        new Thread(() -> {
            try {
                String result = stringRedisTemplate.opsForValue().get("mall:shoppingcart:100004923218866176");
                log.info("线程3执行成功,{}",result);
            }catch (Exception e){
                log.error("线程3 执行正常的key操作报错",e);
            }
        }).run();
```
这里线程2和线程3模拟操作的key和线程1一样是为了保证所有操作落在同一个redis节点上，从而保证先后拿到的是同一个连接。

**报错日志：**

```
20:25:06.424 [main] - 线程1 set null操作报错
 Caller+0	 at com.fcbox.mall.panglu.run.example.RedisRunner.lambda$run$0(RedisRunner.java:31)
org.springframework.data.redis.RedisSystemException: Unknown redis exception; nested exception is java.lang.NullPointerException
	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.getFallback(FallbackExceptionTranslationStrategy.java:48)
	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:38)
	at org.springframework.data.redis.connection.jedis.JedisClusterConnection.convertJedisAccessException(JedisClusterConnection.java:3985)
	at org.springframework.data.redis.connection.jedis.JedisClusterConnection.set(JedisClusterConnection.java:620)
	at org.springframework.data.redis.connection.DefaultStringRedisConnection.set(DefaultStringRedisConnection.java:744)
	at org.springframework.data.redis.core.DefaultValueOperations$10.inRedis(DefaultValueOperations.java:172)
	at org.springframework.data.redis.core.AbstractOperations$ValueDeserializingRedisCallback.doInRedis(AbstractOperations.java:57)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:207)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:169)
	at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:91)
	at org.springframework.data.redis.core.DefaultValueOperations.set(DefaultValueOperations.java:169)
	at com.fcbox.mall.panglu.run.example.RedisRunner.lambda$run$0(RedisRunner.java:29)
	at java.lang.Thread.run(Thread.java:748)
	at com.fcbox.mall.panglu.run.example.RedisRunner.run(RedisRunner.java:33)
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:732)
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:716)
	at org.springframework.boot.SpringApplication.afterRefresh(SpringApplication.java:703)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:304)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1107)
	at com.fcbox.mall.panglu.PangluApplication.main(PangluApplication.java:10)
Caused by: java.lang.NullPointerException: null
	at redis.clients.jedis.Protocol.sendCommand(Protocol.java:102)
	at redis.clients.jedis.Protocol.sendCommand(Protocol.java:87)
	at redis.clients.jedis.Connection.sendCommand(Connection.java:127)
	at redis.clients.jedis.BinaryClient.set(BinaryClient.java:118)
	at redis.clients.jedis.BinaryJedis.set(BinaryJedis.java:210)
	at redis.clients.jedis.BinaryJedisCluster$1.execute(BinaryJedisCluster.java:80)
	at redis.clients.jedis.BinaryJedisCluster$1.execute(BinaryJedisCluster.java:77)
	at redis.clients.jedis.JedisClusterCommand.runWithRetries(JedisClusterCommand.java:117)
	at redis.clients.jedis.JedisClusterCommand.runBinary(JedisClusterCommand.java:58)
	at redis.clients.jedis.BinaryJedisCluster.set(BinaryJedisCluster.java:82)
	at org.springframework.data.redis.connection.jedis.JedisClusterConnection.set(JedisClusterConnection.java:618)
	... 17 common frames omitted
20:25:06.427 [main] - 线程2 执行正常的key操作报错
 Caller+0	 at com.fcbox.mall.panglu.run.example.RedisRunner.lambda$run$1(RedisRunner.java:40)
org.springframework.dao.InvalidDataAccessApiUsageException: ERR Protocol error: invalid bulk length; nested exception is redis.clients.jedis.exceptions.JedisDataException: ERR Protocol error: invalid bulk length
	at org.springframework.data.redis.connection.jedis.JedisExceptionConverter.convert(JedisExceptionConverter.java:64)
	at org.springframework.data.redis.connection.jedis.JedisExceptionConverter.convert(JedisExceptionConverter.java:41)
	at org.springframework.data.redis.PassThroughExceptionTranslationStrategy.translate(PassThroughExceptionTranslationStrategy.java:37)
	at org.springframework.data.redis.FallbackExceptionTranslationStrategy.translate(FallbackExceptionTranslationStrategy.java:37)
	at org.springframework.data.redis.connection.jedis.JedisClusterConnection.convertJedisAccessException(JedisClusterConnection.java:3985)
	at org.springframework.data.redis.connection.jedis.JedisClusterConnection.get(JedisClusterConnection.java:570)
	at org.springframework.data.redis.connection.DefaultStringRedisConnection.get(DefaultStringRedisConnection.java:296)
	at org.springframework.data.redis.core.DefaultValueOperations$1.inRedis(DefaultValueOperations.java:46)
	at org.springframework.data.redis.core.AbstractOperations$ValueDeserializingRedisCallback.doInRedis(AbstractOperations.java:57)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:207)
	at org.springframework.data.redis.core.RedisTemplate.execute(RedisTemplate.java:169)
	at org.springframework.data.redis.core.AbstractOperations.execute(AbstractOperations.java:91)
	at org.springframework.data.redis.core.DefaultValueOperations.get(DefaultValueOperations.java:43)
	at com.fcbox.mall.panglu.run.example.RedisRunner.lambda$run$1(RedisRunner.java:37)
	at java.lang.Thread.run(Thread.java:748)
	at com.fcbox.mall.panglu.run.example.RedisRunner.run(RedisRunner.java:42)
	at org.springframework.boot.SpringApplication.callRunner(SpringApplication.java:732)
	at org.springframework.boot.SpringApplication.callRunners(SpringApplication.java:716)
	at org.springframework.boot.SpringApplication.afterRefresh(SpringApplication.java:703)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:304)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1118)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:1107)
	at com.fcbox.mall.panglu.PangluApplication.main(PangluApplication.java:10)
Caused by: redis.clients.jedis.exceptions.JedisDataException: ERR Protocol error: invalid bulk length
	at redis.clients.jedis.Protocol.processError(Protocol.java:130)
	at redis.clients.jedis.Protocol.process(Protocol.java:164)
	at redis.clients.jedis.Protocol.read(Protocol.java:218)
	at redis.clients.jedis.Connection.readProtocolWithCheckingBroken(Connection.java:341)
	at redis.clients.jedis.Connection.getBinaryBulkReply(Connection.java:260)
	at redis.clients.jedis.BinaryJedis.get(BinaryJedis.java:246)
	at redis.clients.jedis.BinaryJedisCluster$3.execute(BinaryJedisCluster.java:101)
	at redis.clients.jedis.BinaryJedisCluster$3.execute(BinaryJedisCluster.java:98)
	at redis.clients.jedis.JedisClusterCommand.runWithRetries(JedisClusterCommand.java:117)
	at redis.clients.jedis.JedisClusterCommand.runBinary(JedisClusterCommand.java:58)
	at redis.clients.jedis.BinaryJedisCluster.get(BinaryJedisCluster.java:103)
	at org.springframework.data.redis.connection.jedis.JedisClusterConnection.get(JedisClusterConnection.java:568)
	... 17 common frames omitted
20:25:06.429 [main] - 线程3执行成功,sdafsadf
```

## 原因分析

通过分析源码我们定位到jedis写入命令到输出流的地方：

![image](http://static.silence.work/20200216-5.png)

1、这里args会有两个字节数组，第一个是key，第二个是value，在set null操作时候，会导致args数组的第二个直接数组为空，从而导致arg.length报空指针异常，直接中断输出流的写操作，进而导致后面的writeCrLf没有执行。

断点查阅os内的内容为：

```
*3
$3
SET
$36
mall:shoppingcart:100004923218866176
$
```

2、链接释放。

3、紧接着第二个命令执行，断点结果如下：

![image](http://static.silence.work/20200216-6.png)
![image](http://static.silence.work/20200216-7.png)
查阅此时输出流，会发现第二次操作redis，拿到的流对象还是原来的hash值为4000的对象，也就是说会继续在上一次操作的流中写入本次的命令。

此时os的内容为：

```
*3
$3
SET
$36
mall:shoppingcart:100004923218866176
$*2
$3
GET
$36
mall:shoppingcart:100004923218866176

```

我们发现输出流中的内容和前面部分我们猜测的结果一致，正是由于上一个value为null导致了两个命令之间没有正确的分隔符，因此redis在解析命令的时候自然也会无法解析，从而报错。

4、根据redis的RESP协议我们可以猜测正确的结果应该为：


```
*3
$3
SET
$36
mall:shoppingcart:100004923218866176
$value
*2
$3
GET
$36
mall:shoppingcart:100004923218866176
```

到这里问题已经得到了重现并验证。
----

# 小插曲

最后部分说几个定位问题过程中思考的问题。

#### 问题一：当出现set null操作的时候，是否会导致接下来有操作一定会失败？

答案是肯定的。

在查阅spring-data-redis的源码发现，底层使用的是jedis和JedisPool，而JedisPool使用的连接池是commons-pool来管理的连接。

commons-pool本身在管理连接池默认采用的是LIFO策略，即后进先出。连接使用完后，在队首放入，连接使用的时候从队首取，也就是说当前一个连接释放后，下一次一定取到的还是当前连接。（这里前提条件是操作的集群同一个节点，只有操作同一个节点使用的才是同一个连接池）

#### 问题二：非redis集群环境该错误是否会出现？

答案：不会。

仔细查阅源码后发现：

1、集群环境下spring-data-redis封装的连接对象是**JedisClusterConnection**。

2、非集群封装的连接对象是**JedisConnection**。

两者在实现close方法上有个非常关键的操作JedisClusterConnection没有实现。JedisConnection的close方法会判断broken是否为true，为true则代表当前连接属于异常连接，不需要返回到连接池，而是直接销毁，从而保证后续的操作不会再取到当前有异常的连接。
![image](http://static.silence.work/20200216-8.png)

我们看convertJedisAccessException方法，非集群的版本有考虑到这种NPE的场景，但是集群的版本并没有考虑到。

![image](http://static.silence.work/20200216-9.png)


因此我们也可以得出这样一个结论：**这个set value为null会引发的异常本质上是spring封装操作redis集群api没有考虑到出现异常连接释放问题，属于spring-data-redis的一个bug**。

#### 问题3:spring-boot2.0以上版本是否已经修复？

答案是肯定的，spring-boot2.0以上版本在redis的操作上已经替换成了Lettuce。同时spring-data在封装redis操作上已经对value为null做了判空。

----

# 参考资料


[redis通信协议](http://doc.redisfans.com/topic/protocol.html)

[redis源码分析--请求处理](https://blog.csdn.net/chosen0ne/article/details/43053915)

[Redis命令处理过程分析](https://www.jianshu.com/p/6188becd2cea)