title: Spring拓展机制之BeanFactoryPostProcessor
author: Silence
tags:
  - Spring
categories:
  - Spring
date: 2020-10-07 13:42:00
---


# BeanFactoryPostProcessor的作用

熟知BeanPostProcessor的知道，BeanPostProcessor是bean的后置处理器，用于在bean的生命周期来拓展对bean进行自定义处理。按照BeanPostProcessor的理解，BeanFactoryPostProcessor则是BeanFactory的后置处理器，用于对BeanFactory进行自定义处理，例如自定义注册bean，以及修改BeanFactory中注册的bean定义。

官方的解释是：

> BeanFactoryPostProcessor operates on the bean configuration metadata. That is, the Spring IoC container lets a BeanFactoryPostProcessor read the configuration metadata and potentially change it before the container instantiates any beans other than BeanFactoryPostProcessor instances.


# 接口定义


```
@FunctionalInterface
public interface BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean factory after its standard
	 * initialization. All bean definitions will have been loaded, but no beans
	 * will have been instantiated yet. This allows for overriding or adding
	 * properties even to eager-initializing beans.
	 * @param beanFactory the bean factory used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;

}
```
# 调用入口

入口：AbstractApplicationContext#invokeBeanFactoryPostProcessors

而AbstractApplicationContext#invokeBeanFactoryPostProcessors是ApplicationContext执行refresh过程中调用的（spring容器启动过程中非常重要的一个过程）。

## 执行逻辑

执行过程非常简单，就是遍历AbstractApplicationContext#beanFactoryPostProcessors中的BeanFactoryPostProcessor依次执行postProcessBeanFactory方法，传递BeanFactory对象

```
AbstractApplicationContext#invokeBeanFactoryPostProcessors
    -->PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
        -->PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors

	private static void invokeBeanFactoryPostProcessors(
			Collection<? extends BeanFactoryPostProcessor> postProcessors, ConfigurableListableBeanFactory beanFactory) {

		for (BeanFactoryPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessBeanFactory(beanFactory);
		}
	}
```

# 如何添加BeanFactoryPostProcessor

**方式一：手动添加，调用如下方法**

```
AbstractApplicationContext#addBeanFactoryPostProcessor
```

即向beanFactoryPostProcessors变量中添加指定的BeanFactoryPostProcessor

```
	@Override
	public void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor) {
		Assert.notNull(postProcessor, "BeanFactoryPostProcessor must not be null");
		this.beanFactoryPostProcessors.add(postProcessor);
	}
```

**方式二：按照添加一个普通bean的方式添加，例如通过spring的注解、xml、BeanDefinitionRegistry.registerBeanDefinition等**





# 几个重要的实现类

**PropertySourcesPlaceholderConfigurer：**

用于处理属性中定义的${xxx}占位符，对这种格式的进行解析处理为真正的值。

**EventListenerMethodProcessor：**

处理@EventListener注解，即Spring的事件机制。



# 子接口BeanDefinitionRegistryPostProcessor


```
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * Modify the application context's internal bean definition registry after its
	 * standard initialization. All regular bean definitions will have been loaded,
	 * but no beans will have been instantiated yet. This allows for adding further
	 * bean definitions before the next post-processing phase kicks in.
	 * @param registry the bean definition registry used by the application context
	 * @throws org.springframework.beans.BeansException in case of errors
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```


BeanDefinitionRegistryPostProcessor在BeanFactoryPostProcessor基础上增加了postProcessBeanDefinitionRegistry方法，用于回调传递BeanDefinitionRegistry对象，BeanDefinitionRegistry是注册bean的重要类

实现该方法可以向容器中注册bean。


## 重要实现类


**ConfigurationClassPostProcessor**：此类非常重要，执行优先级最高，用于处理@ComponentScan、@Configuration、@Import等注解指定的类，从而批量注册bean，此类后面单独分析。


具体分析见我这篇文章：[Spring BeanFactory后置处理器之ConfigurationClassPostProcessor](http://silence.work/2020/10/31/Spring-BeanFactory%E5%90%8E%E7%BD%AE%E5%A4%84%E7%90%86%E5%99%A8%E4%B9%8BConfigurationClassPostProcessor/)

## 调用逻辑

调用入口同BeanFactoryPostProcessor一样都是在ApplicationContext执行refresh过程中的invokeBeanFactoryPostProcessors调用的。

调用过程在PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors方法中。

当初在多个BeanFactoryPostProcessor以及BeanDefinitionRegistryPostProcessor时，Spring时如何处理调用顺序的呢？

# BeanFactoryPostProcessor的执行顺序

BeanFactoryPostProcessor执行的顺序逻辑都在
PostProcessorRegistrationDelegate的invokeBeanFactoryPostProcessors()方法中。


在保证先执行BeanDefinitionRegistryPostProcessor再执行BeanFactoryPostProcessor的原则上，再通过继承PriorityOrdered、Ordered可以自定义控制BeanFactory中注册的所有BeanFactoryPostProcessor的先后顺序，

## 源码

```
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();

		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
			    //1. 先调用通过AbstractApplicationContext#addBeanFactoryPostProcessor手动添加的BeanDefinitionRegistryPostProcessor
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// 2. 从beanFactory中获取自定义的BeanDefinitionRegistryPostProcessor，同时只执行实现了PriorityOrdered的BeanDefinitionRegistryPostProcessor BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// 3. 继续从BeanFactory中获取BeanDefinitionRegistryPostProcessor，根据实现的Order排序，按顺序执行实现了Order的BeanDefinitionRegistryPostProcessor
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// 4. 继续从beanFactory中获取自定义的BeanDefinitionRegistryPostProcessor，最终执行剩余所有的BeanDefinitionRegistryPostProcessor
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// 5. 执行自定义BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法，也就是父接口的方法
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			//6. 执行手动添加的BeanFactoryPostProcessor#postProcessBeanFactory方法
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// 7. 按顺序执行实现了PriorityOrdered的BeanFactoryPostProcessor
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);


        //// 8. 按顺序执行实现了Ordered的BeanFactoryPostProcessor
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// 9. 执行剩余其他的BeanFactoryPostProcessor
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
```

## 执行过程

**step 1：** 调用通过AbstractApplicationContext#addBeanFactoryPostProcessor手动添加的BeanDefinitionRegistryPostProcessor


**step 2：** 从beanFactory中获取实现了PriorityOrdered的BeanDefinitionRegistryPostProcessor，然后按顺序依次执行


**step 3：**
继续从beanFactory中获取实现了Ordered的BeanDefinitionRegistryPostProcessor，然后按顺序依次执行

**step 4：**
继续从beanFactory中获取自定义的BeanDefinitionRegistryPostProcessor，最终执行剩余所有的BeanDefinitionRegistryPostProcessor


**step 5：**
按顺序执行所有BeanDefinitionRegistryPostProcessor的postProcessBeanFactory方法，也就是父接口BeanFactoryPostProcessor的方法



**step 6：**
执行手动添加的BeanFactoryPostProcessor#postProcessBeanFactory方法

**step 7：**
按顺序执行实现了PriorityOrdered的BeanFactoryPostProcessor

**step 8：**
按顺序执行实现了Ordered的BeanFactoryPostProcessor

**step 9：**
执行剩余其他的BeanFactoryPostProcessor


***特别说明***：对于顺序执行BeanDefinitionRegistryPostProcessor的几步顺序过程，每次都会重新从BeanFactory去获取，而BeanFactoryPostProcessor获取一次就不会再获取了，原因就是因为BeanDefinitionRegistryPostProcessor会向BeanFactory中注册bean，每执行一个BeanDefinitionRegistryPostProcessor后，BeanFactory中注册的bean都发生了变化，因此要重新获取，例如ConfigurationClassPostProcessor继承了PriorityOrdered，该类实现的postProcessBeanDefinitionRegistry方法会批量注册通过注解定义的bean，这里面同时也会添加很多BeanFactoryPostProcessor。









