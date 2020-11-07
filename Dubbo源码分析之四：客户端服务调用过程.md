title: Dubbo源码分析之四：客户端服务调用过程
author: Silence
tags:
  - Dubbo
  - RPC
  - ''
categories:
  - Dubbo
date: 2020-04-12 22:17:00
---
# 1. 分析思路
我们以下面的Demo来分析服务调用的整个过程。

```
    public static void main(String[] args) throws IOException {
        ReferenceConfig<DemoService> reference = new ReferenceConfig<>();
        reference.setApplication(new ApplicationConfig("dubbo-demo-api-consumer"));
        reference.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
        reference.setInterface(DemoService.class);
//        reference.setCheck(false);
        DemoService service = reference.get();
        String message = service.sayHello("dubbo");
        System.out.println(message);
        System.in.read();
    }
```

通过获取服务接口的代理类Proxy，然后发起远程调用。

我们将发起调用的过程拆分为4个过程来分析：

**1、进入invoke调用链**

**2、发送请求**

**3、接收服务端响应**

**4、处理响应结果**

# invoke执行链路

代理类调用方法时实际是交给了InvokerInvocationHandler.invoke()方法，这里也是invoke执行链的开始。

**整体invoke的执行链路时序图：**

![image](http://static.silence.work/request-invoke.png)

## 1. InvokerInvocationHandler.invoke

```
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        ...
        ...
        //实际调用服务接口的入口
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
```

**主要做了两件事情：**
> 1、将调用的方法和参数封装成RpcInvocation作为Rpc调用过程中的参数。这个RpcInvocation代表的是调用过程中的会话域。
> 
> 2、执行invoker.invoke方法

InvokerInvocationHandler 中的 invoker 成员变量类型为 MockClusterInvoker，因此我们接着看MockClusterInvoker。


## 2. MockClusterInvoker.invoke

封装了服务降级的逻辑，我们看代码：

```
 public Result invoke(Invocation invocation) throws RpcException {
        Result result = null;
        //获取mock参数
        String value = directory.getUrl().getMethodParameter(invocation.getMethodName(), MOCK_KEY, Boolean.FALSE.toString()).trim();
        //mock参数不存在，则不走服务降级逻辑
        if (value.length() == 0 || "false".equalsIgnoreCase(value)) {
            //no mock
            result = this.invoker.invoke(invocation);
        } else if (value.startsWith("force")) {
           // force:xxx 直接执行 mock 逻辑，不发起远程调用
            result = doMockInvoke(invocation, null);
        } else {
           // fail:xxx 表示消费方对调用服务失败后，再执行 mock 逻辑，不抛出异常
            try {
                result = this.invoker.invoke(invocation);
                if(result.getException() != null && result.getException() instanceof RpcException){
                    RpcException rpcException= (RpcException)result.getException();
                    if(rpcException.isBiz()){
                        throw  rpcException;
                    }else {
                        result = doMockInvoke(invocation, rpcException);
                    }
                }

            } catch (RpcException e) {
                if (e.isBiz()) {
                    throw e;
                }
                result = doMockInvoke(invocation, e);
            }
        }
        return result;
    }
```
关于服务降级的逻辑这里不分析，后面单独分析

## 3. AbstractClusterInvoker.invoke()
封装了集群失败重试的容错的公共逻辑，也就是当集群环境某个节点调用失败之后，如何选择重试策略，doInvoke由不同容错策略的子类实现。
```

    public Result invoke(final Invocation invocation) throws RpcException {
        checkWhetherDestroyed();
        // 获取上下文Rpc参数并封装到invocation中
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (contextAttachments != null && contextAttachments.size() != 0) {
            ((RpcInvocation) invocation).addAttachments(contextAttachments);
        }
        //列举invoker
        List<Invoker<T>> invokers = list(invocation);
        //获取负载均衡策略
        LoadBalance loadbalance = initLoadBalance(invokers, invocation);
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);
        return doInvoke(invocation, invokers, loadbalance);
    }
```

1、根据RegistryDirectory.list方法获取服务列表invokers；

2、获取负载均衡策略LoadBalance，默认为RandomLoadBalance

3、选择具体的集群策略执行doInvoke。

## 4. FailoverClusterInvoker.doInvoke()
Failover Cluster是dubbo默认的容错策略：失败自动切换。

``` 
      public Result doInvoke(Invocation invocation, final List<Invoker<T>> invokers, LoadBalance loadbalance) throws RpcException {
        List<Invoker<T>> copyInvokers = invokers;
        checkInvokers(copyInvokers, invocation);
        String methodName = RpcUtils.getMethodName(invocation);
        //获取重试次数参数
        int len = getUrl().getMethodParameter(methodName, RETRIES_KEY, DEFAULT_RETRIES) + 1;
        if (len <= 0) {
            len = 1;
        }
        // retry loop.
        RpcException le = null; // last exception.
        List<Invoker<T>> invoked = new ArrayList<Invoker<T>>(copyInvokers.size()); // invoked invokers.
        Set<String> providers = new HashSet<String>(len);
        //按照重试次数循环调用
        for (int i = 0; i < len; i++) {
            //当发生重试的时候，重新调用list方法，获取最新的服务列表
            if (i > 0) {
                checkWhetherDestroyed();
                copyInvokers = list(invocation);
                // check again
                checkInvokers(copyInvokers, invocation);
            }
            //通过负载均衡策略从上面获取的invoker列表获取一个可用的invoker
            Invoker<T> invoker = select(loadbalance, invocation, copyInvokers, invoked);
            invoked.add(invoker);
            //设置 invoked 到 RPC 上下文中
            RpcContext.getContext().setInvokers((List) invoked);
            try {
                // 调用目标 Invoker 的 invoke 方法
                Result result = invoker.invoke(invocation);
                return result;
            } catch (RpcException e) {
                if (e.isBiz()) { // biz exception.
                    throw e;
                }
                le = e;
            } catch (Throwable e) {
                le = new RpcException(e.getMessage(), e);
            } finally {
                providers.add(invoker.getUrl().getAddress());
            }
        }
        //重试也失败，则抛出异常
        throw new RpcException()
    }
```

总结下大概做了哪些事：
> 
> 1、获取服务列表（也就是invoker集合）；
> 
> 2、获取负载均衡策略LoadBalance（默认是RandomLoadBalance）
> 
> 3、获取重试次数参数
> 
> 4、通过负载均衡策略从上面获取的invoker列表中获取一个可用的invoker；
> 
> 5、通过获取到的invoker调用invoke方法，这里的invoker是RegistryDirectory.InvokerDelegate的实例。
> 
> 6、如果调用失败，会重新循环调用，调用前会重新获取服务列表（防止服务列表有服务不可用），循环次数就是第三步获取的重试次数参数

关于第一步获取服务列表invokers和第四步根据负载均衡策略从invokers中选择一个invoker的逻辑，后面单独来分析。

## 5. InvokerWrapper.invoke
Invoker的包装类，但实际包装之后未做任何而外逻辑，因此继续往下看。

```
public class InvokerWrapper<T> implements Invoker<T> {

    private final Invoker<T> invoker;

    private final URL url;

    public InvokerWrapper(Invoker<T> invoker, URL url) {
        this.invoker = invoker;
        this.url = url;
    }
    ...
    ...
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        return invoker.invoke(invocation);
    }

```

## 6. ListenerInvokerWrapper.invoke

ListenerInvokerWrapper主要作用是在构造invoker和销毁invoker增加了回调监听器的逻辑处理。

不过这里dubbo默认并未提供任何InvokerListener，实际执行invoke时也无任何逻辑增强。


## 7.ProtocolFilterWrapper.CallbackRegistrationInvoker.invoke
1、进入所有拦截器的调用链中。

2、调用拦截器invoker之后，将返回的结果通过每个拦截器实现了Listener的方法进行异步回调通知。


```
 static class CallbackRegistrationInvoker<T> implements Invoker<T> {

        private final Invoker<T> filterInvoker;
        private final List<Filter> filters;

        public CallbackRegistrationInvoker(Invoker<T> filterInvoker, List<Filter> filters) {
            this.filterInvoker = filterInvoker;
            this.filters = filters;
        }

        @Override
        public Result invoke(Invocation invocation) throws RpcException {
            //调用拦截器链的invoke
            Result asyncResult = filterInvoker.invoke(invocation);
            //把异步返回的结果加入到上下文中
            asyncResult = asyncResult.whenCompleteWithContext((r, t) -> {
                //循环各个过滤器
                for (int i = filters.size() - 1; i >= 0; i--) {
                    Filter filter = filters.get(i);
                    //如果该Filter属于ListenableFilter，则执行listener的回调逻辑
                    if (filter instanceof ListenableFilter) {
                        Filter.Listener listener = ((ListenableFilter) filter).listener();
                        if (listener != null) {
                            if (t == null) {
                          //调用回调方法onResponse
                            listener.onResponse(r, filterInvoker, invocation);
                            } else {
                             
                            listener.onError(t, filterInvoker, invocation);
                            }
                        }
                    } else {
                    //这个是为了兼容2.7版本之前的逻辑
                        filter.onResponse(r, filterInvoker, invocation);
                    }
                }
            });
            return asyncResult;
        }
        ...
        ...
    }
```

## 8. ProtocolFilterWrapper的buildInvokerChain方法中的invoker实例的invoke方法

这里是经过一系列过滤器的开始。
该方法中是对异常的捕获，调用内部类Listener的onError来回调错误信息。接下来看它经过了哪些拦截器。

```
 private static <T> Invoker<T> buildInvokerChain(final Invoker<T> invoker, String key, String group) {
        Invoker<T> last = invoker;
        List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);

        if (!filters.isEmpty()) {
            for (int i = filters.size() - 1; i >= 0; i--) {
                final Filter filter = filters.get(i);
                final Invoker<T> next = last;
                last = new Invoker<T>() {
                    ...
                    ...
                    @Override
                    public Result invoke(Invocation invocation) throws RpcException {
                        Result asyncResult;
                        try {
                            // 依次调用各个过滤器，获得最终的返回结果
                            asyncResult = filter.invoke(next, invocation);
                        } catch (Exception e) {
                            // onError callback
                            if (filter instanceof ListenableFilter) {
                                Filter.Listener listener = ((ListenableFilter) filter).listener();
                                if (listener != null) {
                                    listener.onError(e, invoker, invocation);
                                }
                            }
                            throw e;
                        }
                        return asyncResult;
                    }
                };
            }
        }

        return new CallbackRegistrationInvoker<>(last, filters);
    }
```
我们看下默认经过了哪些Filter

### 8.1 ConsumerContextFilter

在当前的RpcContext中记录本次调用的一次状态信息，这里注意在finally中会调用removeContext，也就验证了RpcContext记录的上下文仅仅只针对一次调用有效。

```
    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        //将invoker、invocation、本地地址、远程地址等放到上下文
        RpcContext.getContext()
                .setInvoker(invoker)
                .setInvocation(invocation)
                .setLocalAddress(NetUtils.getLocalHost(), 0)
                .setRemoteAddress(invoker.getUrl().getHost(), invoker.getUrl().getPort())
                .setRemoteApplicationName(invoker.getUrl().getParameter(REMOTE_APPLICATION_KEY))
                .setAttachment(REMOTE_APPLICATION_KEY, invoker.getUrl().getParameter(APPLICATION_KEY));
        if (invocation instanceof RpcInvocation) {
            ((RpcInvocation) invocation).setInvoker(invoker);
        }
        try {
            RpcContext.removeServerContext();
            //调用下一个过滤器
            return invoker.invoke(invocation);
        } finally {
            RpcContext.removeContext();
        }
    }
```
该过滤器执行完成后，会回到上一步，循环下一个过滤器调用：FutureFilter

### 8.2 FutureFilter.invoke
处理dubbo的dubbo:method配置的onreturn、onthrow、oninvoke逻辑。

具体参考：[事件通知
](http://dubbo.apache.org/zh-cn/docs/user/demos/events-notify.html)

```
    public Result invoke(final Invoker<?> invoker, final Invocation invocation) throws RpcException {
        fireInvokeCallback(invoker, invocation);
        // need to configure if there's return value before the invocation in order to help invoker to judge if it's
        // necessary to return future.
        return invoker.invoke(invocation);
    }
```

### 8.3 MonitorFilter.invoke

```
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        // 如果设置了监控
        if (invoker.getUrl().hasParameter(MONITOR_KEY)) {
            invocation.setAttachment(MONITOR_FILTER_START_TIME, String.valueOf(System.currentTimeMillis()));
            getConcurrent(invoker, invocation).incrementAndGet(); // count up
        }
            return invoker.invoke(invocation); // proceed invocation chain
    }
```

该过滤器实际用来做监控，监控服务的调用数量。

## 9. AsyncToSyncInvoker.invoke

把异步结果转化为同步结果。

执行完invoke获取到Result之后，根据会话域中设置的invokeMode枚举，如果是同步调用，则会在这个类中进行异步结果转同步的处理（本质就是调用CompletableFuture.get方法）。

关于invokeMode的设置是在下一步AbstractInvoker.invoke执行invoke前设置的。

```
    public Result invoke(Invocation invocation) throws RpcException {
        Result asyncResult = invoker.invoke(invocation);

        try {
            // 如果是同步的调用
            if (InvokeMode.SYNC == ((RpcInvocation) invocation).getInvokeMode()) {
                //从异步结果中get结果
                asyncResult.get(Integer.MAX_VALUE, TimeUnit.MILLISECONDS);
            }
        } catch (InterruptedException e) {
            throw new RpcException("Interrupted unexpectedly while waiting for remoting result to return!  method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (ExecutionException e) {
            Throwable t = e.getCause();
            if (t instanceof TimeoutException) {
                throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
            } else if (t instanceof RemotingException) {
                throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
            }
        } catch (Throwable e) {
            throw new RpcException(e.getMessage(), e);
        }
        return asyncResult;
    }
```


## 10. AbstractInvoker.invoke

这里主要是不同协议的一个公共逻辑，主要是会话域中封装一些值。

例如：附加值、invokeMode、异步调用id。这里重点说明下invokeMode，也就是上一步执行完invoke需要拿来判断是否要做异步转同步的逻辑。

**invokeMode存在3个枚举值：**

- **FUTURE**：如果服务接口定义的返回参数是CompletableFuture，则属于FUTURE模式，FUTURE模式也属于Dubbo提供的一种异步调用方式（2.7版本提供，服务端异步方式）。
- **ASYNC**：如果consumer对服务接口指定了ASYNC，则属于ASYNC异步调用
- **SYNC**：默认值，同步调用。

这个invokeMode值除了上一步异步转同步中用到之外，还会在后面的处理响应结果是否为future的recreate方法中会用到。

```
    @Override
    public Result invoke(Invocation inv) throws RpcException {
        // if invoker is destroyed due to address refresh from registry, let's allow the current invoke to proceed
        if (destroyed.get()) {
            logger.warn("Invoker for service...");
        }
        //在会话域中加入该调用链
        RpcInvocation invocation = (RpcInvocation) inv;
        invocation.setInvoker(this);
        //把附加值放入会话域
        if (CollectionUtils.isNotEmptyMap(attachment)) {
            invocation.addAttachmentsIfAbsent(attachment);
        }
        //把上下文的附加值也放入会话域
        Map<String, String> contextAttachments = RpcContext.getContext().getAttachments();
        if (CollectionUtils.isNotEmptyMap(contextAttachments)) {
            /**
             * invocation.addAttachmentsIfAbsent(context){@link RpcInvocation#addAttachmentsIfAbsent(Map)}should not be used here,
             * because the {@link RpcContext#setAttachment(String, String)} is passed in the Filter when the call is triggered
             * by the built-in retry mechanism of the Dubbo. The attachment to update RpcContext will no longer work, which is
             * a mistake in most cases (for example, through Filter to RpcContext output traceId and spanId and other information).
             */
            invocation.addAttachments(contextAttachments);
        }
        //从配置中得到是什么模式的调用，一共有FUTURE、ASYNC和SYNC
        invocation.setInvokeMode(RpcUtils.getInvokeMode(url, invocation));
        // 如果调用是异步，则在会话域中添加id字段
        RpcUtils.attachInvocationIdIfAsync(getUrl(), invocation);

        try {
            return doInvoke(invocation);
        } catch (InvocationTargetException e) { // biz exception
            Throwable te = e.getTargetException();
            if (te == null) {
                return AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
            } else {
                if (te instanceof RpcException) {
                    ((RpcException) te).setCode(RpcException.BIZ_EXCEPTION);
                }
                return AsyncRpcResult.newDefaultAsyncResult(null, te, invocation);
            }
        } catch (RpcException e) {
            if (e.isBiz()) {
                return AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
            } else {
                throw e;
            }
        } catch (Throwable e) {
            return AsyncRpcResult.newDefaultAsyncResult(null, e, invocation);
        }
    }

```


## 11、DubboInvoker.doInvoke()



这里主要是获取与服务端的Client对象，以及对oneway和非oneway的调用方式对结果进行不同处理，最终通过AsyncRpcResult进行返回。

- oneway：异步无返回值的调用方式，consumer端只管发出去请求，不需要响应结果（返回一个空的AsyncRpcResult）。
- 非oneway：异步或者同步的调用方式，在这里实际都是当作异步的方式进行的处理，然后利用CompletableFuture.whenComplete的方式得到异步通知响应结果，主线程并不需要阻塞。

关于非oneway的方式为什么都是作为异步方式来处理？是因为在前面AsyncToSyncInvoker中已经做了同步的处理。所以这里并不需要等待。

```
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        // 再会话域中添加path、version字段
        inv.setAttachment(PATH_KEY, getUrl().getPath());
        inv.setAttachment(VERSION_KEY, version);
        //获取远程调用的ExchangeClient
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            //获取调用方式参数inOneway
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            //调用超时时间
            int timeout = getUrl().getMethodPositiveParameter(methodName, TIMEOUT_KEY, DEFAULT_TIMEOUT);
            //isoneway：异步无返回值。
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                //返回一个空的结果
                return AsyncRpcResult.newDefaultAsyncResult(invocation);
            } else {
                //同步调用或者异步有返回值
                AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
                CompletableFuture<Object> responseFuture = currentClient.request(inv, timeout);
                //调用之后异步通知结果 
                asyncRpcResult.subscribeTo(responseFuture);
                // save for 2.6.x compatibility, for example, TraceFilter in Zipkin uses com.alibaba.xxx.FutureAdapter
                FutureContext.getContext().setCompatibleFuture(responseFuture);
                return asyncRpcResult;
            }
        } catch (TimeoutException e) {
            //超时异常
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            //调用异常
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
    
    public void subscribeTo(CompletableFuture<?> future) {
        future.whenComplete((obj, t) -> {
            if (t != null) {
                this.completeExceptionally(t);
            } else {
                this.complete((Result) obj);
            }
        });
    }
```

到DubboInvoker，基本invoke的链路基本走完，接下来看发送请求的过程。


# 发送请求过程

**发送请求整体时序图**：

![image](http://static.silence.work/request-2.png)


## 1. ReferenceCountExchangeClient.request；

ReferenceCountExchangeClient 内部定义了一个引用计数变量 referenceCount，每当该对象被引用一次 referenceCount 都会进行自增。每当 close 方法被调用时，referenceCount 进行自减。ReferenceCountExchangeClient 内部仅实现了一个引用计数的功能，其他方法并无复杂逻辑，均是直接调用被装饰对象的相关方法。

```
final class ReferenceCountExchangeClient implements ExchangeClient {
    private final URL url;
    private final AtomicInteger referenceCount = new AtomicInteger(0);

    private ExchangeClient client;

    public ReferenceCountExchangeClient(ExchangeClient client) {
        this.client = client;
        //引用计数+1
        referenceCount.incrementAndGet();
        this.url = client.getUrl();
    }
    ...
    ...
    @Override
    public CompletableFuture<Object> request(Object request, int timeout) throws RemotingException {
        注解调用被包装对象的request
        return client.request(request, timeout);
    }
    }
```
## 2. HeaderExchangeClient.request

封装了心跳检测以及重连的逻辑；
对应的心跳检测和服务重连Task分别为：HeartbeatTimerTask、ReconnectTimerTask。

线程名称为：dubbo-client-idleCheck-。

不过在dubbo选择使用netty4之后已经不再使用自己实现的HeartbeatTimerTask了，而是利用了netty本身提供的心跳机制IdleStateHandler。

重连的任务ReconnectTimerTask依然保留

```
    public HeaderExchangeClient(Client client, boolean startTimer) {
        Assert.notNull(client, "Client can't be null");
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);

        if (startTimer) {
            URL url = client.getUrl();
            //开启重连任务
            startReconnectTask(url);
            //开启心跳任务
            startHeartBeatTask(url);
        }
    }
    
    private void startHeartBeatTask(URL url) {
        //新版本这里返回false
        if (!client.canHandleIdle()) {
            AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
            int heartbeat = getHeartbeat(url);
            long heartbeatTick = calculateLeastDuration(heartbeat);
            //心跳任务，默认是一分钟
            this.heartBeatTimerTask = new HeartbeatTimerTask(cp, heartbeatTick, heartbeat);
            IDLE_CHECK_TIMER.newTimeout(heartBeatTimerTask, heartbeatTick, TimeUnit.MILLISECONDS);
        }
    }

    private void startReconnectTask(URL url) {
        if (shouldReconnect(url)) {
            AbstractTimerTask.ChannelProvider cp = () -> Collections.singletonList(HeaderExchangeClient.this);
            int idleTimeout = getIdleTimeout(url);
            long heartbeatTimeoutTick = calculateLeastDuration(idleTimeout);
            //重连任务
            this.reconnectTimerTask = new ReconnectTimerTask(cp, heartbeatTimeoutTick, idleTimeout);
            IDLE_CHECK_TIMER.newTimeout(reconnectTimerTask, heartbeatTimeoutTick, TimeUnit.MILLISECONDS);
        }
    }
```


## 3. HeaderExchangeChannel.request

- 封装Request；
- 构建DefaultFuture；


### 3.1 封装Request

```
   public CompletableFuture<Object> request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        Request req = new Request();
        req.setVersion(Version.getProtocolVersion());
        req.setTwoWay(true);
        req.setData(request);
        //DefaultFuture
        DefaultFuture future = DefaultFuture.newFuture(channel, req, timeout);
        try {
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```

### 3.2 构建DefaultFuture


```
  public static DefaultFuture newFuture(Channel channel, Request request, int timeout) {
        final DefaultFuture future = new DefaultFuture(channel, request, timeout);
        // timeout check
        timeoutCheck(future);
        return future;
    }
```


这里我们重点看下DefaultFuture类以及构建DefaultFuture的过程：

DefaultFuture本身继承了CompletableFuture，同时也维护了一个静态的缓存FUTURES map对象，这里后面在处理服务端响应Response的时候也会分析DefaultFuture的意义。

```
ublic class DefaultFuture extends CompletableFuture<Object> {
    
    private static final Map<Long, Channel> CHANNELS = new ConcurrentHashMap<>();
    
    //静态的map缓存变量
    private static final Map<Long, DefaultFuture> FUTURES = new ConcurrentHashMap<>();

    //获取当前request对象的id作为key，将当前DefaultFuture存放在缓存变量中
    private DefaultFuture(Channel channel, Request request, int timeout) {
        this.channel = channel;
        this.request = request;
        this.id = request.getId();
        this.timeout = timeout > 0 ? timeout : channel.getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
        // put into waiting map.
        FUTURES.put(id, this);
        CHANNELS.put(id, channel);
    }
}
```

### 3.3 调用超时定时任务

注意在构建DefaultFuture之后还会启动一个定时任务线程检测调用超时。定时线程执行频率根据timeout的配置值决定，默认一秒，接口调用超时就是通过该定时任务实现的。


```
   private static void timeoutCheck(DefaultFuture future) {
        TimeoutCheckTask task = new TimeoutCheckTask(future.getId());
        future.timeoutCheckTask = TIME_OUT_TIMER.newTimeout(task, future.getTimeout(), TimeUnit.MILLISECONDS);
    }
    
    //定时任务
    private static class TimeoutCheckTask implements TimerTask {

        private final Long requestID;

        TimeoutCheckTask(Long requestID) {
            this.requestID = requestID;
        }

        @Override
        public void run(Timeout timeout) {
            DefaultFuture future = DefaultFuture.getFuture(requestID);
            if (future == null || future.isDone()) {
                return;
            }
            //没进if说明超时了
            // create exception response.
            Response timeoutResponse = new Response(future.getId());
            // set timeout status.
            timeoutResponse.setStatus(future.isSent() ? Response.SERVER_TIMEOUT : Response.CLIENT_TIMEOUT);
            timeoutResponse.setErrorMessage(future.getTimeoutMessage(true));
            // 构建超时的response放到DefaultFuture
            DefaultFuture.received(future.getChannel(), timeoutResponse, true);

        }
    }
```



接下来的调用就具体跟远程通信有关了，也就是 进入了Transport层。

## 4. AbstractClient.send

> 1、如果需要重连，则重新连接；
> 
> 2、获取Channel，默认获取NettyChannel。
> 
> 3、通过通道发送消息

```
    @Override
    public void send(Object message, boolean sent) throws RemotingException {
        //如果需要重连，则回重新进行链接
        if (needReconnect && !isConnected()) {
            connect();
        }
        Channel channel = getChannel();
        //TODO Can the value returned by getChannel() be null? need improvement.
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }
        channel.send(message, sent);
    }
```


## 5. NettyChannel.send

这里是真正调用netty api向服务端写入数据的地方；

这里netty在写入数据的时候，默认是异步写入的，
 
这里有一个特别的参数sent来确定是否等待消息发出，这个sent的值可以在MethodConfig中配置。有两个值：

- **true：等待消息发出，消息发送失败将抛出异常。**
- **false：默认值，不等待消息发出，将消息放入 IO 队列，即刻返回**

```
    public void send(Object message, boolean sent) throws RemotingException {
        // whether the channel is closed
        super.send(message, sent);

        boolean success = true;
        int timeout = 0;
        try {
            //写入数据，发送消息，这里的channel是Netty本身的Channel类
            ChannelFuture future = channel.writeAndFlush(message);
            //默认情况下 sent = false
            if (sent) {
                // wait timeout ms
                timeout = getUrl().getPositiveParameter(TIMEOUT_KEY, DEFAULT_TIMEOUT);
                //等待消息发出，若在规定时间没能发出，success 会被置为 false
                success = future.await(timeout);
            }
            Throwable cause = future.cause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
        }
        if (!success) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                    + "in timeout(" + timeout + "ms) limit");
        }
    }
```

## 6. ChannelHandler执行链路

根据服务引用的分析，consumer端启动netty客户端时（NettyClient.open），在ChannelPipeline链中加入了4个ChannelHandler，分别是：

- NettyCodecAdapter.InternalEncoder：编码器
- NettyCodecAdapter.InternalDecoder：解码器
- IdleStateHandler：心跳处理器
- NettyClientHandler：最终的消息处理器

因此在上一步调用writeAndFlush之后，根据出站方向主要会经过NettyClientHandler和编码器NettyCodecAdapter.InternalEncoder进行编码后发送出去。

我们看下NettyClientHandler的write方法：


```
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        super.write(ctx, msg, promise);
        final NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);
        final boolean isRequest = msg instanceof Request;
        //ChannelPromise是ChannelFuture的子类
        //加入写入数据完成后的监听，这里主要是处理出站异常
        promise.addListener(future -> {
            try {
                if (future.isSuccess()) {
                    // if our future is success, mark the future to sent.
                    handler.sent(channel, msg);
                    return;
                }
                //发送消息如果失败后，则提供处理出站异常逻辑，构建失败的Response
                Throwable t = future.cause();
                if (t != null && isRequest) {
                    Request request = (Request) msg;
                    Response response = buildErrorResponse(request, t);
                    handler.received(channel, response);
                }
            } finally {
                NettyChannel.removeChannelIfDisconnected(ctx.channel());
            }
        });
    }
```


接着进入编码器，这里主要列下编码器的调用链路，不做编码的具体分析。

```
InternalEncoder.encode
-->DubboCountCodec.encode
    -->DubboCodec.encode
        -->ExchangeCodec.encode
            -->ExchangeCodec.encodeRequest
```



# 接收服务端响应

接收服务响应和发送消息一样也会经过指定的入站ChannelHandler，先经过解码器NettyCodecAdapter.InternalDecoder，然后经过NettyClientHandler。

![image](http://static.silence.work/receive.png)

## 1. 解码链路


```
InternalDecoder.decode()
    -->DubboCountCodec.decode()
        -->ExchangeCodec.decode()
            -->DubboCodec.decodeBody()
```

## 2. NettyClientHandler.channelRead()


```
NettyClientHandler.channelRead()
-->AbstractPeer.received()
    -->MultiMessageHandler.receive()
        -->HeartbeatHandler.receive()
            -->AllChannelHandler.receive()
```


这里主要看下AllChannelHandler.receive方法：

将接收到的响应派发给业务线程池（DubboClienthandler-服务ip:port-thread-）处理，这里其实就是解放出io线程，将处理逻辑交给业务线程，避免阻塞io线程。

```
  public void received(Channel channel, Object message) throws RemotingException {
        //获取客户端接收消息业务处理线程池
        //DubboClientHandler-192.168.100.12:20880-thread-xxx
        ExecutorService executor = getExecutorService();
        try {
            //将ChannelEventRunnable任务提交给线程池。
            executor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
        	if(message instanceof Request && t instanceof RejectedExecutionException){
        		Request request = (Request)message;
        		if(request.isTwoWay()){
        			String msg = "Server side(" + url.getIp() + "," + url.getPort() + ") threadpool is exhausted ,detail msg:" + t.getMessage();
        			Response response = new Response(request.getId(), request.getVersion());
        			response.setStatus(Response.SERVER_THREADPOOL_EXHAUSTED_ERROR);
        			response.setErrorMessage(msg);
        			channel.send(response);
        			return;
        		}
        	}
            throw new ExecutionException(message, channel, getClass() + " error when process received event .", t);
        }
    }
```

关于这里的派发策略，dubbo提供了5种策略，默认使用AllChannelHandler。

- **all：** 所有消息都派发到线程池，包括请求，响应，连接事件，断开事件，心跳等。默认。
AllChannelHandler。
- **direct**： 所有消息都不派发到线程池，全部在 IO 线程上直接执行。
WrappedChannelHandler。
- **message：** 只有请求响应消息派发到线程池，其它连接断开事件，心跳等消息，直接在 IO 线程上执行。MessageOnlyChannelHandler。
- **execution：** 只有请求消息派发到线程池，不含响应，响应和其它连接断开事件，心跳等消息，直接在 IO 线程上执行。
ExecutionChannelHandler。
- **connection：** 在 IO 线程上，将连接断开事件放入队列，有序逐个执行，其它消息派发到线程池。ConnectionOrderedChannelHandler。


## 3. ChannelEventRunnable线程链路

```
ChannelEventRunnable.run()
    -->DecodeHandler.receive()
        -->HeaderExchangeHandler.received()
            -->HeaderExchangeHandler.handleResponse()
                -->DefaultFuture.received(channel, response);
```

分析到这里我们重点看下DefaultFuture.received方法逻辑。

这里结合前面分析的HeaderExchangeChannel.request，在request前会构造DefaultFuture对象，同时将当前请求对应的DefaultFuture封装到缓存FUTURES中。

这里在接收响应就是通过response的id获取对应请求时的DefaultFuture，然后将响应结果封装到DefaultFuture中，从而实现了如何将异步请求和响应一一对应。

```
    public static void received(Channel channel, Response response, boolean timeout) {
        try {
            // 根据调用编号从 FUTURES 集合中查找指定的 DefaultFuture 对象
            DefaultFuture future = FUTURES.remove(response.getId());
            if (future != null) {
                Timeout t = future.timeoutCheckTask;
                if (!timeout) {
                    // decrease Time
                    t.cancel();
                }
                //封装rsponse响应结果
                future.doReceived(response);
            } else {
                logger.warn("The timeout response finally returned at "
                        + (new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS").format(new Date()))
                        + ", response " + response
                        + (channel == null ? "" : ", channel: " + channel.getLocalAddress()
                        + " -> " + channel.getRemoteAddress()));
            }
        } finally {
            CHANNELS.remove(response.getId());
        }
    }
    
    private void doReceived(Response res) {
        if (res == null) {
            throw new IllegalStateException("response cannot be null");
        }
        if (res.getStatus() == Response.OK) {
            this.complete(res.getResult());
        } else if (res.getStatus() == Response.CLIENT_TIMEOUT || res.getStatus() == Response.SERVER_TIMEOUT) {
            this.completeExceptionally(new TimeoutException(res.getStatus() == Response.SERVER_TIMEOUT, channel, res.getErrorMessage()));
        } else {
            this.completeExceptionally(new RemotingException(channel, res.getErrorMessage()));
        }
    }
```

到此接收服务端响应过程分析完，接下来看处理响应结果过程。


# 处理请求响应结果：recreate()

也就是执行完InvokerInvocationHandler.invoke()之后对AsyncRpcResult结果的处理：

```
AsyncRpcResult.recreate();
```

回过头看在DubboInvoker.doInvoker过程中，会构造好AsyncRpcResult对象，而AsyncRpcResult本身继承了CompletableFuture。会利用CompletableFuture.whenComplete的方式得到响应的结果。


```
    AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
    CompletableFuture<Object> defaultFuture = currentClient.request(inv, timeout);
    asyncRpcResult.subscribeTo(defaultFuture);
```


那在recreate方法中可以大概猜测其实就是帮使用者封装好从CompletableFuture中获取异步响应的结果的逻辑。


```
    @Override
    public Object recreate() throws Throwable {
        RpcInvocation rpcInvocation = (RpcInvocation) invocation;
        FutureAdapter future = new FutureAdapter(this);
        //把future放到Rpc上下文中
        RpcContext.getContext().setFuture(future);
        //如果属于future调用模型，则直接返回future
        if (InvokeMode.FUTURE == rpcInvocation.getInvokeMode()) {
            return future;
        }
        //对于非future模式，则获取响应结果
        return getAppResponse().recreate();
    }
    
    public Result getAppResponse() {
        try {
            //是否已经异步执行完了
            if (this.isDone()) {
                //返回结果
                return this.get();
            }
        } catch (Exception e) {
            // This should never happen;
            logger.error("Got exception when trying to fetch the underlying result from AsyncRpcResult.", e);
        }
        //还没有响应结果，则返回一个空的响应
        return new AppResponse();
    }
    
    
```

这里总结recreate方法处理的三个关键点：

**1、封装异步响应结果future到RpcContext中；**

**2、如果调用方式属于future模式（服务端异步），则直接返回future对象；**

对于Future模式我们在AbstractInvoker.invoke已经有过分析，对于future模式的调用，则返回交给使用者，自己通过调用future的获取结果方法来得到；

**3、不属于future模式，则判断是否已经获取到响应结果。有则调用get返回结果，没有则返回空的AppResponse。**

由于同步调用前面AsyncToSyncInvoker已经做了等待，因此这里一定能get到结果；而对于属于指定了async异步的调用，则需要关键点1中的RpcContext中获取future，然后自己从future中等待获取结果。

# 调用过程中的异步总结

![image](http://static.silence.work/dubbo.png)

1、在发送请求时，默认是利用了netty的异步发送，这里本质上是一次异步；
NettyChannel.send()，不阻塞等待响应结果。

2、netty会将接收到的响应结果交给业务线程池(DubboClienthandler-服务ip:port-thread-)，这是第二次异步（分离IO线程和业务线程）。

代码位置：AllChannelHandler.receive()。

最终将响应结果封装到DefaultFuture（也就是request对应的DefaultFuture）。

3、第二步得到的DefaultFuture会转换为AsyncRpcResult，AsyncRpcResult如何拿到DefaultFuture封装的响应结果也是通过异步获取，这里是第三次异步。

代码位置：DubboInvoker.doInvoke()。


```
    AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
    CompletableFuture<Object> defaultFuture = currentClient.request(inv, timeout);
    asyncRpcResult.subscribeTo(defaultFuture);
```
