title: Spring Bean后置处理器之AutowiredAnnotationBeanPostProcessor
author: Silence
tags:
  - Spring
categories:
  - Spring
date: 2020-10-31 19:01:00
---
# 本文能帮你解答的问题

1. @Autowired注解是如何完成属性自动注入的？
2. 如何改变bean实例化构造函数的选择？
2. @Autowired查找候选者过程是什么？
3. @Qualifier与@Primary的作用？

# 类介绍

## 作用

1. 选择标注了@Autowired的构造函数作为实例化用到的构造函数。
2. 处理@Autowired和@Value注解的属性字段，注入属性值
3. 处理@Autowired和@Value注解的方法，注入对应的值。
4. 处理@Lookup注解的方法。

## 类结构

![image](http://static.silence.work/AutowiredAnnotationBeanPostProcessor.png)

从类图可以看出：

1. AutowiredAnnotationBeanPostProcessor主要继承了InstantiationAwareBeanPostProcessor、SmartInstantiationAwareBeanPostProcessor和MergedBeanDefinitionPostProcessor。
2. 主要实现了determineCandidateConstructors、postProcessMergedBeanDefinition、postProcessPropertyValues方法

### 继承父类说明

#### InstantiationAwareBeanPostProcessor

提供了bean实例化前后对bean的操作以及在属性赋值前，传递属性值的postProcessProperties方法。

#### SmartInstantiationAwareBeanPostProcessor 

主要提供了实例化前自定义选择构造函数的方法。

#### MergedBeanDefinitionPostProcessor

用于处理合并后的BeanDefinition，在bean实例化后、属性赋值前回调。

#### BeanFactoryAware

回调提供BeanFactory对象，在属性赋值后、bean初始化前回调。

### 构造函数

```
	public AutowiredAnnotationBeanPostProcessor() {
		this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```
通过构造函数可以大概猜测该后置处理器的作用。

# 实现方法分析

针对AutowiredAnnotationBeanPostProcessor主要实现了determineCandidateConstructors、postProcessMergedBeanDefinition、postProcessProperties，按照执行的先后顺序依次分析。

## determineCandidateConstructors

**调用时机**：bean实例化前调用。

实现的逻辑主要分两部分，一部分是处理@Lookup注解，一部分是处理构造函数上的@Autowired注解。

### 处理@Lookup注解方法

@Lookup的作用是在方法上的注解，被其标注的方法会被重写，然后根据其返回值的类型，容器调用BeanFactory的getBean()方法来返回一个bean。

使用场景：当一个单例的bean依赖的bean是多例的时候，如果需要单例的bean每次获取依赖的bean都是不同实例，则可以通过@Lookup实现。

@Lookup的主要作用就是在这里实现的。

```
	if (!this.lookupMethodsChecked.contains(beanName)) {
			if (AnnotationUtils.isCandidateClass(beanClass, Lookup.class)) {
				try {
					Class<?> targetClass = beanClass;
					do {
						ReflectionUtils.doWithLocalMethods(targetClass, method -> {
						    //查找方法上的Lookup注解
							Lookup lookup = method.getAnnotation(Lookup.class);
							if (lookup != null) {
								Assert.state(this.beanFactory != null, "No BeanFactory available");
								LookupOverride override = new LookupOverride(method, lookup.value());
								try {
									RootBeanDefinition mbd = (RootBeanDefinition)
											this.beanFactory.getMergedBeanDefinition(beanName);
									mbd.getMethodOverrides().addOverride(override);
								}
								catch (NoSuchBeanDefinitionException ex) {
									throw new BeanCreationException(beanName,
											"Cannot apply @Lookup to beans without corresponding bean definition");
								}
							}
						});
						targetClass = targetClass.getSuperclass();
					}
					while (targetClass != null && targetClass != Object.class);

				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName, "Lookup method resolution failed", ex);
				}
			}
			this.lookupMethodsChecked.add(beanName);
		}
```


### 选择@Autowired的构造函数

当@Autowired标注在某个构造函数的时候，则bean的实例化会使用当前构造函数，同时注入构造函数指定的依赖。

例如下面的例子：

```
@Component
public class Service2 {
    private Service1 service1;

    public Service2() {
        System.out.println(this.getClass() + "无参构造器");
    }

    @Autowired
    public Service2(Service1 service1) {
        System.out.println(this.getClass() + "有参构造器");
        this.service1 = service1;
    }
}
```

源码：

```
		Constructor<?>[] candidateConstructors = this.candidateConstructorsCache.get(beanClass);
		if (candidateConstructors == null) {
			synchronized (this.candidateConstructorsCache) {
				candidateConstructors = this.candidateConstructorsCache.get(beanClass);
				if (candidateConstructors == null) {
					Constructor<?>[] rawCandidates;
					//获取声明的构造函数
					rawCandidates = beanClass.getDeclaredConstructors();
					List<Constructor<?>> candidates = new ArrayList<>(rawCandidates.length);
					Constructor<?> requiredConstructor = null;
					Constructor<?> defaultConstructor = null;
					Constructor<?> primaryConstructor = BeanUtils.findPrimaryConstructor(beanClass);
					int nonSyntheticConstructors = 0;
					for (Constructor<?> candidate : rawCandidates) {
						if (!candidate.isSynthetic()) {
							nonSyntheticConstructors++;
						}
						else if (primaryConstructor != null) {
							continue;
						}
						//查找@Autowired注解
						MergedAnnotation<?> ann = findAutowiredAnnotation(candidate);
                        //。。。
                        //如果存在
						if (ann != null) {
							//required值是否是true
							boolean required = determineRequiredStatus(ann);
							if (required) {
								if (!candidates.isEmpty()) {
									throw new BeanCreationException(beanName,
											"Invalid autowire-marked constructors: " + candidates +
											". Found constructor with 'required' Autowired annotation: " +
											candidate);
								}
								requiredConstructor = candidate;
							}
							candidates.add(candidate);
						}
						else if (candidate.getParameterCount() == 0) {
							defaultConstructor = candidate;
						}
					}
					if (!candidates.isEmpty()) {
						// Add default constructor to list of optional constructors, as fallback.
						if (requiredConstructor == null) {
							if (defaultConstructor != null) {
								candidates.add(defaultConstructor);
							}
							else if (candidates.size() == 1 && logger.isInfoEnabled()) {
								logger.info("Inconsistent constructor declaration on bean with name '" + beanName +
										"': single autowire-marked constructor flagged as optional - " +
										"this constructor is effectively required since there is no " +
										"default constructor to fall back to: " + candidates.get(0));
							}
						}
						candidateConstructors = candidates.toArray(new Constructor<?>[0]);
					}
					else if (rawCandidates.length == 1 && rawCandidates[0].getParameterCount() > 0) {
						candidateConstructors = new Constructor<?>[] {rawCandidates[0]};
					}
					else if (nonSyntheticConstructors == 2 && primaryConstructor != null &&
							defaultConstructor != null && !primaryConstructor.equals(defaultConstructor)) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor, defaultConstructor};
					}
					else if (nonSyntheticConstructors == 1 && primaryConstructor != null) {
						candidateConstructors = new Constructor<?>[] {primaryConstructor};
					}
					else {
						candidateConstructors = new Constructor<?>[0];
					}
				    //将选中的构造函数缓存起来
				    this.candidateConstructorsCache.put(beanClass, candidateConstructors);
				}
			}
		}
		return (candidateConstructors.length > 0 ? candidateConstructors : null);
	}
```

## postProcessMergedBeanDefinition

**调用时机**：在bean实例化后、属性赋值前回调。

**作用**：查找bean对应类中定义的@Autowired注解和@Value注解，并缓存起来

```
	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
	
```
findAutowiringMetadata方法内会调用buildAutowiringMetadata方法，将得到的InjectionMetadata放到injectionMetadataCache缓存起来，buildAutowiringMetadata主要分方法和字段上的@Autowired和@Value解析。


```
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {

		List<InjectionMetadata.InjectedElement> elements = new ArrayList<>();
		Class<?> targetClass = clazz;

		do {
			final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();
            //处理字段上的Value和Autowired注解
			ReflectionUtils.doWithLocalFields(targetClass, field -> {
				MergedAnnotation<?> ann = findAutowiredAnnotation(field);
				if (ann != null) {
					if (Modifier.isStatic(field.getModifiers())) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation is not supported on static fields: " + field);
						}
						return;
					}
					boolean required = determineRequiredStatus(ann);
					currElements.add(new AutowiredFieldElement(field, required));
				}
			});
            /处理方法上的Autowired和Value注解 
			ReflectionUtils.doWithLocalMethods(targetClass, method -> {
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
				if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
					return;
				}
				MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
				if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
					if (Modifier.isStatic(method.getModifiers())) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation is not supported on static methods: " + method);
						}
						return;
					}
					if (method.getParameterCount() == 0) {
						if (logger.isInfoEnabled()) {
							logger.info("Autowired annotation should only be used on methods with parameters: " +
									method);
						}
					}
					boolean required = determineRequiredStatus(ann);
					PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
					currElements.add(new AutowiredMethodElement(method, required, pd));
				}
			});

			elements.addAll(0, currElements);
			targetClass = targetClass.getSuperclass();
		}
		while (targetClass != null && targetClass != Object.class);

		return InjectionMetadata.forElements(elements, clazz);
	}
```

## 	postProcessProperties

**调用时机**：属性赋值前回调。

从缓存中获取上一步缓存的InjectionMetadata调用inject方法注入属性值。

```
	public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
	
	    //从缓存中获取上一步缓存的InjectionMetadata
		InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
		try {
		    //注入
			metadata.inject(bean, beanName, pvs);
		}
		catch (BeanCreationException ex) {
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "Injection of autowired dependencies failed", ex);
		}
		return pvs;
	}

```
InjectionMetadata封装了标注了Autowired和Value注解的字段或者方法，通过InjectedElement表示。

分别由AutowiredFieldElement和AutowiredMethodElement表示，都是AutowiredAnnotationBeanPostProcessor内部类


```
//代表标注了注解的字段
private class AutowiredFieldElement extends InjectionMetadata.InjectedElement {
    //注入逻辑所在的方法
    protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
        //...
    }
}

//代表标注了注解的方法
private class AutowiredMethodElement extends InjectionMetadata.InjectedElement {
    //注入逻辑所在的方法
    protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
        //...
    }
}
```

这里分析下字段的注入逻辑。

- 入口：AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject：

```
		protected void inject(Object bean, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
			Field field = (Field) this.member;
			Object value;
			if (this.cached) {
				value = resolvedCachedArgument(beanName, this.cachedFieldValue);
			}
			else {
			    //封装依赖字段
				DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
				desc.setContainingClass(bean.getClass());
				Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
				Assert.state(beanFactory != null, "No BeanFactory available");
				TypeConverter typeConverter = beanFactory.getTypeConverter();
				try {
				    //获取依赖字段对应的value，这里面逻辑较深，主要涉及很多选择的先后顺序逻辑
					value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
				}
				catch (BeansException ex) {
					throw new UnsatisfiedDependencyException(null, beanName, new InjectionPoint(field), ex);
				}
				synchronized (this) {
					if (!this.cached) {
						if (value != null || this.required) {
							this.cachedFieldValue = desc;
							registerDependentBeans(beanName, autowiredBeanNames);
							if (autowiredBeanNames.size() == 1) {
								String autowiredBeanName = autowiredBeanNames.iterator().next();
								if (beanFactory.containsBean(autowiredBeanName) &&
										beanFactory.isTypeMatch(autowiredBeanName, field.getType())) {
									this.cachedFieldValue = new ShortcutDependencyDescriptor(
											desc, autowiredBeanName, field.getType());
								}
							}
						}
						else {
							this.cachedFieldValue = null;
						}
						this.cached = true;
					}
				}
			}
			if (value != null) {
			 //反射调用字段的set方法，注入value到bean中
			 ReflectionUtils.makeAccessible(field);
				field.set(bean, value);
			}
		}
```

通过查找获取依赖字段对应的value，然后利用反射将value set到对应bean的属性字段中。

如何获取属性的值逻辑主要在DefaultListableBeanFactory#resolveDependency方法中，这里面涉及到@Autowired查找候选者的具体过程。

```
value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
```

# @Autowired查找候选者过程


查找的大致处理逻辑为：

```
按类型找->通过限定符@Qualifier过滤->@Primary->@Priority->根据名称找（字段名称或者方法名称）
```

## 主干源码

- 代码入口

```
DefaultListableBeanFactory#resolveDependency
    ->DefaultListableBeanFactory#doResolveDependency
```


```
DefaultListableBeanFactory#doResolveDependency

	public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

		InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
		try {
			Object shortcut = descriptor.resolveShortcut(this);
			if (shortcut != null) {
				return shortcut;
			}
            
			Class<?> type = descriptor.getDependencyType();
			//处理@Value注解
			Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
			if (value != null) {
			    //。。。
			}

            //处理需要注入多个bean的属性值类型，例如List、Map、Stream
			Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
			if (multipleBeans != null) {
				return multipleBeans;
			}
            //根据type获取需要注入的bean，key为bean的名称。
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			if (matchingBeans.isEmpty()) {
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				return null;
			}

			String autowiredBeanName;
			Object instanceCandidate;
            //如果匹配到的注入的bean存在多个
			if (matchingBeans.size() > 1) {
			    //选择一个满足条件的beaNname，这里主要处理@Primary和@Priority注解
				autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
				if (autowiredBeanName == null) {
					if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
						return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
					}
					else {
						// In case of an optional Collection/Map, silently ignore a non-unique case:
						// possibly it was meant to be an empty collection of multiple regular beans
						// (before 4.3 in particular when we didn't even look for collection beans).
						return null;
					}
				}
				//根据选中的beanName获取bean
				instanceCandidate = matchingBeans.get(autowiredBeanName);
			}
			else {
				// We have exactly one match.
				Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
				autowiredBeanName = entry.getKey();
				instanceCandidate = entry.getValue();
			}

			if (autowiredBeanNames != null) {
				autowiredBeanNames.add(autowiredBeanName);
			}
			if (instanceCandidate instanceof Class) {
				instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
			}
			Object result = instanceCandidate;
			if (result instanceof NullBean) {
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				result = null;
			}
			if (!ClassUtils.isAssignableValue(type, result)) {
				throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
			}
			return result;
		}
		finally {
			ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
		}
	}
```

## 查找过程

### 1. 获取属性的bean类型

从代表了依赖的DependencyDescriptor中获取依赖属性的class type。

```
    Class<?> type = descriptor.getDependencyType();
```

### 2. 处理多bean的属性类型

判断依赖的bean类型是否属于多个bean的类型，例如Stream、数组、List、Map此类。如果是则直接处理成对应的类型并返回，查找候选者直接结束。

```
	Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
	if (multipleBeans != null) {
		return multipleBeans;
	}
```

### 3. 根据类型查找所有的bean

根据依赖属性的type查找bean工厂中所有的bean，得到一个map，key为bean的名称、value为bean。

```
	Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
```

此步代码较深，当注入的bean上标注了@Qualifier，会根据指定的bean名称筛选出满足条件的bean，不满足条件的不会放到map返回。

根据@Qualifier筛选bean的逻辑在：QualifierAnnotationAutowireCandidateResolver#isAutowireCandidate中。

### 4. 根据规则从多个bean中筛选一个

判断上一步获取的所有bean是否超过1个，超过1个则需要继续查找，根据一定规则最终取到一个。


```
	if (matchingBeans.size() > 1) {
		autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
		if (autowiredBeanName == null) {
			if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
				return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
			}
			else {
				return null;
				}
			}
		instanceCandidate = matchingBeans.get(autowiredBeanName);
	}
```

具体筛选出一个bean的逻辑在determineAutowireCandidate方法。

1. 先根据@Primary注解筛选加了@Primary注解的bean名称
2. 没有，则继续根据Priority注解筛选优先级最高的bean名称
3. 依然没有，则尝试获取属性名称和bean名称一样的bean。

```
	protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
		//先根据@Primary注解筛选加了@Primary注解的bean名称
		Class<?> requiredType = descriptor.getDependencyType();
		String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
		if (primaryCandidate != null) {
			return primaryCandidate;
		}
		//没有则继续根据Priority注解筛选优先级最高的bean名称
		String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
		if (priorityCandidate != null) {
			return priorityCandidate;
		}
		// 依然没有？则根据属性名称来匹配bean
		for (Map.Entry<String, Object> entry : candidates.entrySet()) {
			String candidateName = entry.getKey();
			Object beanInstance = entry.getValue();
			if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
					matchesBeanName(candidateName, descriptor.getDependencyName())) {
				return candidateName;
			}
		}
		return null;
	}
```

# 总结

1. @Autowired属性注入是通过bean后置处理器AutowiredAnnotationBeanPostProcessor在bean生命周期不同阶段拦截完成。
2. AutowiredAnnotationBeanPostProcessor通过实现SmartInstantiationAwareBeanPostProcessor的determineCandidateConstructors方法，**在bean实例化前**选择@Autowired注解的构造函数，同时注入属性，从而完成自定义构造函数的选择。
3. 通过实现postProcessMergedBeanDefinition，**在属性赋值前**，缓存属性字段上的@Autowired和@Value注解信息。
4. 通过实现postProcessProperties，**在实例化后，属性赋值前**，查找获取依赖字段对应的value，然后利用反射将value set到对应bean的属性字段中。
5. @Autowired查找候选者过程为：按类型找->通过限定符@Qualifier过滤->@Primary->@Priority->根据名称找（字段名称或者方法名称）