title: Spring拓展机制之BeanPostProcessor
author: Silence
tags:
  - Spring
categories:
  - Spring
date: 2020-10-07 13:36:00
---

# BeanPostProcessor的作用

bean的后置处理器，spring ioc容器管理bean最重要的一个拓展机制，开发者通过向BeanFactory中添加自定义的BeanPostProcessor，就能够在bean的生命周期内对bean的实例化、属性注入、初始化等进行自定义修改。

例如：在bean的实例化过程，spring默认会调用bean的无参构造函数通过反射进行实例化，而开发者想自定义选择构造函数，则可以通过实现BeanPostProcessor来完成。

再或者spring的AOP，本质上也是通过BeanPostProcessor在bean实例化好后，对其进行代理增强，得到其代理类


# BeanPostProcessor接口定义


```
public interface BeanPostProcessor {

	/**
	 * Apply this {@code BeanPostProcessor} to the given new bean instance <i>before</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 */
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

	/**
	 * Apply this {@code BeanPostProcessor} to the given new bean instance <i>after</i> any bean
	 * initialization callbacks (like InitializingBean's {@code afterPropertiesSet}
	 * or a custom init-method). The bean will already be populated with property values.
	 * The returned bean instance may be a wrapper around the original.
	 * <p>In case of a FactoryBean, this callback will be invoked for both the FactoryBean
	 * instance and the objects created by the FactoryBean (as of Spring 2.0). The
	 * post-processor can decide whether to apply to either the FactoryBean or created
	 * objects or both through corresponding {@code bean instanceof FactoryBean} checks.
	 * <p>This callback will also be invoked after a short-circuiting triggered by a
	 * {@link InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation} method,
	 * in contrast to all other {@code BeanPostProcessor} callbacks.
	 * <p>The default implementation returns the given {@code bean} as-is.
	 * @param bean the new bean instance
	 * @param beanName the name of the bean
	 * @return the bean instance to use, either the original or a wrapped one;
	 * if {@code null}, no subsequent BeanPostProcessors will be invoked
	 * @throws org.springframework.beans.BeansException in case of errors
	 * @see org.springframework.beans.factory.InitializingBean#afterPropertiesSet
	 * @see org.springframework.beans.factory.FactoryBean
	 */
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}

}
```


## postProcessBeforeInitialization

1. 实现该方法就能拿到实例化的bean。该方法返回结果也是一个bean，可以对传入的bean进行修改后返回
2. 在一些bean的初始化回调前进行调用，例如InitializingBean的afterPropertiesSet方法前或者自定义的的init-method。
2. 该方法拿到的bean是已经经过了属性填充的bean。


## postProcessAfterInitialization

1. 实现该方法就能拿到实例化的bean。该方法返回结果也是一个bean，可以对传入的bean进行修改后返回
2. 在一些bean的初始化回调后进行调用，例如InitializingBean的afterPropertiesSet方法前或者自定义的的init-method。

## 接口方法调用逻辑

调用入口：


```
AbstractAutowireCapableBeanFactory#initializeBean()
```


通过调用入口initializeBean方法，我们知道是在bean初始化阶段会回调这两个接口方法，分别在执行invokeInitMethods方法前后调用。

![image](http://static.silence.work/BeanPostProcessor-1.png)

applyBeanPostProcessorsBeforeInitialization和applyBeanPostProcessorsAfterInitialization执行的逻辑非常简单，就是遍历AbstractBeanFactory#beanPostProcessors保存的BeanPostProcessor进行调用。


```
	/** BeanPostProcessors to apply in createBean. */
	private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();
```


```
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
		    //传入一个bean得到一个bean
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
	
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessAfterInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

# 如何添加BeanPostProcessor

方式一：手动添加，调用如下方法

AbstractBeanFactory#addBeanPostProcessor

```
@Override
	public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
		Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
		// Remove from old position, if any
		this.beanPostProcessors.remove(beanPostProcessor);
		// Track whether it is instantiation/destruction aware
		if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
			this.hasInstantiationAwareBeanPostProcessors = true;
		}
		if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
			this.hasDestructionAwareBeanPostProcessors = true;
		}
		// Add to end of list
		this.beanPostProcessors.add(beanPostProcessor);
	}
```


通过调用该方法将beanPostProcessor保存到集合变量beanPostProcessors中


```

/** BeanPostProcessors to apply in createBean. */
	private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();
	
```


**方式二：按照添加一个普通bean的方式添加，例如通过spring的注解、xml、BeanDefinitionRegistry.registerBeanDefinition等**



# Spring提供的BeanPostProcessor直接实现类

spring本身提供直接实现BeanPostProcessor接口很少，我们看个比较重要的实现类：ApplicationContextAwareProcessor。

## ApplicationContextAwareProcessor

实现了postProcessBeforeInitialization方法，在初始化前，判断bean是否是ApplicationContext相关的Aware接口实现，如果是，则执行拓展逻辑回调相关Aware方法。


```
class ApplicationContextAwareProcessor implements BeanPostProcessor {
    //。。

	@Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}

	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof EnvironmentAware) {
			((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
		}
		if (bean instanceof EmbeddedValueResolverAware) {
			((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
		}
		if (bean instanceof ResourceLoaderAware) {
			((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
		}
		if (bean instanceof ApplicationEventPublisherAware) {
			((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
		}
		if (bean instanceof MessageSourceAware) {
			((MessageSourceAware) bean).setMessageSource(this.applicationContext);
		}
		if (bean instanceof ApplicationContextAware) {
			((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
		}
	}

}

```

看到这里似乎我们似乎没有看到BeanPostProcessor有多重要，直接实现BeanPostProcessor仅仅只能在bean初始化前后进行一些逻辑处理，那如何在bean实例化过程进行自定义呢？

答案在BeanPostProcessor的子类接口们，往下看

# BeanPostProcessor的子类接口

BeanPostProcessor提供了很多非常重要的子类接口，对BeanPostProcessor只能在bean初始化阶段拓展的不足做了增强。


## InstantiationAwareBeanPostProcessor

BeanPostProcessor的子接口，添加了在bean实例化前后的回调方法，同时还提供了拿到bean的属性回调方法。

![image](http://static.silence.work/BeanPostProcessor-2.png)

### postProcessBeforeInstantiation

```
default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		return null;
}
```
通过实现该接口方法能够直接在这里自定义创建bean的实例，从而跳过spring内部实例化的过程。

#### 调用入口

- AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation


```
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// 判断是否存在InstantiationAwareBeanPostProcessor，存在则applyBeanPostProcessorsBeforeInstantiation
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					//如果拿到的bean不为null，代表已经实例化了，直接这里执行bean的初始化过程
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
	
```

- AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation

遍历AbstractBeanFactory#beanPostProcessors，如果是InstantiationAwareBeanPostProcessor，则执行postProcessBeforeInstantiation方法

```
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}
```


### postProcessAfterInstantiation


```
default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {
		return true;
	}
```

bean实例化紧接着对bean属性赋值过程会调用，通过实现postProcessAfterInstantiation并返回false时，则可以自定义跳过Spring内部属性的赋值操作。

#### 调用入口

- AbstractAutowireCapableBeanFactory#populateBean

populateBean方法是bean实例化后对bean的属性赋值，在该方法内会遍历调用postProcessAfterInstantiation方法，如果返回fase则直接return，阻止后续的操作

![image](http://static.silence.work/BeanPostProcessor-3.png)


### postProcessProperties


```
	default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName)
			throws BeansException {

		return null;
	}
```

bean属性赋值前，回调postProcessProperties，传递PropertyValues对象，PropertyValues中保存了bean实例对象中所有属性值的设置，通过修改PropertyValues可以修改属性值。

#### 调用入口

- AbstractAutowireCapableBeanFactory#populateBean

同postProcessAfterInstantiation一样，也是在属性赋值方法中进行回调。

![image](http://static.silence.work/BeanPostProcessor-4.png)

### 重要实现类

#### AutowiredAnnotationBeanPostProcessor

自动注入非常重要的一个后置器，实现的postProcessProperties会对@Autowired、@Value标注的字段、方法注入值。

具体分析见我这篇文章：[Spring Bean后置处理器之AutowiredAnnotationBeanPostProcessor](http://silence.work/2020/10/31/Spring%E7%9A%84Bean%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8%E4%B9%8BAutowiredAnnotationBeanPostProcessor/)

#### CommonAnnotationBeanPostProcessor

对@Resource标注的字段和方法注入值。

具体分析见我这篇文章：[Spring Bean后置处理器之CommonAnnotationBeanPostProcessor](http://silence.work/2020/10/31/Spring-Bean%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8%E4%B9%8BCommonAnnotationBeanPostProcessor/)

## SmartInstantiationAwareBeanPostProcessor

InstantiationAwareBeanPostProcessor的子接口。增加了3个方法。



### predictBeanType

预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null

```
default Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException {
		return null;
}
```


### determineCandidateConstructors


```
	default Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName)
			throws BeansException {

		return null;
	}
```

bean实例化前，通过回调该方法，从而可以实现自定义选择构造器。

#### 调用入口

- AbstractAutowireCapableBeanFactory#determineConstructorsFromBeanPostProcessors

遍历SmartInstantiationAwareBeanPostProcessor，调用determineCandidateConstructors返回构造函数

```
    /**/
	@Nullable
	protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
			throws BeansException {

		if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
					if (ctors != null) {
						return ctors;
					}
				}
			}
		}
		return null;
	}
```



### getEarlyBeanReference

获得提前暴露的bean引用。主要用于解决循环引用的问题。


### 重要实现类

#### AnnotationAwareAspectJAutoProxyCreator

@EnableAspectJAutoProxy会在spring容器中注册一个bean，AOP的实现，对符合条件的bean，自动生成代理对象。

具体分析见我这篇文章[：Spring Bean后置处理器之AnnotationAwareAspectJAutoProxyCreator
](http://silence.work/2020/10/31/Spring-Bean%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8%E4%B9%8BAnnotationAwareAspectJAutoProxyCreator/)

## MergedBeanDefinitionPostProcessor

BeanPostProcessor的子接口，增加了postProcessMergedBeanDefinition方法。

用于处理合并后的BeanDefinition，在bean实例化后、属性赋值前回调。


```
	void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
```


### 调用入口



```
AbstractAutowireCapableBeanFactory#doCreateBean
    ->AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors
```


```
	protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
```

### 重要实现类

#### AutowiredAnnotationBeanPostProcessor

在 postProcessMergedBeanDefinition 方法中对 @Autowired、@Value 标注的方法、字段进行缓存，为后续属性赋值阶段的AutowiredAnnotationBeanPostProcessor处理做准备


#### CommonAnnotationBeanPostProcessor

在 postProcessMergedBeanDefinition 方法中对 @Resource 标注的字段、@Resource 标注的方法、 @PostConstruct 标注的字段、 @PreDestroy标注的方法进行缓存


## DestructionAwareBeanPostProcessor

BeanPostProcessor的子接口，增加了postProcessBeforeDestruction，用于bean销毁前的回调。

### 重要实现类

#### CommonAnnotationBeanPostProcessor

CommonAnnotationBeanPostProcessor#postProcessBeforeDestruction方法中会调用bean中所有标注了@PreDestroy的方法