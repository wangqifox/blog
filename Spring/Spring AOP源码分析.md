---
title: Spring AOP源码分析
date: 2018/01/16 18:16:00
---

在`invokeBeanFactoryPostProcessors`中注册beanDefinition

`invokeBeanFactoryPostProcessors` -> 
`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors` -> 
`invokeBeanDefinitionRegistryPostProcessors` -> 
`ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(registry)` -> 
`processConfigBeanDefinitions(registry)` -> 
`ConfigurationClassBeanDefinitionReader.loadBeanDefinitions` -> 
`ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass` -> 
`loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars())` -> 
`AspectJAutoProxyRegistrar.registerBeanDefinitions`
<!--more-->
```java
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

	AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

	AnnotationAttributes enableAspectJAutoProxy =
			AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
	if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
		AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
	}
	if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
		AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
	}
}
```

其中`AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary`最终调用的方法为`AopConfigUtils.registerOrEscalateApcAsRequired`：

```java
private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry, Object source) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
		BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
		if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
			int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
			int requiredPriority = findPriorityForClass(cls);
			if (currentPriority < requiredPriority) {
				apcDefinition.setBeanClassName(cls.getName());
			}
		}
		return null;
	}
	RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
	beanDefinition.setSource(source);
	beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
	beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
	registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
	return beanDefinition;
}
```

其中cls为`AnnotationAwareAspectJAutoProxyCreator`，在registry中注册`AnnotationAwareAspectJAutoProxyCreator`的beanDefinition，key为`org.springframework.aop.config.internalAutoProxyCreator`

## bean创建及初始化

bean创建工作在`DefaultListableBeanFactory.doCreateBean`中完成：

1. 调用`AbstractAutowireCapableBeanFactory.createBeanInstance`创建bean实例。
2. 调用`AbstractAutowireCapableBeanFactory.populateBean`填充bean实例。
3. 调用`AbstractAutowireCapableBeanFactory.initializeBean`初始化bean实例

```java
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
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}

	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}
	return wrappedBean;
}
```

重点关注`applyBeanPostProcessorsAfterInitialization`，它调用各个beanPostProcessor的`postProcessAfterInitialization`。

进入`AbstractAutoProxyCreator.postProcessAfterInitialization`

```java
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
	if (bean != null) {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		if (!this.earlyProxyReferences.contains(cacheKey)) {
			return wrapIfNecessary(bean, beanName, cacheKey);
		}
	}
	return bean;
}
```

进入`AbstractAutoProxyCreator.wrapIfNecessary`，这个方法根据bean是否需要代理来包装一层代理

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
		return bean;
	}
	if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
		return bean;
	}
	if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

	// 获取该bean的advice
	Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
	if (specificInterceptors != DO_NOT_PROXY) {
		this.advisedBeans.put(cacheKey, Boolean.TRUE);
		// 如果这个bean的advice不为空，创建代理
		Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
		this.proxyTypes.put(cacheKey, proxy.getClass());
		return proxy;
	}

	this.advisedBeans.put(cacheKey, Boolean.FALSE);
	return bean;
}
```

## createProxy

在bean初始化过程中，如果发现bean的advice不为空，则调用`AbstractAutoProxyCreator.createProxy`创建bean的代理

```java
protected Object createProxy(Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

	if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
		AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
	}

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

	Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
	for (Advisor advisor : advisors) {
		proxyFactory.addAdvisor(advisor);
	}

	proxyFactory.setTargetSource(targetSource);
	customizeProxyFactory(proxyFactory);

	proxyFactory.setFrozen(this.freezeProxy);
	if (advisorsPreFiltered()) {
		proxyFactory.setPreFiltered(true);
	}

	return proxyFactory.getProxy(getProxyClassLoader());
}
```

最后调用`ProxyFactory.getProxy`方法，该方法只有一行代码:

```java
public Object getProxy(ClassLoader classLoader) {
	return createAopProxy().getProxy(classLoader);
}
```

### 初始化AopProxy

初始化AopProxy调用的是`ProxyCreatorSupport.createAopProxy`

```java
protected final synchronized AopProxy createAopProxy() {
	if (!this.active) {
		activate();
	}
	return getAopProxyFactory().createAopProxy(this);
}
```

首先获取aopProxyFactory，再调用`DefaultAopProxyFactory`中的`createAopProxy`方法。这就是生成代理的入口，你会发现传入的参数是this，其实传入的就是this(ProxyCreatorSupport)的父类`AdvisedSupport`中的advisor等生成代理的核心参数。

Spring根据代理的目标对象是否实现了接口，来返回JdkDynamicAopProxy的动态代理或者CGLIB的代理，并且传入advisor核心参数。(JdkDynamicAopProxy这个实现了InvocationHandler，要实现invoke的关键就是传入的advisor)

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
	if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
		Class<?> targetClass = config.getTargetClass();
		if (targetClass == null) {
			throw new AopConfigException("TargetSource cannot determine target class: " +
					"Either an interface or a target is required for proxy creation.");
		}
		if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
			return new JdkDynamicAopProxy(config);
		}
		return new ObjenesisCglibAopProxy(config);
	}
	else {
		return new JdkDynamicAopProxy(config);
	}
}
```

接着看`JdkDynamicAopProxy`：

```java
public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
	Assert.notNull(config, "AdvisedSupport must not be null");
	if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
		throw new AopConfigException("No advisors and no TargetSource specified");
	}
	this.advised = config;
}
```

将传入的AdvisedSupport赋值到advised里。

### 获取代理对象

回到`ProxyFactory.getProxy`方法，方法中先调用`createAopProxy`创建`AopProxy`，再调用`getProxy`获取代理对象。

以`JdkDynamicAopProxy`为例：

```java
public Object getProxy(ClassLoader classLoader) {
	if (logger.isDebugEnabled()) {
		logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
	}
	Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
	findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
	return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

调用`Proxy.newProxyInstance`创建代理类。这是java reflect包提供的原生创建代理类的方法。就是在这里，目标对象的代理对象完成了创建并返回。

回到`AbstractAutowireCapableBeanFactory.initializeBean`方法，wrappedBean在经过`applyBeanPostProcessorsAfterInitialization`处理，调用`AbstractAutoProxyCreator.postProcessAfterInitialization`，wrappedBean变成了一个代理类。

### invoke

当调用aop代理类的方法时，调用的实际上是`JdkDynamicAopProxy.invoke`:

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	MethodInvocation invocation;
	Object oldProxy = null;
	boolean setProxyContext = false;

	TargetSource targetSource = this.advised.targetSource;
	Class<?> targetClass = null;
	Object target = null;

	try {
		if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
			// 判断如果被代理的目标对象要执行的方法是equals则执行JdkDynamicAopProxy方法，然后就返回了，也就是说Spring不对equals方法进行AOP拦截
			return equals(args[0]);
		}
		else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
			// 判断如果被代理的目标对象要执行的方法是hashCode则执行JdkDynamicAopProxy方法，然后就返回了，也就是说Spring不对hashCode方法进行AOP拦截
			return hashCode();
		}
		else if (method.getDeclaringClass() == DecoratingProxy.class) {
			// 如果被代理的对象实现了DecoratingProxy，返回被代理的对象
			return AopProxyUtils.ultimateTargetClass(this.advised);
		}
		else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
				method.getDeclaringClass().isAssignableFrom(Advised.class)) {
			// 如果被代理的对象本身就是实现了Advised接口，也不做处理，直接执行。
			return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
		}

		Object retVal;

		if (this.advised.exposeProxy) {
			// Make invocation available if necessary.
			oldProxy = AopContext.setCurrentProxy(proxy);
			setProxyContext = true;
		}

		// 获取被增强的对象
		target = targetSource.getTarget();
		if (target != null) {
			targetClass = target.getClass();
		}

		// 这个是核心方法，获取该方法的拦截链
		List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		// Check whether we have any advice. If we don't, we can fallback on direct
		// reflective invocation of the target, and avoid creating a MethodInvocation.
		if (chain.isEmpty()) {
			// We can skip creating a MethodInvocation: just invoke the target directly
			// Note that the final invoker must be an InvokerInterceptor so we know it does
			// nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
			Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
			retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
		}
		else {
			// 创建MethodInvocation，将我们之前的代理、目标对象、拦截的method名称、拦截方法的参数、拦截器链全部整合到了ReflectiveMethodInvocation这个类中
			invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
			// 根据拦截链执行joinpoint
			retVal = invocation.proceed();
		}

		// Massage return value if necessary.
		Class<?> returnType = method.getReturnType();
		if (retVal != null && retVal == target &&
				returnType != Object.class && returnType.isInstance(proxy) &&
				!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
			// Special case: it returned "this" and the return type of the method
			// is type-compatible. Note that we can't help if the target sets
			// a reference to itself in another returned object.
			retVal = proxy;
		}
		else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
			throw new AopInvocationException(
					"Null return value from advice does not match primitive return type for: " + method);
		}
		return retVal;
	}
	finally {
		if (target != null && !targetSource.isStatic()) {
			// Must have come from TargetSource.
			targetSource.releaseTarget(target);
		}
		if (setProxyContext) {
			// Restore old proxy.
			AopContext.setCurrentProxy(oldProxy);
		}
	}
}
```

#### AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice

根据给定的方法，获取一个MethodInterceptor对象的列表。

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, Class<?> targetClass) {
	/**
	 * 从缓存中寻找该方法的拦截链是否已经获取过
	 * (可能被代理对象的某个方法被调用过多次，调用第一次就会获取一次，后面多次调用时，则需从缓存中直接获取，无需多次获取，这样能提高性能)
	 * 如果已经获取过，直接返回
	 */
	MethodCacheKey cacheKey = new MethodCacheKey(method);
	List<Object> cached = this.methodCache.get(cacheKey);
	if (cached == null) {
		cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
		this.methodCache.put(cacheKey, cached);
	}
	return cached;
}
```

看看`DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice`方法：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, Class<?> targetClass) {

	// This is somewhat tricky... We have to process introductions first,
	// but we need to preserve order in the ultimate list.
	List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);
	Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
	// 查看advisor数组中是否有实现IntroductionAdvisor接口的子类
	boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);
	AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

	for (Advisor advisor : config.getAdvisors()) {
		if (advisor instanceof PointcutAdvisor) {
			// Add it conditionally.
			PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
			// 判断当前的pointcut是否匹配要增强的类
			if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
				MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
				MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
				// 判断当前pointcut的方法是否匹配要增强的方法
				if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
					if (mm.isRuntime()) {
						// Creating a new object instance in the getInterceptors() method
						// isn't a problem as we normally cache created chains.
						for (MethodInterceptor interceptor : interceptors) {
							interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
						}
					}
					else {
						interceptorList.addAll(Arrays.asList(interceptors));
					}
				}
			}
		}
		else if (advisor instanceof IntroductionAdvisor) {
			IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
			if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}
		else {
			Interceptor[] interceptors = registry.getInterceptors(advisor);
			interceptorList.addAll(Arrays.asList(interceptors));
		}
	}

	return interceptorList;
}
```

#### ReflectiveMethodInvocation.proceed

```java
public Object proceed() throws Throwable {
	//	We start with an index of -1 and increment early.
	if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
		return invokeJoinpoint();
	}

	Object interceptorOrInterceptionAdvice =
			this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
	if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
		// Evaluate dynamic method matcher here: static part will already have
		// been evaluated and found to match.
		InterceptorAndDynamicMethodMatcher dm =
				(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
		if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
			return dm.interceptor.invoke(this);
		}
		else {
			// Dynamic matching failed.
			// Skip this interceptor and invoke the next in the chain.
			return proceed();
		}
	}
	else {
		// It's an interceptor, so we just invoke it: The pointcut will have
		// been evaluated statically before this object was constructed.
		return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
	}
}
```



参考:

> http://www.91display.com/api/blog/detail.do?bid=6966780842

