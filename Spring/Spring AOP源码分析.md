---
title: Spring AOP源码分析
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


参考:

> http://www.91display.com/api/blog/detail.do?bid=6966780842

