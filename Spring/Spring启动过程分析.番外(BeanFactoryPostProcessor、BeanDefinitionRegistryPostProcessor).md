---
title: Spring启动过程分析.番外(BeanFactoryPostProcessor、BeanDefinitionRegistryPostProcessor)
date: 2018/03/07 15:46:00
---

今天复习Spring启动过程的时候，当分析`invokeBeanFactoryPostProcessors`方法的时候，发现自己对`BeanFactoryPostProcessor`和`BeanDefinitionRegistryPostProcessor`这两个接口还是有点一知半解，现在单独开一章来单独分析这两个接口。
<!-- more -->
先来看一下这两个接口定义的翻译：

BeanFactoryPostProcessor：

```java
/**
 * 允许对应用上下文中的bean定义进行自定义的修改，以适应上下文底层bean工厂的bean属性值
 * 应用上下文可以自动检测BeanFactoryPostProcessor的bean定义，在其他bean创建之前应用BeanFactoryPostProcessor。
 * 可以用于自定义配置文件以便配置上下文中的bean属性
 * 查看PropertyResourceConfigurer以及具体的实现了解配置需要的解决方案
 * BeanFactoryPostProcessor可以和bean定义交互以及修改bean定义，但是不能实例化bean。否则会导致过早实例化，违反容器规则引起意外的副作用。如果需要和bean示例交互，可以考虑使用BeanPostProcessor来替代。
 */
public interface BeanFactoryPostProcessor {
	/**
	 * 修改上下文内部的bean factory在其标准初始化之后。所有的bean定义都会被加载，但是不会实例化。允许对bean定义重写或添加属性。
	 */
	void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

BeanDefinitionRegistryPostProcessor：

```java
/**
 * 对BeanFactoryPostProcessor的扩展，在检测BeanFactoryPostProcessor之前进一步注册bean定义。
 * 特别是，BeanDefinitionRegistryPostProcessor可以注册bean定义反过来定义BeanFactoryPostProcessor实例。
 */
public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {

	/**
	 * 修改应用上下文内部的bean定义注册器，在其标准初始化之后。所有常规的bean定义都会被加载，但是没有bean被实例化。
	 * 允许在下一个post-processing阶段之前增加更多的bean定义
	 */
	void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException;

}
```

## 使用

来看一下这两个接口的使用。主要在`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors`方法中使用。

### BeanDefinitionRegistryPostProcessor

首先获取类型上匹配`BeanDefinitionRegistryPostProcessor`的类名称：

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
```

默认情况下，这里获取到的`postProcessorNames`只有一个值：`org.springframework.context.annotation.internalConfigurationAnnotationProcessor`。这个值是在`AnnotationConfigApplicationContext`创建过程中，创建`AnnotatedBeanDefinitionReader`时加入的。`AnnotatedBeanDefinitionReader`构造函数中调用`AnnotationConfigUtils.registerAnnotationConfigProcessors`，在这个方法中，通过以下代码加入了`internalConfigurationAnnotationProcessor`的beanDefinition：

```java
if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
	RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
	def.setSource(source);
	beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
}
```

获取名称之后再调用`beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class)`获取这个bean，比如这里获取到的就是`ConfigurationClassPostProcessor`。

接着调用`ConfigurationClassPostProcessor`的`postProcessBeanDefinitionRegistry`方法，进行配置类的处理，包括：扫描目录加入所有的beanDefinition

到此为止，`BeanDefinitionRegistryPostProcessor`的工作完成了，此时所有的beanDefinition已经被加载。

总结一下，到目前为止，beanDefinition的映射表为：

- `org.springframework.context.annotation.internalConfigurationAnnotationProcessor` -> `org.springframework.context.annotation.ConfigurationClassPostProcessor`
- `org.springframework.context.event.internalEventListenerFactory` -> `org.springframework.context.event.DefaultEventListenerFactory`
- `org.springframework.context.event.internalEventListenerProcessor` -> `org.springframework.context.event.EventListenerMethodProcessor`
- `org.springframework.context.annotation.internalAutowiredAnnotationProcessor` -> `org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor`
- `org.springframework.context.annotation.internalCommonAnnotationProcessor` -> `org.springframework.context.annotation.CommonAnnotationBeanPostProcessor`
- `org.springframework.context.annotation.internalRequiredAnnotationProcessor` -> `org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor`
- `config` -> `EnhancerBySpringCGLIB`

其中前6个beanDefinition是在`AnnotationConfigApplicationContext`构造过程中由`AnnotationConfigUtils.registerAnnotationConfigProcessors`方法加入的。

`config`是Spring中我们的配置类，经过`ConfigurationClassPostProcessor`的`postProcessBeanFactory`方法的处理，包装了一个增强类。

### BeanFactoryPostProcessor

由于`BeanDefinitionRegistryPostProcessor`继承了`BeanFactoryPostProcessor`，因此需要调用`ConfigurationClassPostProcessor`的`postProcessBeanFactory`方法。该方法中使用`ConfigurationClassEnhancer`来增强配置类，以及增加一个`BeanPostProcessor`：`ImportAwareBeanPostProcessor`。

接着获取类型上匹配`BeanFactoryPostProcessor`的类名称：

```java
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);
```

然后调用`postProcessBeanFactory`方法

默认只有`ConfigurationClassPostProcessor`一个`BeanFactoryPostProcessor`，所以后面的并没有执行其他的`postProcessBeanFactory`方法。


