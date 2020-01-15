---
title: Spring之扫描package
date: 2020/01/15 10:21:00
---

在Spring中扫描package是一个常见的操作，比如扫描package为特定的class创建实例。围绕Spring建立的第三方工具通常也需要扫描package然后向Spring注册特定的实例。比如Mybatis，在启动时会扫描mapper所在的package，为底下定义的mapper生成实例（详见[MyBatis探究(三)——MyBatis与Spring的结合及代码探究][1]）。

<!-- more -->

之前在看Spring代码时，对于package扫描的过程基本就忽视了。最近在写代码时需要这样扫描package的功能，乘着这个机会复习一下Spring中扫描package的功能。

使用Spring来扫描某个指定的package是非常便捷的，示例如下：

```java
ClassPathScanningCandidateComponentProvider scanner = new ClassPathScanningCandidateComponentProvider(false);'
Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents("io.github.wangqifox");
for (BeanDefinition candidateComponent : candidateComponents) {
    System.out.println(candidateComponent.getBeanClassName());
}
```

`ClassPathScanningCandidateComponentProvider`是Spring中用于扫描package的类，Spring中的扫描任务都是使用这个类或者继承这个类来完成的。

新建`ClassPathScanningCandidateComponentProvider`时需要传入一个`useDefaultFilters`参数，该参数表示是否注册默认的过滤器（默认包含几个过滤注解的过滤器）。

调用`findCandidateComponents`查找指定package下符合要求的类，该方法调用`scanCandidateComponents`扫描package，步骤如下：

首先根据指定的package生成一个查找路径(`packageSearchPath`)，比如根据`io.github.wangqifox`生成的查找路径为`classpath*:io/github/wangqifox/**/*.class`。

然后调用`PathMatchingResourcePatternResolver.getResources`方法从该查找路径中扫描类：

```java
public Resource[] getResources(String locationPattern) throws IOException {
    Assert.notNull(locationPattern, "Location pattern must not be null");
    if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
        // a class path resource (multiple resources for same name possible)
        if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
            // a class path resource pattern
            return findPathMatchingResources(locationPattern);
        }
        else {
            // all class path resources with the given name
            return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
        }
    }
    else {
        ...
    }
}
```

首先调用`AntPathMatcher.isPattern`方法判断路径中是否包含`*`、`?`这样的通配符：

```java
public boolean isPattern(String path) {
    return (path.indexOf('*') != -1 || path.indexOf('?') != -1);
}
```

我们的查找路径`io/github/wangqifox/**/*.class`是包含通配符的，因此调用`findPathMatchingResources`方法扫描路径匹配的类。

`findPathMatchingResources`首先将查找路径拆成两部分：`rootDirPath`（根路径，比如`classpath*:io/github/wangqifox/`）和`subPattern`（匹配路径，比如`**/*.class`）。

接着调用`getResources(rootDirPath)`扫描`rootDirPath`底下的所有类。`getResources`方法前面出现过，这次因为传入的路径是根路径不包含通配符，所以调用`findAllClassPathResources`扫描路径底下所有的类，它委托给`doFindAllClassPathResources`方法去实现。

`doFindAllClassPathResources`方法通过类加载器(`ClassLoader`)去获取指定路径下的所有资源，以`URL`的形式表示。调用`convertClassLoaderURL`方法将`URL`转换成`UrlResource`的形式返回。

回到`findPathMatchingResources`方法，现在已经得到根路径下所有的资源了。接下来对根目录下的资源进行筛选。资源一共有4中类型：`bundle`类型、`vfs`类型、`jar`类型、普通文件类型。重点关注`jar`类型和本地文件类型。

方法`ResourceUtils.isJarURL`用于判断资源是否是`jar`类型，`jar`类型包括`jar`、`war`、`zip`、`vfszip`、`wsjar`。如果是`jar`类型则调用`doFindPathMatchingJarResources`方法，传入`subPattern`（匹配路径）参数筛选根路径下的资源。

如果是普通文件则调用`doFindPathMatchingFileResources`方法，传入`subPattern`（匹配路径）参数筛选根路径下的资源。

回到`scanCandidateComponents`方法，经过`getResources`的调用，获得了指定路径下所有的类资源。接下来要做的就是对所有的资源做进一步的筛选：

1. 创建资源元数据的reader：`MetadataReader`
2. 调用`boolean isCandidateComponent(MetadataReader metadataReader)`方法判断该资源是否符合候选资源的要求
3. 如果资源通过了筛选，再调用`boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition)`方法进一步筛选

`boolean isCandidateComponent(MetadataReader metadataReader)`方法默认遍历`excludeFilters`和`includeFilters`过滤器来检查资源是否匹配要求。如果不注册过滤器，返回`false`，即package下所有的类都不符合要求：

```java
protected boolean isCandidateComponent(MetadataReader metadataReader) throws IOException {
    for (TypeFilter tf : this.excludeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return false;
        }
    }
    for (TypeFilter tf : this.includeFilters) {
        if (tf.match(metadataReader, getMetadataReaderFactory())) {
            return isConditionMatch(metadataReader);
        }
    }
    return false;
}
```

`boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition)`方法默认会筛选掉接口以及内部类：

```java
protected boolean isCandidateComponent(AnnotatedBeanDefinition beanDefinition) {
    AnnotationMetadata metadata = beanDefinition.getMetadata();
    return (metadata.isIndependent() && (metadata.isConcrete() ||
            (metadata.isAbstract() && metadata.hasAnnotatedMethods(Lookup.class.getName()))));
}
```

经过上面一系列的步骤，最终返回符合我们需求的类定义(`BeanDefinition`)。


































[1]: /articles/mybatis/MyBatis探究(三)——MyBatis与Spring的结合及代码探究.html