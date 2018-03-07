---
title: Spring启动过程分析.番外(BeanPostProcessor)
date: 2018/03/07 20:22:00
---

上一篇文章我们分析了`invokeBeanFactoryPostProcessors`方法中`BeanFactoryPostProcessor`和`BeanDefinitionRegistryPostProcessor`这两个接口发挥的作用。紧接着，Spring调用了`registerBeanPostProcessors`接口在上下文中注册`BeanPostProcessor`。这篇文章我们来分析一下`BeanPostProcessor`接口发挥的作用。
<!-- more -->
首先看一下`BeanPostProcessor`定义的翻译：

```java
/**
 * 允许对新的bean实例进行自定义的修改
 * 应用上下文可以自动检测到BeanPostProcessor的定义，在随后的bean创建中可以使用它们。普通的bean工厂允许注册post-processors，适用于通过该工程创建的所有bean。
 * 通常，通过标记接口填充bean的post-processors会实现postProcessBeforeInitialization，使用代理包装bean的post-processor会实现postProcessAfterInitialization
 */
public interface BeanPostProcessor {
    /**
     * 在任何bean初始化回调(比如，InitializingBean的afterPropertiesSet或者自定义的init-method)之前，将BeanPostProcessor应用于新的bean实例。返回的bean实例可能是经过包装的实例或者是原始的实例。
     */
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    
    /**
     * 在任何bean初始化回调(比如，InitializingBean的afterPropertiesSet或者自定义的init-method)之后，将BeanPostProcessor应用于新的bean实例。bean已经使用属性值进行了填充。返回的bean实例可能是经过包装的实例或者是原始的实例。
     * 对于FactoryBean来说，这个回调会被FactoryBean实例以及FactoryBean创建的对象来调用。post-processor可以决定是适用于FactoryBean或者被创建的对象还是通过相应的bean instanceof FactoryBean来进行检查。
     * 这个回调也可以在InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation方法引发短路之后再进行调用，与其他的BeanPostProcessor回调进行对比。
     */
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

## registerBeanPostProcessors

接着再来看`registerBeanPostProcessors`方法的工作，它直接委托给`PostProcessorRegistrationDelegate.registerBeanPostProcessors`方法来执行。

首先调用`beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false)`来获取所有匹配`BeanPostProcessor`的类名。默认情况下能获得三个类名：

- `org.springframework.context.annotation.internalAutowiredAnnotationProcessor`
- `org.springframework.context.annotation.internalRequiredAnnotationProcessor`
- `org.springframework.context.annotation.internalCommonAnnotationProcessor`

这三个值是在`AnnotationConfigApplicationContext`创建过程中，创建`AnnotatedBeanDefinitionReader`时加入的。`AnnotatedBeanDefinitionReader`构造函数中调用`AnnotationConfigUtils.registerAnnotationConfigProcessors`，在这个方法中，通过以下代码加入了这三个类的beanDefinition：

```java
if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
	RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
	def.setSource(source);
	beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
}

if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
	RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
	def.setSource(source);
	beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
}

if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
	RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
	def.setSource(source);
	beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
}	
```

获取到这三个`BeanPostProcessor`的名称之后，往beanFactory中加了一个名为`BeanPostProcessorChecker`的`BeanPostProcessor`。

注意！这个时候beanFactory的`beanPostProcessors`列表中已经存在了3个`BeanPostProcessor`：

- `ApplicationContextAwareProcessor`
- `ApplicationListenerDetector`
- `ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor`

其中：`ApplicationContextAwareProcessor`和`ApplicationListenerDetector`是在`prepareBeanFactory`方法中加入的。`ImportAwareBeanPostProcessor`是调用`ConfigurationClassPostProcessor`的`postProcessBeanFactory`方法是加入的。

接着将这三个`BeanPostProcessor`按顺序注册到beanFactory中，添加在beanFactory的`beanPostProcessors`列表的末尾。

最后再注册一个`ApplicationListenerDetector`到beanFactory中，由于`ApplicationListenerDetector`已经存在于beanFactory的`beanPostProcessors`列表中，所以这里是先将原来的`ApplicationListenerDetector`删除然后在`beanPostProcessors`列表的末尾加入新的`ApplicationListenerDetector`。

`registerBeanPostProcessors`方法执行完成之后，beanFactory的`beanPostProcessors`列表中总共有7个`BeanPostProcessor`：

- `ApplicationContextAwareProcessor`
- `ConfigurationClassPostProcessor.ImportAwareBeanPostProcessor`
- `PostProcessorRegistrationDelegate.BeanPostProcessorChecker`
- `CommonAnnotationBeanPostProcessor`
- `AutowiredAnnotationBeanPostProcessor`
- `RequiredAnnotationBeanPostProcessor`
- `ApplicationListenerDetector`

