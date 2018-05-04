---
title: Spring与MVC(一)
date: 2018/01/22 14:42:00
---

在请求离开浏览器时，会带有用户所请求内容的信息，至少会包含请求的URL。但是还可能带有其他的信息，例如用户提交的表单信息。

请求旅程的第一站是Spring的DispatcherServlet。与大多数基于Java的Web框架一样，Spring MVC所有的请求都会通过一个前端控制器Servlet。前端控制器是常用的Web应用程序模式，在这里一个单实例的Servlet将请求委托给应用程序的其他组件来执行实际的处理。在Spring MVC中，DispatcherServlet就是前端控制器。

DispatcherServlet的任务就是将请求发送给Spring MVC控制器(controller)。控制器是一个用于处理请求的Spring组件。在典型的应用程序中可能会有多个控制器，DispatcherServlet需要知道应该将请求发送给哪个控制器。所以DispatcherServlet查询一个或多个处理器映射(handler mapping)来确定请求的下一站在哪里。处理器映射会根据请求所携带的URL信息来进行决策。

一旦选择了合适的控制器，DispatcherServlet会将请求发送给选中的控制器。到了控制器，请求会卸下负载(用户提交的信息)并耐心等待控制器处理这些信息。(实际上，设计良好的控制器本身只处理很少甚至不处理工作，而是将业务逻辑委托给一个或多个服务对象进行处理)

控制器在完成逻辑处理后，通常会产生一些信息，这些信息需要返回给用户并在浏览器上显示。这些信息被称为模型(model)。不过仅仅给用户返回原始的信息是不够的——这些信息需要以用户友好的方式进行格式化，一般会是HTML。所以，信息需要发送一个视图(view)，通常会是JSP。

控制器所做的最后一件事就是将模型数据打包，并且标示出用于渲染输出的视图名。它接下来会将请求连同模型和视图名发送回DispatcherServlet。

这样，控制器就不会与特定的视图相耦合，传递给DispatcherServlet的视图名并不直接表示某个特定的JSP。实际上，它甚至并不能确定视图就是JSP。相反，它仅仅传递了一个逻辑名称，这个名称将会用来查找产生结果的真正视图。DispatcherServlet将会使用视图解析器(view resolver)来将逻辑视图名匹配为一个特定的视图实现，它可能是也可能不是JSP。

既然DisPatcherServlet已经知道由哪个视图渲染结果，那请求的任务基本上也就完成了。它的最后一站是视图的实现(可能是JSP)，在这里它交付模型数据。请求的任务就完成了。视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端。
<!-- more -->
## 配置DispatcherServlet

DispatcherServlet是Spring MVC的核心。在这里请求会第一次接触到框架，它要负责将请求路由到其他的组件之中：

```java
public class WebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return new Class[] { RootConfig.class };
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class[] { WebConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

扩展`AbstractAnnotationConfigDispatcherServletInitializer`的任意类都会自动地配置DispatcherServlet和Spring应用上下文，Spring的应用上下文会位于应用程序的Servlet上下文中。

在Servlet 3.0环境中，容器会在类路径中查找实现`javax.servlet.ServletContainerInitializer`接口的类，如果能发现的话，就会用它来配置Servlet容器。

Spring提供了这个接口的实现，名为SpringServletContainerInitializer，这个类反过来又会查找实现WebApplicationInitializer的类并将配置的任务交给它们来完成。Spring 3.2引入了一个便利的WebApplicationInitializer基础实现，也就是AbstractAnnotationConfigDispatcherServletInitializer。因为我们扩展了AbstractAnnotationConfigDispatcherServletInitializer(同时也就是实现了WebApplicationInitializer)，因此当部署到Servlet 3.0容器中的时候，容器会自动发现它，并用它来配置Servlet上下文。

DispatcherServlet加载包含Web组件的bean，如控制器、视图解析器以及处理器映射。而ContextLoaderListener要加载应用中的其他bean，这些bean通常是驱动应用后端的中间层和数据层组件。

实际上，`AbstractAnnotationConfigDispatcherServletInitializer`会同时创建`DispatcherServlet`和`ContextLoaderListener`：

- `getServletConfigClasses()`方法返回的带有@Configuration注解的类将会用来定义DispatcherServlet应用上下文中的bean。
- `getRootConfigClasses()`方法返回的带有@Configuration注解的类将会用来配置ContextLoaderListener创建的应用上下文中的bean。

## 启用Spring MVC

我们有多种方式来配置DispatcherServlet，与之类似，启用Spring MVC组件的方法也不仅一种。以前，Spring是使用XML进行配置的，你可以使用`<mvc:annotation-driven>`启用注解驱动的Spring MVC。

我们所能创建的最简单的Spring MVC配置就是一个带有`@EnableWebMvc`注解的类：

```java
@Configuration
@EnableWebMvc		// 启用Spring MVC
@ComponentScan("love.wangqi")		// 启用组件扫描
public class WebConfig extends WebMvcConfigurerAdapter {
    /**
     * 配置JSP视图解析器
     */
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        resolver.setExposeContextBeansAsAttributes(true);
        return resolver;
    }

    /**
     * 配置静态资源的处理
     */
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

第一件需要注意的事情是WebConfig现在添加了`@ComponentScan`注解，因此将会扫描`love.wangqi`包来查找组件。稍后会看到，我们所编写的控制器将会带有`@Controller`注解，这会使其成为组件扫描时的候选bean。因此，我们不需要再配置类中显式声明任何的控制器。

接下来，我们添加了一个`ViewResolver`bean。更具体来讲，是`InternalResourceViewResolver`。它会查找jsp文件，在查找的时候，它会在视图名称上加一个特定的前缀和后缀（例如，名为home的视图将会解析为`/WEB-INF/views/home.jsp`）。

最后，新的WebConfig类还扩展了`WebMvcConfigurerAdapter`并重写了其`configureDefaultServletHandling()`方法。通过调用`DefaultServletHandlerConfigurer`的`enable()`方法，我们要求`DispatcherServlet`将对静态资源的请求转发到Servlet容器中默认的Servlet上，而不是使用`DispatcherServlet`本身来处理此类请求。

## RootConfig

因为这里聚焦于Web开发，而Web相关的配置通过DispatcherServlet创建的应用上下文都已经配置好了，因此现在的RootConfig相对很简单

```java
@Configuration
@ComponentScan(basePackages = {"love.wangqi"}, excludeFilters = {@ComponentScan.Filter(type = FilterType.ANNOTATION, value = EnableWebMvc.class)})
public class RootConfig {
}
```

唯一需要注意的是RootConfig使用了`@ComponentScan`注解。这样我们就可以用非Web的组件来充实完善RootConfig。

## 控制器

在Spring MVC中，控制器只是方法上添加了`@RequestMapping`注解的类，这个注解声明了它们所要处理的请求。

我们尽量简单，假设控制器类要处理对"/"的请求，并渲染应用的首页:

```java
@Controller
public class HomeController {
    @RequestMapping(value = "/", method = RequestMethod.GET)
    public String home() {
        return "home";
    }
}

```

`@Controller`是一个构造器(stereotype)的注解，它基于`@Component`注解。在这里，它的目的就是辅助实现组件扫描。因为HomeController带有`@Controller`注解，因此组件扫描器会自动找到HomeController，并将其声明为Spring应用上下文中的一个bean。

其实，你也可以让HomeController带有@Component注解，它所实现的效果是一样的，但是在表意性上可能会差一些，无法确定HomeController是什么组件类型。

HomeController唯一的一个方法，也就是home()方法，带有@RequestMapping注解。它的`value`属性指定了这个方法所要处理的请求路径，`method`属性细化了它所处理的HTTP方法。在本例中，当收到对"/"的HTTP GET请求时，就会调用`home()`方法。

你可以看到，`home()`方法其实并没有做太多的事情：它返回了一个String类型的"home"。这个String将会被Spring MVC解读为要渲染的视图名称。`DispatcherServlet`会要求视图解析器将这个逻辑名称解析为实际的视图。

鉴于我们配置`InternalResourceViewResolver`的方式，视图名"home"将会解析为"/WEB-INF/views/home.jsp"路径的JSP


