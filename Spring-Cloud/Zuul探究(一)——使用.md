---
title: Zuul探究(一)——使用
date: 2018/05/02 11:08:25
---

最近开始做一个动态网关平台，集动态路由、动态权限、限流配额等功能为一体，为整个公司所有项目的对外接口提供一个统一的调用管理。之前各个项目的对外接口都是各自管理的，于是每个接口的权限控制不统一，接口的说明也散落在各个项目的接口文档里，导致对接工作十分繁复。

我们选择的技术方案是Zuul。

Zuul探究将是一个系列。从这篇文章开始，我将开始探究Zuul的相关技术，包括：

- Zuul的使用
- Zuul的工作原理
- Zuul动态路由的实现

<!-- more -->

## Zuul传统路由的配置

使用Zuul来构建网关的步骤非常简单，只需要下面几步：

- 创建一个基础的Spring Boot项目，命名为：ZuulTest。并在pom.xml中引入依赖：

    ```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.1.RELEASE</version>
    </parent>
    
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.RC1</spring-cloud.version>
    </properties>
    
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>
    </dependencies>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    
    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/libs-milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
    
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    ```

- 创建应用主类，并使用`@EnableZuulProxy`注解开启Zuul的功能

    ```java
    @EnableZuulProxy
    @SpringBootApplication
    public class ZuulApplication {
        public static void main(String[] args) {
            SpringApplication.run(ZuulApplication.class, args);
        }
    }
    ```

- 创建配置文件`application.yml`，并加入服务名、端口号、路由地址

    ```
    spring:
      application:
        name: zuul-test
    
    server:
      port: 8773
    
    zuul:
      routes:
        hello-world:
          path: /hello-world/**
          url: http://localhost:8600/
    ```

经过以上配置，我们访问`http://127.0.0.1:8773/hello-world`，该请求最终被路由到`http://localhost:8600/`接口上。

## Zuul服务路由的配置

服务路由的配置和传统的路由配置相比，不再需要手动指定路由。

- 首先在pom文件中加入`spring-cloud-starter-netflix-eureka-server`

    ```xml
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
    ```
    
- 然后在`application.yml`文件中加入eureka的配置：

    ```
    eureka:
      instance:
        hostname: localhost
      client:
        service-url:
          default-zone: http://${eureka.instance.hostname}:8761/eureka/
    ```
    
我们可以直接将API网关也看作是Eureka服务治理下的一个普通微服务应用。它除了会将自己注册到Eureka服务注册中心上之外，也会从注册中心获取所有服务以及它们的实例清单。所以，在Eureka的帮助下，API网关服务本身就已经维护了系统中所有serviceId与实例地址的映射关系。当有外部请求到达API网关的时候，根据请求的URL路径找到最佳匹配的path规则，API网关就可以知道要将该请求路由到哪个具体的serviceId上去。由于在API网关中已经知道serviceId对应服务实例的地址清单，那么只要通过Ribbon的负载均衡策略，直接在这些清单中选择一个具体的实例进行转发就能完成路由工作了。

> http://blog.didispace.com/spring-cloud-starter-dalston-6-1/
> http://blog.didispace.com/spring-cloud-starter-dalston-6-2/

