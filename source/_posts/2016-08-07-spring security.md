---
title: Spring Security笔记
published: true
layout: post
tags:
	- SpringSecurity
	- CAS
categories:
	- Spring
---

#### authentication 和 authroization

**authentication: 身份认证** 用来获得request的身份，这需要用户登录提供用户名/密码，通过DB/LDAP/CAS等各种方式确认用户的合法性。

**authroization: 授权** 对于每个用户执行某种操作需要有相应的权限(GrantedAuthority)。通常通过给用户赋予一个或者多个的角色，或者用户组的概念。只有属于某个角色，才能执行相应的操作。

spring security就是为了提供这两种功能而实际的一套框架。它基于spring的依赖注入规范，支持多种身份验证实现。spring security的基本思想并不局限于某种具体需要权限认证的服务，但基于servlet规范的web application也提供了更好的支持，对web application 支持三个维度的授权实现：

+ 请求层级，对具体的请求根据url过滤；
+ 对具体Bean上方法，也是通常所说的服务层；
+ 对特定的domain object;
