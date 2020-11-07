title: Dubbo源码分析之一：前置准备
author: Silence
tags:
  - Dubbo
categories:
  - Dubbo
date: 2020-04-05 21:08:00
---
# 学习思路

从github导出[官方源码](https://github.com/apache/dubbo.git)，同时通过官方提供的[demo](https://github.com/apache/dubbo/tree/master/dubbo-demo/dubbo-demo-api)进行分析，如果对部分细节想深入理解某功能或者某个类，可以找项目内的单元测试进行调试学习。

另外，dubbo源码体系本身很大，为了学习DUBBO远程调用的链路过程，我将其分成如下几个部分进行学习：

**Provider端：**

> 1、本地服务是如何暴露到远程的；
> 
> 2、服务端如何处理相应远程客户端的请求；

**Consumer端：**

> 1、客户端如何获取远程服务的代理类；
> 
> 2、通过代理类执行远程调用的过程；

**说明：这里我分析的dubbo版本选用的是2.7版本。**

# 前置知识准备

## SPI
1、Dubbo的SPI能够解决动态指定不同的key来加载不同的具体实现类。


2、Dubbo的SPI是通过读取配置文件META-INF/dubbo/internal下不同接口配置的多种实现，然后在运行过程中通过指定具体的实现key，来获取具体实现类的实例。

类似于Spring管理bean，通过xml指定bean的名称和实现类，然后利用beanFactory.getBean("beanTest")来获取bean的实例。


**使用方法**：

```
//获取xxx对应的xxxInterface接口实现类示例
ExtensionLoader.getExtensionLoader(xxxInterface.class).getExtension("xxx");
```
想了解具体实现过程可以看单元测试类ExtensionLoaderTest，进行调试学习。

**例如：**

在META-INF/dubbo/internal存在配置文件org.apache.dubbo.common.extension.ext6_wrap.WrappedExt。配置内容为：

```
impl1=org.apache.dubbo.common.extension.ext6_wrap.impl.Ext5Impl1
impl2=org.apache.dubbo.common.extension.ext6_wrap.impl.Ext5Impl2
wrapper1=org.apache.dubbo.common.extension.ext6_wrap.impl.Ext5Wrapper1
wrapper2=org.apache.dubbo.common.extension.ext6_wrap.impl.Ext5Wrapper2
```
通过以下代码即可动态获取具体要获取的实现类的实例对象

```
WrappedExt impl1 = ExtensionLoader.getExtensionLoader(WrappedExt.class).getExtension("impl1");

```

### SPI中的AOP实现

在根据SPI获取指定实现类的时候，如果某接口存在包装类特征的实现类，会将获取的实现类进行包装，从而获取到包装后的实现类，相当于对要获取的原实现类进行了增强。

正如上面WrappedExt接口的例子，在获取impl1实现类的时候，实际上获取到的是经过了Ext5Wrapper1和Ext5Wrapper2包装后的实例。如下面获取的实例：

![image](http://static.silence.work/20200405-01.png)


想了解具体实现过程可以看单元测试类ExtensionLoaderTest.test_getExtension_WithWrapper，进行调试学习。


### spi中的自适应拓展机制

为了解决有些拓展并不想在框架启动阶段被加载，而是希望在拓展方法被调用时，根据运行时传递的参数来选择具体实现类进行加载。这个自适应拓展机制是用@Adaptive来实现的。

例如我们前面讲了Dubbo的SPI是指定key即可获取到对应的实现类实例，而自适应机制是会现获取一个接口的动态代理类，这个动态代理类在程序运营的时候根据传递的参数再来决定获取具体的实现类实例。

**@Adaptive注解：存在两种使用情况**

**1、注解在实现类上：**
Dubbo 不会为该类生成代理类，直接获取有@Adaptive注解注解的实现类。Adaptive 注解在类上的情况很少，在 Dubbo 中，仅有两个类被 Adaptive 注解了，分别是 AdaptiveCompiler 和 AdaptiveExtensionFactory

**2、注解在接口方法上（Adaptive的核心）：**
表示拓展的加载逻辑需由框架自动生成。会动态生成实现类。
Dubbo 会为拓展接口生成具有代理功能的代码。然后通过 javassist 或 jdk 编译这段代码，得到 Class 类，最后再通过反射创建代理类。

**使用方法：**

```
ExtensionLoader.getExtensionLoader(xxxInterface.class).getAdaptiveExtension();
```

想深入理解Dubbo的自适应拓展机制，可以看单元测试
ExtensionLoader_Adaptive_Test。

```
//获取的是一个动态编译类SimpleExt$Adaptive
SimpleExt ext = ExtensionLoader.getExtensionLoader(SimpleExt.class).getAdaptiveExtension();

Map<String, String> map = new HashMap<String, String>();
//指定要使用的子类key
map.put("simple.ext", "impl2");
URL url = new URL("p1", "1.2.3.4", 1010, "path1", map);

//在调用的时候根据url传递的参数决定使用SimpleExt的哪个实现类的echo方法
String echo = ext.echo(url, "haha");

```
动态生成的SimpleExt$Adaptive类代码如下

```
public class SimpleExt$Adaptive implements org.apache.dubbo.common.extension.ext1.SimpleExt {
    public java.lang.String bang(org.apache.dubbo.common.URL arg0, int arg1) {
        throw new UnsupportedOperationException("The method public abstract java.lang.String org.apache.dubbo.common.extension.ext1.SimpleExt.bang(org.apache.dubbo.common.URL,int) of interface org.apache.dubbo.common.extension.ext1.SimpleExt is not adaptive method!");
    }

    public java.lang.String yell(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("key1", url.getParameter("key2", "impl1"));
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.common.extension.ext1.SimpleExt) name from url (" + url.toString() + ") use keys([key1, key2])");
        org.apache.dubbo.common.extension.ext1.SimpleExt extension = (org.apache.dubbo.common.extension.ext1.SimpleExt) ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.extension.ext1.SimpleExt.class).getExtension(extName);
        return extension.yell(arg0, arg1);
    }

    public java.lang.String echo(org.apache.dubbo.common.URL arg0, java.lang.String arg1) {
        if (arg0 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg0;
        String extName = url.getParameter("simple.ext", "impl1");
        if (extName == null)
            throw new IllegalStateException("Failed to get extension (org.apache.dubbo.common.extension.ext1.SimpleExt) name from url (" + url.toString() + ") use keys([simple.ext])");
        org.apache.dubbo.common.extension.ext1.SimpleExt extension = (org.apache.dubbo.common.extension.ext1.SimpleExt) ExtensionLoader.getExtensionLoader(org.apache.dubbo.common.extension.ext1.SimpleExt.class).getExtension(extName);
        return extension.echo(arg0, arg1);
    }
}
```

### 扩展点自动激活

为了解决同时加载多个实现的场景。通过@Activate注解在实现类上，指定value或者group值来获取对应实现类的实例集合。

例如在服务引用的代码中，会获取对应的多个Filter实例，会根据过@Activate指定了group是consumer进行获取。

```
List<ActivateExt1> list = ExtensionLoader.getExtensionLoader(ActivateExt1.class).getActivateExtension(url, new String[]{}, "default_group");
```


## 配置类说明

下图为每个配置模块之间的关系，分为服务提供侧、应用测、消费者侧。
![image](http://static.silence.work/20200405-02.png)

### 应用配置相关
ApplicationConfig：用于配置当前应用信息，不管该应用是提供者还是消费者

RegistryConfig：用于配置连接注册中心相关信息

MonitorConfig：用于配置连接监控中心相关信息

### 服务提供相关
ProviderConfig：服务提供的缺省配置，也就是单个ServiceConfig的缺省配置。

ServiceConfig：暴露单个服务的配置。

ProtocolConfig：用于配置提供服务的协议信息，协议由提供方指定，消费方被动接受。

### 服务消费相关
ConsumerConfig：服务消费的缺省配置，也就是单个ServiceConfig的缺省配置。
ReferenceConfig：单个服务引用的配置。

### 子信息配置

MethodConfig：用于 ServiceConfig 和 ReferenceConfig 指定方法级的配置信息。

ArgumentConfig：用于指定方法参数的配置。

![image](http://static.silence.work/20200405-03.png)

## 关键接口职责说明

### ExtensionLoader

拓展加载器，是Dubbo 实现spi的核心类。



### Proxy

封装了所有接口的透明化代理，远程调用的接口，最终都会在consumer端转变为代理类来进行远程调用。

### Invoker

它代表一个远程服务调用的可执行体，可向它发起 invoke 调用，它有可能是一个本地的实现，也可能是一个远程的实现，也可能一个集群实现。

在获取服务代理类的过程中，会将要调用的远程服务包装成Invoker，然后将Invoker封装成Proxy。我们在分析服务调用过程会对Invoker的认识会比较深刻。

```
public interface Invoker<T> extends Node {

    /**
     * get service interface.
     *
     * @return service interface.
     */
    Class<T> getInterface();

    /**
     * invoke.
     *
     * @param invocation
     * @return result
     * @throws RpcException
     */
    Result invoke(Invocation invocation) throws RpcException;

}
```

### Invocation 

会话域，它持有调用过程中的变量，比如方法名，参数等，在invoker链中进行传递。



### ProxyFactory

根据invoke创建代理类的工厂类。

```
@SPI("javassist")
public interface ProxyFactory {

    /**
     * create proxy.
     *
     * @param invoker
     * @return proxy
     */
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker) throws RpcException;

    /**
     * create proxy.
     *
     * @param invoker
     * @return proxy
     */
    @Adaptive({PROXY_KEY})
    <T> T getProxy(Invoker<T> invoker, boolean generic) throws RpcException;

    /**
     * create invoker.
     *
     * @param <T>
     * @param proxy
     * @param type
     * @param url
     * @return invoker
     */
    @Adaptive({PROXY_KEY})
    <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) throws RpcException;

}
```


### Protocol

是Invoker暴露和引用的主功能入口，它负责 Invoker 的生命周期管理。

```
@SPI("dubbo")
public interface Protocol {

    int getDefaultPort();

    @Adaptive
    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    @Adaptive
    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();

}
```

### Directory

服务目录，保存invoker，可以简单理解为invoker的集合。例如provider是四个节点的集群，则从directory中会返回长度为4的invoker集合。

服务目录中存储了一些和服务提供者有关的信息，通过服务目录，服务消费者可获取到服务提供者的信息，比如 ip、端口、服务协议等。通过这些信息，服务消费者就可通过 Netty 等客户端进行远程调用。

```
public interface Directory<T> extends Node {

    /**
     * get service type.
     *
     * @return service type.
     */
    Class<T> getInterface();

    /**
     * list invokers.
     *
     * @return invokers
     */
    List<Invoker<T>> list(Invocation invocation) throws RpcException;

}
```

### Cluster

 前面说了Directory是代表invoker的一个集合，则Cluster是将多个Invoker封装成一个Invoker。
 
 
```
public interface Cluster {

    @Adaptive
    <T> Invoker<T> join(Directory<T> directory) throws RpcException;
}
```


### Filter

dubbo提供的过滤器，filter能在invoker执行链中进行拦截，方便在调用过程中拓展逻辑。

我们看Filter内部还定义了一个子接口Listener，用于在Filter拦截过程中做回调通知。

dubbo本身提供了很多Filter，我们在分析invoker执行链路过程中会对Filter认识更深刻

```
public interface Filter {
    /**
     * Does not need to override/implement this method.
     */
    Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException;

    /**
     * Filter itself should only be response for passing invocation, all callbacks has been placed into {@link Listener}
     *
     * @param appResponse
     * @param invoker
     * @param invocation
     * @return
     */
    @Deprecated
    default Result onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
        return appResponse;
    }

    interface Listener {

        void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation);

        void onError(Throwable t, Invoker<?> invoker, Invocation invocation);
    }

}
```

### URL

URL 是 Dubbo 配置的载体，通过 URL 可让 Dubbo 的各种配置在各个模块之间传递。
![image](http://static.silence.work/20200405-04.png)

