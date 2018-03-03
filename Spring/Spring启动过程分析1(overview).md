---
title: Spring启动过程分析1(overview)
date: 2017/12/28 08:57:00
---

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
	/**
	 * 调用AnnotationConfigApplicationContext()构造函数，初始化AnnotatedBeanDefinitionReader, ClassPathBeanDefinitionScanner
	 */
	this();
	/**
	 * 注册初始的配置文件
	 */
	register(annotatedClasses);
	/**
	 * 启动主流程
	 */
	refresh();
}
```
<!--more-->
## AnnotationConfigApplicationContext初始化

`AnnotationConfigApplicationContext(Class<?>... annotatedClasses)`调用AnnotationConfigApplicationContext的空构造函数`AnnotationConfigApplicationContext()`进行初始化

```java
public AnnotationConfigApplicationContext() {
	this.reader = new AnnotatedBeanDefinitionReader(this);
	this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

### AnnotatedBeanDefinitionReader

AnnotatedBeanDefinitionReader构造函数：

```java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	Assert.notNull(environment, "Environment must not be null");
	// 设置registry，本质上是annotationConfigApplicationContext实例
	this.registry = registry;
	// 新建conditionEvaluator，用于处理Conditional注释
	this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
	/**
	 * 注册AnnotationConfigProcessors，包括：
	 * ConfigurationClassPostProcessor
	 * AutowiredAnnotationBeanPostProcessor
	 * RequiredAnnotationBeanPostProcessor
	 * CommonAnnotationBeanPostProcessor
	 * PersistenceAnnotationBeanPostProcessor
	 * EventListenerMethodProcessor
	 * DefaultEventListenerFactory
	 */
	AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

`AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry)`中调用`DefaultListableBeanFactory.registerBeanDefinition`来注册各种BeanPostProcessor的BeanDefinition，包括：

- ConfigurationClassPostProcessor
- AutowireAnnotationBeanPostProcessor
- RequireAnnotationBeanPostProcessor
- CommonAnnotationBeanPostProcessor
- EventListenerMethodProcessor
- DefaultEventListenerFactory

### ClassPathBeanDefinitionScanner

ClassPathBeanDefinitionScanner构造函数：

```java
public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters, Environment environment, ResourceLoader resourceLoader) {
	Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
	// 设置registry，本质上是annotationConfigApplicationContext实例
	this.registry = registry;
	
	if (useDefaultFilters) {
		/**
		 * 为@Component注册默认的过滤器
		 * 隐含地注册包含@Component的注释，包括：@Repository, @Service, @Controller
		 * 也支持@ManagedBean以及@Named注释
		 * 保存在List<TypeFilter> includeFilters里
		 * AnnotationTypeFilter
		 */
		registerDefaultFilters();
	}
	// 设置environment，StandardEnvironment
	setEnvironment(environment);
	// 设置resourceLoader，本质上还是annotationConfigApplicationContext实例
	setResourceLoader(resourceLoader);
}
```

## 注册初始配置文件

注册初始配置文件使用了AnnotationConfigApplicationContext构造函数中初始化的AnnotatedBeanDefinitionReader。调用reader.register方法，最终调用的是`AnnotationConfigApplicationContext.registerBean(Class<?> annotatedClass, String name, Class<? extends Annotation>... qualifiers)`，主要步骤如下：

1. 先将Class包装成AnnotatedGenericBeanDefinition

	`AnnotatedGenericBeanDefinition abd = AnnotatedGenericBeanDefinition(annotatedClass)`
	
	其中，metadata为StandardAnnotationMetadata，里面的annotations保存了配置类中的注释(比如：Configuration, ComponentScan等)

2. 调用`AnnotationConfigUtils.processCommonDefinitionAnnotations(abd)`，处理常见的注释

    判断配置文件是否有一下几个注释，分别进行处理：
    
    1. Lazy
    2. Primary
    3. DependsOn
    4. Role
    5. Description

3. 再将AnnotatedGenericBeanDefinition包装成BeanDefinitionHolder

	`BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(abd, beanName)`

4. 调用`AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry)`

5. 最后注册BeanDefinition

	`BeanDefinitionReaderUtils.registerBeanDefinition(definitionHolder, this.registry)`
	
## 注册BeanDefinition

注册BeanDefinition调用的是`DefaultListableBeanFactory.registerBeanDefinition`，该方法在后面的启动过程中还会经常使用。主要的功能是将bean的名称(beanName)和bean的定义(beanDefinition)放入beanDefinitionMap中，将beanName放入beanDefinitionNames中:

```java
this.beanDefinitionMap.put(beanName, beanDefinition)
this.beanDefinitionNames.add(beanName)
```

## Spring启动主流程

注册完初始的配置文件之后，开始执行主流程。主流程全部包含在`AbstractApplicationContext.refresh`函数中

```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		/**
		 * 表示真正做refresh操作之前需要准备做的事情：
		 * 1. 设置Spring容器的启动时间，撤销关闭状态，开启活跃状态
		 * 2. 初始化property
		 * 3. 验证环境信息里必须存在的属性
		 */
		prepareRefresh();

		/**
		 * 获取beanFactory
		 */
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		/**
		 * 对beanFactory进行相关的设置，为后续的使用做准备
		 */
		prepareBeanFactory(beanFactory);

		try {
			/**
			 * BeanFactory设置之后再进行后续的一些BeanFactory操作
			 * 不同的Spring容器做不同的操作。比如GenericWebApplicationContext容器会在BeanFactory中添加ServletContextAwareProcessor用于处理ServletContextAware类型的bean初始化的时候调用setServletContext或者setServletConfig方法(跟ApplicationContextAwareProcessor原理一样)
			 */	
			postProcessBeanFactory(beanFactory);

			/**
			 * 在Spring容器中找出实现了BeanFactoryPostProcessor接口的processor并执行。
			 * Spring容器会委托给PostProcessorRegistrationDelegate的invokeBeanFactoryPostProcessors方法执行
			 */
			invokeBeanFactoryPostProcessors(beanFactory);

			/**
			 * 从Spring容器中找出实现了BeanPostProcessor接口的bean，并设置到BeanFactory的属性中。之后bean被实例化的时候会调用这个BeanPostProcessor。
			 * 该方法委托给了PostProcessorRegistrationDelegate类的registerBeanPostProcessors方法执行。这里的过程跟invokeBeanFactoryPostProcessors类似：
			 * 1. 先找出实现了PriorityOrdered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
			 * 2. 找出实现了Ordered接口的BeanPostProcessor并排序后加到BeanFactory的BeanPostProcessor集合中
			 */
			registerBeanPostProcessors(beanFactory);

			/**
			 * 在Spring容器中初始化一些国际化相关的属性
			 */
			initMessageSource();

			/* 
			 * 在Spring容器中初始化事件广播器，事件广播器用于事件的发布
			 */
			initApplicationEventMulticaster();

			/**
	        * 一个模板方法，不同的Spring容器做不同的事情
	        * 比如web程序的容器AnnotationConfigEmbeddedWebApplicationContext中会调用createEmbeddedServletContainer方法去创建内置的Servlet容器
	        * 目前SpringBoot只支持3中内置的Servlet容器: Tomcat, Jetty, Undertow
	        */
			onRefresh();

			/**
	        * 把Spring容器内的事件监听器和BeanFactory中的事件监听器都添加的事件广播器中。
	        * 然后如果存在early event的话，广播出去。
	        */
			registerListeners();

			/**
           * 实例化BeanFactory中已经被注册但是未实例化的所有实例(懒加载的不需要实例化)。
           * 比如invokeBeanFactoryPostProcessors方法中根据各种注解解析出来的类，在这个时候都会被初始化。
           * 实例化的过程各种BeanPostProcessor开始起作用。
           */
			finishBeanFactoryInitialization(beanFactory);

			/**
           * 完成refresh，调用LifecycleProcessor的onRefresh()方法，发送ContextRefreshedEvent
           */
			finishRefresh();
		}

		catch (BeansException ex) {
			if (logger.isWarnEnabled()) {
				logger.warn("Exception encountered during context initialization - " +
						"cancelling refresh attempt: " + ex);
			}

			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}

		finally {
			// Reset common introspection caches in Spring's core, since we
			// might not ever need metadata for singleton beans anymore...
			resetCommonCaches();
		}
	}
}
```

