---
layout: post
title: OAuth教程--范围(scope)
date: 2018-07-11 3:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/scope/)）**

范围(scope)是一种限制应用访问用户数据的方法。与授予对用户帐户的完全访问权限相比，为应用程序提供一种方式，来限制他们代表用户访问和操作的范围，通常是一种更有效的方式。

某些应用仅使用OAuth来识别用户，因此他们只需要访问用户ID和基本配置文件信息。其他应用可能需要知道更敏感的信息，例如用户的生日，或者他们可能需要能够代表用户发布内容，或修改个人资料数据。如果用户确切知道应用程序可以和不能对他们的帐户做什么，他们将更愿意授权应用程序。范围是一种控制访问权限的方法，可帮助用户识别他们授予应用程序的权限。

- [定义范围](https://www.oauth.com/oauth2-servers/scope/defining-scopes/)
- [用户界面](https://www.oauth.com/oauth2-servers/scope/user-interface/)
- [复选框](https://www.oauth.com/oauth2-servers/scope/checkboxes/)