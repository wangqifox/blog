---
title: Spring启动过程分析2(prepareBeanFactory)
date: 2017/12/28 13:26:00
---

`prepareBeanFactory`方法在`AbstractApplicationContext.refresh`流程中。对前面获取到的beanFactory(ConfigurationListableBeanFactory)进行相关的设置，包括ClassLoader, post-processors，为后续的使用做准备。
<!--more-->
代码如下：

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// 设置classLoader(用于加载bean)
	beanFactory.setBeanClassLoader(getClassLoader());
	// 设置表达式解析器StandardBeanExpressionResolver(解析bean定义中的一些表达式)
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	// 添加属性编辑注册器(注册属性编辑器)
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegister(this, getEnvironment()));
	
	// 添加ApplicationContextAwareProcessor这个BeanPostProcessor，保存在beanFactory的beanPostProcessors列表中
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	
	/** 
	 * 取消
	 * EnvironmentAware, 
	 * EmbeddedValueResolverAware,
	 * ResourceLoaderAware, 
	 * ApplicationEventPublisherAware, 
	 * MessageSourceAware, 
	 * ApplicationContextAware这6个接口的自动注入。
	 * 因为ApplicatioinContextAwareProcessor把这6这个接口的实现工作做了
	 * 
	 * 保存在beanFactory的ignoredDependencyInterfaces集合中
	 * 
	 * ApplicatioinContextAwareProcessor的作用在于为实现*Aware接口的bean调用该Aware接口定义的方法，并传入对应的参数。
	 * 比如实现EnvironmentAware接口的bean在该Processor内部会调用EnvironmentAware接口的setEnvironment方法，并把Spring容器内部的ConfigurationEnvironment传递进去。
	 */
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
	
	/**
	 * 设置特殊类型对应的bean。
	 * beanFactory对应刚刚获取的BeanFactory
	 * ResourceLoader, ApplicationEventPublisher, ApplicationContext这3个接口对应的bean都设置为当前的Spring容器
	 * 
	 * 保存在beanFactory的resolvableDependencies(ConcurrentHashMap)中
	 */
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);
	
	// 注册ApplicationListenerDetector，用于发现实现了ApplicationListener接口的bean
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
	
	// 检查代码织入
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)){
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader)));
	}
	
	/**
	 * 注册默认的environment, systemProperties, systemEnvironment
	 * 保存在beanFactory的singletonObjects(ConcurrentHashMap)、registeredSingletons(LinkedHashSet)、manualSingletonNames(LinkedHashSet)中
	 */
	if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
	}
	if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
		beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
	}
}
```

beanPostProcessors中当前有`ApplicationContextAwareProcessor`、`ApplicationListenerDetector`两个beanPostProcessor

`beanFactory.addBeanPostProcessor`做的工作是增加BeanPostProcessor。在AbstractBeanFactory的`List<BeanPostProcessor> beanPostProcessors`中加入BeanPostProcessor。

`beanFactory.ignoreDependencyInterface`做的工作是忽略特定的类。在AbstractAutowireCapableBeanFactory的`Set<Class<?>> ignoreDependencyInterfaces`中加入需要被忽略的类。

`beanFactory.registerResolvableDependency`做的工作是为某些类注入特定的值。在`DefaultListableBeanFactory`的`Map<Class<?>, Object> resolvableDependencies`中加入依赖的值(dependencyType)、注入的值(autowiredValue)。

`beanFactory.registerSingleton`做的工作是在beanFactory中注册单例。在`DefaultSingletonBeanRegistry`的`Map<String, Object> singletonObjects`中加入(beanName, singletonObject)；在`DefaultSingletonBeanRegistry`的`Set<String> registeredSingletons`中加入beanName

