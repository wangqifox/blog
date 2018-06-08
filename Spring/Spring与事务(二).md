---
title: Spring与事务(二)
date: 2018/01/19 10:04:00
---

上回说到，使用事务的模板`TransactionTemplate`可以极大地减少我们使用事务时的工作，我们只需将我们的业务逻辑写到`TransactionCallback`接口方法中即可。

使用`TransactionTemplate`虽然帮我们省略了一些相同的操作，但是每次数据库操作都要写到`TransactionCallback`中，也业务逻辑还不是分离的。这就引出了AOP代理。

要将Spring AOP和事务结合起来，也有很多的表现形式，但原理都是一样的。

最简单的莫过于在Configuration类中增加`@EnableTransactionManagement`，Spring将标注有`@Transactional`的对象创建出代理对象。
<!-- more -->
## 生成beanDefinition

在Configuration类中增加了`@EnableTransactionManagement`，Spring启动时在解析Configuration时

调用`invokeBeanFactoryPostProcessors`生成beanDefinition

`invokeBeanFactoryPostProcessors` -> 
`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors` -> 
`invokeBeanDefinitionRegistryPostProcessors` -> 
`ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry(registry)` -> 
`processConfigBeanDefinitions(registry)` -> 
`ConfigurationClassParser.parse(candidates)` ->
`processConfigurationClass(configClass)` ->
`doProcessConfigurationClass(configClass, sourceClass)`

### invokeBeanDefinitionRegistryPostProcessors

在处理Import注释时，我们知道`@EnableTransactionManagement`中加了注释`@Import(TransactionManagementConfigurationSelector.class)`，因此会调用`ConfigurationClassParser.processImports`。

```java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, boolean checkForCircularImports) throws IOException {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            selector, this.environment, this.resourceLoader, this.registry);
                    if (this.deferredImportSelectors != null && selector instanceof DeferredImportSelector) {
                        this.deferredImportSelectors.add(
                                new DeferredImportSelectorHolder(configClass, (DeferredImportSelector) selector));
                    }
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                        processImports(configClass, currentSourceClass, importSourceClasses, false);
                    }
                }
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                    ParserStrategyUtils.invokeAwareMethods(
                            registrar, this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass));
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]", ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

importCandidates里传入的是`TransactionManagementConfigurationSelector`。

可以看到，因为传入的`TransactionManagementConfigurationSelector`是`ImportSelector`的子类，于是首先调用`AdviceModeImportSelector.selectImports`获取需要导入的类，能看到是`org.springframework.context.annotation.AutoProxyRegistrar`和`org.springframework.transaction.annotation.ProxyTransactionManagementConfiguration`。然后递归调用`processImports`。

接着看`AutoProxyRegistrar`，它是`ImportBeanDefinitionRegistrar`的子类。调用`ParserStrategyUtils.invokeAwareMethods`执行实现的`Aware`接口。然后调用`configClass.addImportBeanDefinitionRegistrar`在config类中增加importBeanDefinitionRegistrar。

接着看`ProxyTransactionManagementConfiguration`，它既不是`ImportSelector`类也不是`ImportBeanDefinitionRegistrar`类，于是进入else代码块，调用`processConfigurationClass`，将它作为一个配置类来处理。

查看`ProxyTransactionManagementConfiguration`类的代码，我们可以看到`ProxyTransactionManagementConfiguration`被`@Configuraion`注释，它还新建了3个bean：`BeanFactoryTransactionAttributeSourceAdvisor`、`TransactionAttributeSource`、`TransactionInterceptor`。

#### loadBeanDefinitions

回到`ConfigurationClassPostProcessor.processConfigBeanDefinitions`方法，经过`parser.parse`的解析，configClasses中包含了我们自定义的Configuration类以及`ProxyTransactionManagementConfiguration`，还包含通过ComponentScan扫描得到的类。

调用`ConfigurationClassBeanDefinitionReader.loadBeanDefinitions`:

- 加载自定义Configuration类中定义bean的beanDefinition。之前在importBeanDefinitionRegistrar中添加了`AutoProxyRegistrar`，因此调用`registerBeanDefinitions`方法注册了`InfrastructureAdvisorAutoProxyCreator`
- 加载`ProxyTransactionManagementConfiguration`以及其中四个bean（`BeanFactoryTransactionAttributeSourceAdvisor`、`TransactionAttributeSource`、`TransactionInterceptor`、`TransactionalEventListenerFactory`）的beanDefinition。

### invokeBeanFactoryPostProcessors

调用`ConfigurationClassPostProcessor.postProcessBeanFactory`：

- 使用cglib对Configuration类进行增强
- 增加一个`ImportAwareBeanPostProcessor`

## 注册beanPostProcessors

在调用`registerBeanPostProcessors`之前，`beanFactory`的`beanPostProcessors`中有三个beanPostProcessor：

1. ApplicationContextAwareProcessor
2. ApplicationListenerDetector
3. ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor

在调用`invokeBeanFacotryPostProcessors`之后，调用`registerBeanPostProcessors`注册`beanPostProcessor`。注册完之后`beanFactory`的`beanPostProcessors`中有8个beanPostProcessor：

1. ApplicationContextAwareProcessor
2. ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
3. PostProcessorRegistrationDelegate$BeanPostProcessorChecker
4. InfrastructureAdvisorAutoProxyCreator
5. CommonAnnotationBeanPostProcessor
6. AutowiredAnnotationBeanPostProcessor
7. RequiredAnnotationBeanPostProcessor
8. ApplicationListenerDetector


## 创建bean并初始化

与AOP代理类相同，bean创建工作在`DefaultListableBeanFactory.doCreateBean`中完成：

1. 调用`AbstractAutowireCapableBeanFactory.createBeanInstance`创建bean实例。
2. 调用`AbstractAutowireCapableBeanFactory.populateBean`填充bean实例。
3. 调用`AbstractAutowireCapableBeanFactory.initializeBean`初始化bean实例

在`initializeBean`方法中，调用`applyBeanPostProcessorsAfterInitialization`，它调用各个beanPostProcessor的`postProcessAfterInitialization`。

重点是`InfrastructureAdvisorAutoProxyCreator`，它创建了使用事务的类的代理类。

`AbstractAutoProxyCreator.postProcessAfterInitialization` ->
`AbstractAutoProxyCreator.wrapIfNecessary` ->

在`AbstractAutoProxyCreator.wrapIfNecessary`方法中：

```java
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
if (specificInterceptors != DO_NOT_PROXY) {
    this.advisedBeans.put(cacheKey, Boolean.TRUE);
    Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
    this.proxyTypes.put(cacheKey, proxy.getClass());
    return proxy;
}
```

### 寻找拦截器

Spring通过调用`AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean`方法来获取某个bean的所有advice。它实际调用的是`AbstractAdvisorAutoProxyCreator.findEligibleAdvisors`方法，为这个类寻找合适的advice。主要流程如下：

1. 调用`AbstractAdvisorAutoProxyCreator.findCandidateAdvisors()`方法寻找所有的advisor。其中会在Spring容器里寻找名称为`org.springframework.transaction.config.internalTransactionAdvisor`的bean对象，返回的是一个`BeanFactoryTransactionAttributeSourseAdvisor`类对象，它用于在一个使用事务的方法中加入事务通知。
2. 调用`AbstractAdvisorAutoProxyCreator.findAdvisorsThatCanApply()`方法在所有advisor中筛选适合这个类对象的advisor。

    它实际调用的是`AopUtils.canApply`方法来确定advisor的pointcut是否能匹配这个类对象，原理如下：
    
    1. 调用Pointcut的`getMethodMatcher`方法获取`MethodMatcher`对象。该对象是Pointcut的一部分，用于判断目标方法是否匹配advice。
    2. 获取目标对象的所有的方法，然后遍历这些方法，调用`BeanFactoryTransactionAttributeSourseAdvisor`父类`TransactionAttributeSourcePointcut`的`matches`方法来判断目标方法是否匹配Pointcut。其原理是通过是否能在目标方法的注释中找到事务属性(TransactionAttribute)来判断是否匹配Pointcut。

### 创建代理类

通过`getAdvicesAndAdvisorsForBean`方法找到一个拦截器`BeanFactoryTransactionAttributeSourceAdvisor`。因此需要创建一个代理类：

1. 调用`DefaultAopProxyFactory.createAopProxy`来创建一个AopProxy
2. 调用`CglibAopProxy.getProxy`来获取一个代理类

代理类中加入了`DynamicAdvisedInterceptor`、`StaticUnadvisedInterceptor`、`SerializableNoOp`、`StaticDispatcher`、`AdvisedDispatcher`、`EqualsInterceptor`、`HashCodeInterceptor`等一系列的拦截器。

## 执行@Transactional注释的方法

当调用@Transactional注释的方法时，调用的是`DynamicAdvisedInterceptor.intercept`。

首先执行`AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)`获取拦截器列表。调用流程如下：

1. DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice
2. DefaultAdvisorAdapterRegistry.getInterceptors方法返回TransactionInterceptor

获取到的拦截器链中有`TransactionInterceptor`。

接着新建`CglibMethodInvocation`对象，执行`ReflectiveMethodInvocation.proceed`方法

```java
public Object proceed() throws Throwable {
    //    We start with an index of -1 and increment early.
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

其中`interceptorOrInterceptionAdvice`中获得的拦截器是`TransactionInterceptor`，然后执行到`((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this)`语句，进入`TransactionInterceptor.invoke`，然后进入`TransactionAspectSupport.invokeWithinTransaction`。

```java
...

TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
Object retVal = null;
try {
    // This is an around advice: Invoke the next interceptor in the chain.
    // This will normally result in a target object being invoked.
    retVal = invocation.proceedWithInvocation();
}
catch (Throwable ex) {
    // target invocation exception
    completeTransactionAfterThrowing(txInfo, ex);
    throw ex;
}
finally {
    cleanupTransactionInfo(txInfo);
}
commitTransactionAfterReturning(txInfo);
return retVal;

...
```

调用`createTransactionIfNecessary`创建事务：

```java
protected TransactionInfo createTransactionIfNecessary(
        PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

    // If no name specified, apply method identification as transaction name.
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            status = tm.getTransaction(txAttr);
        }
        else {
            if (logger.isDebugEnabled()) {
                logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
                        "] because no transaction manager has been configured");
            }
        }
    }
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}
```

调用`tm.getTransaction`获取事务状态，具体可以参考[Spring与事务(一)]()。

接着调用`prepareTransactionInfo`封装一个事务信息的对象。

回到`TransactionAspectSupport.invokeWithinTransaction`方法，调用`createTransactionIfNecessary`根据需要创建一个事务之后，再调用回调函数：`invocation.proceedWithInvocation`调用用户代码，进入`ReflectiveMethodInvocation.proceed`，：

```java
public Object proceed() throws Throwable {
    //    We start with an index of -1 and increment early.
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

调用`invokeJoinPoint`执行用户的业务代码：

```java
protected Object invokeJoinpoint() throws Throwable {
    if (this.publicMethod) {
        return this.methodProxy.invoke(this.target, this.arguments);
    }
    else {
        return super.invokeJoinpoint();
    }
}
```

回到`TransactionAspectSupport.invokeWithinTransaction`方法。如果`invocation.proceedWithInvocation`执行成功，则调用`commitTransactionAfterReturning`提交事务；否则调用`completeTransactionAfterThrowing`回滚事务。

