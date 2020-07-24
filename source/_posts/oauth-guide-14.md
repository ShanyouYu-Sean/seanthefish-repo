---
layout: post
title: OAuth教程--资源服务器
date: 2018-07-13 14:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

资源服务器是API服务器的OAuth 2.0术语。资源服务器在应用程序获取访问令牌后处理经过身份验证的请求。

大规模部署可能有多个资源服务器。例如，Google的服务拥有数十种资源服务器，例如Google云平台，Google地图，Google云端硬盘，Youtube，Google +等等。这些资源服务器中的每一个都明显是分开的，但它们都共享相同的授权服务器。

![谷歌的一些API](https://ws4.sinaimg.cn/large/006tKfTcly1ftiyxmx2rrj30ox0dbgoc.jpg)

较小的部署通常只有一个资源服务器，并且通常构建为与授权服务器相同的代码库或相同部署的一部分。

### 验证访问令牌

资源服务器将从应用发出的请求中的具有的HTTP Authorization header获取访问令牌。资源服务器需要能够验证访问令牌以确定是否处理请求，并找到关联的用户帐户等。

如果您使用的是自编码访问令牌，则可以在资源服务器中完全验证令牌，而无需与数据库或外部服务器交互。

如果您的令牌存储在数据库中，则验证令牌需要在数据库中查找。

另一种选择是使用令牌自解析规范来构建API以验证访问令牌。这是处理在大量资源服务器上验证访问令牌的好方法，因为这意味着您可以将访问令牌的所有逻辑封装在单个服务器中，通过API将信息暴露给系统的其他部分。令牌自解析endpoint仅供内部使用，因此您需要使用某些内部授权来保护它，或者仅在系统防火墙内的服务器上启用它。

### 验证范围

资源服务器需要知道与访问令牌关联的范围列表。如果访问令牌中的作用域不包含执行指定操作所需的作用域，则服务器负责拒绝请求。

OAuth 2.0规范本身没有定义任何范围，也没有范围的注册表。范围列表由服务决定。

### 错误代码和未经授权的访问

如果访问令牌不允许访问所请求的资源，或者请求中没有访问令牌，则服务器必须回复HTTP 401响应并在相应header中包含WWW-Authenticate。

最简单的WWW-Authenticate标头包括字符串Bearer，表示需要承载令牌。标题还可以指示附加信息，例如“realm”和“scope”。“realm”值用于传统的[HTTP身份验证](https://tools.ietf.org/html/rfc2617)。“scope”值允许资源服务器指示访问资源所需的作用域列表，因此应用程序可以在启动授权流时请求适当的作用域。响应还应包括适当的“error”值，具体取决于发生的错误类型。

> **invalid_request （HTTP 400）** - 请求缺少参数，否则会出现格式错误。
> **invalid_token（HTTP 401）** - 由于其他原因，访问令牌已过期，已撤销，格式错误或无效。客户端可以获取新的访问令牌，然后重试。
> **insufficient_scope （HTTP 403）** - 访问令牌请求范围错误

例如：

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="example",
                  scope="delete",
                  error="insufficient_scope"
```

如果请求没有身份验证，则不需要错误代码或其他错误信息。

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="example"
```