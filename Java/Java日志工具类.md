---
title: Java日志工具类
date: 2017/11/23 15:43:00
---


# Java日志工具类

比较常用的日志工具类有Log4j、SLF4j、Commons-logging(简称jcl)、logback、log4j2(log4j升级版)、jdk logging。大致可分为两类：

- 接口类，包括Commons-logging、SLF4j
- 实现类，Log4j、log4j2、logback、jdk logging

<!--more-->

## 日志接口类工具

Java应用系统中的日志门面，即它们提供了一套通用的接口，具体的实现可以由开发者自由选择。一般我们在程序开发的时候，会选择使用commons-logging或者slf4j这些日志门面，而不是直接使用log4j或者logback这些实现。这样做的好处是，我们可以自由选择日志实现。

- commons-logging

commons-logging是apache最早提供的日志的门面接口。它的主要作用是提供一个日志门面，使用者可以使用不同的日志实现。用户可以自由选择第三方的日志组件作为具体实现，像log4j，或者jdk自带的logging，common-logging会通过动态查找的机制，在程序运行时自动找出真正使用的日志库。common-logging内部有一个Simple logger的简单实现，但是功能很弱。

- SLF4J

SLF4J是Simple Logging Facade for Java的简称，即java的简单日志门面。类似于Apache Common-logging，是对不同日志框架提供的一个门面封装，可以在部署的时候不修改任何配置即可接入一种日志实现方案。但是，它在编译时静态绑定真正的log库。使用SLF4J时，如果你需要使用某一种日志实现，那么你必须选择正确的SLF4J的jar包的集合(各种桥接包，如slf4j-log4j12.jar、slf4j-jcl.jar等；注意：如果有任意两个实现slf4j的包同时出现，那么就可能出现问题)。

SLF4J相对于jcl接口能支持多个参数，并通过{}占位符进行替换，避免logger.isXXXEnabled这种无奈的判断，带来性能提升。

## 日志实现类工具

- log4j

Log4j：经典的一种日志解决方案。内部把日志系统抽象成logger、appender、pattern等实现。我们可以通过配置文件轻松地实现日志系统的管理和多样化配置。apache的一个开源项目，通过使用log4j，我们可以控制日志信息输送的目的地是控制台、文件、GUI组件、甚至是套接口服务器、NT的事件记录器、UNIXSyslog守护进程等；用户也可以控制每一条日志的输出格式；通过定义每一条日志信息的级别，用户能够更加细致地控制日志的生成过程。这些可以通过一个配置文件来灵活地进行配置，而不需要修改程序代码。

- log4j2

Log4j2是Log4j的升级版，与之前的版本log4j 1.x相比有重大的改进，在修正了logback固有的架构问题的同时，改进了许多logback所具有的功能。log4j2与log4j1发生了很大的变化，不兼容。log4j1仅仅作为一个实际的日志框架，slf4j、commons-logging作为门面，统一各种日志框架的混乱格局，现在log4j2也想跳出来充当门面了，也想统一大家了。

- logback

logback也是一种日志实现。logback是由log4j创始人设计的又一个开源日志组件。logback当前分成三个模块：logback-core, logback-classic和logback-access。logback-core是其他两个模块的基础模块。logback-classic是log4j的一个改良版本。此外logback-classic完整实现slf4j api使你可以很方便地更换成其他日志系统如log4j或jdk logging。logback-access访问模块与servlet容器集成提供通过http来访问日志的功能。

## slf4j提供的适配库和桥接库

适配器（旧日志输出到slf4j）：

- slf4j-log4j12: 使用log4j-1.2作为日志输出服务
- slf4j-jdk14: 使用java.util.logging作为日志输出服务
- slf4j-jcl: 使用JCL作为日志输出服务
- slf4j-simple: 日志输出至System.err
- slf4j-nop: 不输出日志
- log4j-slf4j-impl: 使用log4j2作为日志输出服务

logback天然与slf4j适配，不需要额外引入适配库

桥接库（slf4j输出到实现）：

- log4j-over-slf4j: 将使用log4j api输出的日志桥接至slf4j
- jcl-over-slf4j: 将使用jcl api输出的日志桥接至slf4j
- jul-to-slf4j: 将使用java.util.logging输出的日志桥接至slf4j
- log4j-to-slf4j: 将使用log4j2输出的日志桥接至slf4j


参考资料：

> http://tech.lede.com/2017/02/06/rd/server/log4jSearch/
>
> http://www.jianshu.com/p/85d141365d39


