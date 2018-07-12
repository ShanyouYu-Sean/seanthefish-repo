---
layout: post
title: OAuth教程--授权请求
date: 2018-07-10 18:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/authorization/the-authorization-request/)）**

客户端会将用户的浏览器定向到授权服务器以开始OAuth流程。客户端可以使用授权码授予类型或隐式授权。除了response_type参数指定的授权类型外，请求还有许多其他参数来指示请求的细节。

[OAuth 2.0客户端](http://seanthefish.com/2018/06/29/oauth-guide-2/)描述了客户端如何为您的服务构建授权URL。授权服务器第一次看到用户申请此授权请求时，将使用客户端设置的查询参数将用户定向到服务器。此时，授权服务器将需要验证请求并提供授权接口，允许用户批准或拒绝该请求。

### 请求参数

以下参数用于开始授权请求。例如，如果授权服务器URL是`https://authorization-server.com/auth`, 客户端将创建如下的URL并将用户的浏览器指向它：

```html
https://authorization-server.com/auth?response_type=code
&client_id=29352735982374239857
&redirect_uri=https://example-app.com/callback
&scope=create+delete
&state=xcoivjuywkdkhvusuye3kch
```

>**response_type**
>response_type将设置为code，表示应用程序希望在成功时收到授权码。

>**client_id**
>client_id是应用程序的公共标识符。

>**redirect_uri （可选的）**
>redirect_uri不规范所要求的，但你的服务应该需要它。此URL必须与开发人员在创建应用程序时注册的其中一个URL匹配，如果请求不匹配，授权服务器应拒绝该请求。

>**scope （可选的）**
>请求可以具有一个或多个scope值，指示应用程序请求的附加访问。授权服务器需要向用户显示请求的scope。

>**state （推荐的）**
>state应用程序使用该参数来存储特定于请求的数据和/或防止CSRF攻击。授权服务器必须将未修改的state值返回给应用程序。

### 授予类型

当客户端应用程序期望授权码作为响应时，授权码授予类型将与密钥和公共客户端一起工作。要启动授权码授予，客户端将使用查询参数response_type=code以及其他所需参数将用户的浏览器定向到授权服务器。

### 验证授权请求

授权服务器必须首先验证client_id请求中的对应的有效的应用程序。

如果您的服务器允许应用程序注册多个重定向URL，则验证重定向URL有两个步骤。如果请求包含redirect_uri参数，则服务器必须确认它是此应用程序的有效重定向URL。如果redirect_uri请求中没有参数，并且只注册了一个URL，则服务器使用先前注册的重定向URL。否则，如果请求中没有重定向URL，并且没有注册重定向URL，就将会返回一个错误。

如果client_id无效，服务器应立即拒绝请求并向用户显示错误。

### 无效的重定向网址

如果授权服务器检测到重定向URL有问题，则需要通知用户该问题。由于多种原因，重定向网址可能无效，包括：

- 缺少重定向URL参数
- 重定向URL参数无效，例如，如果它是不解析为URL的字符串
- 重定向URL与应用程序的已注册重定向URL之一不匹配

在这些情况下，授权服务器应向用户显示错误，通知他们问题。服务器不得将用户重定向回应用程序。这避免了所谓的“[开放重定向器攻击](https://oauth.net/advisories/2014-1-covert-redirect/)”。如果已注册重定向URL，则服务器应仅将用户重定向到重定向URL。

### 其他错误

通过将用户重定向到重定向URL，并使用查询字符串中的错误代码来处理所有其他错误。有关如何响应错误的详细信息，请参阅“[授权响应](https://www.oauth.com/oauth2-servers/authorization/the-authorization-response/)”部分。

如果请求缺少response_type参数，或者该参数的值是除了code和token之外的任何内容，则服务器会返回invalid_request错误。

由于授权服务器可能要求客户端指定它们是公共的还是机密的，因此它可以拒绝不允许的授权请求。例如，如果客户端指定它们是机密客户端，则服务器可以拒绝使用令牌授权类型的请求。拒绝时，请使用错误代码unauthorized_client。

如果存在无法识别的scope值，授权服务器应拒绝该请求。在这种情况下，服务器可以使用invalid_scope错误代码重定向到回调URL 。

授权服务器需要存储此请求的“state”值，以便将其包含在访问令牌响应中。服务器不得修改或对state值包含的内容做出任何假设，因为它纯粹是为了客户端的便利。