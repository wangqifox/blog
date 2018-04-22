---
title: MyBatis探究(四)——MyBatis与SpringBoot的结合及代码探究
date: 2018/04/22 10:11:00
---

MyBatis提供了MyBatis-Spring-Boot-Starter项目用于MyBatis与SpringBoot的整合。充分利用了SpringBoot自动配置的优点，使用起来非常方便。我们来看一下简单的使用：
<!-- more -->
## MyBatis-Spring-Boot-Starter的简单使用

首先引入MyBatis-Spring-Boot-Starter的包，以maven为例：

```java
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.2</version>
</dependency>
```

接着在application.yml文件中加入mybatis相关的配置：

```
spring:
  datasource:
    name: test
    url: jdbc:mysql://127.0.0.1:3306/mytest
    username: root
    password: root
    type: com.alibaba.druid.pool.DruidDataSource
    driver-class-name: com.mysql.jdbc.Driver
    druid:
      filters: stat
      pool-prepared-statements: true
      max-open-prepared-statements: 20
      max-active: 20
      initial-size: 1
      max-wait: 60000
      min-idle: 1
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      validation-query: select 'x'
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false

mybatis:
  mapper-locations: classpath:mapping/*.xml
  type-aliases-package: love.wangqi.springmybatisdemo.model
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl
```

经过以上两步，我们就配置好了Mybatis，然后就可以正常使用Mybatis了：加入`UserMapper.java`、`UserMapper.xml`，在`UserService`中注入`UserMapper`。

## MyBatis-Spring-Boot-Starter的探究

经过`SpringBoot自动配置探究的分析`，我们知道了SpringBoot自动配置的原理。根据其原理我们来探究一下MyBatis-Spring-Boot-Starter是如何工作的。

首先我们看到，mybatis-spring-boot-starter的pom文件引入了5个包，分别是：spring-boot-starter, spring-boot-starter-jdbc, mybatis-spring-boot-autoconfigure, mybatis, mybatis-spring。

mybatis-spring-boot-autoconfigure是自动配置的关键，查看它底下的文件，看到了`META-INF/spring.factories`，其中定义了`EnableAutoConfiguration`，为`org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration`。

`MybatisAutoConfiguration`这个类实际上代替了我们在Mybatis-Spring中Config类中的各种配置。查看其代码我们发现似曾相识。

它首先通过`@EnableConfigurationProperties(MybatisProperties.class)`注解注入了我们在`application.yml`文件中定义的Mybatis的各种配置。

然后定义了`SqlSessionFactory`, `SqlSessionTemplate`以及一个名为`AutoConfiguredMapperScannerRegistrar`的类，这个类继承了`ImportBeanDefinitionRegistrar`，在`registerBeanDefinitions`方法中扫描并注册各个Mapper类的BeanDefinition。


