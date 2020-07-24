---
layout: post
title: OAuth教程--工具和帮助文档
date: 2018-07-14 02:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

### OAuth 2.0游乐场

`https://www.oauth.com/playground/`

![play](https://ws1.sinaimg.cn/large/006tKfTcly1ftoj9ztdrzj30sg0kw41k.jpg)

OAuth 2.0 Playground通过与真正的OAuth 2.0授权服务器交互，引导您完成各种OAuth流程。

它包含授权码流，PKCE，设备流以及OpenID Connect的简单示例。

### Google OAuth 2.0 Playground

`https://developers.google.com/oauthplayground/`

Google的OAuth 2.0 Playground允许您手动逐步完成从此测试应用程序访问您自己的Google帐户的授权过程。

您可以从可用范围列表中进行选择，点击“授权”以转到标准Google授权页面，重定向会将您返回到OAuth 2.0 Playground。在那里，您可以查看交换访问令牌的授权代码的请求，然后使用访问令牌进行测试以发出API请求。

该工具还允许您通过自定义授权和令牌endpoint来对其他OAuth 2.0服务器进行授权。

### OpenID Connect Debugger

`https://oidcdebugger.com`

OpenID Connect Debugger允许您测试OpenID Connect请求并调试来自服务器的响应。您可以将该工具配置为与任何OpenID服务器（如Google）配合使用。

### 服务器和客户端库目录

`https://oauth.net/code/`

oauth.net网站包含支持OAuth 2.0的服务器，客户端和服务的目录。您可以找到从完整的OAuth 2.0服务器实现到促进流程的每个步骤的库的任何内容，以及客户端库和代理服务。

如果您要提供任何库或服务，也可以将它们添加到目录中。

### jsonwebtoken.io

`https://jsonwebtoken.io`

![jwt](https://ws3.sinaimg.cn/large/006tKfTcly1ftojc727cbj30sg0ovahn.jpg)

jsonwebtoken.io是一个用于调试JSON Web令牌的工具。它允许您粘贴JWT，它将对其进行解码并显示各个组件。如果您向其提供用于签署JWT的秘钥，它还可以验证签名。