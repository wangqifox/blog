---
title: Spring启动过程分析.番外(@Value)
date: 2018/03/14 16:32:00
---

这篇文章我们来分析一下经过@Value注释的变量时如何注入属性值的。

@Value属性值的注入发生在调用`populateBean`给实例化完成的bean填充属性之时。

遍历`BeanPostProcessor`列表，当调用`AutowiredAnnotationBeanPostProcesor`的`postProcessPropertyValues`方法时：

```java
public PropertyValues postProcessPropertyValues(
		PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeanCreationException {
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

可以看到，首先调用`findAutowiringMetadata`函数来找到需要注入的元数据，然后调用`inject`方法来注入相应的属性值。

## InjectMetadata.inject(Object target, String beanName, PropertyValues pvs)

在`inject`方法中遍历`InjectedElement`元素，调用`AutowiredFieldElement.inject`方法：

1. 调用`value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter)`解析表达式
2. 调用`field.set(bean, value)`将解析的值设置到相应的字段中

## DefaultListableBeanFactory.resolveDependency

该方法针对bean定义解析特定的依赖，它最终调用的是`DefaultListableBeanFactory.doResolveDependency`方法：

1. 调用`AutowireCandidateResolver.getSuggestedValue(descriptor)`获得@Value注释中的描述信息。
2. 如果这个描述信息非null且是String类型，调用`resolveEmbeddedValue`来解析描述信息的值。默认情况下调用`AbstractPropertyResolver.resolvePlaceholders`来解析这个表达式
3. 根据需要转换的类型，将解析表达式获得的值转换成响应的类型






