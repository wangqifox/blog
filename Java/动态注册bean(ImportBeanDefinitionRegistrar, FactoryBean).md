---
title: 动态注册bean(ImportBeanDefinitionRegistrar, FactoryBean)
date: 2019/01/14 15:33:00
---

这几天在开发一个内部SDK的过程中遇到一个问题：SDK需要请求很多的HTTP接口，如果在我们的逻辑代码中直接进行HTTP接口的请求会使得代码的耦合性非常高，同时也需要写大量的代码来进行HTTP请求的配置、响应的处理。这促使我去寻找更加通用的工具来满足需求。

<!-- more -->

因为我们的项目中一直在使用Spring Cloud，自然而然地我想到了Feign。Feign([https://github.com/OpenFeign/feign](https://github.com/OpenFeign/feign))是一个基于注解来生成HTTP请求，并且能自动处理请求返回的工具。

```java
interface GitHub {
  @RequestLine("GET /repos/{owner}/{repo}/contributors")
  List<Contributor> contributors(@Param("owner") String owner, @Param("repo") String repo);
}

public static class Contributor {
  String login;
  int contributions;
}

public class MyApp {
  public static void main(String... args) {
    GitHub github = Feign.builder()
                         .decoder(new GsonDecoder())
                         .target(GitHub.class, "https://api.github.com");
  
    // Fetch and print a list of the contributors to this library.
    List<Contributor> contributors = github.contributors("OpenFeign", "feign");
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
  }
}
```

正如上面的例子所示，我们只需要在接口上加上相应的注解，然后通过调用Feign生成的代理类，我们就可以得到接口返回的数据。中间一系列的请求过程都是由Feign自动完成的。可以看到，Feign能够大幅度地简化我们的代码。

美中不足的是，原生的Feign是不支持Spring的，这意味着我们无法享受到Spring依赖注入的便利。幸运的是Spring Cloud提供了`spring-cloud-starter-openfeign`组件，不幸的是`spring-cloud-starter-openfeign`与Spring Cloud以及Spring Boot结合地比较深，这意味着只有使用Spring Cloud才能使用`spring-cloud-starter-openfeign`组件来请求服务。

在我们的SDK中只需要简单地对http接口进行请求，不需要也不能够依赖服务注册提供的服务地址。于是我们需要对`spring-cloud-starter-openfeign`做一个简化，剥离出其中有用的部分。

本文侧重分析如何在Spring中实现动态注册bean。

## 动态注册bean

在Spring中动态注册bean的通用做法是实现`ImportBeanDefinitionRegistrar`接口。

`ImportBeanDefinitionRegistrar`需要配合`@Import`注解，`@Import`注解导入实现了`ImportBeanDefinitionRegistrar`接口的类。

假设我们定义了如下的接口：

```java
@HttpUtil(url = "http://127.0.0.1")
public interface RequestDemo {
    @HttpRequest(path = "/index")
    String test1();

    @HttpRequest(path = "/post", method = "POST")
    String test2();
}
```

目标是调用`test1()`方法的时候请求`GET http://127.0.0.1/index`，调用`test2()`方法的时候请求`POST http://127.0.0.1/post`。为了实现这个目标，核心是实现`ImportBeanDefinitionRegistrar`。

主要的思路是：利用`ClassPathScanningCandidateComponentProvider`获取标注了`HttpUtil`注解的接口，使用`BeanDefinitionReaderUtils`将一个实现了`FactoryBean`的工厂方法的`BeanDefinition`注册到容器中。获取`Bean`的时候会调用工厂方法的`getObject()`方法返回一个代理类。`ImportBeanDefinitionRegistrar`实现类的代码如下：

```java
public class HttpRequestRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware, EnvironmentAware {

    private ResourceLoader resourceLoader;

    private Environment environment;

    @Override
    public void setEnvironment(Environment environment) {
        this.environment = environment;
    }

    @Override
    public void setResourceLoader(ResourceLoader resourceLoader) {
        this.resourceLoader = resourceLoader;
    }

    /**
     * 注册动态bean的beanDefinition
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 获取Class的扫描器
        ClassPathScanningCandidateComponentProvider scanner = getScanner();
        scanner.setResourceLoader(this.resourceLoader);

        // 指定只扫描标注了@HttpUtil注解的接口
        AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(HttpUtil.class);
        scanner.addIncludeFilter(annotationTypeFilter);
        
        // 指定扫描的基础包
        String basePackage = ClassUtils.getPackageName(importingClassMetadata.getClassName());

        // 获取基础包底下所有符合条件的类的定义
        Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
        for (BeanDefinition candidateComponent : candidateComponents) {
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();

                Map<String, Object> attributes = annotationMetadata.getAnnotationAttributes(HttpUtil.class.getCanonicalName());
                // 调用registerHttpClient注册类定义
                registerHttpClient(registry, annotationMetadata, attributes);
            }
        }
    }

    private void registerHttpClient(BeanDefinitionRegistry registry, AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
        String className = annotationMetadata.getClassName();
        // 在HttpFactoryBean中设置url type等信息
        BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(HttpFactoryBean.class);
        definition.addPropertyValue("url", getUrl(attributes));
        definition.addPropertyValue("type", className);
        definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

        AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();
        BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className, null);
        // 针对每一个接口注册一个HttpFactoryBean
        BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
    }

    private String resolve(String value) {
        if (StringUtils.hasText(value)) {
            return this.environment.resolvePlaceholders(value);
        }
        return value;
    }

    private String getUrl(Map<String, Object> attributes) {
        String url = resolve((String) attributes.get("url"));
        return getUrl(url);
    }

    static String getUrl(String url) {
        if (StringUtils.hasText(url) && !(url.startsWith("#{") && url.contains("}"))) {
            if (!url.contains("://")) {
                url = "http://" + url;
            }
            try {
                new URL(url);
            }
            catch (MalformedURLException e) {
                throw new IllegalArgumentException(url + " is malformed", e);
            }
        }
        return url;
    }

    /**
     * 构造Class扫描器
     */
    protected ClassPathScanningCandidateComponentProvider getScanner() {
        return new ClassPathScanningCandidateComponentProvider(false, this.environment) {
            @Override
            protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
                if (beanDefinition.getMetadata().isInterface()) {
                    return !beanDefinition.getMetadata().isAnnotation();
                }
                return false;
            }
        };
    }
}
```

有了`ImportBeanDefinitionRegistrar`的实现类，如何让这个实现类被Spring发现呢？我们需要编写一个注解，并在其中使用`@Import`导入前面的`HttpRequestRegistrar`。

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(HttpRequestRegistrar.class)
public @interface EnableHttpUtil {
}
```

接着将`@EnableHttpUtil`添加到`@Configuration`注解下，这样Spring在启动过程中就会执行`HttpRequestRegistrar`注册动态bean的定义。

```java
@Configuration
@EnableHttpUtil
public class Config {
}
```

我们在`HttpRequestRegistrar`中注册了`HttpFactoryBean`的定义。`HttpFactoryBean`实现了`FactoryBean`，是一个工厂Bean，即`HttpFactoryBean`的目的是创建一个实际的bean。代码如下：

```java
public class HttpFactoryBean implements FactoryBean<Object>, InitializingBean {
    private Class<?> type;

    private String url;

    /**
     * 核心方法，创建真正的对象
     */
    @Override
    public Object getObject() throws Exception {
        if (!this.url.startsWith("http")) {
            this.url = "http://" + this.url;
        }
        return createProxy(this.url);
    }

    /**
     * 创建代理类，在代理类中执行真正的操作
     */
    private Object createProxy(String url) {
        InvocationHandler invocationHandler = createInvocationHandler(url);
        Object proxy = Proxy.newProxyInstance(HttpRequest.class.getClassLoader(), new Class[] {type}, invocationHandler);
        return proxy;
    }

    private InvocationHandler createInvocationHandler(String url) {
        return new InvocationHandler() {
            private HttpHandler httpHandler = new DemoHttpHandler(url);

            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                return httpHandler.handle(method);
            }
        };
    }

    public Class<?> getType() {
        return type;
    }

    public void setType(Class<?> type) {
        this.type = type;
    }

    public String getUrl() {
        return url;
    }

    public void setUrl(String url) {
        this.url = url;
    }

    /**
     * getObjectType是核心方法，只有该方法返回的类型和实际类型匹配时，才能在Spring容器中找到对应类的实例
     */
    @Override
    public Class<?> getObjectType() {
        return this.type;
    }

    @Override
    public void afterPropertiesSet() throws Exception {

    }
}
```

简单起见，我们在代理类中仅仅返回了http请求的url，省略了http请求的过程，对于说明动态注册bean的方法问题不大。

```java
public class DemoHttpHandler implements HttpHandler {
    String url;

    public DemoHttpHandler(String url) {
        this.url = url;
    }

    @Override
    public String handle(Method method) {
        HttpRequest httpRequest = method.getAnnotation(HttpRequest.class);
        String path = httpRequest.path();
        String httpMethod = httpRequest.method();

        return httpMethod + " " + url + path;
    }
}
```

有了前面的铺垫，我们就可以来使用我们的Http请求工具了。

首先新建一个接口来定义http接口：

```java
@HttpUtil(url = "http://127.0.0.1")
public interface RequestDemo {
    @HttpRequest(path = "/index")
    String test1();

    @HttpRequest(path = "/post", method = "POST")
    String test2();
}
```

然后我们就可以直接通过这个接口来进行http请求了：

```java
public class Main {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext annotationConfigApplicationContext = new AnnotationConfigApplicationContext(Config.class);
        RequestDemo requestDemo = annotationConfigApplicationContext.getBean(RequestDemo.class);
        System.out.println(requestDemo.test1());
        System.out.println(requestDemo.test2());
    }
}
```

结果输出：

```java
GET http://127.0.0.1/index
POST http://127.0.0.1/post
```

可以看到我们完成了根据接口定义来动态生成一个bean的目的。

完整代码参考：[https://github.com/wangqifox/spring-demo/tree/master/m3](https://github.com/wangqifox/spring-demo/tree/master/m3)


## spring与feign的整合

有了前面对动态注册bean的了解，我们在脑海里对spring与feign的整合已经有了大体的轮廓了。我们可以参考Spring Cloud对feign的整合来裁剪我们需要的代码。



完整代码参考：[https://github.com/wangqifox/spring-feign](https://github.com/wangqifox/spring-feign)

















> https://zhuanlan.zhihu.com/p/30070328
> https://zhuanlan.zhihu.com/p/30123517

