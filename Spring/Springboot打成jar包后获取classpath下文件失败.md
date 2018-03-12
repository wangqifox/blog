---
title: SpringBoot打成jar包后获取classpath下文件失败
date: 2018/03/12 18:29:00
---

使用如下代码：

```java
ClassPathResource resource = new ClassPathResource("application.yml");
File file = resource.getFile();
FileUtils.readLines(file).forEach(System.out::println);
```

未打包时可以获取到文件，打包成jar后报错。

这是因为打包后Spring试图访问文件系统路径，但无法访问jar包中的路径。因此必须使用`resource.getInputStream()`

```java
ClassPathResource resource = new ClassPathResource("application.yml");
InputStream inputStream = resource.getInputStream();
IOUtils.readLines(inputStream).forEach(System.out::println);
```


> http://blog.csdn.net/qq_18748427/article/details/78606432
> https://smarterco.de/java-load-file-from-classpath-in-spring-boot/

