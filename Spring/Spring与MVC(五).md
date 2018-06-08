---
title: Spring与MVC(五)
date: 2018/01/24 18:47:00
---

上一篇文章[Spring与MVC(四)][1]，我们分析了`DispatcherServlet`在处理请求时是如何找到正确的Controller，以及如何执行。在这篇文章中，我们来看分析一下Spring MVC是如何处理方法参数以及响应返回值的。
<!-- more -->

首先让我们来看一下Spring MVC中两个重要的接口，这两个接口分别对应请求方法参数的处理、响应返回值的处理，分别是`HandlerMethodArgumentResolver`和`HandlerMethodReturnValueHandler`。

## 处理方法参数

请求首先被`DispatcherServlet`截获，通过`handlerMapping`获得`HandlerExecutionChain`，然后调用`getHandlerAdapter()`方法获得`HandlerAdapter`。通过`HandlerAdapter`来调用用户定义的handler，调用流程如下：

1. AbstractHandlerMethodAdapter.handle
2. RequestMappingHandlerAdapter.handleInternal
3. RequestMappingHandlerAdapter.invokeHandlerMethod

`RequestMappingHandlerAdapter.invokeHandlerMethod`方法执行用户的方法。其中对于每个请求都会实例化一个`ServletInvocableHandlerMethod`，然后调用`invokeAndHandle`方法进行处理，它会分别对请求跟响应进行处理：

```java
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    setResponseStatus(webRequest);

    if (returnValue == null) {
        if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
            mavContainer.setRequestHandled(true);
            return;
        }
    }
    else if (StringUtils.hasText(getResponseStatusReason())) {
        mavContainer.setRequestHandled(true);
        return;
    }

    mavContainer.setRequestHandled(false);
    try {
        this.returnValueHandlers.handleReturnValue(
                returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
    catch (Exception ex) {
        if (logger.isTraceEnabled()) {
            logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
        }
        throw ex;
    }
}
```

其中`InvocableHandlerMethod.invokeForRequest`方法负责解析参数、执行用户方法：

```java
public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    if (logger.isTraceEnabled()) {
        logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
                "' with arguments " + Arrays.toString(args));
    }
    Object returnValue = doInvoke(args);
    if (logger.isTraceEnabled()) {
        logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
                "] returned [" + returnValue + "]");
    }
    return returnValue;
}
```

首先通过`getMethodArgumentValues`方法获取请求参数，然后调用用户的controller方法，获得返回值`returnValue`然后返回：

```java
private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {

    MethodParameter[] parameters = getMethodParameters();
    Object[] args = new Object[parameters.length];
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = resolveProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        if (this.argumentResolvers.supportsParameter(parameter)) {
            try {
                args[i] = this.argumentResolvers.resolveArgument(
                        parameter, mavContainer, request, this.dataBinderFactory);
                continue;
            }
            catch (Exception ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
                }
                throw ex;
            }
        }
        if (args[i] == null) {
            throw new IllegalStateException("Could not resolve method parameter at index " +
                    parameter.getParameterIndex() + " in " + parameter.getMethod().toGenericString() +
                    ": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
        }
    }
    return args;
}
```

首先，调用`getMethodParameters`获得用户的controller方法匹配到的参数。然后获取参数的解析器：

```java
private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
            if (logger.isTraceEnabled()) {
                logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
                        parameter.getGenericParameterType() + "]");
            }
            if (methodArgumentResolver.supportsParameter(parameter)) {
                result = methodArgumentResolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}
```

遍历所有的参数解析器，调用每个解析器的`supportsParameter`方法判断它是否能解析parameter。这里有26中参数解析器供选择：

1. RequestParamMethodArgumentResolver
2. RequestParamMapMethodArgumentResolver
3. PathVariableMethodArgumentResolver
4. PathVariableMapMethodArgumentResolver
5. MatrixVariableMethodArgumentResolver
6. MatrixVariableMapMethodArgumentResolver
7. ServletModelAttributeMethodProcessor
8. RequestResponseBodyMethodProcessor
9. RequestPartMethodArgumentResolver
10. RequestHeaderMethodArgumentResolver
11. RequestHeaderMapMethodArgumentResolver
12. ServletCookieValueMethodArgumentResolver
13. ExpressionValueMethodArgumentResolver
14. SessionAttributeMethodArgumentResolver
15. RequestAttributeMethodArgumentResolver
16. ServletRequestMethodArgumentResolver
17. ServletResponseMethodArgumentResolver
18. HttpEntityMethodProcessor
19. RedirectAttributesMethodArgumentResolver
20. ModelMethodProcessor
21. MapMethodProcessor
22. ErrorsMethodArgumentResolver
23. SessionStatusMethodArgumentResolver
24. UriComponentsBuilderMethodArgumentResolver
25. RequestParamMethodArgumentResolver
26. ServletModelAttributeMethodProcessor

第7个`ServletModelAttributeMethodProcessor`和第26个`ServletModelAttributeMethodProcessor`区别在于前者的`annotationNotRequired`属性为`false`，后者的`annotationNotRequired`属性为`true`。

### @RequestParam

看一下被`@RequestParam`注释的参数如何被解析。

首先参数解析器选择的是`RequestParamMethodArgumentResolver`，看一下它的`resolveArgument`方法的主流程是如何工作的：

1. 调用`AbstractNamedValueMethodArgumentResolver.getNamedValueInfo`从`@RequestParam`注释中获取参数的名称
2. 调用`AbstractnamedValueMethodArgumentResolver.resolveStringValue`解析参数名称中可能存在占位符、表达式
3. 调用`RequestParamMethodArgumentResolver.resolveName`从request中获取参数的值
4. 参数值再经过`WebDataBinder`的转化

#### RequestParamMethodArgumentResolver.resolveName

`RequestParamMethodArgumentResolver.resolveName`的调用流程如下：

1. 调用`MultipartResolutionDelegate.resolveMultipartArgument`处理文件上传的情况
2. 调用`getNativeRequest`
3. 调用`ServletWebRequest.getParameterValues`获取相应名称的参数值
    
    `ServletWebRequest.getParameterValues`的调用流程：
    
    1. RequestFacade.getParameterValues
    2. Request.getParameterValues(String name)，根据参数名称获取参数值

#### 参数转化

// TODO

### @RequestHeader

看一下被`@RequestHeader`注释的参数如何被解析。

首先参数解析器选择的是`RequestHeaderMapMethodArgumentResolver`，看一下它的`resolveArgument`方法的主流程是如何工作的：

1. 如果参数的类型是`MultiValueMap`的子类，遍历请求中的header，将header添加到`HttpHeaders`中
2. 否则，遍历请求中的header，将header添加到`LinkedHashMap`中

### @PathVariable

看一下被`@PathVariable`注释的参数如何被解析。

首先参数解析器选择的是`PathVariableMethodArgumentResolver`，看一下它的`resolveArgument`方法的主流程是如何工作的。

它的流程和`RequestParamMethodArgumentResolver`是一致的，因为它们调用的都是`AbstractNamedValueMethodArgumentResolver`父类的`resolveArgument`方法。唯一不同的是`resolveName`方法。

#### PathVariableMethodArgumentResolver.resolveName

`resolveName`方法从request的多个属性中获取`org.springframework.web.servlet.HandlerMapping.uriTemplateVariables`属性，然后从中获取参数的值。

那么`org.springframework.web.servlet.HandlerMapping.uriTemplateVariables`属性何时获得的呢，其实它是在根据请求获取处理方法的时候获得的，即`DispatcherServlet`的`doDispatch`方法执行`getHandler`的时候。具体是在获得最佳匹配函数之后，处理它的时候：`RequestMappingInfoHandlerMapping.handleMatch`：

```java
uriVariables = getPathMatcher().extractUriTemplateVariables(bestPattern, lookupPath);
decodedUriVariables = getUrlPathHelper().decodePathVariables(request, uriVariables);
```

`extractUriTemplateVariables`函数根据模板(bestPattern)和请求的路径(lookupPath)获得路径中参数与值的映射表。

### @RequestBody

看一下被`@RequestBody`注释的参数如何被解析。

首先参数解析器选择的是`RequestResponseBodyMethodProcessor`，看一下它的`resolveArgument`方法的主流程是如何工作的。

1. 调用`RequestResponseBodyMethodProcessor.readWithMessageConverters`获取参数数据，它实际调用的是`AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters`方法
2. 参数值再经过`WebDataBinder`的转化

#### AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters

`AbstractMessageConverterMethodArgumentResolver.readWithMessageConverters`的功能是选用合适的消息转换器(MessageConverter)将数据转换成参数需要的类型，候选的消息转换器有一下10种：

1. ByteArrayHttpMessageConverter（支持byte[]类型）
2. StringHttpMessageConverter（defaultCharset="UTF-8"）（支持String类型）
3. StringHttpMessageConverter（defaultCharset="ISO-8859-1"）（支持String类型）
4. ResourceHttpMessageConverter（支持Resource类型的子类）
5. ResourceRegionHttpMessageConverter
6. SourceHttpMessageConverter（支持javax.xml.transform.stax.StAXSource, javax.xml.transform.dom.DOMSource, javax.xml.transform.stream.StreamSource, javax.xml.transform.sax.SAXSource, javax.xml.transform.Source）
7. AllEncompassingFormHttpMessageConverter（支持MultiValueMap的子类）
8. MappingJackson2HttpMessageConverter（mediaType是`application/json`和`application/*+json`其中之一，且参数是Jackson的ObjectMapper可以反序列化的类型。）
9. MappingJackson2HttpMessageConverter
10. Jaxb2RootElementHttpMessageConverter（参数带着@XmlRootElement注释，或者带着@XmlType。并且mediaType是受支持的）

#### 参数转化

// TODO

### 绑定对象

如果对象没有注释修饰，选择`ServletModelAttributeMethodProcessor`解析器(annotationNotRequired属性为true)。

看一下`ServletModelAttributeMethodProcessor`的`supportsParameter`方法如何工作：

```java
public boolean supportsParameter(MethodParameter parameter) {
    return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
            (this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
}
```

首先判断参数是否有`ModelAttribute`注释，没有。`this.annotationNotRequired`为`true`。再调用`BeanUtils.isSimplePropertry`判断参数的类是否是"简单"属性，如果不是"简单"属性则`ServletModelAttributeMethodProcessor`支持该参数。"简单"属性指的是：原始类型、`String`、`CharSequence`、`Number`、`Date`、`URI`、`URL`、`Locale`、`Class`、或者是相应的数组。

```java
public static boolean isSimpleProperty(Class<?> clazz) {
    Assert.notNull(clazz, "Class must not be null");
    return isSimpleValueType(clazz) || (clazz.isArray() && isSimpleValueType(clazz.getComponentType()));
}

public static boolean isSimpleValueType(Class<?> clazz) {
    return (ClassUtils.isPrimitiveOrWrapper(clazz) || clazz.isEnum() ||
            CharSequence.class.isAssignableFrom(clazz) ||
            Number.class.isAssignableFrom(clazz) ||
            Date.class.isAssignableFrom(clazz) ||
            URI.class == clazz || URL.class == clazz ||
            Locale.class == clazz || Class.class == clazz);
}
```

接着分析`ServletModelAttributeMethodProcessor`的`resolveArgument`方法的主流程是如何工作的。

1. 调用`ModelFactory.getNameForParameter`获取参数的名称，一般是类名称首字母转小写。
2. 调用`createAttribute`来生成绑定的对象，注意这时对象的内容为空
3. 调用`binderFactory.createBinder`来生成一个数据绑定器。默认情况这里具体的类是`ExtendedServletRequestDataBinder`
4. 调用`bindRequestParameters`来绑定请求参数，其中`ServletModelAttributeMethodProcessor`的`bindRequestParameters`方法如下
    
    ```java
    protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
        ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
        ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
        servletBinder.bind(servletRequest);
    }
    ```

    其中servletBinder为`ExtendedServletRequestDataBinder`

    调用dataBinder的`bind`方法填充对象中的数据

    ```java
    public void bind(ServletRequest request) {
        MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
        MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
        if (multipartRequest != null) {
            bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
        }
        addBindValues(mpvs, request);
        doBind(mpvs);
    }
    ```
    
    首先读取请求中所有参数以及参数对应的值生成`MutablePropertyValues`。然后调用`addBindValues`将请求路径中的参数与值合并到mpvs中。
    
    然后调用`doBind`将请求的参数绑定到对象上。

5. 调用`validateIfApplicable`方法，判断生成的参数对象是否符合要求
    
### @RequestBody
    
如果参数带着`@RequestBody`注释，选择`RequestResponseBodyMethodProcessor`。来看一下它的`resolveArgument`方法是如何工作的：

1. 调用`readWithMessageConverters`读取请求内容生成对应的参数对象

    其中messageConverters有如下7个供选择：
    
    1. ByteArrayHttpMessageConverter
    2. StringHttpMessageConverter
    3. ResourceHttpMessageConverter
    4. SourceHttpMessageConverter
    5. AllEncompassingFormHttpMessageConverter
    6. Jaxb2RootElementHttpMessageConverter
    7. MappingJackson2HttpMessageConverter

    **注意`MappingJackson2HttpMessageConverter`并不是默认有的，需要在配置文件中配置**
    
    通过调用messageConverter的`canRead`方法判断这个messageConverter是否可以读取请求生成对应的参数。
    
    找到合适的messageConverter之后执行三步操作：
    
    1. 调用`RequestBodyAdvice`的`beforeBodyRead`方法对请求进行处理。
    2. 调用messageConverter的`read`方法，读取请求内容生成对应的参数对象
    3. 调用`RequestBodyAdvice`的`afterBodyRead`方法对生成的参数对象进行处理

    返回生成的参数对象
    
2. 创建`WebDataBinder`，具体的类是`ExtendedServletRequestDataBinder`
3. 调用`validateIfApplicable`方法，判断生成的参数对象是否符合要求
    
## 返回值的响应

回到`ServletInvocableHandlerMethod.invokeAndHandle`方法，调用`invokeForRequest`方法执行完controller方法得到返回值`returnValue`。

然后调用返回值处理器处理返回值，返回值处理器为`HandlerMethodReturnValueHandlerComposite`，执行`handleReturnValue`：

```java
public void handleReturnValue(Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
    }
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}

private HandlerMethodReturnValueHandler selectHandler(Object value, MethodParameter returnType) {
    boolean isAsyncValue = isAsyncReturnValue(value, returnType);
    for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
        if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
            continue;
        }
        if (handler.supportsReturnType(returnType)) {
            return handler;
        }
    }
    return null;
}
```

首先调用`selectHandler`函数在当前所有可选的处理器中执行`supportsReturnType`方法选择支持的处理器。可选返回值处理器默认有15个：

1. ModelAndViewMethodReturnValueHandler
2. ModelMethodProcessor
3. ViewMethodReturnValueHandler
4. ResponseBodyEmitterReturnValueHandler
5. StreamingResponseBodyReturnValueHandler
6. HttpEntityMethodProcessor
7. HttpHeadersReturnValueHandler
8. CallableMethodReturnValueHandler
9. DeferredResultMethodReturnValueHandler
10. AsyncTaskMethodReturnValueHandler
11. ModelAttributeMethodProcessor
12. RequestResponseBodyMethodProcessor
13. ViewNameMethodReturnValueHandler
14. MapMethodProcessor
15. ModelAttributeMethodProcessor

### ViewNameMethodReturnValueHandler

返回值为字符串时默认选择`ViewNameMethodReturnValueHandler`，我们来看一下它的返回值处理方法`handleReturnValue`:

```java
public void handleReturnValue(Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

    if (returnValue instanceof CharSequence) {
        String viewName = returnValue.toString();
        mavContainer.setViewName(viewName);
        if (isRedirectViewName(viewName)) {
            mavContainer.setRedirectModelScenario(true);
        }
    }
    else if (returnValue != null){
        // should not happen
        throw new UnsupportedOperationException("Unexpected return type: " +
                returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
    }
}
```

我们可以看到它在`ModelAndViewContainer`中设置viewName，如果viewName中包含了"redirect:"字符串，表示重定向。


返回到`RequestMappingHandlerAdapter.invokeHandlerMethod`方法，执行完`invokeAndHandle`之后，再调用`getModelAndView`方法返回一个`ModelAndView`对象，里面包含view以及model。

返回到`DispatcherServlet.doDispatch`方法，获得`ModelAndView`对象，最后调用`processDispatchResult`渲染页面或者处理异常。

## RequestResponseBodyMethodProcessor

如果返回值带着`ResponseBody`注释，选择`RequestResponseBodyMethodProcessor`，我们来看一下它的返回值处理方法`handleReturnValue`：

```java
public void handleReturnValue(Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

    mavContainer.setRequestHandled(true);
    ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
    ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

    // Try even with null return value. ResponseBodyAdvice could get involved.
    writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
}
```

`handleReturnValue`方法调用`AbstractMessageConverterMethodProcessor.writeWithMessageConverters`方法将返回值处理后输出。它的工作步骤如下：

1. 调用`getAcceptableMediaTypes`方法来获取请求头的"ACCEPT"字段指定客户端能够接收的内容类型`requestedMediaTypes`
2. 调用`getProducibleMedaiTypes`方法获取当前能够生成的内容类型`producibleMediaTypes`

    `getProducibleMedaiTypes`遍历当前所有的messageConverters:
    
    1. ByteArrayHttpMessageConverter
    2. StringHttpMessageConverter
    3. ResourceHttpMessageConverter
    4. SourceHttpMessageConverter
    5. AllEncompassingFormHttpMessageConverter
    6. Jaxb2RootElementHttpMessageConverter

    调用`canWrite`方法判断返回的类是否可以由该`converter`来输出，如果可以输出的话添加该`converter`支持的`MediaType`。
    
3. 调用`isCompatibleWith`方法，从`requestedMediaTypes`和`producibleMediaTypes`中挑选兼容的类型
4. 遍历`messageConverters`，调用`canWrite`方法选择合适的messageConverter
5. 获取`RequestResponseBodyAdviceChain`，调用`beforeBodyWrite`对返回值进行处理
6. 在返回头上加入"Content-Disposition"
7. 调用messageConverter的`write`方法，在返回头上加入contentType。然后调用`writeInternal`方法，将转化过后的数据直接写到response中



[1]: /articles/Spring/Spring与MVC(四).html


