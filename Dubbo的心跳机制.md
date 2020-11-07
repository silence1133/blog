title: Dubbo的心跳机制
author: Silence
tags:
  - Dubbo
categories:
  - Dubbo
date: 2020-05-17 11:28:00
---
# 1. 心跳机制

在网络传输中，如何确保客户端和服务端之间通道连接是否可用是一个很重要的问题，比如客户端突然崩溃，服务器端可能在几天内都维护着一个无用的 TCP 连接。
心跳机制能有效的保证连接的可用性。

TCP本身提供了Keep-alive选项用于保证可用的通道。

我们先看下TCP的Keep-alive机制。

# 2. TCP的Keep-alive

在一个时间段内，如果没有任何连接相关的活动，TCP Keep-alive会开始作用，每隔一个时间间隔，发送一个探测报文，该探测报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的 TCP 连接已经死亡，系统内核将错误信息通知给上层应用程序。

这个过程会涉及到操作系统提供的3个变量（通过sysctl 命令查看）：

```
#保活时间，默认2小时
net.ipv4.tcp_keepalive_time = 7200
#探测时间间隔72s
net.ipv4.tcp_keepalive_intvl = 75
#探测次数9次
net.ipv4.tcp_keepalive_probes = 9
```

根据系统提供的参数默认值，如果使用 TCP 自身的 keep-Alive 机制，在 Linux 系统中，最少需要经过2个多小时才可以发现一个“死亡”连接。

另外Keepa alive默认是关闭的，需要开启KeepAlive的应用必须在TCP的socket中单独开启。

除了TCP保活间隔较长，另外KeepAlive机制只是在网络层保证了连接的可用性，如果网络层是通的，应用层不通了也是应该认为链接不可用了。

因此，应用层也需要提供心跳机制来保证应用直接的链接可用性检测。

# 3. Dubbo的心跳机制

Dubbo使用了netty提供给的IdleStateHandler实现的心跳机制。

服务端检测到客户端不可用的链接会关闭链接，客户端
检测到服务端不可用链接会进行重连。

## 3.1 IdleStateHandler介绍


```

//readerIdleTime：读事件超时时间
//writerIdleTime：写事件超时事件
//allIdleTime：读或者写超时事件
public IdleStateHandler(long readerIdleTime, long writerIdleTime, long allIdleTime,TimeUnit unit) {
    this(false, readerIdleTime, writerIdleTime, allIdleTime, unit);
}
```

IdleStateHandler这个类会根据设置的超时参数，循环检测 channelRead 和 write 方法多久没有被调用。

当在 pipeline 中加入 IdleSateHandler 之后，可以在此 pipeline 的任意 Handler 的 userEventTriggered 方法之中检测 IdleStateEvent 事件。

## 3.2 dubbo如何使用IdleStateHandler

我们看下客户端和服务端启动netty服务加入的IdleStateHandler定义。

### 3.2.1 客户端

#### 3.2.1.1 NettyClient.doOpen():
```
    ch.pipeline().addLast("decoder", adapter.getDecoder())
                 .addLast("encoder", adapter.getEncoder())
                 .addLast("client-idle-handler", new IdleStateHandler(60*1000, 0, 0, MILLISECONDS))
                 .addLast("handler", nettyClientHandler);
```
在channelPipeline中加入IdleStateHandler，配置了 read 超时为 60s，也就是客户端会检测超过60s没有读事件的channel。

#### 3.2.1.2 NettyClientHandler.userEventTriggered:

```
   @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // send heartbeat when read idle.
        if (evt instanceof IdleStateEvent) {
            try {
                NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
                if (logger.isDebugEnabled()) {
                    logger.debug("IdleStateEvent triggered, send heartbeat to channel " + channel);
                }
                Request req = new Request();
                req.setVersion(Version.getProtocolVersion());
                req.setTwoWay(true);
                req.setEvent(Request.HEARTBEAT_EVENT);
                channel.send(req);
            } finally {
                NettyChannel.removeChannelIfDisconnected(ctx.channel());
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }
```

每60s检测到没有读事情channel的时候，通过userEventTriggered构建一个心跳request发送给服务端，我们这里注意构建的Request有个TwoWay参数，代表这个请求是双向的，服务端收到会进行回复。

###  3.2.2 服务端

#### 3.2.2.1 NettyServer.doOpen():

```
     ch.pipeline().addLast("decoder", adapter.getDecoder())
                  .addLast("encoder", adapter.getEncoder())
                  .addLast("server-idle-handler", new IdleStateHandler(0, 0, 200*1000, MILLISECONDS))
                  .addLast("handler", nettyServerHandler);
```

在channelPipeline中加入IdleStateHandler，配置了 write/read 超时为 200s。

#### 3.2.2.2 NettyServerHandler.userEventTriggered

```
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // server will close channel when server don't receive any heartbeat from client util timeout.
        if (evt instanceof IdleStateEvent) {
            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
            try {
                logger.info("IdleStateEvent triggered, close channel " + channel);
                channel.close();
            } finally {
                NettyChannel.removeChannelIfDisconnected(ctx.channel());
            }
        }
        super.userEventTriggered(ctx, evt);
    }
```
每200s检测没有读写事件的channel，通过触发userEventTriggered，直接关闭channel。

#### 3.2.2.3 HeartbeatHandler

前面客户端检测到通道读空闲会发送心跳包给服务端，服务端会通过HeartbeatHandler接收到心跳包，然后构建response再发送给客户端，客户端也通过HeartbeatHandler接收心跳响应。

不管是客户端还是服务端在处理心跳包时都会在当前通道设置当前的读时间戳（setReadTimestamp(channel)）。

这个时间戳的意义我们在重连机制中会作为检测链接是否需要重连的重要依据。

```
  @Override
    public void received(Channel channel, Object message) throws RemotingException {
        setReadTimestamp(channel);
        if (isHeartbeatRequest(message)) {
            Request req = (Request) message;
            if (req.isTwoWay()) {
                Response res = new Response(req.getId(), req.getVersion());
                res.setEvent(Response.HEARTBEAT_EVENT);
                channel.send(res);
                if (logger.isInfoEnabled()) {
                    int heartbeat = channel.getUrl().getParameter(Constants.HEARTBEAT_KEY, 0);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Received heartbeat from remote channel " + channel.getRemoteAddress()
                                + ", cause: The channel has no data-transmission exceeds a heartbeat period"
                                + (heartbeat > 0 ? ": " + heartbeat + "ms" : ""));
                    }
                }
            }
            return;
        }
        if (isHeartbeatResponse(message)) {
            if (logger.isDebugEnabled()) {
                logger.debug("Receive heartbeat response in thread " + Thread.currentThread().getName());
            }
            return;
        }
        handler.received(channel, message);
    }
```


### 3.2.3 结论

**1. 客户端每隔60s检测读空闲的channel，检测到读空闲时，则发送一次心跳包。**

**2. 服务端每200s检测一次是否有读或者写是否空闲的通道，如果空闲
则直接将该channel关闭。**

**3. 这里客户端服务端检测空闲时长设置成200s（客户端60s的3倍多），是为了要等待客户端重试3次之后依然没有收到任何的读写事件才能认定channel已经非正常端开了，需要回收链接。**

**4. 虽然Netty的IdleStateHandler是一种单向的心跳机制，但dubbo使用是按照双向设计的（服务端收到心跳响应还会回复心跳给客户端）**

分析到这里，我们知道了服务端如何感知客户端链接的存活情况，然后对不可用的链接直接进行关闭。

**那对于客户端如何感知与服务端链接的可用性呢**？


## 3.3 重连机制

前面心跳机制部分主要是保证了服务端如何感知客户端链接的可用性问题，检测到不可用链接后直接关闭链接。

而对于客户端如何感知到服务端链接的可用，主要通过的一个重连机制的定时任务。

### 3.3.1 重连任务


在dubbo客户端将远程服务调用封装成invoker的过程中，会经过一个HeaderExchangeClient，我们看下HeaderExchangeClient构造函数：

这里会启动两个定时任务startReconnectTask与startHeartBeatTask。

startReconnectTask就是启动一个重连定时任务。（
其中startHeartBeatTask已经在dubbo2.7.x版本中已经不再使用了。）

```
    public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);

        if (startTimer) {
            URL url = client.getUrl();
            startReconnectTask(url);
            startHeartBeatTask(url);
        }
    }

```

重连任务每60s会执行一次。检测到链接断开了或者通道上次读时间超过了60s*3（这里的读超时就是通过HeartbeatHandler接收的心跳包设置的时间戳来判断的），则会进行重连。


```
    @Override
    protected void doTask(Channel channel) {
        try {
            Long lastRead = lastRead(channel);
            Long now = now();

            // Rely on reconnect timer to reconnect when AbstractClient.doConnect fails to init the connection
            if (!channel.isConnected()) {
                try {
                    logger.info("Initial connection to " + channel);
                    ((Client) channel).reconnect();
                } catch (Exception e) {
                    logger.error("Fail to connect to " + channel, e);
                }
            // check pong at client
            } else if (lastRead != null && now - lastRead > idleTimeout) {
                logger.warn("Reconnect to channel " + channel + ", because heartbeat read idle time out: "
                        + idleTimeout + "ms");
                try {
                    ((Client) channel).reconnect();
                } catch (Exception e) {
                    logger.error(channel + "reconnect failed during idle time.", e);
                }
            }
        } catch (Throwable t) {
            logger.warn("Exception when reconnect to remote channel " + channel.getRemoteAddress(), t);
        }
    }
```