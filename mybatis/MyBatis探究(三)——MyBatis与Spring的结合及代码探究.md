---
title: MyBatis探究(三)——MyBatis与Spring的结合及代码探究
date: 2018/04/18 16:39:00
---

MyBatis与Spring的结合依赖于MyBatis-Spring。MyBatis-Spring会帮助你将MyBatis代码无缝地整合到Spring中。使用这个类库中的类，Spring将会加载必要的MyBatis工厂类和session类。这个类库也提供一个简单的方式来注入MyBatis数据映射器和SqlSession到业务层的bean中。而且它也会处理事务，翻译MyBatis的异常到Spring的DataAccessException异常（数据异常）。
<!-- more -->

# Mybatis-Spring配置

## 配置SqlSessionFactory

SqlSessionFactory的作用是生成SqlSession。MyBatis-Spring项目提供了`SqlSessionFactoryBean`类给我们去配置。一般而言，我们需要给出两个参数，一个是数据源，另一个是MyBatis的配置文件路径。示例如下：

```java
@Bean
public SqlSessionFactory sqlSessionFactory() throws Exception {
    SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(dataSource());
    PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
    sqlSessionFactoryBean.setMapperLocations(resolver.getResources("classpath:/love/wangqi/dao/xml/*.xml"));
    return sqlSessionFactoryBean.getObject();
}
```

## 配置SqlSessionTemplate

SqlSessionTemplate是一个模板类，通过调用SqlSession来完成工作。它有两种构建方法，一种是只有一个SqlSessionFactory作为参数；另一种有两个参数，一个是SqlSessionFactory，另一个是执行器类型，它是一个枚举类(`org.apache.ibatis.session.ExecutorType`)。示例如下：

```java
@Bean
public SqlSessionTemplate sqlSessionTemplate() throws Exception {
    SqlSessionTemplate sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory());
    return sqlSessionTemplate;
}
```

SqlSessionTemplate一般不建议使用。应尽量采用Mapper接口的编程方式，更符合面向对象的编程。

## 配置Mapper

### MapperFactoryBean

在MyBatis中，Mapper只需要一个接口，而不是一个实现类，它是由MyBatis体系通过动态代理的形式生成代理对象去运行的，所以Spring也没有办法为其生成实现类。为了处理这个问题，MyBatis-Spring提供了一个MapperFactoryBean类作为中介，我们可以通过配置它来实现我们想要的Mapper。配置MapperFactoryBean有3个参数：

1. mapperInterface：用来制定接口，当我们的接口继承了配置的接口，那么MyBatis就认为它是一个Mapper
2. SqlSessionFactory：当SqlSessionTemplate属性不被配置的时候，MyBatis-Spring才会去设置它。
3. SqlSessionTemplate：当它被设置的时候，SqlSessionFactory将被作废。

实例如下：

```java
@Bean
public MapperFactoryBean userMapper() throws Exception {
    MapperFactoryBean<UserMapper> mapperFactoryBean = new MapperFactoryBean<>();
    mapperFactoryBean.setSqlSessionFactory(sqlSessionFactory());
    mapperFactoryBean.setMapperInterface(UserMapper.class);
    return mapperFactoryBean;
}
```

这样我们就可以使用这个接口进行编程了，它的效果等同于sqlSession.getMapper(UserMapper.class)。

### MapperScannerConfigurer

一个复杂的系统存在许许多多的Mapper，如果需要一个一个配置，那么工作量会很大，不过MyBatis-Spring中提供了MapperScannerConfigurer来自动扫描我们的映射器，大大提高了效率。我们可以配置一下几个属性：

- basePackage：指定让Spring自动扫描什么包，它会逐层深入扫描
- annotationClass：表示如果类被这个注解标识的时候，才进行扫面
- sqlSessionFactoryBeanName：指定在Spring中定义sqlSessionFactory的bean名称。如果它被定义，sqlSessionFactory将不起作用。
- sqlSessionTemplateBeanName：指定在Spring中定义sqlSessionTemplate的bean的名称。如果它被定义，sqlSessionFactoryBeanName将不起作用。
- markerInterface：指定是实现了什么接口就认为它是Mapper。我们需要提供一个公共的接口去标记。

示例如下：

```java
@Bean
public MapperScannerConfigurer mapperScannerConfigurer() {
    MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
    mapperScannerConfigurer.setBasePackage("love.wangqi.dao");
    return mapperScannerConfigurer;
}
```

## 配置事务

MyBatis和Spring结合后是使用Spring AOP去管理事务的，使用Spring AOP是相当简单的，它分为声明式事务和编程式事务两种。大部分场景下使用声明式事务就可以了。

首先需要定义事务处理器：

```java
@Bean
public DataSourceTransactionManager transactionManager() {
    DataSourceTransactionManager transactionManager = new DataSourceTransactionManager();
    transactionManager.setDataSource(dataSource());
    return transactionManager;
}
```

然后开启Spring的事务管理：在Config类加上`@EnableTransactionManagement`注解。

Spring会自动使用任意存在的容器事务，在上面附加一个SqlSession。如果没有开启事务，或者需要基于事务配置，Spring会开启一个新的容器管理事务。

# 代码解析

## SqlSessionFactory的生成

SqlSessionFactory由`SqlSessionFactoryBean.getObject()`方法来创建：

```java
public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
        afterPropertiesSet();
    }
    return this.sqlSessionFactory;
}
```

`getObject()`方法调用`afterPropertiesSet()`来生成`sqlSessionFactory`，`afterPropertiesSet()`方法调用`buildSqlSessionFactory()`方法，这个是真正构建`SqlSessionFactory`的方法：

1. 根据配置构建`Configuration`
2. 根据`Configuration`调用`SqlSessionFactoryBuilder.build`方法来构建SqlSessionFactory

这两个步骤和纯Mybatis构建`SqlSessionFactory`的步骤相似。

## Mapper构建

MyBatis-Spring相比于纯MyBatis提供了`MapperFactoryBean`和`MapperScannerConfigurer`来协助生成Mapper的bean实例，代替了MyBatis从`SqlSession`中手动获取Mapper，充分利用了Spring依赖注入的特性。

### 生成BeanDefinition

#### MapperFactoryBean

首先分析一下MapperFactoryBean的工作原理。在Spring中bean实例的生成主要分为两步：生成BeanDefinition、生成Mapper实例。

以我们的例子为例，userMapper在Config文件中被定义，因此它的BeanDefinition在处理Config文件的时候生成。

代码的调用流程如下：

1. AbstractApplicationContext.refresh
2. AbstractApplicationContext.invokeBeanFactoryPostProcessors
3. PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors
4. PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors
5. ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry
6. ConfigurationClassPostProcessor.processConfigBeanDefinitions

beanDefinition的生成在`processConfigBeanDefinitions`方法中完成，重点关注两个步骤：

1. 首先调用`parse`方法解析Config类，调用流程如下：

    1. ConfigurationClassParser.parse
    2. ConfigurationClassParser.processConfigurationClass
    3. ConfigurationClassParser.doProcessConfigurationClass
    
    在`doProcessConfigurationClass`方法中处理Config类中被`@Bean`注释的方法：
    
    ```java
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }
    ```
    
    这段代码将Config类中被`@Bean`注释的方法封装成`BeanMethod`类，保存在configClass中。

2. 调用`loadBeanDefinitions`方法加载configClass中注册的BeanDefinition，调用流程如下：

    1. ConfigurationClassBeanDefinitionReader.loadBeanDefinitions
    2. ConfigurationClassBeanDefinitionReader.loadBeanDefinitionsForConfigurationClass
    
    在`loadBeanDefinitionsForConfigurationClass`方法中加载Config类中被`@Bean`注释的方法：
    
    ```java
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }
    ```
    
    遍历Config类中所有的BeanMethod，调用`loadBeanDefinitionForBeanMethod`将其加载为BeanDefinition，真实的类为`ConfigurationClassBeanDefinitionReader$ConfigurationClassBeanDefinition`。
    
#### MapperScannerConfigurer
    
`MapperScannerConfigurer`和`MapperFactoryBean`一样，也定义在Config类中，使用`@Bean`注释。因此`MapperScannerConfigurer`本身BeanDefintion的生成流程和`MapperFactoryBean`是一样的。

不同的是`MapperScannerConfigurer`继承了`BeanDefinitionRegistryPostProcessor`，它是一个可以注册BeanDefinition的类。

在`PostProcessorRegistrationDelegate.invokeBeanFactoryProcessors`方法中，多次执行了`invokeBeanDefinitionRegistryPostProcessors`方法，它会调用`MapperScannerConfigurer`中的`postProcessBeanDefinitionRegistry`方法：

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    if (this.processPropertyPlaceHolders) {
        processPropertyPlaceHolders();
    }

    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);
    scanner.setAddToConfig(this.addToConfig);
    scanner.setAnnotationClass(this.annotationClass);
    scanner.setMarkerInterface(this.markerInterface);
    scanner.setSqlSessionFactory(this.sqlSessionFactory);
    scanner.setSqlSessionTemplate(this.sqlSessionTemplate);
    scanner.setSqlSessionFactoryBeanName(this.sqlSessionFactoryBeanName);
    scanner.setSqlSessionTemplateBeanName(this.sqlSessionTemplateBeanName);
    scanner.setResourceLoader(this.applicationContext);
    scanner.setBeanNameGenerator(this.nameGenerator);
    scanner.registerFilters();
    scanner.scan(StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}
```

`MapperScannerConfigurer.postProcessBeanDefinitionRegistry`方法中调用`ClassPathBeanDefinitionScanner.scan`方法来扫描package下定义的接口，它的调用流程如下：

1. ClassPathBeanDefinitionScanner.scan
2. ClassPathMapperScanner.doScan
3. ClassPathBeanDefinitionScanner.doScan
4. ClassPathMapperScanner.processBeanDefinition

`ClassPathMapperScanner.doScan`方法首先调用`ClassPathBeanDefinitionScanner.doScan`扫描指定的package，生成并注册BeanDefinition。BeanDefinition的真实类为`ScannedGenericBeanDefinition`。

然后调用`ClassPathMapperScanner.processBeanDefinitions`来处理`doScan`方法返回的BeanDefinition，主要的功能就是：指定这个bean对应的类为`MapperFactoryBean`，设置一些属性值。

### 生成Mapper实例

根据我们的例子，Mapper实例在我们生成被依赖实例的时候生成。换句话说，在`UserService`中注入`UserMapper`的时候生成`UserMapper`实例。调用流程如下：

1. AbastractApplicationContext.finishBeanFactoryInitialization
2. DefaultListableBeanFactory.preInstantiateSingletons
3. AbstractBeanFactory.getBean
4. AbstractBeanFactory.doGetBean
5. DefaultSingletonBeanRegistry.getSingleton
6. ObjectFactory.getObject
7. AbstractAutowireCapableBeanFactory.createBean
8. AbstractAutowireCapableBeanFactory.doCreateBean

在`doCreateBean`方法中调用`createBeanInstance`创建`UserService`的bean实例，接着调用`populateBean`方法填充依赖的对象。调用流程如下：

1. AbstractAutowireCapableBeanFactory.populateBean
2. AutowiredAnnotationBeanPostProcessor.postProcessPropertyValues
3. InjectionMetadata.inject
4. AutowiredAnnotationBeanPostProcessor$AutowiredFieldElement.inject
5. DefaultListableBeanFactory.resolveDependency
6. DefaultListableBeanFactory.doResolveDependency
7. DependencyDescriptor.resolveCandidate

`resolveCandidate`方法根据beanName、requiredType、beanFactory获取userMapper的bean实例。调用beanFactory.getBean方法来获取userMapper的bean实例。

1. AbstractBeanFactory.getBean
2. AbstractBeanFactory.doGetBean
3. DefaultSingletonBeanRegistry.getSingleton
4. ObjectFactory.getObject
5. AbstractAutowireCapableBeanFactory.createBean
6. AbstractAutowireCapableBeanFactory.doCreateBean

`AbstractAutowireCapableBeanFactory.doCreateBean`为`userMapper`创建了类为`MapperFactoryBean`的实例。

回到`AbstractBeanFactory.doGetBean`，调用`DefaultSingletonBeanRegistry.getSingleton`获得`MapperFactoryBean`的实例之后接着调用`AbstractBeanFactory.getObjectForBeanInstance`方法检查bean实例是否是`FactoryBean`，如果是则返回由`FactoryBean`创建的Bean实例。调动流程如下：

1. AbstractBeanFactory.getObjectForBeanInstance
2. FactoryBeanRegistrySupport.getObjectFromFactoryBean
3. FactoryBeanRegistrySupport.doGetObjectFromFactoryBean
4. factory.getObject

`MapperFactoryBean.getObject`方法的代码如下：

```java
public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
}
```

可以看到`getObject`方法和原生Mybatis是一样的，通过SqlSession来获得UserMapper的实例，我们知道它是使用`MapperProxy`构建的代理类。

## 与Spring事务的结合

### 事务代理类的生成

我们来看看一段最简单的事务代码是如何执行的，它处于UserService中：

```java
@Transactional
public void insertUser(User user) {
    userMapper.insetUser(user);
}
```

我们知道，userService的bean生成之后，需要调用`populateBean`方法来填充bean实例和`initializeBean`方法来初始化bean。`initializeBean`方法中会调用`AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization`来遍历所有BeanPostProcessor的`postProcessAfterInitialization`方法。

因为在Config文件中加入了`@EnableTransactionManagement`注释开启了事务支持，因此加入了名为`InfrastructureAdvisorAutoProxyCreator`的BeanPostProcessor。在`InfrastructureAdvisorAutoProxyCreator.postProcessAfterInitialization`方法中返回经过动态代理包装的UserService实例。调用流程如下：

1. AbstractAutowireCapableBeanFactory.doCreateBean
2. AbstractAutowireCapableBeanFactory.initializeBean
3. AbstractAutowireCapableBeanFactory.applyBeanPostProcessorsAfterInitialization
4. AbstractAutoProxyCreator.postProcessAfterInitialization
5. AbstractAutoProxyCreator.wrapIfNecessary

在`AbstractAutoProxyCreator.wrapIfNecessary`方法中调用`AbstractAdvisorAutoProxyCreator.getAdvicesAndAdvisorsForBean`获取Advisor列表。这里能获取到一个`BeanFactoryTransactionAttributeSourceAdvisor`。

Advisor列表不为空，则调用`AbstractAutoProxyCreator.createProxy`来创建代理。

因此我们可以发现UserService最后生成的是一个由`JdkDynamicAopProxy`支持的代理类。

### 事务代理类的执行

通过上文事务代理类的生成，我们知道了UserService的实例是一个由`JdkDynamicAopProxy`支持的代理类。

于是，当执行`UserService.insertUser`方法时，执行的是`JdkDynamicAopProxy.invoke`方法。`invoke`方法中调用`AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice`来获取`insertUser`方法相关的拦截方法。调用流程如下：

1. JdkDynamicAopProxy.invoke
2. AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice
3. DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice
4. DefaultAdvisorAdapterRegistry.getInterceptors

`DefaultAdvisorAdapterRegistry.getInterceptors`方法中获取到`BeanFactoryTransactionAttributeSourceAdvisor`中的`MethodInterceptor`：`TransactionInterceptor`。

回到`JdkDynamicAopProxy.invoke`方法，通过`AdvisedSupport.getInterceptorsAndDynamicInterceptionAdvice`方法获取了拦截器链。之后创建`ReflectiveMethodInvocation`，再调用`proceed`方法执行整个链条中的方法。调动流程如下：

1. ReflectiveMethodInvocation.proceed
2. TransactionInterceptor.invoke
3. TransactionAspectSupport.invokeWithinTransaction

`TransactionAspectSupport.invokeWithinTransaction`方法有几个步骤：

1. 获取事务属性`TransactionAttribute`，包括事务传播行为、隔离级别等
2. 获取事务管理器`PlatformTransactionManager`，这是我们的在Config中定义的`DataSourceTransactionManager`
3. 调用`TransactionAspectSupport.createTransactionIfNecessary`，返回`TransactionInfo`

    在该方法中调用`DataSourceTransactionManager.getTransaction`方法获取事务状态`TransactionStatus`。包括：获取连接、建立事务。建立的连接保存在`DataSourceTransactionObject`中，`DataSourceTransactionObject`保存在`DefaultTransactionStatus`中，`DefaultTransactionStatus`保存在`TransactionInfo`中。这个过程都在`DataSourceTransactionManager.doBegin`方法中执行。`doBegin`方法还有一个步骤，代码如下：
    
    ```java
    if (txObject.isNewConnectionHolder()) {
        TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
    }
    ```
    
    这几句代码将建立的连接保存在`TransactionSynchronizationManager`的静态变量resources中，resources是一个`ThreadLocal`类型的变量。

4. 调用`ReflectiveMethodInvocation.proceed`方法。在其中调用`UserService.insertUser`的真实方法，其中调用`UserMapper.insertUser`方法。

    关于Mapper中方法的执行上一篇文章有过分析，这里我们关注`insertUser`方法中连接的获取。调用流程如下：
    
    1. BaseExecutor.getConnection
    2. SpringManagedTransaction.getConnection
    3. SpringManagedTransaction.openConnection
    4. DataSourceUtils.getConnection
    5. TransactionSynchronizationManager.getResource
    6. TransactionSynchronizationManager.doGetResource
    
    `TransactionSynchronizationManager.doGetResource`方法的代码如下：
    
    ```java
    private static Object doGetResource(Object actualKey) {
        Map<Object, Object> map = resources.get();
        if (map == null) {
            return null;
        }
        Object value = map.get(actualKey);
        // Transparently remove ResourceHolder that was marked as void...
        if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
            map.remove(actualKey);
            // Remove entire ThreadLocal if empty...
            if (map.isEmpty()) {
                resources.remove();
            }
            value = null;
        }
        return value;
    }
    ```

    可以看到，它从我们之前绑定连接的`resources`中获取连接。这里解释了为什么Spring中的事务和Mybatis执行使用了同一个连接。

