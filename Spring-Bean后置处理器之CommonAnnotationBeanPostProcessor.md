title: Spring Bean后置处理器之CommonAnnotationBeanPostProcessor
author: Silence
tags:
  - Spring
categories:
  - Spring
date: 2020-10-31 19:03:00
---

# 本文能帮你解答的问题
1. @PostConstruct和@PreDestroy注解标注的方法是在什么阶段调用的？
2. @Resource是如何完成属性自动注入的？
3. @Resource相比@Autowired查找候选者的过程差异是什么？

# 类定义

## 作用

1. 对@PostConstruct和@PreDestroy注解的处理，完成bean初始化前和销毁前的回调

2. 对@Resource注解的处理，完成依赖字段的注入

## 类结构
![image](http://static.silence.work/CommonAnnotationBeanPostProcessor.png)


### 继承父类

#### InstantiationAwareBeanPostProcessor

提供了bean实例化前后对bean的操作以及在属性赋值前，传递属性值的postProcessProperties方法。

#### InitDestroyAnnotationBeanPostProcessor

继承了DestructionAwareBeanPostProcessor和MergedBeanDefinitionPostProcessor接口。

主要实现的方法有：postProcessMergedBeanDefinition、postProcessBeforeInitialization、postProcessBeforeDestruction、requiresDestruction。


##### MergedBeanDefinitionPostProcessor

用于处理合并后的BeanDefinition，在bean实例化后、属性赋值前回调。

##### DestructionAwareBeanPostProcessor

用于bean销毁前的回调。

### 构造函数

```
	public CommonAnnotationBeanPostProcessor() {
		setOrder(Ordered.LOWEST_PRECEDENCE - 3);
		setInitAnnotationType(PostConstruct.class);
		setDestroyAnnotationType(PreDestroy.class);
		ignoreResourceType("javax.xml.ws.WebServiceContext");
	}
```
通过构造函数可以大概猜测该后置处理器的作用。

# 实现方法分析

由于CommonAnnotationBeanPostProcessor继承的InitDestroyAnnotationBeanPostProcessor也是具体类，且提供了很多逻辑，这里先分析InitDestroyAnnotationBeanPostProcessor。

## InitDestroyAnnotationBeanPostProcessor

主要是对@PostConstruct和@PreDestroy注解的处理。

### postProcessMergedBeanDefinition

bean实例化前调用。

主要是解析当前bean中添加了@PostConstruct和@PreDestroy注解的方法，缓存起来，给后续实际时机点回调用
```
	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		LifecycleMetadata metadata = findLifecycleMetadata(beanType);
		metadata.checkConfigMembers(beanDefinition);
	}
```

### postProcessBeforeInitialization

bean初始化前调用。

从缓存中获取bean对应的@PostConstruct元素，通过invokeInitMethods执行对应的方法。

```
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			metadata.invokeInitMethods(bean, beanName);
		}
		catch (InvocationTargetException ex) {
			throw new BeanCreationException(beanName, "Invocation of init method failed", ex.getTargetException());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Failed to invoke init method", ex);
		}
		return bean;
	}
```

### postProcessBeforeDestruction

bean销毁前调用，
从缓存中获取bean对应的@PreDestroy元素，通过invokeInitMethods执行对应的方法。

```
	public void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException {
		LifecycleMetadata metadata = findLifecycleMetadata(bean.getClass());
		try {
			metadata.invokeDestroyMethods(bean, beanName);
		}
		catch (InvocationTargetException ex) {
			String msg = "Destroy method on bean with name '" + beanName + "' threw an exception";
			if (logger.isDebugEnabled()) {
				logger.warn(msg, ex.getTargetException());
			}
			else {
				logger.warn(msg + ": " + ex.getTargetException());
			}
		}
		catch (Throwable ex) {
			logger.warn("Failed to invoke destroy method on bean with name '" + beanName + "'", ex);
		}
	}
```

## CommonAnnotationBeanPostProcessor

主要是添加对@Resource注解的处理逻辑。

### postProcessMergedBeanDefinition

除了调用父类查找@PostConstruct和@PreDestroy注解的方法之外，添加了查找@Resource并缓存的逻辑。

```
	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		super.postProcessMergedBeanDefinition(beanDefinition, beanType, beanName);
		InjectionMetadata metadata = findResourceMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
```

### postProcessProperties

属性赋值前调用。

找到bean对应@Resource注解的字段，并调用inject方法完成属性的注入。

```
	@Override
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
		InjectionMetadata metadata = findResourceMetadata(beanName, bean.getClass(), pvs);
		try {
			metadata.inject(bean, beanName, pvs);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of resource dependencies failed", ex);
		}
		return pvs;
	}
```

处理套路和AutowiredAnnotationBeanPostProcessor处理@Autowired注解字段注入一样。

完成属性的注入主要在CommonAnnotationBeanPostProcessor#autowireResource完成。

# @Resource查找候选者过程

查找的主要过程如下：

```
先按Resource的name值作为bean名称找->按名称（字段名称、方法名称、set属性名称）找->按类型找->通过限定符@Qualifier过滤->@Primary->@Priority->根据名称找（字段名称或者方法参数名称）

```

查找源码：

CommonAnnotationBeanPostProcessor#autowireResource

```
protected Object autowireResource(BeanFactory factory, LookupElement element, @Nullable String requestingBeanName)
			throws NoSuchBeanDefinitionException {

		Object resource;
		Set<String> autowiredBeanNames;
		String name = element.name;
		if (factory instanceof AutowireCapableBeanFactory) {
			AutowireCapableBeanFactory beanFactory = (AutowireCapableBeanFactory) factory;
			DependencyDescriptor descriptor = element.getDependencyDescriptor();
			//根据name查找，如果查找不到则调用beanFactory.resolveDependency查找
			if (this.fallbackToDefaultTypeMatch && element.isDefaultName && !factory.containsBean(name)) {
				autowiredBeanNames = new LinkedHashSet<>();
				resource = beanFactory.resolveDependency(descriptor, requestingBeanName, autowiredBeanNames, null);
				if (resource == null) {
					throw new NoSuchBeanDefinitionException(element.getLookupType(), "No resolvable resource object");
				}
			}
			else {
			    //bean factory中有对应bean名称的bean
				resource = beanFactory.resolveBeanByName(name, descriptor);
				autowiredBeanNames = Collections.singleton(name);
			}
		}
		else {
			resource = factory.getBean(name, element.lookupType);
			autowiredBeanNames = Collections.singleton(name);
		}

		if (factory instanceof ConfigurableBeanFactory) {
			ConfigurableBeanFactory beanFactory = (ConfigurableBeanFactory) factory;
			for (String autowiredBeanName : autowiredBeanNames) {
				if (requestingBeanName != null && beanFactory.containsBean(autowiredBeanName)) {
					beanFactory.registerDependentBean(autowiredBeanName, requestingBeanName);
				}
			}
		}

		return resource;
	}
```

主要过程是：

1. 根据依赖的bean名称判断bean factory中是否存在这个bean，如果不存在，则会走@Autowired查找bean的逻辑。
2. 存在，则直接根据名称获取bean即可。

逻辑比较简单，在@Autowired之上添加了先按照名称查找的逻辑。

# 总结

1. 父类InitDestroyAnnotationBeanPostProcessor实现了postProcessBeforeInitialization方法，完成了@PostConstruct注解方法在bean实例化前的调用。实现了postProcessBeforeDestruction方法完成了@PostConstruct注解方法在bean销毁前的调用。
2. 同@Autowired一样， CommonAnnotationBeanPostProcessor通过实现postProcessProperties方法，利用反射完成属性的注入。
3. @Resource查找候选者过程相比@Autowired多了先通过名称查找的过程，名称没有查找到则会执行@Autowired查找后续者的过程。