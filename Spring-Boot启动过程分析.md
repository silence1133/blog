title: Spring Boot启动过程分析
author: Silence
tags:
  - Spring-Boot
  - ''
  - ''
categories:
  - Spring-Boot
date: 2020-10-07 14:07:00
---


# 构造SpringApplication


```
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		//main方法的类
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		//此参数决定了后续构造哪个ApplicationContext
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
		//实例化好classpath下所有的ApplicationContextInitializer类
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
	    //实例化好classpath下所有的ApplicationListener类
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

## 1. webApplicationType设置

由classpath下是否存在以下两个类来决定是否用web环境的Applicaiton

```
	private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
			"org.springframework.web.context.ConfigurableWebApplicationContext" };
```


## 2. 实例所有的ApplicationContextInitializer

这里加载的是classpath下META-INF/spring.factories下配置的所有ApplicationContextInitializer，加载到后，利用反射进行实例化。


```

private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// Use names and ensure unique to protect against duplicates
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		//从spring.factories中加载到所有的class之后，通过反射实例化所有的ApplicationContextInitializer
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
}

```


这里重点看下加载下META-INF/spring.factories的过程：

- 加载入口：SpringFactoriesLoader#loadSpringFactories


```
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
       //有缓存直接命中
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}
         //第一次没命中缓存，则往下执行
		try {
		    //取到所有jar下的META-INF/spring.factories文件路径
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			//遍历处理
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
				    //key
					String factoryTypeName = ((String) entry.getKey()).trim();
					for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryTypeName, factoryImplementationName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
```

加载所有jar下的META-INF/spring.factories文件，将接口路径作为key，value是一个List，存放同一个接口的不同实现类的路径放到map对象中，最后封装成一个cache大的map缓存起来(key为类加载对象)。
![image](http://static.silence.work/spring-boot-1.png)

## 3. 实例化所有的ApplicationListener

上一步已经缓存了，直接从缓存中获取ApplicationListener的类并实例化。

关于ApplicationListener何时会触发，在SpringApplicationRunListener部分会分析到：<a href="#123" target="_self">实例化SpringApplicationRunListener并触发starting方法</a>



# 进入run方法

## 主干逻辑

```
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		//实例化好classpath下的SpringApplicationRunListener，封装到SpringApplicationRunListeners
		SpringApplicationRunListeners listeners = getRunListeners(args);
		//调用starting
		listeners.starting();
		try {
			ApplicationArguments 
			//准备环境变量对象，同时这里会触发listeners的environmentPrepared
			applicationArguments = new DefaultApplicationArguments(args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
		    //异常处理
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```


<span id = "123">

### 实例化SpringApplicationRunListener并触发starting方法

</span>

![image](http://static.silence.work/spring-boot-2.png)

实例化SpringApplicationRunListener同前面实例化ApplicationContextInitializer一样，这里只是多了一步封装将所有的对象封装成一个SpringApplicationRunListeners，然后触发所有的starting方法。

**特别说明：** sprint boot本身提供了一个EventPublishingRunListener，实现了SpringApplicationRunListener的每一个方法，通过spring的event机制，发送相应的Event。

这里会构造ApplicationStartingEvent对象，来通知监听了ApplicationStartingEvent对应的ApplicationListener。


```
public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    //...
    
	@Override
	public void starting() {
		this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
	}
	//...
}

//事件通知逻辑
	public void multicastEvent(ApplicationEvent event) {
		multicastEvent(event, resolveDefaultEventType(event));
	}

	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```




### 准备环境变量对象以及配置

- 入口：SpringApplication#prepareEnvironment

```
	private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		// 根据webApplicationType选择指定的环境对象
		ConfigurableEnvironment environment = getOrCreateEnvironment();
		//将main 的args参数设置到环境变量中，同时读取spring.profiles.active指定的环境profile，设置到环境变量对象中
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		//添加一个ConfigurationPropertySourcesPropertySource到propertySources中，value是自己
		ConfigurationPropertySources.attach(environment);
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```

环境对象enviroment结构如下：

![image](http://static.silence.work/spring-boot-3.png)


**1. web环境下用StandardServletEnvironment，非web环境StandardEnvironment。**


```
	private ConfigurableEnvironment getOrCreateEnvironment() {
		if (this.environment != null) {
			return this.environment;
		}
		switch (this.webApplicationType) {
		case SERVLET:
			return new StandardServletEnvironment();
		case REACTIVE:
			return new StandardReactiveWebEnvironment();
		default:
			return new StandardEnvironment();
		}
	}
```


**2. 将main 的args参数设置到环境变量的propertySources中，同时读取spring.profiles.active指定的环境profile，设置到环境变量对象的activeProfiles变量中。**


```
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
		if (this.addConversionService) {
			ConversionService conversionService = ApplicationConversionService.getSharedInstance();
			environment.setConversionService((ConfigurableConversionService) conversionService);
		}
		configurePropertySources(environment, args);
		configureProfiles(environment, args);
}
```



**3. 添加一个ConfigurationPropertySourcesPropertySource到propertySources中，value是自己。**


```
	public static void attach(Environment environment) {
		Assert.isInstanceOf(ConfigurableEnvironment.class, environment);
		MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
		PropertySource<?> attached = sources.get(ATTACHED_PROPERTY_SOURCE_NAME);
		if (attached != null && attached.getSource() != sources) {
			sources.remove(ATTACHED_PROPERTY_SOURCE_NAME);
			attached = null;
		}
		if (attached == null) {
		    //添加逻辑
			sources.addFirst(new ConfigurationPropertySourcesPropertySource(ATTACHED_PROPERTY_SOURCE_NAME,
					new SpringConfigurationPropertySources(sources)));
		}
	}
```



**4. 触发所有SpringApplicationRunListener的environmentPrepared，传递environment。**
 
发送ApplicationEnvironmentPreparedEvent事件

```
SpringApplicationRunListeners#environmentPrepared：

    void environmentPrepared(ConfigurableEnvironment environment) {
		for (SpringApplicationRunListener listener : this.listeners) {
			listener.environmentPrepared(environment);
		}
	}
	
EventPublishingRunListener#environmentPrepared

    public void environmentPrepared(ConfigurableEnvironment environment) {
		this.initialMulticaster
				.multicastEvent(new ApplicationEnvironmentPreparedEvent(this.application, this.args, environment));
	}
```

<font color='red'>**问题点：application.properties文件在哪里读取的。**</font>

通过事件驱动发布ApplicationEnvironmentPreparedEvent事件，ConfigFileApplicationListener监听了该事件，从而触发该类的onApplicationEvent方法。

设置入口：ConfigFileApplicationListener#onApplicationEvent。

最终执行逻辑在：ConfigFileApplicationListener.Loader#load()


### createApplicationContext

构造ApplicationContext，根据webApplicationType构造具体的ApplicationContext对象。

- AnnotationConfigApplicationContext：默认
- AnnotationConfigServletWebServerApplicationContext：web环境下
- AnnotationConfigReactiveWebServerApplicationContext：？


```
	protected ConfigurableApplicationContext createApplicationContext() {
		Class<?> contextClass = this.applicationContextClass;
		if (contextClass == null) {
			try {
				switch (this.webApplicationType) {
				case SERVLET:
					contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
					break;
				case REACTIVE:
					contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
					break;
				default:
					contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
				}
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
			}
		}
		//构造
		return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```

构造过程就是通过反射调用具体ApplicationContext的构造函数

AnnotationConfigServletWebServerApplicationContext：

```
	public AnnotationConfigServletWebServerApplicationContext() {
	    //注解的Bean读取器
		this.reader = new AnnotatedBeanDefinitionReader(this);
		//扫描指定类路径中注解Bean定义的扫描器
		this.scanner = new ClassPathBeanDefinitionScanner(this);
	}
```
[AnnotatedBeanDefinitionReader与ClassPathBeanDefinitionScanner](https://cloud.tencent.com/developer/article/1497799)

### 实例化SpringBootExceptionReporter

异常报告器。

SpringBootExceptionReporter的实现类为FailureAnalyzers，FailureAnalyzers封装了所有的FailureAnalyzer的实例。此过程相当于是实例化所有的FailureAnalyzer。

<font color='red'>问题点，什么时候执行？</font>

在try catch中执行，例如run方法中出现异常时。

```
exceptionReporters = {ArrayList@4787}  size = 1
 0 = {FailureAnalyzers@4793} 
  classLoader = {Launcher$AppClassLoader@1724} 
  analyzers = {ArrayList@4794}  size = 20
   0 = {ValidationFailureAnalyzer@4796} 
   1 = {BeanCurrentlyInCreationFailureAnalyzer@4797} 
   2 = {BeanDefinitionOverrideFailureAnalyzer@4798} 
   3 = {BeanNotOfRequiredTypeFailureAnalyzer@4799} 
   4 = {BindFailureAnalyzer@4800} 
   5 = {BindValidationFailureAnalyzer@4801} 
   6 = {UnboundConfigurationPropertyFailureAnalyzer@4802} 
   7 = {ConnectorStartFailureAnalyzer@4803} 
   8 = {NoSuchMethodFailureAnalyzer@4804} 
   9 = {NoUniqueBeanDefinitionFailureAnalyzer@4805} 
   10 = {PortInUseFailureAnalyzer@4806} 
   11 = {ValidationExceptionFailureAnalyzer@4807} 
   12 = {InvalidConfigurationPropertyNameFailureAnalyzer@4808} 
   13 = {InvalidConfigurationPropertyValueFailureAnalyzer@4809} 
   14 = {NoSuchBeanDefinitionFailureAnalyzer@4810} 
   15 = {FlywayMigrationScriptMissingFailureAnalyzer@4811} 
   16 = {DataSourceBeanCreationFailureAnalyzer@4812} 
   17 = {HikariDriverConfigurationFailureAnalyzer@4813} 
   18 = {ConnectionFactoryBeanCreationFailureAnalyzer@4814} 
   19 = {NonUniqueSessionRepositoryFailureAnalyzer@4815} 
```


### prepareContext

前面构建好ApplicationContext之后，这步主要是对applicationContext设置一些参数

```
rivate void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
		//1. 设置环境变量
		context.setEnvironment(environment);
		postProcessApplicationContext(context);
		//2. 执行所有的ApplicationContextInitializer.initialize方法
		applyInitializers(context);
		//3. 执行SpringApplicationRunListener的contextPrepared
		listeners.contextPrepared(context);
		//4. 输出profile日志
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		if (beanFactory instanceof DefaultListableBeanFactory) {
			((DefaultListableBeanFactory) beanFactory)
					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.lazyInitialization) {
			context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
		}
		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		//5. 加载bean到applicationContext中
		load(context, sources.toArray(new Object[0]));
		//6. 执行SpringApplicationRunListener的contextLoaded
		listeners.contextLoaded(context);
	}
```


#### 1. 设置环境配置对象

		
```
context.setEnvironment(environment);
```


#### 2. 执行ApplicationContextInitializer的initialize

**拓展点执行**，这里执行的所有ApplicationContextInitializer都会有一些重要的逻辑，主要是添加了很多BeanFactoryProcessor。


```
public interface ApplicationContextInitializer<C extends ConfigurableApplicationContext> {

	/**
	 * Initialize the given application context.
	 * @param applicationContext the application to configure
	 */
	void initialize(C applicationContext);

}
```



#### 3. 执行SpringApplicationRunListener的contextPrepared

发送ApplicationContextInitializedEvent事件


#### 4. 输出选择的profile日志


```
 The following profiles are active: dev
```

#### 5. 加载source bean到applicationContext中

也就是main的类

#### 6. 执行SpringApplicationRunListener的contextLoaded

此过程做了两件事件：

- 遍历application中的ApplicationListener，针对继承了ApplicationContextAware的ApplicationListener实现类，调用setApplicationContext方法。


- 发送ApplicationPreparedEvent事件。



```
	public void contextLoaded(ConfigurableApplicationContext context) {
		for (ApplicationListener<?> listener : this.application.getListeners()) {
			if (listener instanceof ApplicationContextAware) {
				((ApplicationContextAware) listener).setApplicationContext(context);
			}
			context.addApplicationListener(listener);
		}
		this.initialMulticaster.multicastEvent(new ApplicationPreparedEvent(this.application, this.args, context));
	}
```

### refreshContext

- 入口：SpringApplication#refreshContext

主干逻辑在：AbstractApplicationContext#refresh


**此过程其实就是执行ApplicationContext的refresh方法，属于spring ApplicationContext的内容，这里只做简单的分析。**


```
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```


#### 1. prepareRefresh()

准备刷新的上下文环境。

*无重要逻辑，不分析。*

#### 2. obtainFreshBeanFactory

初始化BeanFactory。

*由于AnnotationConfigServletWebServerApplicationContext继承的GenericApplicationContext，GenericApplicationContext的refreshBeanFactory无重要逻辑（也不会进行bean的读取），不分析。*

#### 3. prepareBeanFactory

主要是beanFactory的一些设置，填充拓展功能，例如添加BeanPostProcessor、注册bean等。

```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		beanFactory.setBeanClassLoader(getClassLoader());
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```

- **设置beanClassLoader**


- **设置BeanExpressionResolver**：表达式处理器


- **设置ResourceEditorRegistrar**


- **添加ApplicationContextAwareProcessor**

是一个BeanPostProcessor，在postProcessBeforeInitialization触发相关Aware的set方法

```
org.springframework.context.EnvironmentAware
org.springframework.context.EmbeddedValueResolverAware
org.springframework.context.ResourceLoaderAware
org.springframework.context.ApplicationEventPublisherAware
org.springframework.context.MessageSourceAware
org.springframework.context.ApplicationContextAware
```

- **设置忽略autowiring的bean**


```
	    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```


- **registerResolvableDependency:给特定的依赖类型注册自动装配的值**


```
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```


- **设置ApplicationListenerDetector**

也是一个BeanPostProcessor，这个processor的作用是在postProcessAfterInitialization方法中将继承了ApplicationListener的bean添加到applicationContext中。

- **注册指定的bean**

environment、systemProperties、systemEnvironment

#### 4. postProcessBeanFactory

继续上一步对BeanFactory的功能填充。

1. 添加了一个BeanPostProcessor：WebApplicationContextServletContextAwareProcessor。
此类作用是为了bean初始化前调用ServletContextAware、ServletConfigAware的set方法。


2. 在beanFactory注册相关WebApplicationScopes("request", "session", "globalSession")

```
	protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		beanFactory.addBeanPostProcessor(new WebApplicationContextServletContextAwareProcessor(this));
		beanFactory.ignoreDependencyInterface(ServletContextAware.class);
		registerWebApplicationScopes();
	}
```


#### 5. invokeBeanFactoryPostProcessors

执行所有手动注册的BeanFactoryPostProcessor的postProcessBeanFactory方法。如果实现了BeanDefinitionRegistryPostProcessor会先执行postProcessBeanDefinitionRegistry方法。

核心逻辑入口：

```
PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
```

这里会执行一个非常关键的BeanFactoryPostProcessor：
ConfigurationClassPostProcessor，该Processor会批量扫描所有注解标注的bean并注册到容器中。


**问题：这些BeanFactoryPostProcessor在哪里添加的？**

在前面的执行ApplicationContextInitializer的initialize过程。


#### 6. registerBeanPostProcessors

加载所有的BeanPostProcessor，并按照顺序注册到BeanFactory中，保存到AbstractBeanFactory#beanPostProcessors这个集合变量中。

关键逻辑入口：PostProcessorRegistrationDelegate#registerBeanPostProcessors()

#### 7. initMessageSource
初始化消息资源。

*无重要逻辑，忽略*

#### 8. initApplicationEventMulticaster

注册ApplicationEventMulticaster的bean到beanFactory，如果没有自定义ApplicationEventMulticaster，则使用SimpleApplicationEventMulticaster。

#### 9. onRefresh

对于ServletWebServerApplicationContext多了主要逻辑是createWebServer。主要是初始化好web服务器，并启动。


```
ServletWebServerApplicationContext#createWebServer

private void createWebServer() {
		WebServer webServer = this.webServer;
		ServletContext servletContext = getServletContext();
		if (webServer == null && servletContext == null) {
			ServletWebServerFactory factory = getWebServerFactory();
			this.webServer = factory.getWebServer(getSelfInitializer());
			getBeanFactory().registerSingleton("webServerGracefulShutdown",
					new WebServerGracefulShutdownLifecycle(this.webServer));
			getBeanFactory().registerSingleton("webServerStartStop",
					new WebServerStartStopLifecycle(this, this.webServer));
		}
		else if (servletContext != null) {
			try {
				getSelfInitializer().onStartup(servletContext);
			}
			catch (ServletException ex) {
				throw new ApplicationContextException("Cannot initialize servlet context", ex);
			}
		}
		initPropertySources();
	}


```

#### 10.registerListeners


获取所有的ApplicaitonListener，添加到前面创建好的SimpleApplicationEventMulticaster中。

```
protected void registerListeners() {
		// 获取静态指定的ApplicationListener
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

        //获取BeanFactory注册的ApplicationListener，这里只是拿到beanName。
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```



#### 11. finishBeanFactoryInitialization

实例化所有的非懒加载单例bean。

也就是遍历所有的beanDefinitionNames，调用BeanFactory的getBean逻辑。

实例化所有bean的关键入口：DefaultListableBeanFactory#preInstantiateSingletons

#### 12. finishRefresh

1. 获取LifecycleProcessor，并调用onRefresh。一般是DefaultLifecycleProcessor的onRefresh。


2. 发布ContextRefreshedEvent事件。



```
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```



### afterRefresh

无任何逻辑

### 输出启动完成日志


```
Started Application in 49.522 seconds (JVM running for 60.352)
```

### 执行SpringApplicationRunListener的started方法

发送ApplicationStartedEvent事件


### callRunners

拿到所有的ApplicationRunner、CommandLineRunner，执行run方法。


```
	private void callRunners(ApplicationContext context, ApplicationArguments args) {
		List<Object> runners = new ArrayList<>();
		runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
		runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
		AnnotationAwareOrderComparator.sort(runners);
		for (Object runner : new LinkedHashSet<>(runners)) {
			if (runner instanceof ApplicationRunner) {
				callRunner((ApplicationRunner) runner, args);
			}
			if (runner instanceof CommandLineRunner) {
				callRunner((CommandLineRunner) runner, args);
			}
		}
	}
```

### 执行SpringApplicationRunListener的running方法

发送ApplicationReadyEvent事件

# 启动过程总结


![image](http://static.silence.work/spring-boot-4.png)

Spring Boot启动过程主要是在Spring的ApplicationContext的refresh前后添加了一些逻辑。

**在refresh前**主要做自动装配，从META/spring.factories中加载到拓展的配置类并实例化，例如各种AutoConfiguration、ApplicationContextInitializer、ApplicationListener等。同时根据环境准备好相应的ApplicationContext子类。

**在refresh后**触发ApplicationStartedEvent事件，以及提供类似于ApplicationRunner、CommandLineRunner这样的钩子方便开发者在spring-boot启动后做一些自定义拓展。

