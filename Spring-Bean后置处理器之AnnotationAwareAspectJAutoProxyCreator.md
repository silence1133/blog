title: Spring Bean后置处理器之AnnotationAwareAspectJAutoProxyCreator
author: Silence
tags:
  - Spring
categories:
  - Spring
date: 2020-10-31 19:03:00
---
[TOC]

# 本文能帮你解答的问题

1. AnnotationAwareAspectJAutoProxyCreator后置器的作用是什么?
2. Spring AOP自动增强bean是如何实现的。
2. 如何在spring上下文添加AnnotationAwareAspectJAutoProxyCreator？
3. 如何利用ProxyFactory硬编码实现一个bean的增强？
4. AnnotationAwareAspectJAutoProxyCreator是在bean的生命周期什么阶段完成bean的增强？


# 类介绍

## 作用

AnnotationAwareAspectJAutoProxyCreator也是一个bean的后置处理器，是Spring AOP完成bean自动增强的关键类，用于在bean创建过程中扫描@Aspect注解的bean，创建Advisor，从而实现对bean的代理增强。

## 类继承关系

![image](http://static.silence.work/AnnotationAwareAspectJAutoProxyCreator.png)


核心逻辑主要在AbstractAutoProxyCreator，bean后置器的相关方法都是在该抽象类实现的。

AbstractAutoProxyCreator继承的类主要是后置处理器接口SmartInstantiationAwareBeanPostProcessor以及支持代理的通用类ProxyProcessorSupport。

实现的后置器方法主要有predictBeanType、getEarlyBeanReference、postProcessBeforeInstantiation、postProcessAfterInitialization。

## 如何添加AnnotationAwareAspectJAutoProxyCreator

通过添加注解@EnableAspectJAutoProxy，Spring容器启动时就会自动扫描到该注解，从而添加AnnotationAwareAspectJAutoProxyCreator这个BeanPostProcessor。

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

	/**
	 * 是否基于类来创建代理，而不是基于接口来创建代理，
	 当为true的时候会使用cglib来直接对目标类创建代理对象（cglib是基于类创建代理）
	 默认为false，意味着当目标bean有接口，则用java动态代理创建，否则用cglib
	 */
	boolean proxyTargetClass() default false;

	/**
     * 是否需要将代理对象暴露在ThreadLocal中，当为true的时候
     * 可以通过org.springframework.aop.framework.AopContext#currentProxy获取当前代理对象
	 */
	boolean exposeProxy() default false;

}
```

而@EnableAspectJAutoProxy注解上通过Import了AspectJAutoProxyRegistrar类，AspectJAutoProxyRegistrar类会调用AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry)，向BeanDefinitionRegistry中添加该BeanPostProcessor。


# AOP相关知识点

在分析AnnotationAwareAspectJAutoProxyCreator的具体实现之前，我们先准备相关AOP的知识点。

## Spring中AOP的几个概念

### JoinPoint
连接点，连接点就是被拦截到的程序执行点，因为Spring只支持方法类型的连接点，所以在Spring中连接点就是被拦截到的方法。

### Advice

通知，需要在目标对象中增强的功能，通知中有2个重要的信息：方法的什么地方，执行什么操作，这2个信息通过通知来指定。

在spring中接口定义是Advice，同时有不同子类用于在方法不同阶段做功能增强，对应注解通过@Before、@Around、@After、@AfterReturning、@AfterThrowing表示。

### Pointcut

切入点，用来指定需要将通知使用到哪些地方，比如需要用在哪些类的哪些方法上，切入点就是做这个配置的，在Spring中对应接口Pointcut，对应注解@Pointcut

### Advisor

顾问，Pointcut与Advice组合在一起叫Advisor，代表在什么地方（Pointcut）添加什么样的增强功能（Advice），对应接口为Advisor，对应注解为@Aspect。

AOP相关参考:http://www.itsoku.com/article/296

## 硬编码AOP实例
示例：

在DemoService调用say方法前后增加开始和结束的输出逻辑，同时添加say方法调用的耗时。

代码说明：

通过定义好pointcut和Advice，然后组合得到Advisor，利用ProxyFactory，得到增强后的代理类。


```
   static class DemoService{
        public void say(){
            System.out.println("hello word");
        }
    }

    @Test
    public void  test4(){
        DemoService target = new DemoService();
        //定义pointcut
        Pointcut pointcut = new Pointcut() {

            //增强类的匹配规则
            @Override
            public ClassFilter getClassFilter() {
                return clazz -> DemoService.class.isAssignableFrom(clazz);
            }

            //增强方法的匹配规则
            @Override
            public MethodMatcher getMethodMatcher() {
                return new MethodMatcher() {
                    //匹配方法名称
                    @Override
                    public boolean matches(Method method, Class<?> targetClass) {
                        return "say".equals(method.getName());
                    }

                    @Override
                    public boolean isRuntime() {
                        return false;
                    }

                    @Override
                    public boolean matches(Method method, Class<?> targetClass, Object... args) {
                        return false;
                    }
                };
            }
        };
        //定义Advice，MethodInterceptor是Advice的子类，在方法前后增强
        MethodInterceptor advice = invocation -> {
            System.out.println("start");
            long starTime = System.nanoTime();
            //连接点执行
            Object result = invocation.proceed();
            long endTime = System.nanoTime();
            System.out.println("end！");
            System.out.println("耗时:" + (endTime - starTime));
            return result;
        };

        //创建Advisor，将pointcut和advice组装起来
        DefaultPointcutAdvisor advisor = new DefaultPointcutAdvisor(pointcut, advice);

        //通过spring提供的代理创建工厂来创建代理
        ProxyFactory proxyFactory = new ProxyFactory();
        //为工厂指定目标对象
        proxyFactory.setTarget(target);
        //调用addAdvisor方法，为目标添加增强的功能，即添加Advisor，可以为目标添加很多个Advisor
        proxyFactory.addAdvisor(advisor);
        DemoService demoService = (DemoService) proxyFactory.getProxy();
        demoService.say();
    }
```

输出：

```
start
hello word
end！
耗时:13044107
```



# 实现方法分析

## predictBeanType

实例化前预测bean的类型，这里会从创建的代理缓存中获取bean，如果存在则返回代理的对象类型。

```
	public Class<?> predictBeanType(Class<?> beanClass, String beanName) {
		if (this.proxyTypes.isEmpty()) {
			return null;
		}
		Object cacheKey = getCacheKey(beanClass, beanName);
		return this.proxyTypes.get(cacheKey);
	}
```
## getEarlyBeanReference

用于处理循环引用，这里不分析

## postProcessBeforeInstantiation

bean实例化前调用，主要是判断是否是要代理增强的bean，并缓存起来

```
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);
        
		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			//如果是基础类（是指是否属于Advisor、Pointcut AOP等提供的基础类）
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				//是，会把基础bean缓存起来，value为false（代表不是要增强的bean）
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

	   //省略部分代码
	   
		return null;
	}

```

### 判断是否是需要增强的bean

不是，会缓存到map，value记录为false

```
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

#### isInfrastructureClass

AnnotationAwareAspectJAutoProxyCreator覆写了该方法，增加了注解的判断逻辑，所有的判断逻辑为：


```
Advice.class.isAssignableFrom(beanClass) ||
Pointcut.class.isAssignableFrom(beanClass) ||
Advisor.class.isAssignableFrom(beanClass) ||
AopInfrastructureBean.class.isAssignableFrom(beanClass) ||
(AnnotationUtils.findAnnotation(clazz, Aspect.class) != null && !compiledByAjc(clazz))
```

即是不是AOP的基础类，包括是否是基础类的实现或者是否是添加了@Aspect注解。

#### shouldSkip

AspectJAwareAdvisorAutoProxyCreator覆写了该方法，添加了findCandidateAdvisors逻辑，主要逻辑也在findCandidateAdvisors方法。

```
	@Override
	protected boolean shouldSkip(Class<?> beanClass, String beanName) {
		// TODO: Consider optimization by caching the list of the aspect names
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		for (Advisor advisor : candidateAdvisors) {
			if (advisor instanceof AspectJPointcutAdvisor &&
					((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
				return true;
			}
		}
		return super.shouldSkip(beanClass, beanName);
	}
```

findCandidateAdvisors主要做的事情是查找所有的Advisor，并缓存起来。

```

AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

	protected List<Advisor> findCandidateAdvisors() {
		// Add all the Spring advisors found according to superclass rules.
		//查找继承了Advisor的类
		List<Advisor> advisors = super.findCandidateAdvisors();
		// Build Advisors for all AspectJ aspects in the bean factory.
		if (this.aspectJAdvisorsBuilder != null) {
		    //查找标注了@Aspect注解的类
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
		return advisors;
	}

```

## postProcessAfterInitialization

bean初始化后调用，调用wrapIfNecessary来对bean进行代理增强，返回出去，这样拿到的bean实际是经过代理增强的bean。

```
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (this.earlyProxyReferences.remove(cacheKey) != bean) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}
```

AbstractAutoProxyCreator#wrapIfNecessary：

```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}
		
		//前面判断是否是需要代理增强的类，是才会走到下面的逻辑
		// Create proxy if we have advice.
		//获取当前bean的advisors，也就是要针对当前bean增强的逻辑
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
		    //将当前bean标示为要代理增强的bean
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			//创建代理
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

主要做了3件事：

**1. 判断是否是需要增强的类。**


**2. 是，则取得当前bean对应的advisor数组**


```
AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean
    -->AbstractAdvisorAutoProxyCreator#findEligibleAdvisors
    
    protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
	
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
	    //获取所有的Advisor
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		//获取与当前bean有关的Advisor
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		extendAdvisors(eligibleAdvisors);
		if (!eligibleAdvisors.isEmpty()) {
		    //按照规则对多个Advisor进行排序
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}
```


**3. 根据拿到的advisor数组给当前bean创建代理。**

AbstractAutoProxyCreator#createProxy

```
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
        //定义创建代理的ProxyFactory
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
        //准备好Advisor
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		//添加Advisors
		proxyFactory.addAdvisors(advisors);
		//设置目标对象
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);
        
		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
        //创建代理类
		return proxyFactory.getProxy(getProxyClassLoader());
	}
```
析到这里，我们从beanFactory中获取的bean就是经过AOP增强的bean了。

# 总结

1. AnnotationAwareAspectJAutoProxyCreator实现了两个关键类SmartInstantiationAwareBeanPostProcessor和ProxyProcessorSupport。
2. 通过实现postProcessBeforeInstantiation，在bean实例化前扫描@Aspect注解，解析到需要增强的Advisor，并缓存起来。
3. 通过实现postProcessAfterInitialization，在bean初始化后，利用缓存的Advisor，创建代理类，完成bean的增强。