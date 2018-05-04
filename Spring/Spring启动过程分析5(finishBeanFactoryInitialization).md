---
title: Spring启动过程分析5(finishBeanFactoryInitialization)
date: 2018/01/02 15:15:00
---

实例化BeanFactory中已经被注册但是未实例化的所有实例(懒加载的不需要实例化)。比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。实例化的过程各种BeanPostProcessor开始起作用。
<!--more-->
```java
/**
 * 初始化所有剩余的单例bean
 */
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
	// 为容器注册conversion service，用于转化属性值
	if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) && beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
		beanFactory.setConversionService(beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
	}

	/**
	 * 如果没有的话(比如PropertyPlaceholderConfigurer)，注册一个默认的嵌入值解析器(value resoler)。
	 * 用于解析通过注解导入的值。
	 */
	if (!beanFactory.hasEmbeddedValueResolver()) {
		beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
			@Override
			public String resolveStringValue(String strVal) {
				return getEnvironment().resolvePlaceholders(strVal);
			}
		});
	}

	// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
	String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
	for (String weaverAwareName : weaverAwareNames) {
		getBean(weaverAwareName);
	}

	// Stop using the temporary ClassLoader for type matching.
	beanFactory.setTempClassLoader(null);

	// Allow for caching all bean definition metadata, not expecting further changes.
	beanFactory.freezeConfiguration();

	// 实例化所有剩余的(非延迟初始化)单例对象
	beanFactory.preInstantiateSingletons();
}
```

## 实例化单例对象

`beanFactory.preInstantiateSingletons`调用的是`DefaultListableBeanFactory.preInstantiateSingletons`：

```java
public void preInstantiateSingletons() throws BeansException {
	if (this.logger.isDebugEnabled()) {
		this.logger.debug("Pre-instantiating singletons in " + this);
	}

	// 获取当前所有bean的名称
	List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

	// 触发所有非延迟加载单例beans的初始化，主要步骤为调用getBean
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
				boolean isEagerInit;
				if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
					isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
						@Override
						public Boolean run() {
							return ((SmartFactoryBean<?>) factory).isEagerInit();
						}
					}, getAccessControlContext());
				}
				else {
					isEagerInit = (factory instanceof SmartFactoryBean &&
							((SmartFactoryBean<?>) factory).isEagerInit());
				}
				if (isEagerInit) {
					getBean(beanName);
				}
			}
			else {
				getBean(beanName);
			}
		}
	}

	// 触发post-initialization的回调，主要步骤为调用afterSingletonsInstantiated
	for (String beanName : beanNames) {
		Object singletonInstance = getSingleton(beanName);
		// 如果bean是SmartInitializingSingleton实例，则调用其中实现的afterSingletonsInstantiated方法
		if (singletonInstance instanceof SmartInitializingSingleton) {
			final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
			if (System.getSecurityManager() != null) {
				AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}
				}, getAccessControlContext());
			}
			else {
				smartSingleton.afterSingletonsInstantiated();
			}
		}
	}
}
```

### bean初始化

bean初始化调用的是`AbstractBeanFactory.getBean`，最终调用的是`AbstractBeanFactory.doGetBean`

```java
protected <T> T doGetBean(final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly) throws BeansException {

	final String beanName = transformedBeanName(name);
	Object bean;

	// 检查是否已经手动注册过了
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}
	else {
		// 检查是否有循环引用
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// 检查bean definition是否存在
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// 确保当前bean所依赖的bean都被初始化，循环迭代实例化bean
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dep : dependsOn) {
					if (isDependent(beanName, dep)) {
						throw new BeanCreationException(mbd.getResourceDescription(), beanName,
								"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
					}
					registerDependentBean(dep, beanName);
					getBean(dep);
				}
			}

			// 创建bean实例。真正实例化bean的方法是createBean(beanName, mbd, args)，两者的区别在于，单例(Singleton)被缓存起来，而Prototype是不用缓存的。
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					@Override
					public Object getObject() throws BeansException {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}
			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}
			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; consider " +
							"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// 检查要求的类型是否匹配真实的类型
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to convert bean '" + name + "' to required type '" +
						ClassUtils.getQualifiedName(requiredType) + "'", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```

### 创建bean实例

创建bean实例的方法为：`AbstractAutowireCapableBeanFactory.createBean`：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating instance of bean '" + beanName + "'");
	}
	RootBeanDefinition mbdToUse = mbd;

	// 解析bean定义，获取解析过的Class
	Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
	if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
		mbdToUse = new RootBeanDefinition(mbd);
		mbdToUse.setBeanClass(resolvedClass);
	}

	// Prepare method overrides.
	try {
		mbdToUse.prepareMethodOverrides();
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
				beanName, "Validation of method overrides failed", ex);
	}

	try {
		// 在这里，beanPostProcessor可以创建目标bean的代理，然后返回
		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
		if (bean != null) {
			return bean;
		}
	}
	catch (Throwable ex) {
		throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
				"BeanPostProcessor before instantiation of bean failed", ex);
	}

	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
	if (logger.isDebugEnabled()) {
		logger.debug("Finished creating instance of bean '" + beanName + "'");
	}
	return beanInstance;
}
```

#### doCreateBean

`createBean`主要的函数为`doCreateBean`：

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

	// 实例化bean
	BeanWrapper instanceWrapper = null;
	if (mbd.isSingleton()) {
		instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
	}
	if (instanceWrapper == null) {
		instanceWrapper = createBeanInstance(beanName, mbd, args);
	}
	final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
	Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
	mbd.resolvedTargetType = beanType;

	// 调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition来修改merged bean definition
	synchronized (mbd.postProcessingLock) {
		if (!mbd.postProcessed) {
			try {
				applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
			}
			catch (Throwable ex) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Post-processing of merged bean definition failed", ex);
			}
			mbd.postProcessed = true;
		}
	}

   /**
    * 缓存单例，用于解决循环引用的问题。
    * lifecycle接口比如BeanFactoryAware可能会引发循环引用
    */
	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences && isSingletonCurrentlyInCreation(beanName));
	if (earlySingletonExposure) {
		if (logger.isDebugEnabled()) {
			logger.debug("Eagerly caching bean '" + beanName +
					"' to allow for resolving potential circular references");
		}
		addSingletonFactory(beanName, new ObjectFactory<Object>() {
			@Override
			public Object getObject() throws BeansException {
				return getEarlyBeanReference(beanName, mbd, bean);
			}
		});
	}

	// 初始化bean实例
	Object exposedObject = bean;
	try {
		// 注入成员以及属性
		populateBean(beanName, mbd, instanceWrapper);
		if (exposedObject != null) {
			// 初始化实例
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
	}
	catch (Throwable ex) {
		if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
			throw (BeanCreationException) ex;
		}
		else {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
		}
	}

	if (earlySingletonExposure) {
		Object earlySingletonReference = getSingleton(beanName, false);
		if (earlySingletonReference != null) {
			if (exposedObject == bean) {
				exposedObject = earlySingletonReference;
			}
			else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
				String[] dependentBeans = getDependentBeans(beanName);
				Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
				for (String dependentBean : dependentBeans) {
					if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
						actualDependentBeans.add(dependentBean);
					}
				}
				if (!actualDependentBeans.isEmpty()) {
					throw new BeanCurrentlyInCreationException(beanName,
							"Bean with name '" + beanName + "' has been injected into other beans [" +
							StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
							"] in its raw version as part of a circular reference, but has eventually been " +
							"wrapped. This means that said other beans do not use the final version of the " +
							"bean. This is often the result of over-eager type matching - consider using " +
							"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
				}
			}
		}
	}

	// Register bean as disposable.
	try {
		registerDisposableBeanIfNecessary(beanName, bean, mbd);
	}
	catch (BeanDefinitionValidationException ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
	}

	return exposedObject;
}
```

创建bean主要分为三个步骤：

1. 实例化bean
2. 注入成员以及变量
3. 执行初始化步骤

### 实例化bean

实例化bean调用的是`AbstractAutowireCapableBeanFactory.createBeanInstance`：

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
	// 解析bean的定义
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

   // 判断bean是否是public
	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}
   
   // 如果bean有工厂方法，则使用工厂方法来实例化，然后直接返回
	if (mbd.getFactoryMethodName() != null)  {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// 如果显式指定了构造函数则调用`autowireConstructor`，否则调用`instantiateBean`
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	// 如果可以确定构造函数，则直接实例化bean
	if (resolved) {
		if (autowireNecessary) {
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			return instantiateBean(beanName, mbd);
		}
	}

	// 如果不能确定构造函数，则尝试从BeanPostProcessor中获取构造函数
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// 如果还是无法确定构造函数，则使用默认的无参构造函数
	return instantiateBean(beanName, mbd);
}
```

#### autowireConstructor

在构造函数中注入相应的参数

```java
protected BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd, Constructor<?>[] ctors, Object[] explicitArgs) {

	return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
}
```

`autowireConstructor`实际的工作在于寻找构造函数、参数，然后调用`InstantiationStrategy.instantiate`来实例化bean。

```java
beanInstance = this.beanFactory.getInstantiationStrategy().instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
```

##### InstantiationStrategy.instantiate

```java
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner, final Constructor<?> ctor, Object... args) {
    /**
     * 如果bean中有被容器重写的方法，则调用instantiateWithMethodInjection实例化bean
     * 否则，调用BeanUtils.instantiateClass实例化bean
     */
    if (bd.getMethodOverrides().isEmpty()) {
        if (System.getSecurityManager() != null) {
				// use own privileged to change accessibility (when security is on)
				AccessController.doPrivileged(new PrivilegedAction<Object>() {
					@Override
					public Object run() {
						ReflectionUtils.makeAccessible(ctor);
						return null;
					}
				});
		  }
		  /**
		   * 调用ctor.newInstance(args)来实例化
		   */
	     return BeanUtils.instantiateClass(ctor, args);
    }
    else {
        /**
         * 使用cglib来生成一个子类
         */
        return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
    }
}
```

#### AbstractAutowireCapableBeanFactory.instantiateBean

使用默认的构造函数实例化bean

```java
/**
 * 使用默认的构造函数实例化bean
 */
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
	try {
		Object beanInstance;
		final BeanFactory parent = this;
		if (System.getSecurityManager() != null) {
			beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					return getInstantiationStrategy().instantiate(mbd, beanName, parent);
				}
			}, getAccessControlContext());
		}
		else {
			beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
		}
		BeanWrapper bw = new BeanWrapperImpl(beanInstance);
		initBeanWrapper(bw);
		return bw;
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
	}
}
```

实际调用的是`SimpleInstantiationStrategy.instantiate`：

##### SimpleInstantiationStrategy.instantiate

```java
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
	// Don't override the class with CGLIB if no overrides.
	if (bd.getMethodOverrides().isEmpty()) {
		Constructor<?> constructorToUse;
		synchronized (bd.constructorArgumentLock) {
			constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
			if (constructorToUse == null) {
				final Class<?> clazz = bd.getBeanClass();
				if (clazz.isInterface()) {
					throw new BeanInstantiationException(clazz, "Specified class is an interface");
				}
				try {
					if (System.getSecurityManager() != null) {
						constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
							@Override
							public Constructor<?> run() throws Exception {
								return clazz.getDeclaredConstructor((Class[]) null);
							}
						});
					}
					else {
						constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
					}
					bd.resolvedConstructorOrFactoryMethod = constructorToUse;
				}
				catch (Throwable ex) {
					throw new BeanInstantiationException(clazz, "No default constructor found", ex);
				}
			}
		}
		// 如果有无参构造函数就反射生成子类
		return BeanUtils.instantiateClass(constructorToUse);
	}
	else {
		// 生成CGLIB子类
		return instantiateWithMethodInjection(bd, beanName, owner);
	}
}
```

### 注入成员以及变量

bean实例化完成之后，调用`AbstractAutowireCapableBeanFactory.populateBean`注入其中的成员以及变量：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
	PropertyValues pvs = mbd.getPropertyValues();

	if (bw == null) {
		if (!pvs.isEmpty()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
		}
		else {
			// 如果bean为null，则跳过填充的阶段
			return;
		}
	}

	/**
	 * 调用InstantiationAwareBeanPostProcessor来修改bean的状态。可以用于支持字段注入
	 */
	boolean continueWithPropertyPopulation = true;

	if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
					continueWithPropertyPopulation = false;
					break;
				}
			}
		}
	}

   // 如果调用InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation返回false，则跳过填充阶段。表示不再需要继续填充
	if (!continueWithPropertyPopulation) {
		return;
	}

   // 如果自动装配的类型有AUTOWIRE_BY_NAME和AUTOWIRE_BY_TYPE，则分别尝试根据名称自动装配、根据类型自动装配
	if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
			mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
		MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

		// Add property values based on autowire by name if applicable.
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
			autowireByName(beanName, mbd, bw, newPvs);
		}

		// Add property values based on autowire by type if applicable.
		if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
			autowireByType(beanName, mbd, bw, newPvs);
		}

		pvs = newPvs;
	}

	boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
	boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

	if (hasInstAwareBpps || needsDepCheck) {
		PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
		if (hasInstAwareBpps) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					/**
					 * 调用InstantiationAwareBeanPostProcessor的具体子类的ibp.postProcessPropertyValues方法注入属性。
					 * 当我们使用@Resource注解的时候，具体的子类是CommonAnnotationBeanPostProcessor，
					 * 如果使用的是@Autowired注解，则具体的子类是AutowiredAnnotationBeanPostProcessor。
					 */
					pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
					if (pvs == null) {
						return;
					}
				}
			}
		}
		if (needsDepCheck) {
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}
	}

	applyPropertyValues(beanName, mbd, bw, pvs);
}
```

- ApplicationContextAwareProcessor
- ImportAwareBeanPostProcessor
- BeanPostProcessorChecker
- CommonAnnotationBeanPostProcessor
    
    支持`@PostConstruct`、`@PreDestroy`注释
    
- AutowiredAnnotationBeanPostProcessor

    支持`@Autowired`、`@Value`注释

- RequiredAnnotationBeanPostProcessor

    支持`@Required`注释

- ApplicationListenerDetector

#### postProcessPropertyValues

以AutowiredAnnotationBeanPostprocessor为例，它委托给InjectionMetadata对象来完成属性注入，`AutowiredAnnotationBeanPostprocessor.postProcessPropertyValues`：

```java
public PropertyValues postProcessPropertyValues(PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
	/**
	 * 委托给InjectionMetadata对象来完成属性注入
	 */
	InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
	try {
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

- InjectionMetadata.inject(Object target, String beanName, PropertryValues pvs)

	```java
	public void inject(Object target, String beanName, PropertyValues pvs) throws Throwable {
		Collection<InjectedElement> elementsToIterate = (this.checkedElements != null ? this.checkedElements : this.injectedElements);
		if (!elementsToIterate.isEmpty()) {
			boolean debug = logger.isDebugEnabled();
			for (InjectedElement element : elementsToIterate) {
				if (debug) {
					logger.debug("Processing injected element of bean '" + beanName + "': " + element);
				}
				element.inject(target, beanName, pvs);
			}
		}
	}
	```
- InjectionMetadata.inject(Object target, String requestingBeanName, PropertyValues pvs)

	```java
	protected void inject(Object target, String requestingBeanName, PropertyValues pvs) throws Throwable {
		if (this.isField) {
			Field field = (Field) this.member;
			ReflectionUtils.makeAccessible(field);
			field.set(target, getResourceToInject(target, requestingBeanName));
		}
		else {
			if (checkPropertySkipping(pvs)) {
				return;
			}
			try {
				Method method = (Method) this.member;
				ReflectionUtils.makeAccessible(method);
				method.invoke(target, getResourceToInject(target, requestingBeanName));
			}
			catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
		}
		
	}
	```

可以看到，如果是成员变量则调用`field.set`来设置变量，如果是方法则调用`method.invoke`来调用方法

#### applyPropertyValues

注入属性值

最终调用的方法为`AbstractNestablePropertyAccessor.setPropertyValue`:

```java
protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
	if (tokens.keys != null) {
		processKeyedProperty(tokens, pv);
	}
	else {
		processLocalProperty(tokens, pv);
	}
}
```

其中processKeyedProperty设置array、list、map三种类型的属性；processLocalProperty设置field、bean类型的属性。

field类型的属性通过`DirectFieldAccessor.setValue`来设置：

```java
public void setValue(Object object, Object value) throws Exception {
	try {
		ReflectionUtils.makeAccessible(this.field);
		this.field.set(object, value);
	}
	catch (IllegalAccessException ex) {
		throw new InvalidPropertyException(getWrappedClass(), this.field.getName(),
				"Field is not accessible", ex);
	}
}
```

bean类型的属性通过`BeanWrapperImpl.setValue`来设置：

```java
public void setValue(final Object object, Object valueToApply) throws Exception {
   // 获取属性写入的方法
	final Method writeMethod = (this.pd instanceof GenericTypeAwarePropertyDescriptor ?
			((GenericTypeAwarePropertyDescriptor) this.pd).getWriteMethodForActualAccess() :
			this.pd.getWriteMethod());
	if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers()) && !writeMethod.isAccessible()) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged(new PrivilegedAction<Object>() {
				@Override
				public Object run() {
					writeMethod.setAccessible(true);
					return null;
				}
			});
		}
		else {
			writeMethod.setAccessible(true);
		}
	}
	final Object value = valueToApply;
	if (System.getSecurityManager() != null) {
		try {
			AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
				@Override
				public Object run() throws Exception {
					writeMethod.invoke(object, value);
					return null;
				}
			}, acc);
		}
		catch (PrivilegedActionException ex) {
			throw ex.getException();
		}
	}
	else {
	   // 使用writeMethod类设置属性
		writeMethod.invoke(getWrappedInstance(), value);
	}
}
```

### bean初始化

在完成bean实例化，注入成员以及变量之后，执行bean的一系列初始化操作。包括调用init方法，执行`bean post processors`。

bean初始化调用的是`AbstractAutowireCapableBeanFactory.initializeBean`
	
```java
/**
 * 初始化bean实例，调用初始化方法以及bean post processors
 */
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedAction<Object>() {
			@Override
			public Object run() {
				invokeAwareMethods(beanName, bean);
				return null;
			}
		}, getAccessControlContext());
	}
	else {
		// 如果bean实现了Aware、BeanNameAware、BeanClassLoaderAware、BeanFactoryAware接口，则调用其中实现的各种方法
		invokeAwareMethods(beanName, bean);
	}
	
	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		// 在初始化bean之前，调用BeanPostProcessors的postProcessBeforeInitialization方法
		wrappedBean = applyBeanPostProcessorBeforeInitialization(wrappedBean, beanName);
	}
	
	try {
		// 调用初始化方法
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	
	if (mbd == null || !mbd.isSynthetic()) {
		// 初始化完成之后，调用BeanPostProcessors的postProcessAfterInitialization方法
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```

#### applyBeanPostProcessorBeforeInitialization

调用beanPostProcessors中各个beanPostProcessor的postProcessBeforeInitialization方法

#### 调用初始化方法

初始化方法为`AbstractAutowireCapableBeanFactory.invokeInitMethods`：

```java
protected void invokeInitMethods(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
	boolean isInitializingBean = (bean instanceof InitializingBean);
	if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
		if (logger.isDebugEnabled()) {
			logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
		}
		if (System.getSecurityManager() != null) {
			try {
				AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
					@Override
					public Object run() throws Exception {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}
				}, getAccessControlContext());
			}
			catch (PrivilegedActionException pae) {
				throw pae.getException();
			}
		}
		else {
			// 如果bean实现了InitializingBean接口，则调用实现的afterPropertiesSet方法
			((InitializingBean) bean).afterPropertiesSet();
		}
		
		if (mbd != null) {
			String initMethodName = mbd.getInitMethodName();
			if (initMethodName != null && !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				// 执行自定义的初始化方法
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
}
```

##### 调用自定义的初始化方法

自定义的初始化方法为`AbstractAutowireCapableBeanFactory.invokeCustomInitMethod`

```java
protected void invokeCustomInitMethod(String beanName, final Object bean, RootBeanDefinition mbd) throws Throwable {
	// 获取自定义初始化方法的名称
	String initMethodName = mbd.getInitMethodName();
	/**
	 * 如果允许访问非public的方法，调用BeanUtils.findMethod来找到初始化方法；
	 * 否则调用ClassUtils.getMethodIfAvailable获取初始化方法
	 */
	final Method initMethod = (mbd.isNonPublicAccessAllowed() ? BeanUtils.findMethod(bean.getClass(), initMethodName) : ClassUtils.getMethodIfAvailable(bean.getClass(), initMethodName));
	if (initMethod == null) {
		if (mbd.isEnforceInitMethod()) {
			throw new BeanDefinitionValidationException("Couldn't find an init method named '" + initMethodName + "' on bean with name '" + beanName + "'");
		}
		else {
			if (logger.isDebugEnabled()) {
				logger.debug("No default init method named '" + initMethodName + "' found on bean with name '" + beanName + "'");
			}
			// Ignore non-existent default lifecycle methods.
			return;
		}
	}
	
	if (logger.isDebugEnabled()) {
		logger.debug("Invoking init method  '" + initMethodName + "' on bean with name '" + beanName + "'");
	}
	
	/**
	 * 调用invoke，执行前面获取的初始化方法
	 */
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
			@Override
			public Object run() throws Exception {
				ReflectionUtils.makeAccessible(initMethod);
				return null;
			}
		});
		try {
			AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
				@Override
				public Object run() throws Exception {
					initMethod.invoke(bean);
					return null;
				}
			}, getAccessControlContext());
		}
		catch (PrivilegedActionException pae) {
			InvocationTargetException ex = (InvocationTargetException) pae.getException();
			throw ex.getTargetException();
		}
	}
	else {
		try {
			ReflectionUtils.makeAccessible(initMethod);
			initMethod.invoke(bean);
		}
		catch (InvocationTargetException ex) {
			throw ex.getTargetException();
		}
	}
}
```

##### applyBeanPostProcessorsAfterInitialization

调用beanPostProcessors中各个beanPostProcessor的postProcessAfterInitialization方法


