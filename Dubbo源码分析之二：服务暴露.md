title: Dubbo源码分析之二：服务暴露
author: Silence
tags:
  - Dubbo
categories:
  - Dubbo
date: 2020-04-06 21:10:00
---
# 开篇
所谓服务暴露，是指生产者Provider一侧提供的服务要能暴露出来，以便能被消费者发现服务并调用到。

只有读明白了服务的暴露过程，我们才能为后面读懂服务端如果接收响应作准备。

例如本地有一个service类，需要将该service暴露出去，写法如下：

```
public class Application {
    public static void main(String[] args) throws Exception {
        ServiceConfig<DemoServiceImpl> service = new ServiceConfig<>();
        service.setApplication(new ApplicationConfig("dubbo-demo-api-provider"));
        service.setRegistry(new RegistryConfig("zookeeper://127.0.0.1:2181"));
        service.setInterface(DemoService.class);
        service.setRef(new DemoServiceImpl());
        service.export();
        System.in.read();
    }
}
```

通过将本地的service封装到ServiceConfig，然后利用
service.export()方法进行服务暴露。

因此服务暴露的关键入口服务暴露的入口方法是：ServiceConfig.export();


# 服务暴露过程

服务暴露过程会分本地服务暴露和远程服务暴露。但不管是本地暴露还是远程暴露，服务暴露过程总体都分为两个过程：

**1、具体服务转变成Invoker；**（通过ProxyFactory类的getInvoker方法完成的）

**2、Invoker转变成Exporter；**（通过Protocol的export方法完成）

![image](http://static.silence.work/20200406-01.png)


这里我们先看本地服务暴露的过程

## 本地服务暴露


```
  private void exportLocal(URL url) {
        URL local = URLBuilder.from(url)
                .setProtocol(LOCAL_PROTOCOL)
                .setHost(LOCALHOST_VALUE)
                .setPort(0)
                .build();
        //PROXY_FACTORY = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
        Invoker invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local);
        //根据invoker获取export
        Exporter<?> exporter = protocol.export(invoker);
        exporters.add(exporter);
        logger.info("Export dubbo service " + interfaceClass.getName() + " to local registry url : " + local);
    }
```


### 具体服务转变成Invoker

```
    Invoker invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, local);
```

PROXY_FACTORY是通过dubbo的adative机制动态编译生成的类：ProxyFactory$Adaptive。

利用adative机制在执行getInvoker时根据local设置的proxy参数动态选择哪个ProxyFactory实现类的getInvoker方法，Dubbo默认使用的是JavassistProxyFactory。

最后生成的Invoker代码如下：

```
    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                //此行代码将会是调用服务实现类最终的调用入口
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```

### Invoker转变成Exporter


```
    Exporter<?> exporter = protocol.export(invoker);
```

这里的protocal也是根据Adaptive机制动态编译生成的Protocol。

根据invoker.url.protocol的参数来选择使用相应的Protocol实现类来执行export方法；这里本地服务暴露选择的是InjvmProtocol。

InjvmProtocol.export执行的逻辑比较简单，只是做了一层包装：


```
Exporter exporter = new InjvmExporter<T>(invoker, invoker.getUrl().getServiceKey(), exporterMap);
```


这里需要注意的是，Protocol通过spi提供的类包含有ProtocolFilterWrapper、ProtocolListenerWrapper两个包装类。

**因此在执行InjvmProtocol的export方法之前会经过两个类的增强。**

#### 1.ProtocolFilterWrapper.export


```
protocol.export(buildInvokerChain(invoker, SERVICE_FILTER_KEY, CommonConstants.PROVIDER))
```

**增强逻辑：**

通过责任链模式对invoke加了很多的Filter，也就是在调用真正的invoer之前会经过很多filter。


```
List<Filter> filters = ExtensionLoader.getExtensionLoader(Filter.class).getActivateExtension(invoker.getUrl(), key, group);
```

Dubbo默认提供了很多的Filter，包括provider端和consumer端，这个本身也是dubbo开放出的一种拓展能力。

关于Filter后面单独说明。


#### 2.ProtocolListenerWrapper.export


```
List<InvokerListener> listeners = Collections.unmodifiableList(ExtensionLoader.getExtensionLoader(ExporterListener.class)                  .getActivateExtension(invoker.getUrl(), EXPORTER_LISTENER_KEY));
new ListenerExporterWrapper<T>(protocol.export(invoker),listeners);
```

**增强逻辑：**

对获取的的Exporter包装了一层ListenerExporterWrapper类。目的是为了在export之前执行自定义的监听操作，可以理解成是对export的一个拓展点。dubbo本地没提供实际的ExporterListener。

```
public interface ExporterListener {

    void exported(Exporter<?> exporter) throws RpcException;
    
    void unexported(Exporter<?> exporter);
}
```

### 本地服务暴露总结

1、利用javassist将实际的服务执行类转变成基础的invoker。

2、接着对获取的invoker进行增强增加一些Filter，得到一个包装后的invoker。

3、然后构造InjvmProtocol，将invoker转变成InjvmExporter。


## 暴露远程服务


```
//1、ref转变成invoker
 Invoker<?> invoker = PROXY_FACTORY.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(EXPORT_KEY, url.toFullString()));
 DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

//2、invoker转变成export
 Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
```

### 具体服务转变成Invoker

获取invoker的过程和本地服务暴露一致。
只是对invoker多了一次包装，DelegateProviderMetaDataInvoker，然而实际该类无任何逻辑。

### Invoker转变成Exporter

invoker转化为export时，最终是使用了RegistryProtocol来进行export。


```
public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //获取注册中心的地址
        //zookeeper://127.0.0.1:2181/org.apache.dubbo.registry.RegistryService.....
        URL registryUrl = getRegistryUrl(originInvoker);
        // 获取provider的地址
        //dubbo://10.204.246.136:20880/org.apache.dubbo.demo.DemoService...
        URL providerUrl = getProviderUrl(originInvoker);
        
        //获取订阅的地址
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        
        providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
        //doLocalExport
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);

        // url to registry
        final Registry registry = getRegistry(originInvoker);
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
        //to judge if we need to delay publish
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            register(registryUrl, registeredProviderUrl);
            providerInvokerWrapper.setReg(true);
        }

        // Deprecated! Subscribe to override rules in 2.6.x or before.
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);

        exporter.setRegisterUrl(registeredProviderUrl);
        exporter.setSubscribeUrl(overrideSubscribeUrl);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<>(exporter);
    }
```

我们重点看下RegistryProtocol.export方法，这个方法做了很多事情，包括：

> 1. 调用 doLocalExport 启动netty
> 
> 2. 向注册中心注册服务
> 
> 3. 向注册中心进行订阅 override 数据
> 
> 4. 创建并返回 DestroyableExporter


#### doLocalExport(originInvoker, providerUrl)

最终是使用DubboProtocol进行export。在执行DubboProtocol.export之前，和本地服务在执行InjvmProtocol.export一样也会经过两个包装类的增强，这里我们直接看DubboProtocol的export逻辑

```
@Override
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service.
        //获取服务标识，理解成服务坐标也行。由服务组名，服务名，服务版本号以及端口组成
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        。。。
        //启动服务器
        openServer(url);
        //优化序列化
        optimizeSerialization(url);

        return exporter;
    }
```

doLocalExport启动netty服务端这里方法名称容易理解为做本地服务导出的逻辑，实际上不是，该方法主要的逻辑就是启动netty服务端。

也就是上一个过程看到的openServer(url);

调用链路如下：

```
DubboProtocol.openServer(URL url)
    ->DubboProtocol.createServer(URL url)
        ->Exchangers.bind(URL url, ExchangeHandler handler)
            ->Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))
                ->ExtensionLoader.getExtensionLoader(Transporter.class).getAdaptiveExtension().bind(url, handler)
                    ->NettyTransporter.bind(URL url, ChannelHandler listener)
                        ->new NettyServer(url, listener)
```

其中NettyTransporter.bind方法仅有一行代码为创建NettyServer。

我们接下来看创建NettyServer的过程


```
  public NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        //包装ChannelHandler，其中url封装了dubbo服务端处理的线程池
        ChannelHandler handler = ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME));
        localAddress = getUrl().toInetSocketAddress();
        //获取绑定的ip
        String bindIp = getUrl().getParameter(Constants.BIND_IP_KEY, getUrl().getHost());
        int bindPort = getUrl().getParameter(Constants.BIND_PORT_KEY, getUrl().getPort());
        if (url.getParameter(ANYHOST_KEY, false) || NetUtils.isInvalidLocalHost(bindIp)) {
            bindIp = ANYHOST_VALUE;
        }
        bindAddress = new InetSocketAddress(bindIp, bindPort);
        //最大可接受连接数
        this.accepts = url.getParameter(ACCEPTS_KEY, DEFAULT_ACCEPTS);
        this.idleTimeout = url.getParameter(IDLE_TIMEOUT_KEY, DEFAULT_IDLE_TIMEOUT);
        try {
            //调用模板方法 doOpen 启动服务器
            doOpen();
        } catch (Throwable t) {
        }
    }
```
我们看doOpen()，基本就是netty启动服务端的代码了，Dubbo2.7版本默认使用的是netty4，在doOpen()的时候会提供两个我们需要留意的处理器：


```
NettyServerHandler：处理客户端请求的Handler，provider接收Consumer端调用的入口。

IdleStateHandler：netty提供的心跳机制，检测远端是否存活，如果不存活或活跃则对空闲Socket连接进行处理避免资源的浪费。
```

```
 protected void doOpen() throws Throwable {
        bootstrap = new ServerBootstrap();
        //boss线程组
        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        //work线程组，线程数：cpu核心数+1
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        // FIXME: should we use getTimeout()?
                        int idleTimeout = UrlUtils.getIdleTimeout(getUrl());
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())
                                .addLast("encoder", adapter.getEncoder())
                                .addLast("server-idle-handler", new IdleStateHandler(0, 0, idleTimeout, MILLISECONDS))
                                .addLast("handler", nettyServerHandler);
                    }
                });
        // bind
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();

    }
```


#### 向注册中心注册服务


```
        //1.创建注册中心
        final Registry registry = getRegistry(originInvoker);
        final URL registeredProviderUrl = getRegisteredProviderUrl(providerUrl, registryUrl);
        ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                registryUrl, registeredProviderUrl);
        //to judge if we need to delay publish
        boolean register = providerUrl.getParameter(REGISTER_KEY, true);
        if (register) {
            //注册节点
            register(registryUrl, registeredProviderUrl);
            providerInvokerWrapper.setReg(true);
        }
```


##### 1.创建注册中心

```
RegistryProtocol.getRegistry(registryUrl, registeredProviderUrl);
-->ZookeeperRegistryFactory.createRegistry(URL url);
    ---->AbstractRegistry(URL url)
        ---->FailbackRegistry(URL url)
            ---->ZookeeperRegistry.(URL url, ZookeeperTransporter zookeeperTransporter)
```

- AbstractRegistry(URL url)中会初始化好注册中心本地文件的缓存。

- FailbackRegistry(URL url)会初始化好一个HashedWheelTimer定时线程。用于定时处理注册失败、订阅失败的请求。


```
retryTimer = new HashedWheelTimer(new NamedThreadFactory("DubboRegistryRetryTimer", true), retryPeriod, TimeUnit.MILLISECONDS, 128);
```

- ZookeeperRegistry(URL url, ZookeeperTransporter zookeeperTransporter)连接zk，同时加入重连的监听器

```
    zkClient = zookeeperTransporter.connect(url);
    zkClient.addStateListener(state -> {
        if (state == StateListener.RECONNECTED) {
            try {
                recover();
            } catch (Exception e) {
                logger.error(e.getMessage(), e);
            }
        }
    });
```



##### 2.注册节点

所谓的服务注册，本质上是将服务配置数据写入到 Zookeeper 的某个路径的节点下

```
RegistryProtocol.register(registryUrl, registeredProviderUrl);
---->FailbackRegistry.register(URL url)
    ---->ZookeeperRegistry.doRegister(URL url)
        ---->CuratorZookeeperClient.create(url)
```

```
   public void doRegister(URL url) {
        try {
            注册的节点：/dubbo/org.apache.dubbo.demo.DemoService/providers/dubbo%3A%2F%2F10.204.246.115%3A20880%2Forg.apache.dubbo.demo.DemoService%3Fanyhost%3Dtrue%26application。。。。。
            zkClient.create(toUrlPath(url), url.getParameter(DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```

这里有几个知识点说明下：

1、注册的节点路径为：/${group}/${serviceInterface}/providers/${url}

```
例如：
/dubbo/org.apache.dubbo.demo.DemoService/providers/dubbo://192.168.100.12:20880/org.apache.dubbo.demo.DemoService?anyhost=true&application=dubbo-demo-api-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=org.apache.dubbo.demo.DemoService&methods=sayHello&pid=60121&release=&side=provider&timestamp=1586141806238
```

2、注册的叶子节点（${url}）为临时节点

注册的节点如图：

![image](http://static.silence.work/20200406-02.png)



#### 向注册中心进行订阅

```
RegistryProtocol.subscribe(url, listener);
---->FailbackRegistry.subscribe(URL url, NotifyListener listener)
    ---->ZookeeperRegistry.doSubscribe(URL url,NotifyListener listener)
```

这里的订阅主要是订阅两个节点providers和configurators。

**1、创建configurators节点（订阅用）：**

/dubbo/org.apache.dubbo.demo.DemoService/configurators

**2、启动加入订阅：**

```
ChildListener = (parentPath, currentChilds) -> ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds))
List<String> children = zkClient.addChildListener(path, zkListener);

```

**3、收到订阅后的处理：**

```
FailbackRegistry.notify
    -->AbstractRegistry.notify
        -->AbstractRegistry.saveProperties(URL url)
            --> registryCacheExecutor.execute(new SaveProperties(version));
```


启动异步线程(DubboSaveRegistryCache-)更新注册中心信息到本地文件缓存。


```
    private class SaveProperties implements Runnable {
        private long version;

        private SaveProperties(long version) {
            this.version = version;
        }

        @Override
        public void run() {
            doSaveProperties(version);
        }
    }

```


本地文件缓存地址为：

```
//home目录+/.dubbo/dubbo-registry-+priver应用名称+注册中心地址和端口
//例如：/Users/silence/.dubbo/dubbo-registry-dubbo-demo-api-provider-127.0.0.1-2181.cache
System.getProperty("user.home") + "/.dubbo/dubbo-registry-" + url.getParameter(APPLICATION_KEY) + "-" + url.getAddress().replaceAll(":", "-") + ".cache"。
```
存储内容为：

```
org.apache.dubbo.demo.DemoService=empty\://192.168.100.12\:20880/org.apache.dubbo.demo.DemoService?anyhost\=true&application\=dubbo-demo-api-provider&bind.ip\=192.168.100.12&bind.port\=20880&category\=configurators&check\=false&deprecated\=false&dubbo\=2.0.2&dynamic\=true&generic\=false&interface\=org.apache.dubbo.demo.DemoService&methods\=sayHello&pid\=50839&release\=&side\=provider&timestamp\=1573271457782
```

# 总结

通过前面的分析，本质上服务暴露就是将本地接接口方法转成invoker然后封装成Exporter，在这个封装过程中会启动netty服务端监听请求，同时将服务地址注册到注册中心。在远程服务暴露完成后我们会得到DestroyableExporter，DestroyableExporter经过包装后的结构如下图：


![image](http://static.silence.work/20200406-03.png)


其中在ProtocolFilterWrapper.CallbackRegistrationInvoker中封装了一些Filter。