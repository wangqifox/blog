---
title: Shiro读书笔记
date: 2017/10/19 08:29:25
---

## 第一章

- Subject：主体，可以看到主体可以是任何可以与应用交互的"用户"
- SecurityManager：相当于SpringMVC中的DispatcherServlet或者Struts2中的FilterDispatcher；是Shiro的心脏；所有具体的交互都通过SecurityManager进行控制；它管理着所有Subject、且负责进行认真和授权、会话、缓存的管理
- Authenticator：认证器，负责主题认证。其需要认证策略(Authentication Strategy)，即什么情况下算用户认证通过了。
- Authorizer：授权器，或者叫访问控制器，用来决定主体是否有权限进行相应的操作；即控制着用户能访问应用中的哪些功能。

