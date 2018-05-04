---
title: Spring与AOP
date: 2018/03/18 10:53:00
---

AOP是Spring的两大特性之一，本文再来回顾一下AOP的原理

## 示例

首先定义一个切点：

```java
public interface Performance {
    void perform();
}

@Component
public class PerformanceImpl implements Performance {
    @Override
    public void perform() {
        System.out.println("performing...");
    }
}
```

然后定义一个切面：

```java
@Aspect
@Component
public class Audience {
    @Pointcut("execution(* love.test.Performance.perform(..))")
    public void performance() {}

    @Before("performance()")
    public void silenceCellPhone() {
        System.out.println("Silencing cell phones");
    }

    @After("performance()")
    public void after() {
        System.out.println("after");
    }

    @AfterReturning("performance()")
    public void applause() {
        System.out.println("CLAP CLAP CLAP");
    }
}
```

最后定义一个配置类和一个启动类：

```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan
public class ConcertConfig {
}

public class Main {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ConcertConfig.class);
        Performance performance = context.getBean(Performance.class);
        performance.perform();
        context.close();
    }
}
```

最后的输出为

```
Silencing cell phones
performing...
after
CLAP CLAP CLAP
```

## 解析配置类

执行到`invokeBeanFactoryPostProcessors`方法，获取实现了`BeanDefinitionRegistryPostProcessor`接口的类：`ConfigurationClassPostProcessor`。

在`ConfigurationClassPostProcessor`的`processConfigBeanDefinitions`方法中处理配置类。

首先调动`ConfigurationClassParser.parse`方法解析配置类。核心方法是`processImports(configClass, sourceClass, getImports(sourceClass), true)`。

首先调用`getImports(sourceClass)`方法获取配置类中的`Import`注释。这里只有一个：`org.springframework.context.annotation.AspectJAutoProxyRegistrar`。然后在`getImports`方法中加入到`ConfigurationClass`的`importBeanDefinitionRegistrars`中：

```java
Class<?> candidateClass = candidate.loadClass();
ImportBeanDefinitionRegistrar registrar = BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
ParserStrategyUtils.invokeAwareMethods(registrar, this.environment, this.resourceLoader, this.registry);
configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
```

## 加载beanDefinition

调用`this.reader.loadBeanDefinitions(configClass)`根据配置类加载额外的beanDefinition。
 在之前的配置类解析中，因为`@EnableAspectJAutoProxy`注释的存在，`ConfigurationClass`的`importBeanDefinitionRegistrars`中有一个key为`AspectJAutoProxyRegistrar`，value为`StandardAnnotationMetadata`的map。

因此在`loadBeanDefinitionsForConfigurationClass`方法中调用`loadBeanDefinitionsFromRegistrars`，进入`AspectJAutoProxyRegistrar.registerBeanDefinitions`方法，该方法会注册一个名为"org.springframework.aop.config.internalAutoProxyCreator"的beanDefintion，类为`org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator`。

## 注册BeanPostProcessor

经过前面加载了beanDefinition，在`registerBeanPostProcessors`方法中将`AnnotationAwareAspectJAutoProxyCreator`注册为`BeanPostProcessor`。

## bean实例化与初始化

bean创建工作在`DefaultListableBeanFactory.doCreateBean`中完成：

1. 调用`AbstractAutowireCapableBeanFactory.createBeanInstance`创建bean实例。
2. 调用`AbstractAutowireCapableBeanFactory.populateBean`填充bean实例。
3. 调用`AbstractAutowireCapableBeanFactory.initializeBean`初始化bean实例

在调用initializeBean来初始化bean实例的代码中，`applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName)`方法遍历所有的`BeanPostProcessor`，调用其`postProcessAfterInitialization`方法。

重点关注`AnnotationAwareAspectJAutoProxyCreator.postProcessAfterInitialization`，它调用`AbstractAutoProxyCreator.wrapIfNecessary`如果需要代理则返回经过包装的代理类：

1. 调用`AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean`寻找指定bean的Advisor。
2. 如果返回的advisor不为null，说明需要创建代理。调用`createProxy`方法对给定的bean创建一个AOP代理，其调用`ProxyFactory.getProxy`方法，该方法只有一行代码:

```java
public Object getProxy(ClassLoader classLoader) {
	return createAopProxy().getProxy(classLoader);
}
```

先调用`createAopProxy`创建`AopProxy`，再调用`getProxy`获取代理对象。

回到`AbstractAutowireCapableBeanFactory.initializeBean`方法，wrappedBean再经过`applyBeanPostProcessorsAfterInitialization`处理，调用`AbstractAutoProxyCreator.postProcessAfterInitialization`，wrappedBean变成了一个代理类。

## AOP代理对象的调用

当调用aop代理类的方法时，调用的实际上是`JdkDynamicAopProxy.invoke`:

1. 首先调用`List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)`获取这个方法的拦截链
2. 调用`MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain)`创建MethodInvocation，将我们之前的代理、目标对象、拦截的method名称、拦截方法的参数、拦截器链全部整合到了ReflectiveMethodInvocation这个类中
3. 然后调用`ReflectiveMethodInvocation.proceed`执行一串拦截链


