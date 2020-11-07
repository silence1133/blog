title: RocketMQ之NameServer管理路由分析
author: Silence
tags:
  - 消息中间件
  - RocketMQ
categories:
  - 消息中间件
date: 2019-12-30 12:48:00
---
# 启动过程

NameServer的本质其实就是用来管理topic的路由信息，
本文重点围绕路由信息是如何管理的来进行分析。
我们这里先简单分析下NameServer的启动过程

**入口：NamesrvStartup的main方法。**

## 初始化配置

主要就是填充两个配置类：NamesrvConfig、NettyServerConfig。

配置参数来源主要从指定的config文件或者--属性名 属性值获得。

**NamesrvConfig**：主要封装的是nameserver的一些业务参数；


```
public class NamesrvConfig {
    //rocketmq主目录
    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));
    //存储KV配置属性的持久化路径
    private String kvConfigPath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "kvConfig.json";
    //默认配置文件路径 -c指定
    private String configStorePath = System.getProperty("user.home") + File.separator + "namesrv" + File.separator + "namesrv.properties";
    private String productEnvName = "center";
    private boolean clusterTest = false;
    private boolean orderMessageEnable = false;
}
```


**NettyServerConfig**：主要是netty的网络参数


```
public class NettyServerConfig implements Cloneable {
    //namserver 监听端口
    private int listenPort = 8888;
    //netty work线程池个数
    private int serverWorkerThreads = 8;
    //
    private int serverCallbackExecutorThreads = 0;
    //IO线程池线程个数
    private int serverSelectorThreads = 3;
    //oneway方式发送消息请求并发度
    private int serverOnewaySemaphoreValue = 256;
    //异步方式发送消息最大并发度
    private int serverAsyncSemaphoreValue = 64;
    //网络链接最大空闲时间，默认120s。链接空闲时间超过该值，链接则断开
    private int serverChannelMaxIdleTimeSeconds = 120;
    //socket发送缓冲区大小
    private int serverSocketSndBufSize = NettySystemConfig.socketSndbufSize;
    //socket接受缓冲区大小
    private int serverSocketRcvBufSize = NettySystemConfig.socketRcvbufSize;
    private boolean serverPooledByteBufAllocatorEnable = true;
    private boolean useEpollNativeSelector = false;
}
```

## 构建并初始化NamesrvController

### 构建NameSrvController

```
    public NamesrvController(NamesrvConfig namesrvConfig, NettyServerConfig nettyServerConfig) {
        this.namesrvConfig = namesrvConfig;
        this.nettyServerConfig = nettyServerConfig;
        //kv存储操作类
        this.kvConfigManager = new KVConfigManager(this);
        //管理路由信息类
        this.routeInfoManager = new RouteInfoManager();
        this.brokerHousekeepingService = new BrokerHousekeepingService(this);
        this.configuration = new Configuration(
            log,
            this.namesrvConfig, this.nettyServerConfig
        );
        this.configuration.setStorePathFromConfig(this.namesrvConfig, "configStorePath");
    }
```

**RouteInfoManager**： 维护了topic、broker、cluster、filter这些东西的路由信息，该类是Nameserver管理路由的核心类，我们在下个部分管理路由部分重点分析。

### 初始化

初始化过程我们重点关注两点：

1、启动定时任务：每10s扫描一次不活跃的broker，并移除不活跃broker。

2、注册DefaultRequestProcessor。DefaultRequestProcessor是接收并处理不同请求的关键类，核心方法是processRequest。例如处理：GET_ROUTEINTO_BY_TOPIC、REGISTER_BROKER等一系列请求。


```
   public boolean initialize() {
        //加载kv配置
        this.kvConfigManager.load();
        //启动nettyserver
        this.remotingServer = new NettyRemotingServer(this.nettyServerConfig, this.brokerHousekeepingService);
        this.remotingExecutor =
            Executors.newFixedThreadPool(nettyServerConfig.getServerWorkerThreads(), new ThreadFactoryImpl("RemotingExecutorThread_"));
        //构建Process：DefaultRequestProcessor
        this.registerProcessor();
        //定时任务：扫码不活跃broker
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.routeInfoManager.scanNotActiveBroker();
            }
        }, 5, 10, TimeUnit.SECONDS);
        //定时任务：打印KV配置
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                NamesrvController.this.kvConfigManager.printAllPeriodically();
            }
        }, 1, 10, TimeUnit.MINUTES);
        ...
        ...
        return true;
    }
```

## 添加jvm关闭钩子以及启动netty服务

添加jvm关闭的钩子，当jvm关闭的时候能够关闭NamesrvController，做一些资源的销毁工作。

---


# 路由管理

Nameserver的核心就是为producer和consumer维护topic的路由信息，只有搞明白了路由管理，也就读懂了Nameserver。

而路由管理本质其实就是维护路由元信息，而这个路由元信息都存放在RouteInfoManager中，我们先看下RouteInfoManager中的几个缓存map，路由的管理本质就是在管理这几个缓存map。

```
public class RouteInfoManager {
    private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    //每个topic对应的队列信息
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    //每个broker对应的broker信息
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    //集群信息
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    //broker的状态信息
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    //broker的过滤列表
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
}
```

**clusterAddrTable**：
维护了集群对应存在哪些broker，例如：

```
//集群名称为DefaultCluster的集群存在brokerName为Silence.local的节点broker
{DefaultCluster=[Silence.local]}
```
**brokerAddrTable**：存储每个broker对应的broker信息，包括broker名称和broker地址。

```
{Silence.local=BrokerData [brokerName=Silence.local, brokerAddrs={0=192.168.100.12:10911}]}
```

**topicQueueTable**：
存储了每个topic对应的队列信息。

```
{Silence.local=[QueueData [brokerName=Silence.local, readQueueNums=1, writeQueueNums=1, perm=7, topicSynFlag=0]],
SELF_TEST_TOPIC=[QueueData [brokerName=Silence.local, readQueueNums=1, writeQueueNums=1, perm=6, topicSynFlag=0]],
TBW102=[QueueData [brokerName=Silence.local, readQueueNums=8, writeQueueNums=8, perm=7, topicSynFlag=0]], 
BenchmarkTest=[QueueData [brokerName=Silence.local, readQueueNums=1024, writeQueueNums=1024, perm=6, topicSynFlag=0]],
DefaultCluster=[QueueData [brokerName=Silence.local, readQueueNums=16, writeQueueNums=16, perm=7, topicSynFlag=0]],
OFFSET_MOVED_EVENT=[QueueData [brokerName=Silence.local, readQueueNums=1, writeQueueNums=1, perm=6, topicSynFlag=0]]}
```

**brokerLiveTable**：维护每个broker对应用于检测是否存活的信息，很关键的一个信息就是最近接收心跳包的时间戳，该时间戳就是用来判定broker是否存活的关键。

```
{192.168.100.12:10911=BrokerLiveInfo [lastUpdateTimestamp=1577630583522, dataVersion=DataVersion[timestamp=1577628891855, counter=0], channel=[id: 0x16f5c0ca, L:/127.0.0.1:9876 ! R:/127.0.0.1:49603], haServerAddr=192.168.100.12:10912]}
```



## 路由注册

RocketMQ的路由注册的触发是在broker启动时通过发送心跳包给NameServer。

同时broker有一个定时器，每隔30s向NameServer发送心跳包，Nameserver收到broker的心跳包后会更新brokerLiveTable缓存的lastUpdateTimestamp时间戳，同时Nameserver每10s会扫描brokerLiveTable的lastUpdateTimestamp时间戳，如果时间间隔已经有120s没有更新了，Nameserver会将broker视为故障broker，从而移除该broker的路由信息。

broker发送心跳包的定时任务代码是在BrokerController的start方法，这里不做详细介绍，我们重点看下Nameserver接收心跳包的逻辑。

```
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
                } catch (Throwable e) {
                    log.error("registerBrokerAll Exception", e);
                }
            }
    }, 1000 * 10, Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), TimeUnit.MILLISECONDS);

```

### Nameserver处理心跳包

**入口**：DefaultRequestProcessor.processRequest处理 RequestCode.REGISTER_BROKER的请求类型，请求的处理最终转发给了RouteInfoManager的registerBroker方法。

心跳包传过来的参数有
clusterName、brokerAddr、brokerName、brokerId、filterServerList、topicConfigSerializeWrapper。

**topicConfigSerializeWrapper**：封装了broker存储的所有topic信息。（也就是说每次心跳都会把所有topic信息发送过来）


**Nameserver处理心跳包做的主要事情就是更新前面讲的RouteInfoManager中的几个重要缓存map**。

**1、更新clusterAddrTable**

```
    Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
    if (null == brokerNames) {
            brokerNames = new HashSet<String>();
            this.clusterAddrTable.put(clusterName, brokerNames);
    }
    brokerNames.add(brokerName);
```


**2、更新brokerAddrTable**

```
    boolean registerFirst = false;
    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
    if (null == brokerData) {
        registerFirst = true;
        brokerData = new BrokerData(clusterName, brokerName, new HashMap<Long, String>());
        this.brokerAddrTable.put(brokerName, brokerData);
    }
    String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
    //标示当前心跳包的broker是否是第一次注册，后面一步会用到
    registerFirst = registerFirst || (null == oldAddr);
```


**3、更新topicQueueTable**

只要当心跳包包含了topic信息，并且当前broker是master，才会更新topicQueueTable。

```
    if (null != topicConfigWrapper
        && MixAll.MASTER_ID == brokerId) {
            if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
            || registerFirst) {
                ConcurrentMap<String, TopicConfig> tcTable =topicConfigWrapper.getTopicConfigTable();
                if (tcTable != null) {
                    for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                                this.createAndUpdateQueueData(brokerName, entry.getValue());
                    }
                }
            }
        }
```


**4、更新brokerLiveTable**

每一次心跳都会重新替换broker对应的BrokerLiveInfo信息，这个类记录了当前时间戳，也就是前面讲的会根据这个时间来判断broker是否已经断开了，下个部分我们重点分析是如何判断broker已经断开了。

```
    BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                    new BrokerLiveInfo(
                        System.currentTimeMillis(),
                        topicConfigWrapper.getDataVersion(),
                        channel,
                        haServerAddr));
```

**5、更新filterServerTable**

注册broker的过滤器列表

```
    if (filterServerList != null) {
            if (filterServerList.isEmpty()) {
               this.filterServerTable.remove(brokerAddr);
            } else {
               this.filterServerTable.put(brokerAddr, filterServerList);
            }
        }
```

**注意：这里有一点需要说明下，由于多个broker会同时发送心跳包到Nameserver，意味着会有并发写的问题，因此源码中用了写锁，只有获取到写锁才能进入注册的逻辑，否则等待，因此保证了注册路由信息的串行执行。**


## 路由删除

此部分我们主要分析，Nameserver是如何判断宕机的broker，并将对应的路由信息删除的。

路由信息的删除主要有两个触发点：

**1、nameserver定时扫描brokerLiveTable，检测每个broker对应最近一次更新的时间戳，如果相距当前时间超过120s，则移除roker。**

入口：RouteInfoManager.scanNotActiveBroker

这个其实就是前面nameserver启动过程中提到的每10s执行一次的定时器。

onChannelDestroy是最终执行路由删除的逻辑

```
    public void scanNotActiveBroker() {
        Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
        while (it.hasNext()) {
            Entry<String, BrokerLiveInfo> next = it.next();
            long last = next.getValue().getLastUpdateTimestamp();
            //BROKER_CHANNEL_EXPIRED_TIME=1000 * 60 * 2
            if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
                RemotingUtil.closeChannel(next.getValue().getChannel());
                it.remove();
                log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
                //最终执行路由删除的逻辑
                this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
            }
        }
    }
```

**2、broker正常关闭的时候，也会通知nameserver移除该broker。**

最终也是执行onChannelDestroy进行路由的删除，因此我们接下来重点分析onChannelDestroy方法。

### onChannelDestroy

方法代码虽然看起来很长，但逻辑其实很简单，这里不贴代码。

1、从brokerLiveTable中读取要删除的brokerAddr。

这里获取的读锁，主要防止还有写锁在执行。

2、根据要移除的broker，同样是操作RouteInfoManager中的几个重要缓存map，只不过是执行删除策略。

这里同样也会获取相应的写锁。防止并发操作。

## 路由查找

也就是客户端获取对应topic路由信息的过程。客户端会每30秒定时拉取一次路由信息。

路由查找的入口是：RouteInfoManage.pickupTopicRouteData(String topic);

最终通过TopicRouteData封装路由信息返回给客户端，实际整个过程就是通过RouteInfoManage缓存的几个map封装TopicRouteData的过程，我们这里看下TopicRouteData的结构：


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