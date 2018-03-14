---
title: Spring启动过程分析.番外(Bean的实例化过程)
date: 2018/03/14 08:40:00
---

Bean的示例化在Spring的启动过程中基本上算是最后的步骤。经过了前面`BeanDefinitionRegistryPostProcesser`和`BeanFactoryPostProcessor`注册BeanDefintion，`registerBeanPostProcessors`方法注册`BeanPostProcessor`。到了最后实例化的过程。
<!-- more -->
实例化的过程经过一下几层调用：
`AbstractApplicationContext.finishBeanFactoryInitialization` -> `DefaultListableBeanFactory.preInstantiateSingletons` -> 
`AbstractBeanFactory.getBean` ->
`AbstractBeanFactory.doGetBean` 

简单来说就是先获取当前所有的beanDefinitionNames，根据beanDefinitionNames调用`AbstractBeanFactory.doGetBean`来实例化bean。

OK，我们来分析一下`doGetBean`的过程，这是一个复杂的过程。

首先调用`getSingleton`方法在已注册的单例列表中寻找给定名称的单例并返回。

如果返回的不是null，说明这个bean已经注册为单例了，于是调用`getObjectForBeanInstance`返回给定bean实例的对象，如果是FactoryBean则返回它创建的对象，否则返回bean实例本身。

如果`getSingleton`方法返回的是null，则说明这个bean没有被注册为单例。于是开始另外的步骤：

1. 调用`getParentBeanFactory`方法返回父级bean factory。如果父级bean factory不为null，则调用父级bean factory的`getBean`方法来获取bean并返回。
2. 调用`getMergedLocalBeanDefinition`来获取bean definition。
3. 调用`BeanDefinition.getDependsOn`方法获取这个bean依赖的bean的名称。如果依赖的bean不为空，则调用`registerDependentBean`来注册这个bean所依赖的bean，这样当这个bean被销毁时同时销毁这些依赖的bean。然后调用`getBean`来实例化这些依赖的bean
4. 如果bean是单例。调用`getSingleton`方法返回该bean的单例，接着调用`getObjectForBeanInstance`返回给定bean实例的对象，如果是FactoryBean则返回它创建的对象，否则返回bean实例本身。
5. 如果bean是原型(`prototype`)。先调用`beforePrototypeCreation`执行原型创建前的回调，然后调用`createBean`创建bean实例，接着调用`afterPrototypeCreation`执行原型创建后的回调。最后调用`getObjectForBeanInstance`返回给定bean实例的对象，如果是FactoryBean则返回它创建的对象，否则返回bean实例本身。
6. 如果bean既不是单例也不是原型，则在其`scopes`map中获取另外的scope，然后调用scope的`get`方法获取bean实例，最后调用`getObjectForBeanInstance`返回给定bean实例的对象，如果是FactoryBean则返回它创建的对象，否则返回bean实例本身。

## DefaultSingletonBeanRegistry.getSingleton

bean的实例化过程中最核心的方法是：`getSingleton`，其中的调用方式是：

```java
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
```

可以看到调用`getSingleton`方法的同时传入一个`ObjectFactory`，其中`getObject`方法中调用`createBean`方法来创建bean实例。

首先来看`getSingleton`本身：

1. 首先从`beanFactory`的`singletonObjects`中获取相应名称的单例bean。如果返回的单例bean不为null则直接返回，否则进入后面的步骤。
2. 调用`beforeSingletonCreation`执行单例创建前的回调
3. 调动`singletonFactory.getObject`方法获得单例实例
4. 调用`afterSingletonCreation`执行单例创建后的回调
5. 调用`addSingleton`方法将给定的单例对象添加到factory的单例缓存中

### createBean

可以看到创建单例的过程中最核心的步骤是调用`singletonFactory.getObject`方法获得单例实例，而`getObject`调用的是`AbstractAutowireCapableBeanFactory.createBean`方法创建bean实例。

我们来分析一下这个核心方法：

1. 调用`resolveBeanClass`方法，确保bean class被真正地解析。然后克隆一份bean definition
2. 调用`AbstractBeanDefinition.prepareMethodOverrides`方法，验证并准备方法重写。
3. 调用`resolveBeforeInstantiation`方法，在bean初始化前调用`BeanPostProcessor`的`postProcessBeforeInstantiation`。如果调用`postProcessBeforeInstantiation`方法之后返回了一个非null的bean，则马上调用`BeanPostProcessor`的`postProcessAfterInitialization`方法，如果这1次/2次调用之后返回的bean非null，则返回这个bean。否则进入下面的步骤。
4. 最后调用`doCreateBean`方法，得到创建的实例并返回

#### doCreateBean

这是真正创建bean的方法。它的步骤如下：

1. 调用`createBeanInstance`方法，为特定的bean创建一个新的实例，使用恰当的实例化策略：工程方法，构造函数自动装配，简单实例化
2. 根据`createBeanInstance`方法返回的`BeanWrapper`类型，获取对实例象以及对象Class
3. 对bean定义调用`MergedBeanDefinitionPostProcessor`的`postProcessMergedBeanDefinition`方法
4. 调用`populateBean`，使用bean definition中的属性值来填充bean实例
5. 调用`initializeBean`，初始化给定的bean实例，包括调用factory的回调，以及init方法和bean post processor
6. 最后调用`registerDisposableBeanIfNecessary`

下面分别对用到的几个核心方法做分析

##### createBeanInstance

1. 调用`resolveBeanClass`确保bean class已经被解析过了。
2. 如果bean中有工厂方法，则返回调用`instantiateUsingFactoryMethod`方法返回使用工厂方法实例化的bean。否则进入下面的步骤
3. 如果bean基于构造器的依赖关系，则调用`autowireConstructor`方法使用构造器实例化bean并返回
4. 否则，调用`instantiateBean`使用默认的构造器实例化给定的bean并返回

##### populateBean

1. 调用`getPropertyValues`方法获取属性值
2. 调用`InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation`来处理实例化之后的bean
3. 如果bean是通过名称进行依赖注入的，则调用`autowireByName`填充缺失的属性值；如果bean是通过类型进行依赖注入的，则调用`autowireByType`填充缺失的属性值。
4. 调用`InstantiationAwareBeanPostProcessor.postProcessPropertyValues`在属性值被赋予bean之前处理给定的属性值
5. 最后调用`applyPropertyValues`应用属性值

##### initializeBean

1. 调用`invokeAwareMethods`。如果bean实现了`BeanNameAware`接口，调用`setBeanName`设置bean名称；如果bean实现了`BeanClassLoaderAware`接口，调用`setBeanClassLoader`设置classLoader；如果bean实现了`BeanFactoryAware`接口，调用`setBeanFactory`设置beanFactory
2. 调用`BeanPostProcessor`的`postProcessBeforeInitialization`方法
3. 调用`invokeInitMethods`，检查bean是否实现了`InitializingBean`，或者自定义了一个init方法，如果有则调用必要的回调函数。
4. 调用`BeanPostProcessor`的`postProcessAfterInitialization`方法

