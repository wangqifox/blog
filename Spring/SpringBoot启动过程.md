---
title: SpringBoot启动过程
date: 2018/03/02 10:46:25
---

SpringBoot的启动很简单，通用的代码如下：

```java
@SpringBootApplication
public class SpringBootDemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootDemoApplication.class, args);
    }
}
```

`SpringApplication.run`方法实际执行的方法如下：

```java
public static ConfigurableApplicationContext run(Object[] sources, String[] args) {
    return new SpringApplication(sources).run(args);
}
```
<!-- more -->
## 初始化SpringApplication

SpringApplication的构造函数中调用了`initialize`方法来初始化SpringApplication：

```java
private void initialize(Object[] sources) {
    if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources));
    }
    this.webEnvironment = deduceWebEnvironment();
    setInitializers((Collection) getSpringFactoriesInstances(
            ApplicationContextInitializer.class));
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

1. 调用`deduceWebEnvironment`来判断当前的应用是否是web应用，并设置到`webEnvironment`属性中。`deduceWebEnvironment`方法通过获取

    ```java
    javax.servlet.Servlet
    org.springframework.web.context.ConfigurableWebApplicationContext
    ```
    
    这两个类来判断，如果能获得这两个类则说明是web应用，否则不是。
    
2. 调用`getSpringFactoriesInstances`从spring.factories文件中找出key为ApplicationContextInitializer的类并实例化，然后调用`setInitializers`方法设置到`SpringApplication`的`initializers`属性中。这个过程就是找出所有的应用程序初始化器。细节详见:[获取初始化器](#id1)
    
    当前的初始化器有如下几个:
    
    ```java
    DelegatingApplicationContextInitializer
    ContextIdApplicationContextInitializer
    ConfigurationWarningsApplicationContextInitializer
    ServerPortInfoApplicationContextInitializer
    SharedMetadataReaderFactoryContextInitializer
    AutoConfigurationReportLoggingInitializer
    ```

3. 调用`getSpringFactoriesInstances`从spring.factories文件中找出key为ApplicationListener的类并实例化，然后调用`setListeners`方法设置到`SpringApplication`的`listeners`属性中。这个过程就是找出所有的应用程序事件监听器。细节详见:[获取监听器](#id2)

    当前的事件监听器有如下几个：
    
    ```java
    ConfigFileApplicationListener
    AnsiOutputApplicationListener
    LoggingApplicationListener
    ClasspathLoggingApplicationListener
    BackgroundPreinitializer
    DelegatingApplicationListener
    ParentContextCloserApplicationListener
    ClearCachesApplicationListener
    FileEncodingApplicationListener
    LiquibaseServiceLocatorApplicationListener
    ```

4. 调用`deduceMainApplicationClass`方法找出main类，就是这里的`SpringBootDemoApplication`类

## 运行SpringApplication

### SpringApplicationRunListeners和SpringApplicationRunListener类

SpringApplicationRunListeners内部持有SpringApplicationRunListener集合和1个Log日志类。用于SpringApplicationRunListener监听器的批量执行。

SpringApplicationRunListener用于监听SpringApplication的run方法的执行，它定义了5个步骤：

1. starting：run方法执行的时候立马执行，对应的事件类型是ApplicationStartedEvent
2. environmentPrepared：ApplicationContext创建之前并且环境信息准备好的时候调用，对应的事件类型是ApplicationEnvironmentPreparedEvent
3. contextPrepared：ApplicationContext创建好并且在source加载之前调用一次，没有具体的对应事件
4. contextLoaded：ApplicationContext创建并加载之后并在refresh之前调用，对应的事件类型是ApplicationPreparedEvent
5. finished：run方法结束之前调用，对应事件的类型是ApplicationReadyEvent或ApplicationFailedEvent

SpringApplicationRunListener目前只有一个实现类EventPublishingRunListener，详见[获取SpringApplicationRunListeners](#id3)。它把监听的过程封装成了SpringApplicationEvent事件并让内部属性ApplicationEventMulticaster接口的实现类SimpleApplicationEventMulticaster广播出去，广播出去的事件对象会被SpringApplication中的listeners属性进行处理。

所以说SpringApplicationRunListener和ApplicationListener之间的关系是通过ApplicationEventMulticaster广播出去的SpringApplicationEvent所联系起来的。

### run方法

初始化SpringApplication完成之后，调用`run`方法运行：

```java
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch();    // 构造一个任务执行观察者
    stopWatch.start();    // 开始执行，记录开始时间
    ConfigurableApplicationContext context = null;
    FailureAnalyzers analyzers = null;
    configureHeadlessProperty();
    // 获取SpringApplicationRunListeners，内部只有一个EventPublishingRunListener
    SpringApplicationRunListeners listeners = getRunListeners(args);
    // 封装成SpringApplicationEvent事件然后广播出去给SpringApplication中的listeners所监听
    // 这里接受ApplicationStartedEvent事件的listener会执行相应的操作
    listeners.starting();
    try {
        // 构造一个应用程序参数持有类
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 准备并配置环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 打印banner图形
        Banner printedBanner = printBanner(environment);
        // 创建Spring容器
        context = createApplicationContext();
        analyzers = new FailureAnalyzers(context);
        // 配置Spring容器
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 容器上下文刷新，详见Spring的启动分析
        refreshContext(context);
        
        // 容器创建完成之后调用afterRefresh方法
        afterRefresh(context, applicationArguments);
        // 调用监听器，广播Spring启动结束的事件
        listeners.finished(context, null);
        // 停止任务执行观察者
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        return context;
    }
    catch (Throwable ex) {
        handleRunFailure(context, listeners, analyzers, ex);
        throw new IllegalStateException(ex);
    }
}
```

run方法执行完成之后，Spring容器也已经初始化完成，各种监听器和初始化器也做了相应的工作。具体步骤的分析见下文。

#### 配置并准备环境：

```java
private ConfigurableEnvironment prepareEnvironment(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
    // 创建应用程序的环境信息。如果是web程序，创建StandardServletEnvironment；否则，创建StandardEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 配置环境信息。比如profile，命令行参数
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    // 广播出ApplicationEnvironmentPreparedEvent事件给相应的监听器执行
    listeners.environmentPrepared(environment);
    // 环境信息的校对
    if (!this.webEnvironment) {
        environment = new EnvironmentConverter(getClassLoader())
                .convertToStandardEnvironmentIfNecessary(environment);
    }
    return environment;
}
```

#### 创建Spring容器上下文：

```java
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            // 判断是否是web应用，
            // 如果是则创建AnnotationConfigEmbeddedWebApplicationContext，否则创建AnnotationConfigApplicationContext
            contextClass = Class.forName(this.webEnvironment
                    ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
                    "Unable create a default ApplicationContext, "
                            + "please specify an ApplicationContextClass",
                    ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
}
```

#### 配置Spring容器上下文：

```java
private void prepareContext(ConfigurableApplicationContext context,
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments, Banner printedBanner) {
    // 设置Spring容器上下文的环境信息
    context.setEnvironment(environment);
    // Spring容器创建之后做一些额外的事
    postProcessApplicationContext(context);
    // SpringApplication的初始化器开始工作
    applyInitializers(context);
    // 遍历调用SpringApplicationRunListener的contextPrepared方法。目前只是将这个事件广播器注册到Spring容器中
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }

    // 把应用程序参数持有类注册到Spring容器中，并且是一个单例
    context.getBeanFactory().registerSingleton("springApplicationArguments",
            applicationArguments);
    if (printedBanner != null) {
        context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
    }

    // 加载sources，sources是main方法所在的类
    Set<Object> sources = getSources();
    Assert.notEmpty(sources, "Sources must not be empty");
    // 将sources加载到应用上下文中。最终调用的是AnnotatedBeanDefinitionReader.registerBean方法
    load(context, sources.toArray(new Object[sources.size()]));
    // 广播出ApplicationPreparedEvent事件给相应的监听器执行
    // 执行EventPublishingRunListener.contextLoaded方法
    listeners.contextLoaded(context);
}
```

#### Spring容器创建之后回调方法postProcessApplicationContext：

```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    // 如果SpringApplication设置了实例命名生成器，则注册到Spring容器中
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(
                AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                this.beanNameGenerator);
    }
    // 如果SpringApplication设置了资源加载器，设置到Spring容器中
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context)
                    .setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context)
                    .setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
}
```

#### 初始化器开始工作：

```java
protected void applyInitializers(ConfigurableApplicationContext context) {
    // 遍历每个初始化器，调用对应的initialize方法
    for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
    }
}
```

首先调用`getInitializers`方法获取之前取得的初始化器。之后调用初始化器的`initialize`方法。

#### Spring容器创建完成之后会调用afterRefresh方法：

```java
protected void afterRefresh(ConfigurableApplicationContext context,
        ApplicationArguments args) {
    callRunners(context, args);
}

private void callRunners(ApplicationContext context, ApplicationArguments args) {
    List<Object> runners = new ArrayList<Object>();
    // 找出Spring容器中ApplicationRunner接口的实现类
    runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
    // 找出Spring容器中CommandLineRunner接口的实现类
    runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
    // 对runners进行排序
    AnnotationAwareOrderComparator.sort(runners);
    // 遍历runners依次执行
    for (Object runner : new LinkedHashSet<Object>(runners)) {
        if (runner instanceof ApplicationRunner) {
            callRunner((ApplicationRunner) runner, args);
        }
        if (runner instanceof CommandLineRunner) {
            callRunner((CommandLineRunner) runner, args);
        }
    }
}
```


## 一些细节

### <span id="id1"/>获取初始化器

初始化器的获取由`SpringApplication.getSpringFactoriesInstances`方法完成：

```java
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
        Class<?>[] parameterTypes, Object... args) {
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 读取ApplicationContextInitializer的实现类
    Set<String> names = new LinkedHashSet<String>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 实例化ApplicationContextInitializer的实现类
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

`SpringFactoriesLoader.loadFactoryNames`方法获取`ApplicationContextInitializer`接口实现的类：

```java
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
    // 获取接口类的名称
    String factoryClassName = factoryClass.getName();
    try {
        // 获取FACTORIES_RESOURCE_LOCATION(META-INF/spring.factories)的多个位置
        Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        List<String> result = new ArrayList<String>();
        /**
         * urls有
         * spring-boot/META-INF/spring.factories
         * spring-beans/META-INF/spring.factories
         * spring-boot-autoconfigure/META-INF/spring.factories
         * 
         */
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            // 从META-INF/spring.factories文件中加载配置
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            // 从配置中读取ApplicationContextInitializer的实现类
            String factoryClassNames = properties.getProperty(factoryClassName);
            result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
        }
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() +
                "] factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```


### <span id="id2"/>获取监听器

获取监听器的方法与获取初始化器的方法一致，唯一的区别在于获取`org.springframework.context.ApplicationListener`接口的实现类

### <span id="id3"/>获取SpringApplicationRunListeners

首先看`getRunListeners`方法：

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(logger, getSpringFactoriesInstances(
            SpringApplicationRunListener.class, types, this, args));
}
```

可以看到通过调用构造函数来实例化`SpringApplicationRunListeners`，传入的参数有logger以及调用`getSpringFactoriesInstance`获得的`SpringApplicationRunListener`集合。

再看`getSpringFactoriesInstance`方法，它和获取初始化器的方法一样，获取的接口类型是`org.springframework.boot.SpringApplicationRunListener`。获取到的实现类为`org.springframework.boot.context.event.EventPublishRunListener`。

### SpringApplicationRunListeners的工作原理

以启动过程中的`listeners.starting()`方法为例：

```java
public void starting() {
    for (SpringApplicationRunListener listener : this.listeners) {
        listener.starting();
    }
}
```

`this.listeners`中只有一个元素：`EventPublishingRunListener`。它的`starting`方法如下：

```java
public void starting() {
    this.initialMulticaster
            .multicastEvent(new ApplicationStartedEvent(this.application, this.args));
}
```

其中`this.initialMulticaster`是`SimpleApplicationEventMulticaster`的实例。`multicastEvent`方法如下：

```java
public void multicastEvent(ApplicationEvent event) {
    multicastEvent(event, resolveDefaultEventType(event));
}

public void multicastEvent(final ApplicationEvent event, ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        Executor executor = getTaskExecutor();
        if (executor != null) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    invokeListener(listener, event);
                }
            });
        }
        else {
            invokeListener(listener, event);
        }
    }
}
```

首先调用`getApplicationListeners`方法，根据event的type获得`ApplicationListener`列表，其中type为`ApplicationStartedEvent`。`getApplicationListeners`方法中调用`retrieveApplicationListeners`获取支持eventType的listener:

```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
        ResolvableType eventType, Class<?> sourceType, ListenerRetriever retriever) {

    LinkedList<ApplicationListener<?>> allListeners = new LinkedList<ApplicationListener<?>>();
    Set<ApplicationListener<?>> listeners;
    Set<String> listenerBeans;
    synchronized (this.retrievalMutex) {
        // 获取所有的listener
        listeners = new LinkedHashSet<ApplicationListener<?>>(this.defaultRetriever.applicationListeners);
        listenerBeans = new LinkedHashSet<String>(this.defaultRetriever.applicationListenerBeans);
    }
    // 遍历listeners，调用supportsEvent判断listener是否支持eventType
    for (ApplicationListener<?> listener : listeners) {
        if (supportsEvent(listener, eventType, sourceType)) {
            if (retriever != null) {
                retriever.applicationListeners.add(listener);
            }
            allListeners.add(listener);
        }
    }
    // 遍历listenerBeans，调用supportsEvent判断listener是否支持eventType
    if (!listenerBeans.isEmpty()) {
        BeanFactory beanFactory = getBeanFactory();
        for (String listenerBeanName : listenerBeans) {
            try {
                Class<?> listenerType = beanFactory.getType(listenerBeanName);
                if (listenerType == null || supportsEvent(listenerType, eventType)) {
                    ApplicationListener<?> listener =
                            beanFactory.getBean(listenerBeanName, ApplicationListener.class);
                    if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
                        if (retriever != null) {
                            retriever.applicationListenerBeans.add(listenerBeanName);
                        }
                        allListeners.add(listener);
                    }
                }
            }
            catch (NoSuchBeanDefinitionException ex) {
                // Singleton listener instance (without backing bean definition) disappeared -
                // probably in the middle of the destruction phase
            }
        }
    }
    AnnotationAwareOrderComparator.sort(allListeners);
    return allListeners;
}
```

`ApplicationStartedEvent`事件返回的是4个listener：

- LoggingApplicationListener
- BackgroundPreinitializer
- DelegatingApplicationListener
- LiquibaseServiceLocatorApplicationListener

回到`multicastEvent`方法，调用`getTaskExecutor`获取executor。executor为空，调用`invokeListener`：

```java
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();
    if (errorHandler != null) {
        try {
            doInvokeListener(listener, event);
        }
        catch (Throwable err) {
            errorHandler.handleError(err);
        }
    }
    else {
        doInvokeListener(listener, event);
    }
}
```

获取errorHandler，errorHandler为空，调用`doInvokeListener`方法：

```java
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
    try {
        listener.onApplicationEvent(event);
    }
    catch (ClassCastException ex) {
        String msg = ex.getMessage();
        if (msg == null || msg.startsWith(event.getClass().getName())) {
            // Possibly a lambda-defined listener which we could not resolve the generic event type for
            Log logger = LogFactory.getLog(getClass());
            if (logger.isDebugEnabled()) {
                logger.debug("Non-matching event type for listener: " + listener, ex);
            }
        }
        else {
            throw ex;
        }
    }
}
```

可以看到，`doInvokeListener`方法直接调用了listener的`onApplicationEvent`方法。


