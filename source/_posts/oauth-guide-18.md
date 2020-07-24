---
layout: post
title: OAuth教程--令牌自解析endpoint（Token Introspection Endpoint）
date: 2018-07-13 18:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

当OAuth 2.0客户端向资源服务器发出请求时，资源服务器需要某种方式来验证访问令牌。OAuth 2.0核心规范没有定义资源服务器应如何验证访问令牌，只是提到它需要在资源和授权服务器之间协调。在某些情况下，特别是对于小型服务，两个endpoint都是同一系统的一部分，并且可以在内部共享令牌信息，例如在数据库中。在两个endpoint位于不同服务器上的较大系统中，这导致了两个服务器之间通信的专有和非标准协议。

OAuth 2.0 令牌自解析扩展定义了一个协议，该协议返回有关访问令牌的信息，供资源服务器或其他内部服务器使用。

### 令牌自解析规范

令牌自解析规范可以在`https://tools.ietf.org/html/rfc7662`找到

### 自解析endpoint

令牌自解析endpoint需要能够返回有关令牌的信息，因此您很可能将其构建在令牌endpoint所在的相同位置。两个endpoint需要共享数据库，或者如果您已实现自编码令牌，则需要共享密钥。

### 令牌信息请求

该请求将是一个POST请求，仅包含名为“token”的参数。预计此endpoint不会公开给开发人员。不应允许最终用户客户端使用此endpoint，因为响应可能包含开发人员无权访问的特权信息。保护endpoint的一种方法是将其放在无法从外部访问的内部服务器上，或者可以使用HTTP basic auth证进行保护。

```http
POST /token_info HTTP/1.1
Host: authorization-server.com
Authorization: Basic Y4NmE4MzFhZGFkNzU2YWRhN
 
token=c1MGYwNDJiYmYxNDFkZjVkOGI0MSAgLQ
```

### 令牌信息响应

令牌自解析Endpoint应该使用具有下面列出的属性的JSON对象进行响应。必须的只有“active”属性，其余属性是可选的。自解析规范中的一些属性专门用于JWT令牌，因此我们将仅在此处介绍基本的属性。如果您有关于可能有用的令牌的其他信息，您还可以在响应中添加其他属性。

> **active**
> 必须。这是所呈现的令牌当前是否处于活动状态的布尔值。如果此授权服务器已发出令牌，用户尚未撤消该令牌且未过期，则该值应为“true”。

> **scope**
> 一个JSON字符串，以空格分隔,包含与此标记关联的范围列表。

> **client_id**
颁发令牌的OAuth 2.0客户端的客户端标识符。

> **username**
> 授权此令牌的用户的可读标识符。

> **exp**
> unix时间戳（整数时间戳，UTC自1970年1月1日以来的秒数），指示此令牌何时到期。

### 示例响应

下面是自解析endpoint返回的响应示例。

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
 
{
  "active": true,
  "scope": "read write email",
  "client_id": "J8NFmU4tJVgDxKaJFmXTWvaHO",
  "username": "aaronpk",
  "exp": 1437275311
}
```

### 错误响应

如果自解析endpoint可公开访问，则endpoint必须首先验证身份验证。如果身份验证无效，则endpoint应使用`HTTP 401`状态代码和`invalid_client`响应进行响应。

```http
HTTP/1.1 401 Unauthorized
Content-Type: application/json; charset=utf-8
 
{
  "error": "invalid_client",
  "error_description": "The client authentication was invalid"
}
```

任何其他错误都被视为“inactive”令牌。

- 请求的令牌不存在或无效
- 令牌已过期
- 令牌发送给不同的客户端而不是发出此请求

在任何这些情况下，它都不被视为错误响应，并且endpoint仅返回非活动标志。

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
 
{
  "active": false
}
```

### 安全考虑因素

使用令牌自解析endpoint意味着,任何资源服务器都将依赖endpoint来确定访问令牌当前是否处于活动状态。这意味着自解析endpoint独自负责决定API请求是否成功。因此，endpoint必须针对令牌的状态执行所有适用的检查，例如检查令牌是否已过期，验证签名等。

#### 令牌钓鱼攻击

如果自解析endpoint保持打开且不受限制，则它为攻击者提供了一种方法，用于轮询endpoint捕获有效令牌。为防止这种情况发生，服务器必须要求使用endpoint对客户端进行身份验证，或者仅通过其他方式（如防火墙）使endpoint只用于内部服务器。

请注意，资源服务器也是钓鱼攻击的潜在目标，并且应该采取诸如速率限制之类的对策来防止这种情况。

#### 高速缓存

自解析endpoint的使用者可能希望出于性能原因缓存endpoint的响应。因此，在决定缓存时考虑性能和安全性权衡非常重要。例如，较短的缓存到期时间将导致更高的安全性，因为资源服务器将不得不更频繁地查询自解析endpoint，但是这将导致endpoint上的负载增加。较长的到期时间会使一个窗口打开，其中令牌可能实际上已过期或被撤销，但由于缓存时间有剩余，仍然能够在资源服务器上被查到。

缓解问题的一种方法是，使用者永远不会缓存至超出令牌的到期时间，这将在自解析响应的“exp”参数中返回。

#### 限制信息

自解析endpoint不一定需要为同一令牌的所有查询返回相同的信息。例如，两个不同的资源服务器（如果它们在进行自解析请求时进行身份验证）可能会获得令牌状态的不同视图。这可用于限制有关返回到特定资源服务器的令牌的信息。这使得多个资源服务器在互不知情上可以使用相同的令牌。