---
title: Spring启动过程分析.番外(注释整理)
date: 2018/01/06 15:15:00
---

## @PropertySource

指定配置文件的地址

- name：指定属性源的名称
- value：指定属性文件的位置
- ignoreResourceNotFound：默认为false。如果设为true，忽略value中不存在的属性文件
- encoding：指定属性文件的编码
- factory：指定一个自定义的PropertySourceFactory类
<!--more-->
## @ComponentScan

指定扫描的包名，将所有被@Component, @Service, @Repository, @Controller注释的类注册为bean

- name：basePackages的别名
- basePackages：需要扫描的包
- basePackageClasses：类型安全的basePackages
- nameGenerator：指定BeanNameGenerator类，用于命名components
- scopeResolver：指定ScopeMetadataResolver类，用于解析components的scope
- scopedProxy：指定是否应该为components生成代理
- resourcePattern：只为符合resourcePattern的类生成bean
- useDefaultFilters：指定是否应该自动检测被@Component, @Service, @Repository, @Controller注释的类
- includeFilters：指定扫描哪种类型
- excludeFilters：指定不扫描哪种类型
- lazyInit：是否懒加载，默认为false
- Filter
	- type：过滤器的类型
	- value：classes的别名
	- classes：指定哪些类作为过滤器
	- pattern：指定符合pattern的类作为过滤器

## @Import

导入一个或多个由@Configuration注释的配置类。如果要导入XML或者其他非代码的bean定义，需要使用@ImportResource。

- value：由@Configuration注释、实现了ImportSelector接口（决定应该导入哪些配置类）、实现了ImportBeanDefinitionRegistrar接口（注册另外的beanDefinition）的类，或者一般的component类

## @ImportResource

导入一个或多个包含bean定义的资源。默认情况下，使用GroovyBeanDefinitionReader来处理".groovy结尾的文件"，使用XmlBeanDefinitionReader来处理XML文件。

- value：locations的别名
- locations：导入资源的位置
- reader：用于出来资源的自定义BeanDefinitionReader

## @Bean

表示该方法生成一个由Spring容器管理的bean

- value：name的别名
- name：bean的名称
- autowire：指定根据名称还是类型自动装配
- initMethod：指定bean初始化时调用的方法
- destroyMethod：指定bean关闭时调用的方法

## @Required

表示方法(通常是JavaBean的setter方法)是"必须的"。这意味着，setter方法必须被配置为依赖注入一个值

