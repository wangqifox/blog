---
title: Spring启动过程分析4(registerBeanPostProcessors)
date: 2017/12/29 08:42:00
---

从Spring容器中找出实现了BeanPostProcessor接口的bean，并设置到BeanFactory的属性中。之后bean被实例化的时候会调用这个BeanPostProcessor。

该方法委托给了PostProcessorRegistrationDelegate类的registerBeanPostProcessors方法执行。
<!--more-->
```java
protected void registerBeanPostProcessors(ConfigurableListableBeanFactory beanFactory) {
	PostProcessorRegistrationDelegate.registerBeanPostProcessors(beanFactory, this);
}
```

registerBeanPostProcessors代码如下：

```java
public static void registerBeanPostProcessors(
		ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {
	
	// 获取beanFactory中所有BeanPostProcessor类的名称
	// internalAutowiredAnnotationProcessor, internalRequiredAnnotationProcessor, internalCommonAnnotationProcessor, internalAutoProxyCreator
	String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

	/**
	 * 注册BeanPostProcessorChecker，主要功能是BeanPostProcessor实例化期间bean被创建时打印日志，
	 * 或者在bean被BeanPostProcessors处理失败时打印日志
	 */
	int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
	beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

	// 将BeanPostProcessors分为实现了PriorityOrdered接口、实现了Ordered接口、其他，三种类型
	// priorityOrderedPostProcessors中有AutowiredAnnotationBeanPostProcessor, RequiredAnnotationBeanPostProcessor, CommonAnnotationBeanPostProcessor
	// orderedPostProcessorNames中有AnnotationAwareAspectJAutoProxyCreator
	List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
	List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
	List<String> orderedPostProcessorNames = new ArrayList<String>();
	List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
	for (String ppName : postProcessorNames) {
		if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			priorityOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
			orderedPostProcessorNames.add(ppName);
		}
		else {
			nonOrderedPostProcessorNames.add(ppName);
		}
	}

	/**
	 * 注册实现了PriorityOrdered接口的BeanPostProcessors，
	 * 当前主要有
	 * AutowiredAnnotationBeanPostProcessor
	 * RequiredAnnotationBeanPostProcessor
	 * CommonAnnotationBeanPostProcessor
	 */
	sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
	/** 
	 * 注册到beanFactory中。添加到List<BeanPostProcessor> beanPostProcessors中。
	 * 当前beanPostProcessors中有：
	 * ApplicationContextAwareProcessor
	 * ApplicationListenerDetector
	 * ImportAwareBeanPostProcessor
	 * BeanPostProcessorChecker
	 * CommonAnnotationBeanPostProcessor
	 * AutowiredAnnotationBeanPostProcessor
	 * RequiredAnnotationBeanPostProcessor
	 */
	registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

	// 注册实现了Ordered接口的BeanPostProcessors	List<BeanPostProcessor> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
	for (String ppName : orderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		orderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	sortPostProcessors(beanFactory, orderedPostProcessors);
	/** 
	 * 注册到beanFactory中。添加到List<BeanPostProcessor> beanPostProcessors中。
	 * 当前beanPostProcessors中有：
	 * ApplicationContextAwareProcessor
	 * ApplicationListenerDetector
	 * ImportAwareBeanPostProcessor
	 * BeanPostProcessorChecker
	 * CommonAnnotationBeanPostProcessor
	 * AutowiredAnnotationBeanPostProcessor
	 * RequiredAnnotationBeanPostProcessor
	 * AnnotationAwareAspectJAutoProxyCreator
	 */
	registerBeanPostProcessors(beanFactory, orderedPostProcessors);

	// 注册其他的BeanPostProcessors
	List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
	for (String ppName : nonOrderedPostProcessorNames) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		nonOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

	// 最后, 注册所有内部BeanPostProcessors.
	sortPostProcessors(beanFactory, internalPostProcessors);
	registerBeanPostProcessors(beanFactory, internalPostProcessors);

	// 重新注册ApplicationListenerDetector，将ApplicationListenerDetector移到beanPostProcessors列表的末尾
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}
```

注册完成后，beanFactory的beanPostProcessors中有7个BeanPostProcessor：

1. ApplicationContextAwareProcessor
2. ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
3. PostProcessorRegistrationDelegate$BeanPostProcessorChecker
4. CommonAnnotationBeanPostProcessor
5. AutowiredAnnotationBeanPostProcessor
6. RequireAnnotationBeanPostProcessor
7. ApplicationListenerDetector

这里的过程跟invokeBeanFactoryPostProcessors类似：

1. 先找出实现了PriorityOrdered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
2. 找出实现了Ordered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中

注册BeanPostProcessor的代码如下，`PostProcessorRegistrationDelegate.registerBeanPostProcessors`：

```java
private static void registerBeanPostProcessors(ConfigurableListBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {
	for (BeanPostProcessor postProcessor : postProcessors) {
		beanFactory.addBeanPostProcessor(postProcessor);
	}
}
```

真正调用的函数是`AbstractBeanFactory.addBeanPostProcessor`:

```java
public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
	Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
	this.beanPostProcessors.remove(beanPostProcessor);
	this.beanPostProcessors.add(beanPostProcessor);
	if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
		this.hasInstantiationAwareBeanPostProcessors = true;
	}
	if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
		this.hasDestructionAwareBeanPostProcessors = true;
	}
}
```

可以看到，beanPostProcessor被保存在AbstractBeanFactory中的`List<BeanPostProcessor> beanPostProcessors`

## registerBeanPostProcessors中用到的几个函数

### getBeanNamesForType

`beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false)`获取匹配BeanPostProcessor类型的bean名称。

调用的是`DefaultListableBeanFactory.getBeanNamesForType`

```java
public String[] getBeanNamesForType(Class<?> type, boolean includeNonSingletons, boolean allowEagerInit) {
	if (!isConfigurationFrozen() || type == null || !allowEagerInit) {
		return doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, allowEagerInit);
	}
	Map<Class<?>, String[]> cache = (includeNonSingletons ? this.allBeanNamesByType : this.singletonBeanNamesByType);
	String[] resolvedBeanNames = cache.get(type);
	if (resolvedBeanNames != null) {
		return resolvedBeanNames;
	}
	resolvedBeanNames = doGetBeanNamesForType(ResolvableType.forRawClass(type), includeNonSingletons, true);
	if (ClassUtils.isCacheSafe(type, getBeanClassLoader())) {
		cache.put(type, resolvedBeanNames);
	}
	return resolvedBeanNames;
}
```

真正工作的是同一个类下的`doGetBeanNamesForType`:

```java
private String[] doGetBeanNamesForType(ResolvableType type, boolean includeNonSingletons, boolean allowEagerInit) {
	List<String> result = new ArrayList<String>();

	// 检查所有的beanDefinitionNames
	for (String beanName : this.beanDefinitionNames) {
		// Only consider bean as eligible if the bean name
		// is not defined as alias for some other bean.
		if (!isAlias(beanName)) {
			try {
				// 根据beanName获取RootBeanDefinition，如果有父类，合并父类的定义
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				// Only check bean definition if it is complete.
				if (!mbd.isAbstract() && (allowEagerInit ||
						((mbd.hasBeanClass() || !mbd.isLazyInit() || isAllowEagerClassLoading())) &&
								!requiresEagerInitForType(mbd.getFactoryBeanName()))) {
					// In case of FactoryBean, match object created by FactoryBean.
					boolean isFactoryBean = isFactoryBean(beanName, mbd);
					BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
					boolean matchFound =
							(allowEagerInit || !isFactoryBean ||
									(dbd != null && !mbd.isLazyInit()) || containsSingleton(beanName)) &&
							(includeNonSingletons ||
									(dbd != null ? mbd.isSingleton() : isSingleton(beanName))) &&
							isTypeMatch(beanName, type);
					if (!matchFound && isFactoryBean) {
						// In case of FactoryBean, try to match FactoryBean instance itself next.
						beanName = FACTORY_BEAN_PREFIX + beanName;
						matchFound = (includeNonSingletons || mbd.isSingleton()) && isTypeMatch(beanName, type);
					}
					if (matchFound) {
						result.add(beanName);
					}
				}
			}
			catch (CannotLoadBeanClassException ex) {
				if (allowEagerInit) {
					throw ex;
				}
				// Probably contains a placeholder: let's ignore it for type matching purposes.
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Ignoring bean class loading failure for bean '" + beanName + "'", ex);
				}
				onSuppressedException(ex);
			}
			catch (BeanDefinitionStoreException ex) {
				if (allowEagerInit) {
					throw ex;
				}
				// Probably contains a placeholder: let's ignore it for type matching purposes.
				if (this.logger.isDebugEnabled()) {
					this.logger.debug("Ignoring unresolvable metadata in bean definition '" + beanName + "'", ex);
				}
				onSuppressedException(ex);
			}
		}
	}

	// 检查手动注册的单例
	for (String beanName : this.manualSingletonNames) {
		try {
			// In case of FactoryBean, match object created by FactoryBean.
			if (isFactoryBean(beanName)) {
				if ((includeNonSingletons || isSingleton(beanName)) && isTypeMatch(beanName, type)) {
					result.add(beanName);
					// Match found for this bean: do not match FactoryBean itself anymore.
					continue;
				}
				// In case of FactoryBean, try to match FactoryBean itself next.
				beanName = FACTORY_BEAN_PREFIX + beanName;
			}
			// Match raw bean instance (might be raw FactoryBean).
			if (isTypeMatch(beanName, type)) {
				result.add(beanName);
			}
		}
		catch (NoSuchBeanDefinitionException ex) {
			// Shouldn't happen - probably a result of circular reference resolution...
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to check manually registered singleton with name '" + beanName + "'", ex);
			}
		}
	}

	return StringUtils.toStringArray(result);
}
```

`getMergedLocalBeanDefinition`获取bean的RootBeanDefinition，如果bean有父类则合并父类的定义，否则返回bean本身的定义：

```java
protected RootBeanDefinition getMergedBeanDefinition(
		String beanName, BeanDefinition bd, BeanDefinition containingBd)
		throws BeanDefinitionStoreException {

	synchronized (this.mergedBeanDefinitions) {
		RootBeanDefinition mbd = null;

		// Check with full lock now in order to enforce the same merged instance.
		if (containingBd == null) {
			mbd = this.mergedBeanDefinitions.get(beanName);
		}

		if (mbd == null) {
			if (bd.getParentName() == null) {
				// Use copy of given root bean definition.
				if (bd instanceof RootBeanDefinition) {
					mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
				}
				else {
					mbd = new RootBeanDefinition(bd);
				}
			}
			else {
				// Child bean definition: needs to be merged with parent.
				BeanDefinition pbd;
				try {
					String parentBeanName = transformedBeanName(bd.getParentName());
					if (!beanName.equals(parentBeanName)) {
						pbd = getMergedBeanDefinition(parentBeanName);
					}
					else {
						BeanFactory parent = getParentBeanFactory();
						if (parent instanceof ConfigurableBeanFactory) {
							pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
						}
						else {
							throw new NoSuchBeanDefinitionException(parentBeanName,
									"Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
									"': cannot be resolved without an AbstractBeanFactory parent");
						}
					}
				}
				catch (NoSuchBeanDefinitionException ex) {
					throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
							"Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
				}
				// Deep copy with overridden values.
				mbd = new RootBeanDefinition(pbd);
				mbd.overrideFrom(bd);
			}

			// Set default singleton scope, if not configured before.
			if (!StringUtils.hasLength(mbd.getScope())) {
				mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
			}

			// A bean contained in a non-singleton bean cannot be a singleton itself.
			// Let's correct this on the fly here, since this might be the result of
			// parent-child merging for the outer bean, in which case the original inner bean
			// definition will not have inherited the merged outer bean's singleton status.
			if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
				mbd.setScope(containingBd.getScope());
			}

			// Cache the merged bean definition for the time being
			// (it might still get re-merged later on in order to pick up metadata changes)
			if (containingBd == null && isCacheBeanMetadata()) {
				this.mergedBeanDefinitions.put(beanName, mbd);
			}
		}

		return mbd;
	}
}
```

#### isTypeMatch

`isTypeMatch`判断名称为beanName的bean是否是type类型：

```java
public boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException {
	String beanName = transformedBeanName(name);

	// 根据beanName从singletonObjects中获取单例对象
	Object beanInstance = getSingleton(beanName, false);
	if (beanInstance != null) {
		if (beanInstance instanceof FactoryBean) {
			if (!BeanFactoryUtils.isFactoryDereference(name)) {
				Class<?> type = getTypeForFactoryBean((FactoryBean<?>) beanInstance);
				return (type != null && typeToMatch.isAssignableFrom(type));
			}
			else {
				return typeToMatch.isInstance(beanInstance);
			}
		}
		else if (!BeanFactoryUtils.isFactoryDereference(name)) {
			if (typeToMatch.isInstance(beanInstance)) {
				// Direct match for exposed instance?
				return true;
			}
			else if (typeToMatch.hasGenerics() && containsBeanDefinition(beanName)) {
				// Generics potentially only match on the target class, not on the proxy...
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				Class<?> targetType = mbd.getTargetType();
				if (targetType != null && targetType != ClassUtils.getUserClass(beanInstance) &&
						typeToMatch.isAssignableFrom(targetType)) {
					// Check raw class match as well, making sure it's exposed on the proxy.
					Class<?> classToMatch = typeToMatch.resolve();
					return (classToMatch == null || classToMatch.isInstance(beanInstance));
				}
			}
		}
		return false;
	}
	else if (containsSingleton(beanName) && !containsBeanDefinition(beanName)) {
		// null instance registered
		return false;
	}

	// No singleton instance found -> check bean definition.
	BeanFactory parentBeanFactory = getParentBeanFactory();
	if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
		// No bean definition found in this factory -> delegate to parent.
		return parentBeanFactory.isTypeMatch(originalBeanName(name), typeToMatch);
	}

	// Retrieve corresponding bean definition.
	RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);

	Class<?> classToMatch = typeToMatch.resolve();
	if (classToMatch == null) {
		classToMatch = FactoryBean.class;
	}
	Class<?>[] typesToMatch = (FactoryBean.class == classToMatch ?
			new Class<?>[] {classToMatch} : new Class<?>[] {FactoryBean.class, classToMatch});

	// Check decorated bean definition, if any: We assume it'll be easier
	// to determine the decorated bean's type than the proxy's type.
	BeanDefinitionHolder dbd = mbd.getDecoratedDefinition();
	if (dbd != null && !BeanFactoryUtils.isFactoryDereference(name)) {
		RootBeanDefinition tbd = getMergedBeanDefinition(dbd.getBeanName(), dbd.getBeanDefinition(), mbd);
		Class<?> targetClass = predictBeanType(dbd.getBeanName(), tbd, typesToMatch);
		if (targetClass != null && !FactoryBean.class.isAssignableFrom(targetClass)) {
			return typeToMatch.isAssignableFrom(targetClass);
		}
	}

	Class<?> beanType = predictBeanType(beanName, mbd, typesToMatch);
	if (beanType == null) {
		return false;
	}

	// Check bean class whether we're dealing with a FactoryBean.
	if (FactoryBean.class.isAssignableFrom(beanType)) {
		if (!BeanFactoryUtils.isFactoryDereference(name)) {
			// If it's a FactoryBean, we want to look at what it creates, not the factory class.
			beanType = getTypeForFactoryBean(beanName, mbd);
			if (beanType == null) {
				return false;
			}
		}
	}
	else if (BeanFactoryUtils.isFactoryDereference(name)) {
		// Special case: A SmartInstantiationAwareBeanPostProcessor returned a non-FactoryBean
		// type but we nevertheless are being asked to dereference a FactoryBean...
		// Let's check the original bean class and proceed with it if it is a FactoryBean.
		beanType = predictBeanType(beanName, mbd, FactoryBean.class);
		if (beanType == null || !FactoryBean.class.isAssignableFrom(beanType)) {
			return false;
		}
	}

	ResolvableType resolvableType = mbd.targetType;
	if (resolvableType == null) {
		resolvableType = mbd.factoryMethodReturnType;
	}
	if (resolvableType != null && resolvableType.resolve() == beanType) {
		return typeToMatch.isAssignableFrom(resolvableType);
	}
	return typeToMatch.isAssignableFrom(beanType);
}
```

判断是否符合类型的方法主要是`Resolvable.isInstance`:

```java
public boolean isInstance(Object obj) {
	return (obj != null && isAssignableFrom(obj.getClass()));
}
```

```java
private boolean isAssignableFrom(ResolvableType other, Map<Type, Type> matchedBefore) {
	Assert.notNull(other, "ResolvableType must not be null");

	// If we cannot resolve types, we are not assignable
	if (this == NONE || other == NONE) {
		return false;
	}

	// 如果是数组，判断数组的component type，即数组元素的类型
	// Deal with array by delegating to the component type
	if (isArray()) {
		return (other.isArray() && getComponentType().isAssignableFrom(other.getComponentType()));
	}

	if (matchedBefore != null && matchedBefore.get(this.type) == other.type) {
		return true;
	}

	// Deal with wildcard bounds
	WildcardBounds ourBounds = WildcardBounds.get(this);
	WildcardBounds typeBounds = WildcardBounds.get(other);

	// In the form X is assignable to <? extends Number>
	if (typeBounds != null) {
		return (ourBounds != null && ourBounds.isSameKind(typeBounds) &&
				ourBounds.isAssignableFrom(typeBounds.getBounds()));
	}

	// In the form <? extends Number> is assignable to X...
	if (ourBounds != null) {
		return ourBounds.isAssignableFrom(other);
	}

	// Main assignability check about to follow
	boolean exactMatch = (matchedBefore != null);  // We're checking nested generic variables now...
	boolean checkGenerics = true;
	Class<?> ourResolved = null;
	if (this.type instanceof TypeVariable) {
		TypeVariable<?> variable = (TypeVariable<?>) this.type;
		// Try default variable resolution
		if (this.variableResolver != null) {
			ResolvableType resolved = this.variableResolver.resolveVariable(variable);
			if (resolved != null) {
				ourResolved = resolved.resolve();
			}
		}
		if (ourResolved == null) {
			// Try variable resolution against target type
			if (other.variableResolver != null) {
				ResolvableType resolved = other.variableResolver.resolveVariable(variable);
				if (resolved != null) {
					ourResolved = resolved.resolve();
					checkGenerics = false;
				}
			}
		}
		if (ourResolved == null) {
			// Unresolved type variable, potentially nested -> never insist on exact match
			exactMatch = false;
		}
	}
	if (ourResolved == null) {
		ourResolved = resolve(Object.class);
	}
	Class<?> otherResolved = other.resolve(Object.class);

	// We need an exact type match for generics
	// List<CharSequence> is not assignable from List<String>
	// ClassUtils.isAssignable是主要的判断代码，判断ourResolved和otherResolved两个Class是否有可赋值关系
	if (exactMatch ? !ourResolved.equals(otherResolved) : !ClassUtils.isAssignable(ourResolved, otherResolved)) {
		return false;
	}

	if (checkGenerics) {
		// Recursively check each generic
		ResolvableType[] ourGenerics = getGenerics();
		ResolvableType[] typeGenerics = other.as(ourResolved).getGenerics();
		if (ourGenerics.length != typeGenerics.length) {
			return false;
		}
		if (matchedBefore == null) {
			matchedBefore = new IdentityHashMap<Type, Type>(1);
		}
		matchedBefore.put(this.type, other.type);
		for (int i = 0; i < ourGenerics.length; i++) {
			if (!ourGenerics[i].isAssignableFrom(typeGenerics[i], matchedBefore)) {
				return false;
			}
		}
	}

	return true;
}
```

`ClassUtils.isAssignable`代码如下：

```java
public static boolean isAssignable(Class<?> lhsType, Class<?> rhsType) {
	Assert.notNull(lhsType, "Left-hand side type must not be null");
	Assert.notNull(rhsType, "Right-hand side type must not be null");
	if (lhsType.isAssignableFrom(rhsType)) {
		return true;
	}
	if (lhsType.isPrimitive()) {
		Class<?> resolvedPrimitive = primitiveWrapperTypeMap.get(rhsType);
		if (lhsType == resolvedPrimitive) {
			return true;
		}
	}
	else {
		Class<?> resolvedWrapper = primitiveTypeToWrapperMap.get(rhsType);
		if (resolvedWrapper != null && lhsType.isAssignableFrom(resolvedWrapper)) {
			return true;
		}
	}
	return false;
}
```


