---
title: spring实战读书笔记
date: 2017/10/15 15:19:00
---

### 1.2.1 bean的生命周期

Spring自带了多种类型的应用上下文：

- AnnotationConfigApplicationContext：从一个或多个基于Java的配置类中加载Spring应用上下文
- AnnotationConfigWebApplicationContext：从一个或多个基于Java的配置类中加载Spring Web应用上下文
- ClassPathXmlApplicationContext：从类路径下的一个或多个XML配置文件中加载上下文定义，把应用上下文的定义文件作为类资源
- FileSystemXmlApplicationContext：从文件系统下的一个或多个XML配置文件中加载上下文定义
- XmlWebApplicationContext：从Web应用下的一个或多个XML配置文件中加载上下文定义
<!--more-->

使用FileSystemXmlApplicationContext和使用ClassPathXmlApplicationContext的区别在于：FileSystemXmlApplication在指定的文件系统路径下查找xml文件；而ClassPathXmlApplicationContext是在所有路径(包含jar文件)下查找xml文件

在bean准备就绪之前，bean工厂执行了若干启动步骤：

1. 实例化
2. 填充属性：Spring将值和bean的引用注入到bean对应的属性中
3. 调用BeanNameAware的setBeanName()方法：如果bean实现了BeanNameAware接口，Spring将bean的ID传递给setBeanName方法
4. 调用BeanFactoryAware的setBeanFactory()方法：如果bean实现了BeanFactoryAware接口，Spring将调用setBeanFactory()方法，将BeanFactory容器示例传入
5. 调用ApplicationContextAware的setApplicationContext()方法：如果bean实现了ApplicationContextAware接口，Spring将调用setApplicationContext()方法，将bean所在的应用上下文的引用传入进来
6. 调用BeanPostProcessor的预初始化方法：如果bean实现了BeanPostProcessor接口，Spring将调用他们的postProcessBeforeInitialization()方法
7. 调用InitializingBean的afterPropertiesSet()方法：如果bean实现了InitializingBean接口，Spring将调用他们的afterPropertiesSet()方法。类似地，如果bean使用init-method声明了初始化方法，该方法也会被调用
8. 调用自定义的初始化方法
9. 调用BeanPostProcessor的初始化方法：如果bean实现了BeanPostProcessor接口，Spring将调用他们的postProcessAfterInitialization()方法
10. bean可以使用了:此时，bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁
11. 调用DisposableBean的destroy()方法：如果bean实现了DisposableBean接口，Spring将调用它的destroy()接口方法。同样，如果bean使用destroy-method声明了销毁方法，该方法也会被调用
12. 调用自定义销毁方法

### 4.1.1 定义AOP术语

- 通知(Advice)

切面的工作被称为通知。通知定义了切面是什么以及何时使用。Spring切面可以应用5中类型的通知：

	- 前置通知(Before)：在目标方法被调用之前调用通知功能
	- 后置通知(After)：在目标方法完成之后调用通知，此时不会关心方法的输出是什么
	- 返回通知(After-returning)：在目标方法成功执行之后调用通知
	- 异常通知(After-throwing)：在目标方法抛出异常后调用通知
	- 环绕通知(Around)：通知包裹了被通知的方法，在被通知的方法调用之前和调用之后执行自定义的行为

- 连接点(Join Point)

连接点是在应用执行过程中能够插入切面的一个点。这个点可以是调用方法时、抛出异常时、甚至修改一个字段时。切面代码可以利用这些点插入到应用的正常流程中，并添加新的行为。

- 切点(Pointcut)

如果说通知定义了切面的"什么"和"何时"的话，那么切点就定义了"何处"。切点的定义会匹配通知所要织入的一个或多个连接点。我们通常使用明确的类和方法名称，或是利用正则表达式定义所匹配的类和方法名称来指定这些切点。

- 切面(Aspect)

通知和切点共同定义了切面的全部内容——它是什么，在何时和何处完成其功能

- 引入(Introduction)

引入允许我们向现有的类添加新方法或属性。

- 织入(Weaving)

织入是把切面应用到目标对象并创建新的代理对象的过程。切面在指定的连接点被织入到目标对象中。在目标对象的声明周期里有多个点可以进行织入：

	- 编译期：切面在目标类编译时被织入。这种方法需要特殊的编译器。AspectJ的织入编译器就是以这种方式织入切面的
	- 类加载期：切面在目标类加载到JVM时被织入。这种方式需要特殊的类加载器(ClassLoader)，它可以在目标类被引入应用之前增强该目标类的字节码。AspectJ 5的加载时织入(load-time weaving, LTW)就支持以这种方式织入切面
	- 运行期：切面在应用运行的某个时刻被织入。一般情况下，在织入切面时，AOP容器会为目标对象动态地创建一个代理对象。Spring AOP就是以这种方式织入切面的。

不管你是使用JavaConfig还是XML，AspectJ自动代理都会为使用@Aspect注解的bean创建一个代理，这个代理会围绕着所有该切面的切点所匹配的bean。

我们需要记住的是，Spring的AspectJ自动代理仅仅使用@AspectJ作为创建切面的指导，切面依然是基于代理的。在本质上，它依然是Spring基于代理的切面。这一点非常重要，因为这意味着尽管使用的是@AspectJ注解，但我们依然限于代理方法的调用。如果想利用AspectJ的所有能力，我们必须在运行时使用AspectJ并且不依赖Spring来创建基于代理的切面


### 5.1.1

在请求离开浏览器时，会带有用户所请求内容的信息，至少会包含请求的URL。但是还可能带有其他的信息，例如用户提交的表单信息。

请求旅程的第一站是Spring的DispatcherServlet。与大多数基于Java的Web框架一样，Spring MVC所有的请求都会通过一个前端控制器Servlet。前端控制器是常用的Web应用程序模式，在这里一个单实例的Servlet将请求委托给应用程序的其他组件来执行实际的处理。在Spring MVC中，DispatcherServlet就是前端控制器。

DispatcherServlet的任务就是将请求发送给Spring MVC控制器(controller)。控制器是一个用于处理请求的Spring组件。在典型的应用程序中可能会有多个控制器，DispatcherServlet需要知道应该将请求发送给哪个控制器。所以DispatcherServlet查询一个或多个处理器映射(handler mapping)来确定请求的下一站在哪里。处理器映射会根据请求所携带的URL信息来进行决策。

一旦选择了合适的控制器，DispatcherServlet会将请求发送给选中的控制器。到了控制器，请求会卸下负载(用户提交的信息)并耐心等待控制器处理这些信息。(实际上，设计良好的控制器本身只处理很少甚至不处理工作，而是将业务逻辑委托给一个或多个服务对象进行处理)

控制器在完成逻辑处理后，通常会产生一些信息，这些信息需要返回给用户并在浏览器上显示。这些信息被称为模型(model)。不过仅仅给用户返回原始的信息是不够的——这些信息需要以用户友好的方式进行格式化，一般会是HTML。所以，信息需要发送一个视图(view)，通常会是JSP。

控制器所做的最后一件事就是将模型数据打包，并且标示出用于渲染输出的视图名。它接下来会将请求连同模型和视图名发送回DispatcherServlet。

这样，控制器就不会与特定的视图相耦合，传递给DispatcherServlet的视图名并不直接表示某个特定的JSP。实际上，它甚至并不能确定视图就是JSP。相反，它仅仅传递了一个逻辑名称，这个名称将会用来查找产生结果的真正视图。DispatcherServlet将会使用视图解析器(view resolver)来讲逻辑视图名匹配为一个特定的视图实现，它可能是也可能不是JSP。

既然DisPatcherServlet已经知道由哪个视图渲染结果，那请求的任务基本上也就完成了。它的最后一站是视图的实现(可能是JSP)，在这里它交付模型数据。请求的任务就完成了。视图将使用模型数据渲染输出，这个输出会通过响应对象传递给客户端。

### 5.1.2

在Servlet 3.0环境中，容器会在类路径中查找实现javax.servlet.ServletContainerInitializer接口的类，如果能发现的话，就会用它来配置Servlet容器。

Spring提供了这个接口的实现，名为SpringServletContainerInitializer，这个类反过来又会查找实现WebApplicationInitializer的类并将配置的任务交给它们来完成。Spring 3.2引入了一个便利的WebApplicationInitializer基础实现，也就是AbstractAnnotationConfigDispatcherServletInitializer。因为我们扩展了AbstractAnnotationConfigDispatcherServletInitializer(同时也就是实现了WebApplicationInitializer)，因此当部署到Servlet 3.0容器中的时候，容器会自动发现它，并用它来配置Servlet上下文。

DispatcherServlet加载包含Web组件的bean，如控制器、视图解析器以及处理器映射，而ContextLoaderListener要加载应用中的其他bean。这些bean通常是驱动应用后端的中间层和数据层组件。

实际上，AbstractAnnotationConfigDispatcherServletInitializer会同时创建DispatcherServlet和ContextLoaderListener。getServletConfigClasses()方法返回的带有@Configuration注解的类将会用来定义DispatcherServlet应用上下文中的bean。getRootConfigClasses()方法返回的带有@Configuration注解的类将会用来配置ContextLoaderListener创建的应用上下文中的bean。

### 5.1.3

Java校验API所提供的校验注解

|注解|描述|
|---|----|
|@AssertFalse|所注解的元素必须是Boolean类型，并且值为false|
|@AssertTrue|所注解的元素必须是Boolean类型，并且值为true|
|@DecimalMax|所注解的元素必须是数字，并且它的值要小于或等于给定的BigDecimalString值|
|@DecimalMin|所注解的元素必须是数字，并且它的值要大于或等于给定的BigDecimalString值|
|@Digits|所注解的元素必须是数字，并且它的值必须有指定的位数|
|@Future|所注解的元素的值必须是一个将来的日期|
|@Max|所注解的元素必须是数字，并且它的值要小于或等于给定的值|
|@Min|所注解的元素必须是数字，并且它的值要大于或等于给定的值|
|@NotNull|所注解的元素的值必须不能为null|
|@Null|所注解的元素的值必须为null|
|@Past|所注解的元素的值必须是一个已过去的日期|
|@Pattern|所注解的元素的值必须匹配给定的正则表达式|
|@Size|所注解的元素的值必须是String、集合或数组，并且它的长度要符合给定的范围|

### 7.1.1

AbstractAnnotationConfigDispatcherServletInitializer所完成的事情其实比看上去要多。在SpittrWebAppInitializer中我们所编写的三个方法仅仅是必须要重载的abstract的方法。但实际上还有更多的方法可以进行重载，从而实现额外的配置。

此类的方法之一就是customizeRegistration()。在AbstractAnnotationConfigDispatcherServletInitializer将DispatcherServlet注册到Servlet容器中之后，就会调用customizeRegistration()，并将Servlet注册后得到的Registration.Dynamic传递进来。通过重载customizeRegistration()方法，我们可以对DispatcherServlet进行额外的配置。

## 7.3

Spring提供了多种方式将异常转换为响应：

- 特定的Spring异常将会自动映射为指定的HTTP状态码
- 异常上可以添加@ResponseStatus注解，从而将其映射为某一个HTTP状态码
- 在方法上可以添加@ExceptionHandler注解，使其用来处理异常

## 7.4

控制器通知(controller advice)是任意带有@ControllerAdvice注解的类，这个类会包含一个或多个如下类型的方法：

- @ExceptionHandler注解标注的方法
- @InitBinder注解标注的方法
- @ModelAttribute注解标注的方法

在带有@ControllerAdvice注解的类中，以上所述的这些方法会运用到整个应用程序所有控制器中带有@RequestMapping注解的方法上。

@ControllerAdvice最为实用的一个场景就是将所有的@ExceptionHandler方法收集到一个类中，这样所有控制器的异常就能在一个地方进行一致的处理。

## 7.5

当控制器方法返回的String值以"redirect:"开头的话，那么这个String不是用来查找视图的，而是用来指导浏览器进行重定向的路径。

一般来讲，当一个处理器方法完成之后，该方法所指定的模型数据将会复制到请求中，并作为请求中的属性，请求会转发(forward)到视图上进行渲染。因为控制器方法和视图所处理的是同一个请求，所以在转发的过程中，请求属性能够得以保持。

当控制器的结果是重定向的话，原始的请求就结束了，并且会发起一个新的GET请求。原始请求中所带有的模型数据也就随着请求一起消亡了。在新的请求属性中，没有任何的模型数据，这个请求必须要自己计算数据。

对于重定向来说，模型并不能用来传递数据。但是我们也有一些其他方案，能够从发起重定向的方法传递给处理重定向方法中：

- 使用URL模板以路径变量和/或查询参数的形式传递数据
- 通过flash属性发送数据


