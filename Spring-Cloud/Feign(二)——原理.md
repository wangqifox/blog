---
title: Feign(二)——原理
date: 2018/07/17 10:39:25
---

Feign是一个伪客户端，即它不做任何的请求处理。Feign通过处理注解生成request，从而实现简化HTTP API开发的目的，即开发人员可以使用注解的方式定制request api模板，在发送http request请求之前，feign通过处理注解的方式替换掉request模板中的参数，这种实现方式显得更为直接、可理解。

<!-- more -->

现在让我们通过代码来一探Feign的究竟

# FeignClientsRegistrar

首先看一下`@EnableFeignClients`注解：

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    String[] value() default {};
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
    Class<?>[] defaultConfiguration() default {};
    Class<?>[] clients() default {};
}
```

它导入了`FeignClientsRegistrar`，这是一个用于注册beanDefinition的类，代码如下：

```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    registerDefaultConfiguration(metadata, registry);
    registerFeignClients(metadata, registry);
}
```

`registerDefaultConfiguration`方法注册`FeignClientSpecification`的beanDefinition。

`registerFeignClients`方法代码如下：

```java
public void registerFeignClients(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    scanner.setResourceLoader(this.resourceLoader);

    Set<String> basePackages;

    Map<String, Object> attrs = metadata
            .getAnnotationAttributes(EnableFeignClients.class.getName());
    AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(
            FeignClient.class);
    final Class<?>[] clients = attrs == null ? null
            : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        scanner.addIncludeFilter(annotationTypeFilter);
        basePackages = getBasePackages(metadata);
    }
    else {
        final Set<String> clientClasses = new HashSet<>();
        basePackages = new HashSet<>();
        for (Class<?> clazz : clients) {
            basePackages.add(ClassUtils.getPackageName(clazz));
            clientClasses.add(clazz.getCanonicalName());
        }
        AbstractClassTestingTypeFilter filter = new AbstractClassTestingTypeFilter() {
            @Override
            protected boolean match(ClassMetadata metadata) {
                String cleaned = metadata.getClassName().replaceAll("\\$", ".");
                return clientClasses.contains(cleaned);
            }
        };
        scanner.addIncludeFilter(
                new AllTypeFilter(Arrays.asList(filter, annotationTypeFilter)));
    }

    for (String basePackage : basePackages) {
        Set<BeanDefinition> candidateComponents = scanner
                .findCandidateComponents(basePackage);
        for (BeanDefinition candidateComponent : candidateComponents) {
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                // verify annotated class is an interface
                AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
                Assert.isTrue(annotationMetadata.isInterface(),
                        "@FeignClient can only be specified on an interface");

                Map<String, Object> attributes = annotationMetadata
                        .getAnnotationAttributes(
                                FeignClient.class.getCanonicalName());

                String name = getClientName(attributes);
                registerClientConfiguration(registry, name,
                        attributes.get("configuration"));

                registerFeignClient(registry, annotationMetadata, attributes);
            }
        }
    }
}
```

`registerFeignClients`方法从basePackage中扫描被`@FeignClient`注释的类：

- 调用`registerClientConfiguration`注册`@FeignClient`注释中指定的configuration
- 调用`registerFeignClient`注册该类的beanDefinition

`registerFeignClient`方法的代码如下：

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
        AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();
    BeanDefinitionBuilder definition = BeanDefinitionBuilder
            .genericBeanDefinition(FeignClientFactoryBean.class);
    validate(attributes);
    definition.addPropertyValue("url", getUrl(attributes));
    definition.addPropertyValue("path", getPath(attributes));
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    definition.addPropertyValue("type", className);
    definition.addPropertyValue("decode404", attributes.get("decode404"));
    definition.addPropertyValue("fallback", attributes.get("fallback"));
    definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

    String alias = name + "FeignClient";
    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

    boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

    beanDefinition.setPrimary(primary);

    String qualifier = getQualifier(attributes);
    if (StringUtils.hasText(qualifier)) {
        alias = qualifier;
    }

    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
            new String[] { alias });
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

`registerFeignClient`方法将注解的信息连同类名一起取出，赋给BeanDefinitionBuilder，然后根据BeanDefinitionBuilder得到beanDefinition，最后注册beanDefinition。

# ReflectiveFeign

当使用Feign Client时，如上文的示例：

```java
@Autowired
ComputeClient computeClient;
```

使用`ReflectiveFeign`新建一个`ComputeClient`的bean实例：

```java
public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
            continue;
        } else if(Util.isDefault(method)) {
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
        } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
        defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
}
```

在`ReflectiveFeign`的`newInstance`方法中，使用JDK代理创建一个代理类，其中的handler是`ReflectiveFeign#FeignInvocationHandler`。

# FeignInvocationHandler

前文我们看到，通过`@Autowired`注入的`ComputeClient`是一个代理类，它的handler是`ReflectiveFeign#FeignInvocationHandler`，其中的拦截器方法如下：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
        try {
            Object
                    otherHandler =
                    args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
            return equals(otherHandler);
        } catch (IllegalArgumentException e) {
            return false;
        }
    } else if ("hashCode".equals(method.getName())) {
        return hashCode();
    } else if ("toString".equals(method.getName())) {
        return toString();
    }
    return dispatch.get(method).invoke(args);
}
```

实际上它将方法的运行委托给了dispatch中的`MethodHandler`来执行，这里`MethodHandler`的实现类是`SynchronousMethodHandler`。

# SynchronousMethodHandler

`SynchronousMethodHandler`类是方法真正执行的类，其`invoke`方法如下：

```java
public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
        try {
            return executeAndDecode(template);
        } catch (RetryableException e) {
            retryer.continueOrPropagate(e);
            if (logLevel != Logger.Level.NONE) {
                logger.logRetry(metadata.configKey(), logLevel);
            }
            continue;
        }
    }
}
```

1. 根据参数生成`RequestTemplate`对象，该对象就是http请求的模板
2. 调用`executeAndDecode`方法发送请求并返回响应结果

## executeAndDecode

`executeAndDecode`方法通过`RequestTemplate`生成`Request`对象。

然后调用`Client.execute`方法发送请求，接收`Response`。

最后根据Feign方法的返回值来解码`Response`，即将请求返回的响应转成需要的类型返回。

# Client组件

Client组件是一个非常重要的组件，Feign最终发送request请求以及接收response响应，都是由Client组件完成的。Client的实现类为`LoadBalancerFeignClient`：

```java
@Configuration
class DefaultFeignLoadBalancedConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
                              SpringClientFactory clientFactory) {
        return new LoadBalancerFeignClient(new Client.Default(null, null),
                cachingFactory, clientFactory);
    }
}
```

默认情况下，注入`Client.Default()`，它使用的网络请求框架为HttpURLConnection。

怎样使用其他的网络请求框架呢？

我们来看看`HttpClientFeignLoadBalancedConfiguration`：

```java
@Configuration
@ConditionalOnClass(ApacheHttpClient.class)
@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
class HttpClientFeignLoadBalancedConfiguration {
    ... //省略代码
    @Bean
    @ConditionalOnMissingBean(Client.class)
    public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
                                SpringClientFactory clientFactory, HttpClient httpClient) {
        ApacheHttpClient delegate = new ApacheHttpClient(httpClient);
        return new LoadBalancerFeignClient(delegate, cachingFactory, clientFactory);
    }
    ... //省略代码
}
```

从代码的`@ConditionalOnClass(ApacheHttpClient.class)`注释可以看出，只需要在pom文件中加上HttpClient的classpath就行了，另外需要在配置文件上加上`feign.httpclient.enabled = true`，从`@ConditionalOnProperty`注释可以看出，这个配置可以不写，在默认情况下就是`true`。

因此，如果要使用HttpClient，在pom文件上加上：

```
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-httpclient</artifactId>
    <version>8.18.0</version>
</dependency>
```

同理，我们来看看`OkHttpFeignLoadBalancedConfiguration`：

```java
@Configuration
@ConditionalOnClass(OkHttpClient.class)
@ConditionalOnProperty(value = "feign.okhttp.enabled")
class OkHttpFeignLoadBalancedConfiguration {
    ... //省略代码
    @Bean
    @ConditionalOnMissingBean(Client.class)
    public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
                              SpringClientFactory clientFactory, okhttp3.OkHttpClient okHttpClient) {
        OkHttpClient delegate = new OkHttpClient(okHttpClient);
        return new LoadBalancerFeignClient(delegate, cachingFactory, clientFactory);
    }
    ... //省略代码
}
```

只需要在pom文件中加上OkHttp的classpath就行了，另外需要在配置文件上加上`feign.okhttp.enabled = true`，这个配置不能省略，因为它默认不是`true`。

因此，如果要使用Okhttp，在pom文件上加上：

```
<dependency>
    <groupId>com.netflix.feign</groupId>
    <artifactId>feign-okhttp</artifactId>
    <version>RELEASE</version>
</dependency>
```

# LoadBalancerFeignClient

`SynchronousMethodHandler`的`executeAndDecode`方法中调用`LoadBalancerFeignClient.execute`方法执行请求。

```java
public Response execute(Request request, Request.Options options) throws IOException {
    try {
        URI asUri = URI.create(request.url());
        String clientName = asUri.getHost();
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
                this.delegate, request, uriWithoutHost);

        IClientConfig requestConfig = getClientConfig(options, clientName);
        return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
                requestConfig).toResponse();
    }
    catch (ClientException e) {
        IOException io = findIOException(e);
        if (io != null) {
            throw io;
        }
        throw new RuntimeException(e);
    }
}
```

`lbClient()`方法获取`FeignLoadBalancer`实例，然后调用其中的`executeWithLoadBalancer`方法：

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);

    try {
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
                    } 
                    catch (Exception e) {
                        return Observable.error(e);
                    }
                }
            })
            .toBlocking()
            .single();
    } catch (Exception e) {
        Throwable t = e.getCause();
        if (t instanceof ClientException) {
            throw (ClientException) t;
        } else {
            throw new ClientException(e);
        }
    }
    
}
```

`LoadBalancerCommand.submit`方法中以负载均衡的方式选择服务实例，其主要功能在`selectServer()`方法中：

```java
private Observable<Server> selectServer() {
    return Observable.create(new OnSubscribe<Server>() {
        @Override
        public void call(Subscriber<? super Server> next) {
            try {
                Server server = loadBalancerContext.getServerFromLoadBalancer(loadBalancerURI, loadBalancerKey);   
                next.onNext(server);
                next.onCompleted();
            } catch (Exception e) {
                next.onError(e);
            }
        }
    });
}
```

服务实例的选择由`LoadBalancerContext.getServerFromLoadBalancer`方法完成，其中的流程与Ribbon选择服务实例的步骤基本一致。

回到`executeWithLoadBalancer`方法，服务实例确定之后，调用`FeignLoadBalancer.execute`方法执行请求：

```java
public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
        throws IOException {
    Request.Options options;
    if (configOverride != null) {
        RibbonProperties override = RibbonProperties.from(configOverride);
        options = new Request.Options(
                override.connectTimeout(this.connectTimeout),
                override.readTimeout(this.readTimeout));
    }
    else {
        options = new Request.Options(this.connectTimeout, this.readTimeout);
    }
    Response response = request.client().execute(request.toRequest(), options);
    return new RibbonResponse(request.getUri(), response);
}
```

首先调用`request.client()`获取Client，前面说到这里是Client是`Client.Default`。调用其中的`execute`方法执行真正的请求并返回响应：

```java
public Response execute(Request request, Options options) throws IOException {
    HttpURLConnection connection = convertAndSend(request, options);
    return convertResponse(connection).toBuilder().request(request).build();
}
```




> https://cloud.tencent.com/developer/article/1009212

