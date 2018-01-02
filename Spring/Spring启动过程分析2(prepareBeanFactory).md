---
title: Spring启动过程分析2(prepareBeanFactory)
---

`prepareBeanFactory`方法在`AbstractApplicationContext.refresh`流程中。对前面获取到的beanFactory(ConfigurationListableBeanFactory)进行相关的设置，包括ClassLoader, post-processors，为后续的使用做准备。

代码如下：

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// 设置classLoader(用于加载bean)
	beanFactory.setBeanClassLoader(getClassLoader());
	// 设置表达式解析器(解析bean定义中的一些表达式)
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	// 添加属性编辑注册器(注册属性编辑器)
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegister(this, getEnvironment()));
	
	// 添加ApplicationContextAwareProcessor这个BeanPostProcessor
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
	 * /
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
	
	/* 设置特殊类型对应的bean。
	 * beanFactory对应刚刚获取的BeanFactory
	 * ResourceLoader, ApplicationEventPublisher, ApplicationContext这3个接口对应的bean都设置为当前的Spring容器
	 * /
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);
	
	// 注册ApplicationListenerDetector
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
	
	// 检查代码织入
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)){
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader)));
	}
	
	// 注册environment, systemProperties, systemEnvironment
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