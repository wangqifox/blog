---
title: SpringBoot自动配置探究
date: 2018/04/03 10:46:25
---

SpringBoot之所以能大幅减少程序员的配置工作，主要是因为有自动配置的功能。这篇文章我们来自己实现一个简单的能自动配置的工程。
<!-- more -->
## 自定义自动配置

首先定义一个`User`类：

```java
public class User {
    private String name;
    private Integer age;

    public User(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "{name:" + name + ", " + "age:" + age + "}";
    }
}
```

然后定义一个加载User属性的类`UserProperties`：

```java
@ConfigurationProperties(prefix = "user")
public class UserProperties {
    private String name;
    private Integer age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }
}
```

`UserProperties`能从application.yml文件中读取以user开头的配置，配置如下：

```
user:
  name: wangqi
  age: 21
```

要想达到自动配置的效果，需要定义一个配置类，然后在`META-INF/spring.factories`文件中加入自动配置的类。

配置类如下定义：

```java
@Configuration
@EnableConfigurationProperties(UserProperties.class)
@ConditionalOnClass(UserProperties.class)
@ConditionalOnProperty(prefix = "user", name = "name")
public class UserAutoConfiguration {
    @Autowired
    private UserProperties userProperties;

    @Bean
    public User user() {
        return new User(userProperties.getName(), userProperties.getAge());
    }
}
```

在`META-INF/spring.factories`文件中加入自动配置的类：

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=love.wangqi.UserAutoConfiguration
```

最后我们新建一个类来测试一下自动配置的效果：

```java
public class MainApplication {
    @Autowired
    UserProperties userProperties;

    @Bean
    User user() {
        return new User(userProperties.getName(), userProperties.getAge());
    }

    public static void main(String[] args) {
        ApplicationContext context = SpringApplication.run(MainApplication.class, args);
        User user = context.getBean(User.class);
        System.out.println(user);
    }

}
```

执行程序，输出`{name:wangqi, age:21}`。随着我们改变application.yml配置，会输出相应修改的结果。

## 自动配置代码探究

上面我们讨论了如何自定义自动配置的类，下面我们来探究一下这个自动配置是如何发生的。

我们直接看自动配置的发生点，调用流程如下：

1. SpringApplication.run
2. SpringApplication.refreshContext
3. AbstractApplicationContext.refresh
4. AbstractApplicationContext.invokeBeanFactoryPostProcessors
5. PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors
6. PostProcessorRegistrationDelegate.invokeBeanDefinitionRegistryPostProcessors
7. ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry
8. ConfigurationClassPostProcessor.processConfigBeanDefinitions
9. ConfigurationClassParser.parse
10. ConfigurationClassParser.processDeferredImportSelectors

在`ConfigurationClassParser.processDeferredImportSelectors`有以下几个步骤：

1. 调用`AutoConfigurationImportSelector.selectImports`获取所有`META-INF/spring.factories`文件中定义的EnableAutoConfiguration类。
2. 然后调用`ConfigurationClassParser.processImports`处理前面获取的EnableAutoConfiguration类

    调用`ConfigurationClassParser.processConfigurationClass`方法将`UserAutoConfiguration`当成Config类来解析。于是它注册User类的BeanDefinition，在随后的实例化过程实例化User。



