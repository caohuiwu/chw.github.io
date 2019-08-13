---
title: CAS Shiro 单点登录 权限控制
date: 2019-03-05 14:25:22
tags: CAS Shiro 单点登录 权限控制
categories: 单点登录
---
这篇文章，将从登陆、登出流程，详细讲解CAS+Shiro的执行过程。
## 开始
uum：一个基于RBAC（Role-Based Access Control ）的权限系统（CAS client + shiro）
sso：CAS server，负责认证授权

> ### 登陆流程:

这里将以时序图的方式展现。

<!--more-->
![cas-login](cas-sso/cas-login.svg)

> ### 登出流程:

这里将以时序图的方式展现。
![cas-login](cas-sso/cas-logout.svg)

**局限性：**
>  * 采用shiro，各业务系统保存了用户信息，无统一保存用户信息容器，当用户改变，无法立即全局生效；并且退出时，需要发送MQ去清空各业务系统的信息，将导致MQ压力增大。
>  * 若设计不合理，SSO认证过程会很长，并且用户授权信息将很大，增加SSO服务压力。

**个人感悟**
针对上面现阶段系统的局限性，以下是个人优化想法：
>  * 认证过程，只验证用户的登录名密码。CAS client根据需要，再调用服务去获取用户授权信息，并将信息统一存放至Redis。
>  * 用户退出，清空Redis中的用户信息，

参考博客：
[shiro 登录授权过程详细解析](https://my.oschina.net/u/2415799/blog/865526)
[CAS 4.1.10 版本服務端原始碼解讀](https://www.codetw.com/lyxepf.html)
[从http验证流程解析CAS单点登录](https://www.jianshu.com/p/5ef9407c71af)
[CAS的详细登录流程](https://blog.csdn.net/qq_34246546/article/details/79493208)
[前端关于单点登录SSO的知识](https://juejin.im/post/5b8116afe51d4538d23db11e)
<!--more-->