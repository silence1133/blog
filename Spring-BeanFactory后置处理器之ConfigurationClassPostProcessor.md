title: Spring BeanFactory后置处理器之ConfigurationClassPostProcessor
author: Silence
tags:
  - Spring
categories:
  - Spring
date: 2020-10-31 19:04:00
---
# 本文能帮你解答的问题

1. ConfigurationClassPostProcessor的作用是什么？
2. @Configuration、@Component、@ComponentScan、@ComponentScans、@Bean、@Import等注解是如何完成bean注册的？
3. @Conditional注解如何控制配置类是否需要解析？
4. @Configuration的bean为什么要增强？增强了什么逻辑？

# 类介绍

## 作用

非常重要的一个BeanFactory后置处理器，通过扫描配置类完成各种注解的解析，自动完成所有BeanDefinition的注册。

例如通过对@ComponentScan、@Bean、@Import等注解的解析完成bean的注册。


## 类继承关系


![image](http://static.silence.work/ConfigurationClassPostProcessor.png)

继承了3个Aware接口、用于控制执行顺序的Ordered接口，以及BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor。

### Aware

用于拿到ClassLoader、ResourceLoader以及Environment对象。

### Ordered

通过实现getOrder控制执行该BeanFactory后置处理器的优先级。

ConfigurationClassPostProcessors实现的getOrder放回的是Integer.MAX_VALUE，用于保证最先执行。

了解BeanFactoryProcessor执行顺序的知道，实现了PriorityOrdered的后置处理器会优先于实现了Ordered的后置处理器，因此ConfigurationClassPostProcessors还实现了PriorityOrdered以确保第一个执行。

```
	@Override
	public int getOrder() {
		return Ordered.LOWEST_PRECEDENCE;  // within PriorityOrdered
	}
```

### BeanDefinitionRegistryPostProcessor

通过实现postProcessBeanDefinitionRegistry方法，能拿到BeanDefinitionRegistry对象，BeanDefinitionRegistry是注册、修改BeanDefinition的重要类，因此通实现该方法可以向容器中对BeanDefinition进行自定义新增或者修改等。

### BeanFactoryPostProcessor

通过实现postProcessBeanFactory，能拿到BeanFactory对象，是在所有到BeanDefinition注册完成后，实例化前回调，从而可以对bean做进一步的修改


## 添加ConfigurationClassPostProcessor的策略

当开启注解扫描bean，spring则会自动添加该BeanFactory后置处理器，来完成自动扫描注册的bean。

xml的ApplicationContext环境下开启方式：

```
<context:annotation-config/>或者<context:component-scan/>
```

对于AnnotationConfigApplicationContext默认会自动添加ConfigurationClassPostProcessor。（通过构造函数最终调用到AnnotationConfigUtils#registerAnnotationConfigProcessors()完成beanDefition添加，在refresh过程完成该bean的注册）



# 方法源码解读

这里从实现的postProcessBeanDefinitionRegistry和postProcessBeanFactory按照执行的先后顺序进行分析。

其中触发这两个方法的时机都是在
AbstractApplicationContext#refresh的invokeBeanFactoryPostProcessors过程触发。


```
AbstractApplicationContext#refresh
    -->AbstractApplicationContext#invokeBeanFactoryPostProcessors
        -->PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors(beanFactory,beanFactoryPostProcessors)
```
## postProcessBeanDefinitionRegistry


```
@Override
	public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		int registryId = System.identityHashCode(registry);
		if (this.registriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanDefinitionRegistry already called on this post-processor against " + registry);
		}
		if (this.factoriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + registry);
		}
		this.registriesPostProcessed.add(registryId);
		processConfigBeanDefinitions(registry);
	}
```

ConfigurationClassPostProcessor#processConfigBeanDefinitions：


```
	public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
		List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
		String[] candidateNames = registry.getBeanDefinitionNames();
        /1. 寻找所有配置类的BeanDefinition
        for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			//是否是配置类，是则添加到待处理的集合中
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}
        //没有return不处理
		// Return immediately if no @Configuration classes were found
		if (configCandidates.isEmpty()) {
			return;
		}
        
        //2. 排序配置类
		// Sort by previously determined @Order value, if applicable
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});
		
        //省略部分代码
        
        //准备好配置解析类
		// Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
		    //3. 解析配置类
			parser.parse(candidates);
			parser.validate();

			Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
			configClasses.removeAll(alreadyParsed);

			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			//4. 注册解析到的所有BeanDefinition
			this.reader.loadBeanDefinitions(configClasses);
			alreadyParsed.addAll(configClasses);
            
            //省略部分代码
		}
		while (!candidates.isEmpty());
        
        //省略部分代码
	}
```

processConfigBeanDefinitions方法较长，这里拆分几个过程。

### step1. 寻找所有配置类的BeanDefinition


从当前已经注册的所有BeanDefinition中寻找属于配置类的BeanDefinition，放到configCandidates，用于后续的处理。

```
	List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
	String[] candidateNames = registry.getBeanDefinitionNames();
		for (String beanName : candidateNames) {
			BeanDefinition beanDef = registry.getBeanDefinition(beanName);
			if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
				if (logger.isDebugEnabled()) {
					logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
				}
			}
			//如果是配置类，是则放到configCandidates中
			else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
				configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
			}
		}
```

<font color='red'>**问题：什么类才叫配置类？**</font>

带着这个问题看下ConfigurationClassUtils.checkConfigurationClassCandidate方法逻辑。


```
public static boolean checkConfigurationClassCandidate(
			BeanDefinition beanDef, MetadataReaderFactory metadataReaderFactory) {

		String className = beanDef.getBeanClassName();
		if (className == null || beanDef.getFactoryMethodName() != null) {
			return false;
		}
        //取得类上的注解信息
		AnnotationMetadata metadata;
		if (beanDef instanceof AnnotatedBeanDefinition &&
				className.equals(((AnnotatedBeanDefinition) beanDef).getMetadata().getClassName())) {
			// Can reuse the pre-parsed metadata from the given BeanDefinition...
			metadata = ((AnnotatedBeanDefinition) beanDef).getMetadata();
		}
		else if (beanDef instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) beanDef).hasBeanClass()) {
			// Check already loaded Class if present...
			// since we possibly can't even load the class file for this Class.
			Class<?> beanClass = ((AbstractBeanDefinition) beanDef).getBeanClass();
			//继承了这些类的都不是配置类
			if (BeanFactoryPostProcessor.class.isAssignableFrom(beanClass) ||
					BeanPostProcessor.class.isAssignableFrom(beanClass) ||
					AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
					EventListenerFactory.class.isAssignableFrom(beanClass)) {
				return false;
			}
			metadata = AnnotationMetadata.introspect(beanClass);
		}
		else {
            //省略部分代码
		}
        
        //获取Configuration注解的值信息
		Map<String, Object> config = metadata.getAnnotationAttributes(Configuration.class.getName());
		//如果是@Configuration的bean，则属于配置类，会设置CONFIGURATION_CLASS_ATTRIBUTE属性为full。
		if (config != null && !Boolean.FALSE.equals(config.get("proxyBeanMethods"))) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_FULL);
		}
		//如果满足isConfigurationCandidate，也是配置类，设置CONFIGURATION_CLASS_ATTRIBUTE属性为lite
		else if (config != null || isConfigurationCandidate(metadata)) {
			beanDef.setAttribute(CONFIGURATION_CLASS_ATTRIBUTE, CONFIGURATION_CLASS_LITE);
		}
		else {
		    //都不满足不是配置类。
			return false;
		}

		// It's a full or lite configuration candidate... Let's determine the order value, if any.
		//设置排序值用于后续对配置类进行排序处理
		Integer order = getOrder(metadata);
		if (order != null) {
			beanDef.setAttribute(ORDER_ATTRIBUTE, order);
		}

		return true;
	}
	
```


ConfigurationClassUtils#isConfigurationCandidate：

```
	static {
		candidateIndicators.add(Component.class.getName());
		candidateIndicators.add(ComponentScan.class.getName());
		candidateIndicators.add(Import.class.getName());
		candidateIndicators.add(ImportResource.class.getName());
	}
	
	
    public static boolean isConfigurationCandidate(AnnotationMetadata metadata) {
		// Any of the typical annotations found?
		//只要存在上面4种注解中的一种都是配置类
		for (String indicator : candidateIndicators) {
			if (metadata.isAnnotated(indicator)) {
				return true;
			}
		}
		// Finally, let's look for @Bean methods...
		return metadata.hasAnnotatedMethods(Bean.class.getName());
	}

```

**总结一下配置类的定义：**

1. 有@Configuration注解的类
2. 有Component、ComponentScan、Import、ImportResource注解的类
3. 方法上有@Bean注解的类

特别说明：满足1和满足2、3对于配置类的beanDefinition的CONFIGURATION_CLASS_ATTRIBUTE值分别设置为full和lite的原因，这个在postProcessBeanFactory方法中会有答案。


### step2. 排序配置类

根据@Order标注的顺序来排序所有@Configuration标注的BeanDefinition


```
		// Sort by previously determined @Order value, if applicable
		configCandidates.sort((bd1, bd2) -> {
			int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
			int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
			return Integer.compare(i1, i2);
		});
```

### step3. 解析配置类及部分beanDefinition的注册

解析配置类中的各种注解，例如@ComponentScan、@Import、@Bean等，同时完成部分beanDefinition的注册，类似与@Component注解的bean，@Configration的bean，对于@Import、@Bean指定的bean会在step4完成beanDefinition的注册。

```
		// Parse each @Configuration class
		ConfigurationClassParser parser = new ConfigurationClassParser(
				this.metadataReaderFactory, this.problemReporter, this.environment,
				this.resourceLoader, this.componentScanBeanNameGenerator, registry);

		Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
		Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
		do {
			parser.parse(candidates);
			parser.validate();
		}
		while (!candidates.isEmpty());
```
ConfigurationClassParser.parse调用链路：

```
ConfigurationClassParser#parse()
    ->ConfigurationClassParser#processConfigurationClass
        ->ConfigurationClassParser#doProcessConfigurationClass
```

```
protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
        //处理@Conditional注解，条件判断当前配置类是否需要处理
	    if (this.conditionEvaluator.shouldSkip(configClass.getMetadata(), ConfigurationPhase.PARSE_CONFIGURATION)) {
			return;
		}
        //部分代码省略
		// Recursively process the configuration class and its superclass hierarchy.
		SourceClass sourceClass = asSourceClass(configClass);
		do {
		    //解析配置类的关键方法
			sourceClass = doProcessConfigurationClass(configClass, sourceClass);
		}
		while (sourceClass != null);
        //缓存起来，这里得到的configClass就是经过解析后封装好的ConfigurationClass，用于后续做经一步的处理
		this.configurationClasses.put(configClass, configClass);
	}
```
processConfigurationClass处理逻辑是先判断当前配置类是否需要解析，如果需要则然后调用doProcessConfigurationClass去做解析。

#### @Conditional判断配置类是否需要解析

通过ConditionEvaluator#shouldSkip()，判断当前配置类是否需要解析，这里传入的phase参数为PARSE_CONFIGURATION，代表是解析配置阶段。

```
    public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
		//不存在@Conditional注解，无需跳过
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}
		
		//省略部分代码
		
		//一个配置类可能有很多@Conditional
		List<Condition> conditions = new ArrayList<>();
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}
        //排序
		AnnotationAwareOrderComparator.sort(conditions);

        //按顺序调用matches方法
		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			
			//是否继承的ConfigurationCondition，是，则取指定的判断阶段
		    if (condition instanceof ConfigurationCondition) {
				requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
			}
			//判断如果当前处理阶段和Condition指定的阶段一致，则调用match方法。
			if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
				return true;
			}
		}

		return false;
	}

```

**特别说明：ConditionEvaluator#shouldSkip方法在两个阶段都会调用，一个是配置类的解析阶段，一个是bean的注册阶段，如果需要指定只在某个阶段生效，需要实现ConfigurationCondition来指定处理的阶段，ConfigurationPhase参数来区分不同的阶段**，具体使用参考[@Conditional通过条件来控制bean的注册](https://mp.weixin.qq.com/s?__biz=MzA5MTkxMDQ4MQ==&mid=2648934205&idx=1&sn=5407aa7c49eb34f7fb661084b8873cfe&chksm=88621f03bf1596159eeb40d75620db03457f4aa831066052ebc6e1efc2d7b18802a49a7afe8a&token=332995799&lang=zh_CN#rd)


#### 解析@PropertySource

@PropertySource用于指定配置文件地址，完成配置的加载，封装到Environment中。

例如：

```
@PropertySource("classpath:test.properties")
```

```
	// Process any @PropertySource annotations
	for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
		sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
		if (this.environment instanceof ConfigurableEnvironment) {
		    //解析
			processPropertySource(propertySource);
		}
		else {
			logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
					"]. Reason: Environment must implement ConfigurableEnvironment");
		}
	}
```
```
ConfigurationClassParser#processPropertySource：
	
	private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
		//配置文件的名称
		String name = propertySource.getString("name");
		if (!StringUtils.hasLength(name)) {
			name = null;
		}
		//配置文件的编码
		String encoding = propertySource.getString("encoding");
		if (!StringUtils.hasLength(encoding)) {
			encoding = null;
		}
		//配置文件路径
		String[] locations = propertySource.getStringArray("value");
		Assert.isTrue(locations.length > 0, "At least one @PropertySource(value) location is required");
		boolean ignoreResourceNotFound = propertySource.getBoolean("ignoreResourceNotFound");

		Class<? extends PropertySourceFactory> factoryClass = propertySource.getClass("factory");
		PropertySourceFactory factory = (factoryClass == PropertySourceFactory.class ?
				DEFAULT_PROPERTY_SOURCE_FACTORY : BeanUtils.instantiateClass(factoryClass));
        //遍历路径
		for (String location : locations) {
			try {
				String resolvedLocation = this.environment.resolveRequiredPlaceholders(location);
				//加载到配置文件封装成Resource
				Resource resource = this.resourceLoader.getResource(resolvedLocation);
				//将封装成的PropertySource添加到environment的PropertySources中
				addPropertySource(factory.createPropertySource(name, new EncodedResource(resource, encoding)));
			}
			catch (IllegalArgumentException | FileNotFoundException | UnknownHostException ex) {
		        //代码省略
			}
		}
	}
	
```

#### 解析@ComponentScan、@ComponentScans

@ComponentScan用于指定自动扫描bean的包路径。

关于该注解的详细用法参考[@ComponentScan、@ComponentScans详解](https://mp.weixin.qq.com/s?__biz=MzA5MTkxMDQ4MQ==&mid=2648934150&idx=1&sn=6e466720d78f212cbd7d003bc5c2eec2&chksm=88621f38bf15962e324888161d0b91f34c26e4b8a53da87f1364e5af7010dbdcabc9fb555476&token=1346356013&lang=zh_CN#rd)

**主干代码：**

```
		// Process any @ComponentScan annotations
		Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
		if (!componentScans.isEmpty() &&
				!this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
			for (AnnotationAttributes componentScan : componentScans) {
				// The config class is annotated with @ComponentScan -> perform the scan immediately
				//委托给componentScanParser完成bean的扫描与注册
				Set<BeanDefinitionHolder> scannedBeanDefinitions =
						this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
				// Check the set of scanned definitions for any further config classes and parse recursively if needed
				//遍历所有扫描到的BeanDefinition
				for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
					BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
					if (bdCand == null) {
						bdCand = holder.getBeanDefinition();
					}
					//如果扫描到的是配置类，则递归走处理配置类的方法
					if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
						parse(bdCand.getBeanClassName(), holder.getBeanName());
					}
				}
			}
		}
```

主干逻辑：

> 1. 根据包路径扫描bean并注册
> 
> 2. 如果是配置类，递归走配置类的解析，通过反复对配置类解析，最终完成所有扫描到的bean注册


**bean扫描并注册过程委托给ComponentScanAnnotationParser#parse：**

```
public Set<BeanDefinitionHolder> parse(AnnotationAttributes componentScan, final String declaringClass) {
		ClassPathBeanDefinitionScanner scanner = new ClassPathBeanDefinitionScanner(this.registry,
				componentScan.getBoolean("useDefaultFilters"), this.environment, this.resourceLoader);
        //nameGenerator，用于获取bean名称生成器，如果没有配置，则取Spring默认
		Class<? extends BeanNameGenerator> generatorClass = componentScan.getClass("nameGenerator");
		boolean useInheritedGenerator = (BeanNameGenerator.class == generatorClass);
		scanner.setBeanNameGenerator(useInheritedGenerator ? this.beanNameGenerator :
				BeanUtils.instantiateClass(generatorClass));
        //scopedProxy，作用？
		ScopedProxyMode scopedProxyMode = componentScan.getEnum("scopedProxy");
		if (scopedProxyMode != ScopedProxyMode.DEFAULT) {
			scanner.setScopedProxyMode(scopedProxyMode);
		}
		else {
			Class<? extends ScopeMetadataResolver> resolverClass = componentScan.getClass("scopeResolver");
			scanner.setScopeMetadataResolver(BeanUtils.instantiateClass(resolverClass));
		}
        //resourcePattern，需要扫描包中的那些资源，默认是：**/*.class，即会扫描指定包中所有的class文件
		scanner.setResourcePattern(componentScan.getString("resourcePattern"));
        //过滤器：用来配置被扫描出来的那些类会被作为组件注册到容器中
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("includeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addIncludeFilter(typeFilter);
			}
		}
		//过滤器，和includeFilters作用刚好相反，用来对扫描的类进行排除的，被排除的类不会被注册到容器中
		for (AnnotationAttributes filter : componentScan.getAnnotationArray("excludeFilters")) {
			for (TypeFilter typeFilter : typeFiltersFor(filter)) {
				scanner.addExcludeFilter(typeFilter);
			}
		}
        //是否延迟初始化被注册的bean
		boolean lazyInit = componentScan.getBoolean("lazyInit");
		if (lazyInit) {
			scanner.getBeanDefinitionDefaults().setLazyInit(true);
		}

        //basePackages：扫描包路径
		Set<String> basePackages = new LinkedHashSet<>();
		String[] basePackagesArray = componentScan.getStringArray("basePackages");
		for (String pkg : basePackagesArray) {
			String[] tokenized = StringUtils.tokenizeToStringArray(this.environment.resolvePlaceholders(pkg),
					ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
			Collections.addAll(basePackages, tokenized);
		}
		//basePackageClasses：指定一些类，spring容器会扫描这些类所在的包及其子包中的类
		for (Class<?> clazz : componentScan.getClassArray("basePackageClasses")) {
			basePackages.add(ClassUtils.getPackageName(clazz));
		}
		//如果没有配置，则取配置类所在的包路径
		if (basePackages.isEmpty()) {
			basePackages.add(ClassUtils.getPackageName(declaringClass));
		}

		scanner.addExcludeFilter(new AbstractTypeHierarchyTraversingFilter(false, false) {
			@Override
			protected boolean matchClassName(String className) {
				return declaringClass.equals(className);
			}
		});
		return scanner.doScan(StringUtils.toStringArray(basePackages));
	}
```
该方法主要就是将注解设置的参数封装到ClassPathBeanDefinitionScanner中，然后进一步委托给ClassPathBeanDefinitionScanner.doScan进行解析。

**ClassPathBeanDefinitionScanner.doScan：**


```
	protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
		Assert.notEmpty(basePackages, "At least one base package must be specified");
		Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
		//遍历包路径
		for (String basePackage : basePackages) {
		    //扫描到所有的BeanDefinition。
		    //这里的BeanDefinition只是一个很原始的BeanDefinition，很多参数需要进一步设置
			Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
			//遍历设置每一个BeanDefinition的参数
			for (BeanDefinition candidate : candidates) {
				ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
				//scope
				candidate.setScope(scopeMetadata.getScopeName());
				//benaName
				String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
				if (candidate instanceof AbstractBeanDefinition) {
					postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
				}
				if (candidate instanceof AnnotatedBeanDefinition) {
					AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
				}
				if (checkCandidate(beanName, candidate)) {
					BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
					definitionHolder =
							AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
					beanDefinitions.add(definitionHolder);
					//完成beanDefinition的注册
					registerBeanDefinition(definitionHolder, this.registry);
				}
			}
		}
		return beanDefinitions;
	}
```

到这里完成了所有扫描到的beanDefinition的注册。

#### 解析@Import

用于批量导入指定的bean，例如当某个Bean上没有@Component注解时，或者只是指定具体某几个bean，则可以通过该注解完成指定bean的导入。

@Import除了导入普通的bean外，还可以导入如下几种bean：
> 1. @Configuration标注的类
> 2. 继承了ImportBeanDefinitionRegistrar的类
> 3. 继承了ImportSelector或者DeferredImportSelector的类

具体使用参考[@import详解](https://mp.weixin.qq.com/s?__biz=MzA5MTkxMDQ4MQ==&mid=2648934173&idx=1&sn=60bb7d58fd408db985a785bfed6e1eb2&chksm=88621f23bf15963589f06b7ce4e521a7c8d615b1675788f383cbb0bcbb05b117365327e1941a&token=704646761&lang=zh_CN#rd)


```
	// Process any @Import annotations
	processImports(configClass, sourceClass, getImports(sourceClass), true);
```

**ConfigurationClassParser#processImports**


```
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
			Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
        //省略部分代码
		else {
			this.importStack.push(configClass);
			try {
			    //遍历了有@Import注解的所有类
				for (SourceClass candidate : importCandidates) {
				    //类是否继承了ImportSelector
					if (candidate.isAssignable(ImportSelector.class)) {
						// Candidate class is an ImportSelector -> delegate to it to determine imports
						Class<?> candidateClass = candidate.loadClass();
						ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
								this.environment, this.resourceLoader, this.registry);
						if (selector instanceof DeferredImportSelector) {
							this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
						}
						else {
							String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
							Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
							//递归调用processImports方法
							processImports(configClass, currentSourceClass, importSourceClasses, false);
						}
					}
					//是否继承了ImportBeanDefinitionRegistrar
					else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
						// Candidate class is an ImportBeanDefinitionRegistrar ->
						// delegate to it to register additional bean definitions
						Class<?> candidateClass = candidate.loadClass();
						ImportBeanDefinitionRegistrar registrar =
								ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
										this.environment, this.resourceLoader, this.registry);
						configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
					}
					//都不是，则作为配置类进行处理，实际是递归调用processConfigurationClass方法
					else {
						// Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
						// process it as an @Configuration class
						this.importStack.registerImport(
								currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
						processConfigurationClass(candidate.asConfigClass(configClass));
					}
				}
			}
            //省略部分代码
		}
	}
```
主要对@Import注解标注的类分3种情况进行处理（ImportSelector、ImportBeanDefinitionRegistrar、普通类）。

##### ImportSelector

ImportSelector分两种情况：

1. 如果是直接实现的ImportSelector，则获取到自定义的ImportSelector，调用selectImports获取到要导入的指定bean，递归调用processImports，出口为processConfigurationClass方法。
2. 如果是DeferredImportSelector，待所有配置类解析后才会执行导入。

```
            //这个candidate就是解析到的@Import value指定的类
			Class<?> candidateClass = candidate.loadClass();
			//实例化实现的ImportSelectorbean
			ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
			this.environment, this.resourceLoader, this.registry);
			//如果实现的是DeferredImportSelector（ImportSelector的子类）
			if (selector instanceof DeferredImportSelector) {
			    //将当前selector bean先缓存下来，这里不执行selectImports
				this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
			}
			else {
			    //直接实现的ImportSelector，直接调用selectImports，获取importClassNames
				String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
				Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
				//递归调用processImports
				processImports(configClass, currentSourceClass, importSourceClasses, false);
			}
```



##### ImportBeanDefinitionRegistrar

先缓存起来，先不执行实现了ImportBeanDefinitionRegistrar的方法，在step4完成bean的注册

```
		Class<?> candidateClass = candidate.loadClass();
		ImportBeanDefinitionRegistrar registrar = ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
										this.environment, this.resourceLoader, this.registry);
		configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
```


##### 普通类

递归调用processConfigurationClass方法。

这里是也对直接实现ImportSelector解析递归调用processImports的出口。

```
        this.importStack.registerImport(currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
        //当成配置类进行处理
		processConfigurationClass(candidate.asConfigClass(configClass));
```

#### 解析@ImportResource

用于引入xml形式的bean配置文件，此步并未真正去解析，只是先缓存起来，step4完成解析和bean的注册。

```
		AnnotationAttributes importResource =
				AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}
```

#### 解析标注了@Bean的methods

此步只是通过扫描添加了@Bean注解的方法，封装成BeanMethod缓存起来，真正的bean解析以及注册在step4。延迟导入。

```
		Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}
	    // Process default methods on interfaces
		processInterfaces(configClass, sourceClass);
```


### step4. 完成@Bean、@Import等bean的解析与注册

经过前一步对配置类的解析，每一个配置类解析后会封装成ConfigurationClass，这一步针对封装好的ConfigurationClass继续完成BeanDefinition的注册。例如前面解析到的@Bean方法定义的bean、@Import value为ImportBeanDefinitionRegistrar、@ImportedResources指定的bean。

```
			// Read the model and create bean definitions based on its content
			if (this.reader == null) {
				this.reader = new ConfigurationClassBeanDefinitionReader(
						registry, this.sourceExtractor, this.resourceLoader, this.environment,
						this.importBeanNameGenerator, parser.getImportRegistry());
			}
			this.reader.loadBeanDefinitions(configClasses);
```

ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass

```
	private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        //省略部分代码

		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		//处理@Bean的beanDefinitoin注册
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}
        //处理@ImportedResources的beanDefinitoin注册
		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		//处理@Import下value是ImportBeanDefinitionRegistrar和DeferredImportSelector指定的beanDefinitoin注册
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```

#### @Bean的解析与注册

根据@Bean指定的方法，解析注解相关值，构造好BeanDefinition相关属性，最后通过BeanDefinitionRegistry完成bean的注册。

ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod
```
	private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
	    //对应的配置类
		ConfigurationClass configClass = beanMethod.getConfigurationClass();
		//注解信息
		MethodMetadata metadata = beanMethod.getMetadata();
		//方法名称
		String methodName = metadata.getMethodName();
       
        //省略部分代码
        //@Bean注解的属性值
		AnnotationAttributes bean = AnnotationConfigUtils.attributesFor(metadata, Bean.class);
		Assert.state(bean != null, "No @Bean annotation attributes");
        //获取bean名称，如果注解指定了则取指定的名称，否则取方法名称
		// Consider name and any aliases
		List<String> names = new ArrayList<>(Arrays.asList(bean.getStringArray("name")));
		String beanName = (!names.isEmpty() ? names.remove(0) : methodName);
        //注册别名
		// Register aliases even when overridden
		for (String alias : names) {
			this.registry.registerAlias(beanName, alias);
		}

        //省略部分代码

        //开始构造BeanDefinition
		ConfigurationClassBeanDefinition beanDef = new ConfigurationClassBeanDefinition(configClass, metadata);
		beanDef.setResource(configClass.getResource());
		beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

		if (metadata.isStatic()) {
		      //省略部分代码
		}
		else {
		    //设置bean的实例化的工厂bean
			// instance @Bean method
			beanDef.setFactoryBeanName(configClass.getBeanName());
			//设置实例化的工厂方法
			beanDef.setUniqueFactoryMethodName(methodName);
		}

		if (metadata instanceof StandardMethodMetadata) {
			beanDef.setResolvedFactoryMethod(((StandardMethodMetadata) metadata).getIntrospectedMethod());
		}

		beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
		beanDef.setAttribute(org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
				SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

		AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

		Autowire autowire = bean.getEnum("autowire");
		if (autowire.isAutowire()) {
			beanDef.setAutowireMode(autowire.value());
		}

		boolean autowireCandidate = bean.getBoolean("autowireCandidate");
		if (!autowireCandidate) {
			beanDef.setAutowireCandidate(false);
		}
        //指定bean初始化回调的方法
		String initMethodName = bean.getString("initMethod");
		if (StringUtils.hasText(initMethodName)) {
			beanDef.setInitMethodName(initMethodName);
		}
        //指定bean销毁时回调的方法
		String destroyMethodName = bean.getString("destroyMethod");
		beanDef.setDestroyMethodName(destroyMethodName);

		// 设置scope
		ScopedProxyMode proxyMode = ScopedProxyMode.NO;
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(metadata, Scope.class);
		if (attributes != null) {
			beanDef.setScope(attributes.getString("value"));
			proxyMode = attributes.getEnum("proxyMode");
			if (proxyMode == ScopedProxyMode.DEFAULT) {
				proxyMode = ScopedProxyMode.NO;
			}
		}
		
        //省略部分代码
		
		//注册
		this.registry.registerBeanDefinition(beanName, beanDefToRegister);
	}

```



#### @ImportedResources的解析与注册

//不分析

#### @Import注解value是ImportBeanDefinitionRegistrar

逻辑比较简单，就是从缓存中拿到前面解析到@Import value是ImportBeanDefinitionRegistrar的所有类，循环调用registerBeanDefinitions，完成自定义BeanDefinition的注册。

```
	private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, AnnotationMetadata> registrars) {
		registrars.forEach((registrar, metadata) ->
				registrar.registerBeanDefinitions(metadata, this.registry, this.importBeanNameGenerator));
	}
```


## postProcessBeanFactory

```
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		int factoryId = System.identityHashCode(beanFactory);
		if (this.factoriesPostProcessed.contains(factoryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called on this post-processor against " + beanFactory);
		}
		this.factoriesPostProcessed.add(factoryId);
		//如果postProcessBeanDefinitionRegistry没有执行，则通过这个判断来调用processConfigBeanDefinitions重新执行。
		if (!this.registriesPostProcessed.contains(factoryId)) {
			// BeanDefinitionRegistryPostProcessor hook apparently not supported...
			// Simply call processConfigurationClasses lazily at this point then.
			processConfigBeanDefinitions((BeanDefinitionRegistry) beanFactory);
		}
        //利用cglib增强@Configuration的bean
		enhanceConfigurationClasses(beanFactory);
		//添加一个bean后置处理器ImportAwareBeanPostProcessor
		beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
	}
```

### CGLIB增强Configuration类

问题：为什么要增强Configuration类？

带着这个疑问看下enhanceConfigurationClasses方法。

ConfigurationClassPostProcessor#enhanceConfigurationClasses：

```
	public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
		Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
		for (String beanName : beanFactory.getBeanDefinitionNames()) {
			BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
			Object configClassAttr = beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
			MethodMetadata methodMetadata = null;
			if (beanDef instanceof AnnotatedBeanDefinition) {
				methodMetadata = ((AnnotatedBeanDefinition) beanDef).getFactoryMethodMetadata();
			}
			if ((configClassAttr != null || methodMetadata != null) && beanDef instanceof AbstractBeanDefinition) {
				// Configuration class (full or lite) or a configuration-derived @Bean method
				// -> resolve bean class at this point...
				AbstractBeanDefinition abd = (AbstractBeanDefinition) beanDef;
				if (!abd.hasBeanClass()) {
					try {
						abd.resolveBeanClass(this.beanClassLoader);
					}
					catch (Throwable ex) {
						throw new IllegalStateException(
								"Cannot load configuration class: " + beanDef.getBeanClassName(), ex);
					}
				}
			}
			//是否是full类型的配置类
			if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {
                //省略部分代码
                //存下来，在下面代码for循环内进行增强处理
				configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
			}
		}
		//没有full类型的配置类，注解return
		if (configBeanDefs.isEmpty()) {
			// nothing to enhance -> return immediately
			return;
		}
        
        //遍历Configuration类，进行增强
		ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
		for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
			AbstractBeanDefinition beanDef = entry.getValue();
			// If a @Configuration class gets proxied, always proxy the target class
			beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
			// Set enhanced subclass of the user-specified bean class
			Class<?> configClass = beanDef.getBeanClass();
			//增强
			Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
			if (configClass != enhancedClass) {
				beanDef.setBeanClass(enhancedClass);
			}
		}
	}
```

看enhancer.enhance做了哪些增强：

入口：ConfigurationClassEnhancer#newEnhancer

```
	private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
		Enhancer enhancer = new Enhancer();
		enhancer.setSuperclass(configSuperClass);
		//继承了EnhancedConfiguration接口
		enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
		enhancer.setUseFactory(false);
		enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
		enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
		//添加增强的拦截器
		enhancer.setCallbackFilter(CALLBACK_FILTER);
		enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
		return enhancer;
	}
	
	
	private static final ConditionalCallbackFilter CALLBACK_FILTER = new ConditionalCallbackFilter(CALLBACKS);
	
	//两个拦截器
	// The callbacks to use. Note that these callbacks must be stateless.
	private static final Callback[] CALLBACKS = new Callback[] {
			new BeanMethodInterceptor(),
			new BeanFactoryAwareMethodInterceptor(),
			NoOp.INSTANCE
	};


```

EnhancedConfiguration接口继承了BeanFactoryAware接口，因此增强能使得Configuration类能拿到BeanFactory，**这是增强的第一个作用**。

增强添加了两个拦截器，BeanMethodInterceptor和BeanFactoryAwareMethodInterceptor。

BeanMethodInterceptor：拦截@Bean修饰的方法用于，保证只能被调用一次，确保返回的bean是单例的，**这是增强第二个作用**。

BeanFactoryAwareMethodInterceptor：拦截setBeanFactory方法，set $$beanFactory变量值。

### 添加ImportAwareBeanPostProcessor


```
	beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
```
ImportAwareBeanPostProcessor的作用是在属性赋值的时候，对是EnhancedConfiguration的bean调用setBeanFactory方法（搭配上面会将Configuration类增强为EnhancedConfiguration类来理解）。

# 总结

1. ConfigurationClassPostProcessor利用了BeanDefinitionRegistryPostProcessor能自定义注册beanDefinition的拓展能力，以及BeanFactoryPostProcessor能对注册的beanDefinition做修改的拓展能力，从而达到通过注解形式（脱离xml）实现bean的自动注册。
2. 通过在自定义注册beanDefinition阶段，扫描到已经注册的配置类，取得配置类上相关注解元素（即相关配置），从而完成beanDefinition的添加
3. 上一步注册的beanDefinition可能依然是一个配置类，则需要反复执行上一步的扫描解析注册过程，直到没有配置类为止。
4. 配置类的定义包括：有@Configuration注解、Component、ComponentScan、Import、ImportResource、方法上有@Bean注解的类，只要满足有某一个注解即为配置类。

