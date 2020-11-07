title: Dubbo源码分析之三：服务引用
author: Silence
tags:
  - Dubbo
categories:
  - Dubbo
date: 2020-04-12 12:01:00
---
# 1. 开篇
在Dubbo中，有两种方式引用服务，第一种是使用服务直连的方式引用服务，第二种方式是基于注册中心进行引用。本文重点分析基于注册中心进行服务引用的过程。


```
    ReferenceConfig<DemoService> reference = new ReferenceConfig<>();
    reference.setApplication(new ApplicationConfig("dubbo-demo-api-consumer"));
    reference.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
    reference.setInterface(DemoService.class);
    reference.setTimeout(Integer.MAX_VALUE);
    DemoService service = reference.get();
    String message = service.sayHello("dubbo");
```


服务引用的入口：ReferenceConfig.get();

# 2. 服务引用

和服务暴露的过程类似，服务引用也主要分两个过程：

**1. 服务接口转变成invoker（通过Protocol.refer完成）**

**2. invoker转成Proxy（通过ProxyFactory.getProxy完成）**

![image](http://static.silence.work/20200412-1.png)

## 2.1 前置过程

在进入服务接口转变成invoker方法前会经过如下调用链路


```
ReferenceConfig.get()
    -->ReferenceConfig.checkAndUpdateSubConfigs
        -->ReferenceConfig.init()
            -->ReferenceConfig.createProxy(map)
```


### 2.1.1 ReferenceConfig.init()

这里主要是处理相关配置，会在init()方法中将相关参数封装在map中，map中主要封装的参数有服务接口名、方法名、要注册的ip等信息等。


```
0 = {HashMap$Node@1727} "side" -> "consumer"
1 = {HashMap$Node@1728} "application" -> "dubbo-demo-api-consumer"
2 = {HashMap$Node@1729} "register.ip" -> "10.204.246.193"
3 = {HashMap$Node@1730} "release" -> 
4 = {HashMap$Node@1731} "methods" -> "sayHello"
5 = {HashMap$Node@1732} "lazy" -> "false"
6 = {HashMap$Node@1733} "sticky" -> "false"
7 = {HashMap$Node@1734} "dubbo" -> "2.0.2"
8 = {HashMap$Node@1735} "pid" -> "806"
9 = {HashMap$Node@1736} "interface" -> "org.apache.dubbo.demo.DemoService"
10 = {HashMap$Node@1737} "timeout" -> "2147483647"
11 = {HashMap$Node@1738} "timestamp" -> "1575251740703"
```
然后进入createProxy(map)，我们主要分析引用服务的过程。

### 2.1.2 ReferenceConfig.createProxy(map);

```
  private T createProxy(Map<String, String> map) {
        //是否本地调用
        if (shouldJvmRefer(map)) {
            URL url = new URL(LOCAL_PROTOCOL, LOCALHOST_VALUE, 0, interfaceClass.getName()).addParameters(map);
            //直接使用InjvmProtocol 的 refer 方法生成 Invoker 实例
            invoker = REF_PROTOCOL.refer(interfaceClass, url);
            if (logger.isInfoEnabled()) {
                logger.info("Using injvm service " + interfaceClass.getName());
            }
        } else {
            urls.clear(); // reference retry init will add url to urls, lead to OOM
            //是否指定了直连的url
            if (url != null && url.length() > 0) { // user specified URL, could be peer-to-peer address, or register center's address.
                //分割多个url
                String[] us = SEMICOLON_SPLIT_PATTERN.split(url);
                if (us != null && us.length > 0) {
                    for (String u : us) {
                        URL url = URL.valueOf(u);
                        if (StringUtils.isEmpty(url.getPath())) {
                            url = url.setPath(interfaceName);
                        }
                        // 检测 url 协议是否为 registry，若是，表明用户想使用指定的注册中心
                        if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                            urls.add(url.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        } else {
                            urls.add(ClusterUtils.mergeUrl(url, map));
                        }
                    }
                }
            } else { // assemble URL from register center's configuration
                // if protocols not injvm checkRegistry
                if (!LOCAL_PROTOCOL.equalsIgnoreCase(getProtocol())){
                    //校验注册中心
                    checkRegistry();
                    //获取注册中心地址
                    List<URL> us = loadRegistries(false);
                    if (CollectionUtils.isNotEmpty(us)) {
                        for (URL u : us) {
                            URL monitorUrl = loadMonitor(u);
                            if (monitorUrl != null) {
                                map.put(MONITOR_KEY, URL.encode(monitorUrl.toFullString()));
                            }
                            urls.add(u.addParameterAndEncoded(REFER_KEY, StringUtils.toQueryString(map)));
                        }
                    }
                    if (urls.isEmpty()) {
                        throw new IllegalStateException("No such any registry to reference " + interfaceName + " on the consumer " + NetUtils.getLocalHost() + " use dubbo version " + Version.getVersion() + ", please config <dubbo:registry address=\"...\" /> to your spring config.");
                    }
                }
            }
            //上面的ifelse目的很简单：处理不同情况，最终得到urls集合。
            //只有一个url，直接调用refer
            if (urls.size() == 1) {
                invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
            } else {
                List<Invoker<?>> invokers = new ArrayList<Invoker<?>>();
                URL registryURL = null;
                //遍历所有url，遍历执行refer，然后放到invoker集合中
                for (URL url : urls) {
                    invokers.add(REF_PROTOCOL.refer(interfaceClass, url));
                    if (REGISTRY_PROTOCOL.equals(url.getProtocol())) {
                        registryURL = url; // use last registry url
                    }
                }
                //利用Cluster的join方法将集群invokers合并成一个invoker
                if (registryURL != null) { // registry url is available
                    // use RegistryAwareCluster only when register's CLUSTER is available
                    URL u = registryURL.addParameter(CLUSTER_KEY, RegistryAwareCluster.NAME);
                    // The invoker wrap relation would be: RegistryAwareClusterInvoker(StaticDirectory) -> FailoverClusterInvoker(RegistryDirectory, will execute route) -> Invoker
                    invoker = CLUSTER.join(new StaticDirectory(u, invokers));
                } else { // not a registry url, must be direct invoke.
                    invoker = CLUSTER.join(new StaticDirectory(invokers));
                }
            }
        }
        ...
        ...
        // 根据invoker生成proxy
        return (T) PROXY_FACTORY.getProxy(invoker);
    }
```
**此段代码逻辑较长，这里做下总结：**

1、检查是否为本地调用，若是，则调用 InjvmProtocol 的 refer 方法生成 InjvmInvoker 实例；

2、封装Url对象：若不是本地调用，则读取直连配置项，或注册中心 url。如果是通过注册中心来进行调用，则先校验所有的注册中心，然后加载注册中心的url，遍历每个url，加入监控中心url配置，最后把每个url保存到urls并将读取到的 url 存储到 urls 中。

3、如果urls只有一个url，则直接调用REF_PROTOCOL.refer获取invoker；如果不是，则遍历urls，根据每个url都调用REF_PROTOCOL.refer得到对应invoker，然后再通过 Cluster 合并invokers集合为一个invoker。

4、利用PROXY_FACTORY.getProxy将上面得到的invoker转化为proxy代理类；

**通过上面总结的四点，我们发现两行非常关键的代码：**

1、通过refer获取invoker；

```
Invoker invoker = REF_PROTOCOL.refer(interfaceClass, url);
```

2、通过invoker获取proxy。

```
PROXY_FACTORY.getProxy(invoker);
```
引用服务主要围绕这两行代码逐一分析。也就是前面提到的两个过程。


## 2.2 服务接口转invoker

```
Invoker invoker = REF_PROTOCOL.refer(interfaceClass, url);
```

也就是根据Protocol的refer方法创建invoker。我们知道invoker是Dubbo 的核心模型，代表一个可执行体，我们搞清楚invoker的生成过程会对我们认识Dubbo的远程调用链路起到很大帮助。

这里的REF_PROTOCOL是根据SPI的自适应机制动态生成的Protocol$Adaptive，它的refer方法实际就是根据url中的protocol参数选择指定的Protocol实现类。这里我们是通过注册中心进行服务引用，因此实际进入的是RegistryProtocol.refer方法。


**注：** 在服务暴露过程分析中我们获取RegistryProtocol时讲到由于Protocol本身SPI配置有配置两个包装类：ProtocolFilterWrapper和ProtocolListenerWrapper
所以我们获取的都是经过包装后的RegistryProtocol。

不过在服务引用这里没有做任何增强逻辑
```

//Protocol$Adaptive.refer
invoker = REF_PROTOCOL.refer(interfaceClass, urls.get(0));
    ->ProtocolListenerWrapper.refer
        ->ProtocolFilterWrapper.refer
            ->RegistryProtocol.refer
```

- **接下来我们分析RegistryProtocol.refer方法**


```
    public <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException {
        //将registry://替换为zookeeper://
        url = URLBuilder.from(url) .setProtocol(url.getParameter(REGISTRY_KEY, DEFAULT_REGISTRY))
                .removeParameter(REGISTRY_KEY)
                .build();
        //获取注册中心实例        
        Registry registry = registryFactory.getRegistry(url);
        if (RegistryService.class.equals(type)) {
            return proxyFactory.getInvoker((T) registry, type, url);
        }
        // 获取 group 配置，目的是获取配置的cluster
        // group="a,b" or group="*"
        Map<String, String> qs = StringUtils.parseQueryString(url.getParameterAndDecoded(REFER_KEY));
        String group = qs.get(GROUP_KEY);
        if (group != null && group.length() > 0) {
            if ((COMMA_SPLIT_PATTERN.split(group)).length > 1 || "*".equals(group)) {
                return doRefer(getMergeableCluster(), registry, type, url);
            }
        }
        return doRefer(cluster, registry, type, url);
    }
```

首先为 url 设置协议头，然后根据 url 参数加载注册中心实例，没有则创建。创建注册中心过程和服务暴露过程获取注册中心实例基本一致。

然后获取 group 配置，根据 group 配置决定doRefer方法的 第一个参数的类型，然后调用doRefer();

- **我们继续看doRefer方法：**

```
private <T> Invoker<T> doRefer(Cluster cluster, Registry registry, Class<T> type, URL url) {
        //创建 RegistryDirectory 实例
        RegistryDirectory<T> directory = new RegistryDirectory<T>(type, url);
        directory.setRegistry(registry);
        directory.setProtocol(protocol);
        // all attributes of REFER_KEY
        Map<String, String> parameters = new HashMap<String, String>(directory.getUrl().getParameters());
        //生成服务消费者链接
        URL subscribeUrl = new URL(CONSUMER_PROTOCOL, parameters.remove(REGISTER_IP_KEY), 0, type.getName(), parameters);
        if (!ANY_VALUE.equals(url.getServiceInterface()) && url.getParameter(REGISTER_KEY, true)) {
            directory.setRegisteredConsumerUrl(getRegisteredConsumerUrl(subscribeUrl, url));
            //注册服务消费者，在 consumers 目录下新节点
            registry.register(directory.getRegisteredConsumerUrl());
        }
        directory.buildRouterChain(subscribeUrl);
        //订阅 providers、configurators、routers 等节点数据
        directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));
        //获取invoker，一个注册中心可能有多个服务提供者，因此这里需要将多个服务提供者合并为一个
        Invoker invoker = cluster.join(directory);
        ProviderConsumerRegTable.registerConsumer(invoker, url, subscribeUrl, directory);
        return invoker;
    }
```
**这里总结下doRefer主要做了以下几个事情**：

1. 创建一个 RegistryDirectory 实例，然后生成服务者消费者链接。例如：

```
consumer://192.168.3.69/org.apache.dubbo.demo.DemoService?xxxx
```

2. 将上面获取的消费者链接向注册中心进行注册，在consumer目录下。

3. 订阅 providers、configurators、routers 等节点下的数据，完成订阅后，RegistryDirectory 会收到这几个节点下的子节点信息

4. 如果是集群服务提供，则通过cluster.join合并为一个invoker。

关于RegistryProtocol.refer方法我们需要重点关注订阅的过程以及cluster.join这两个过程。

这里先看RegistryDirectory.subscribe的订阅逻辑

### 2.2.1 RegistryDirectory.subscribe

```
    directory.subscribe(subscribeUrl.addParameter(CATEGORY_KEY,
                PROVIDERS_CATEGORY + "," + CONFIGURATORS_CATEGORY + "," + ROUTERS_CATEGORY));
```

RegistryDirectory是一个动态服务目录，会随注册中心配置的变化进行动态调整。因此 RegistryDirectory 实现了 NotifyListener 接口，通过这个接口获取注册中心变更通知。

RegistryDirectory.subscribe在进行订阅之后会立即同步执行回调逻辑，回调的过程实际就是根据服务地址封装invoker的主要过程，回调的链路如下：

```
RegistryDirectory.subscribe
-->FailbackRegistry.subscribe
    -->ZookeeperRegistry.doSubscribe
        -->FailbackRegistry.notify
            -->AbstractRegistry.notify
                -->RegistryDirectory.notify
                    -->refreshInvoker(urls)
```

这里我们只关注监听providers节点获取的回调通知逻辑。

这里我们从refreshInvoker开始分析。

#### 2.2.1.1 refreshInvoker

```
    private void refreshInvoker(List<URL> invokerUrls) {
        // invokerUrls 仅有一个元素，且 url 协议头为 empty，此时表示禁用所有服务
        if (invokerUrls.size() == 1
                && invokerUrls.get(0) != null
                && EMPTY_PROTOCOL.equals(invokerUrls.get(0).getProtocol())) {
            this.forbidden = true; // Forbid to access
            this.invokers = Collections.emptyList();
            routerChain.setInvokers(this.invokers);
            // 销毁所有 Invoker
            destroyAllInvokers(); // Close all invokers
        } else {
            this.forbidden = false; // Allow to access
            //缓存的urlInvokerMap
            Map<String, Invoker<T>> oldUrlInvokerMap = this.urlInvokerMap; // local reference
            if (invokerUrls == Collections.<URL>emptyList()) {
                invokerUrls = new ArrayList<>();
            }
            if (invokerUrls.isEmpty() && this.cachedInvokerUrls != null) {
                invokerUrls.addAll(this.cachedInvokerUrls);
            } else {
                this.cachedInvokerUrls = new HashSet<>();
                this.cachedInvokerUrls.addAll(invokerUrls);//Cached invoker urls, convenient for comparison
            }
            if (invokerUrls.isEmpty()) {
                return;
            }
            //将注册中心的服务列表地址转变Invoker
            //key为url，value为invoker
            Map<String, Invoker<T>> newUrlInvokerMap = toInvokers(invokerUrls);// Translate url list to Invoker map

            List<Invoker<T>> newInvokers = Collections.unmodifiableList(new ArrayList<>(newUrlInvokerMap.values()));
            // 合并多个组的 Invoker
            this.invokers = multiGroup ? toMergeInvokerList(newInvokers) : newInvokers;
            this.urlInvokerMap = newUrlInvokerMap;

            try {
                // 销毁无用 Invoker
                destroyUnusedInvokers(oldUrlInvokerMap, newUrlInvokerMap); // Close the unused Invoker
            } catch (Exception e) {
                logger.warn("destroyUnusedInvokers error. ", e);
            }
        }
    }
```

1、首先会根据入参 invokerUrls 的数量和协议头判断是否禁用所有的服务，如果禁用，则将 forbidden 设为 true，并销毁所有的 Invoker。若不禁用，则通过toInvokers将 url 转成 Invoker，得到 <url, Invoker> 的映射关系。

2、新的 Invoker 列表生成后，会通过destroyUnusedInvokers，销毁无用的 Invoker，避免服务消费者调用已下线的服务的服务。


#### 2.2.1.2 toInvoker(invokerUrls)

接下来看看toInvokers(invokerUrls)的逻辑，这里只看关键代码：

```
private Map<String, Invoker<T>> toInvokers(List<URL> urls) {
    Map<String, Invoker<T>> newUrlInvokerMap = new HashMap<>();
    //遍历服务列表urls
    for (URL providerUrl : urls) {
        Invoker invoker = protocol.refer(serviceType, url)
        invoker = new InvokerDelegate<>(invoker, url, providerUrl);
        newUrlInvokerMap.put(key, invoker);
    }
}
```
**主要做了两件事情：**

1. 遍历服务列表，利用protocol.refer将url转变为Invoker。
2. 将服务地址与对应Invoker缓存起来。


这里的protocal会根据自适应机制选择DubboProtocol调用refer，同样在获取DubboProtocol也会经过两个包装类ProtocolListenerWrapper和ProtocolFilterWrapper，这里增强的逻辑和服务暴露获取invoker一样，只是封装的Filter是consuemr端的Filter。


通过refer的调用链最终获得的Invoker对象是RegistryDirectory.InvokerDelegate。

```
Protocol$Adaptive.refer()
-->ProtocolFilterWrapper.refer()
    -->AbstractProtocol.refer()
        -->DubboProtocol.protocolBindingRefer()
            -->getClients(url)
```


DubboProtocol.protocolBindingRefer()逻辑如下：

```
    @Override
    public <T> Invoker<T> protocolBindingRefer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);
        //获取客户端实例
        ExchangeClient[] clients = getClients(url);
        // create rpc invoker.
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, clients, invokers);
        invokers.add(invoker);
        return invoker;
    }

```
这里会先获取ExchangeClient，然后封装到DubboInvoker。


我们看下getClients(url)方法，

##### getClients(url)

ExchangeClient封装了客户端与服务端通信的对象，例如NettyClient。

获取ExchangeClient链路较长，这里贴下关键链路


```
DubboProtocol.getClients(url)
-->getSharedClient()
    -->buildReferenceCountExchangeClient
        -->initClient()
            -->Exchangers.connect()
                -->HeaderExchanger.connect()
                    -->Transporters.connect()
                        -->NettyTransporter.connect()
                            -->new NettyClient()
```

在构造NettyClient(),首先是创建netty 客户端 bootstrap：


```
protected void doOpen() throws Throwable {
        final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
                .option(ChannelOption.SO_KEEPALIVE, true)
                .option(ChannelOption.TCP_NODELAY, true)
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(NioSocketChannel.class);

        bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, Math.max(3000, getConnectTimeout()));
        bootstrap.handler(new ChannelInitializer() {

            @Override
            protected void initChannel(Channel ch) throws Exception {
                int heartbeatInterval = UrlUtils.getHeartbeat(getUrl());
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        //客户端发送心跳的Handler .addLast("client-idle-handler", new IdleStateHandler(heartbeatInterval, 0, 0, MILLISECONDS))
                        .addLast("handler", nettyClientHandler);
                String socksProxyHost = ConfigUtils.getProperty(SOCKS_PROXY_HOST);
                if(socksProxyHost != null) {
                    int socksProxyPort = Integer.parseInt(ConfigUtils.getProperty(SOCKS_PROXY_PORT, DEFAULT_SOCKS_PROXY_PORT));
                    Socks5ProxyHandler socks5ProxyHandler = new Socks5ProxyHandler(new InetSocketAddress(socksProxyHost, socksProxyPort));
                    ch.pipeline().addFirst(socks5ProxyHandler);
                }
            }
        });
    }
```

接着调用doConnect()链接服务端。

```
protected void doConnect() throws Throwable {
    ChannelFuture future = bootstrap.connect(getConnectAddress());
    boolean ret = future.awaitUninterruptibly(getConnectTimeout(), MILLISECONDS);
    NettyClient.this.channel = newChannel;
}
```

##### new DubboInvoker<T>(serviceType, url, clients, invokers)

没有太多逻辑，主要是将服务接口url和上一步获取ExchangeClient封装成DubboInvoker。


### 2.2.2 cluster.join方法

将上一个过程经过directory.subscribe之后得到的RegistryDirectory转变成Invoker。

```
Invoker invoker = cluster.join(directory);
```

这里的cluster会根据spi的自适应进行加载，获取到默认的cluser：FailoverCluster。同样的Cluster也默认提供了包装类MockClusterWrapper。

MockClusterWrapper.join方法实际就是返回了MockClusterInvoker。


```
public class MockClusterWrapper implements Cluster {

    private Cluster cluster;

    public MockClusterWrapper(Cluster cluster) {
        this.cluster = cluster;
    }
    //包转mock机制
    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new MockClusterInvoker<T>(directory,
                this.cluster.join(directory));
    }

}
```
在封装MockClusterInvoker之前会先调用this.cluster.join(directory)。这里的cluser是FailoverCluster。

我们看FailoverCluster.join方法：

```
public class FailoverCluster implements Cluster {

    public final static String NAME = "failover";

    @Override
    public <T> Invoker<T> join(Directory<T> directory) throws RpcException {
        return new FailoverClusterInvoker<T>(directory);
    }

}

```

join方法从代码上看起来比较简单：其实就是返回FailoverClusterInvoker。

**那么我们这里总结下cluster.join的逻辑**：

将RegistryDirecory封装到FailoverClusterInvoker，然后再将FailoverClusterInvoker封装成MockClusterInvoker。

这里我们得到的invoker对应是这么一个关系：

![image](http://static.silence.work/20200412-2.png)



---

## 2.3 创建代理类


```
PROXY_FACTORY.getProxy(invoker);
```


根据自适应机制会默认选择使用javassist创建代理类，使用的是JavassistProxyFactory.getProxy方法

```
 public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
     return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
}

```
这里生成代理类和jdk的套路一样，只不过Proxy.getProxy方法是dubbo利用javassist的api实现的。

另外，new InvokerInvocationHandler(invoker)：实际就是和jdk的动态代理一样，用于拦截代理方法。

Proxy.getProxy(interfaces),生成代理类是dubbo利用了javassist的动态编译生成的代理类。

```
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        Class<?>[] parameterTypes = method.getParameterTypes();
        if (method.getDeclaringClass() == Object.class) {
            return method.invoke(invoker, args);
        }
        if ("toString".equals(methodName) && parameterTypes.length == 0) {
            return invoker.toString();
        }
        if ("hashCode".equals(methodName) && parameterTypes.length == 0) {
            return invoker.hashCode();
        }
        if ("equals".equals(methodName) && parameterTypes.length == 1) {
            return invoker.equals(args[0]);
        }
        //将请求的参数封装到RpcInvocation中
        return invoker.invoke(new RpcInvocation(method, args)).recreate();
    }
```

# 3. 总结

**服务引用的本质其实就是如何将远程服务url封装为一个可执行的Invoker。**

通过前面RegistryDirectory.subscribe过程中的notify过程分析，我们知道，RegistryDirecory利用订阅后立即同步回调，将服务接口url封装对应的Invoker，然后封装到RegistryDirectory，因此RegistryDirectory的只能我们可以理解为封装注册中心形式的服务目录。

服务目录中存储了一些和服务提供者有关的信息也就是一个invokers集合，invokers集合封装了服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用。

也就是RegistryDirecory封装的就是一个invoker集合，简单看下这个directory封装的invoker集合是什么样子。
![image](http://static.silence.work/20200412-3.png)


最后我通过一张图来表示通过引用服务获取的invoker结构如下图：

![image](http://static.silence.work/20200412-4.png)

这个图说明了服务引用过程是如何一层一层将要调用的远程服务包装成最终的Invoker，关于这里包装的一层一层Invoker的职责我们会在服务调用过程得到进一步认识。
