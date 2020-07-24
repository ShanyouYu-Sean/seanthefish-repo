---
layout: post
title: OAuth教程--进行身份验证请求
date: 2018-07-13 7:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

无论您使用哪种授权类型，或者您是否使用了客户端密钥，您现在都可以使用API​​来使用OAuth 2.0 Bearer Token。

API服务器可以通过两种方式接受Bearer Token。一个在HTTP Authorization标头中，另一个在post body参数中。它取决于它支持的服务，因此您需要检查文档以确定。

在HTTP标头中传入访问令牌时，您应该发出如下请求：

```http
POST /resource/1/update HTTP/1.1
Authorization: Bearer RsT5OjbzRn430zqMLgV3Ia"
Host: api.authorization-server.com
 
description=Hello+World
```

如果服务接受帖子正文中的访问令牌，那么您可以发出如下请求：

```http
POST /resource/1/ HTTP/1.1
Host: api.authorization-server.com
 
access_token=RsT5OjbzRn430zqMLgV3Ia
&description=Hello+World
```

请注意，由于OAuth 2.0规范实际上并不需要上面的任何一个选项，因此您必须阅读与之交互的特定服务的API文档，以了解它们是否支持post body参数或HTTP标头。

访问令牌不应由您的应用程序解析。你应用程序唯一应该做的就是使用它来发出API请求。一些服务将使用结构化令牌（如JWT）作为其访问令牌，但客户端在这种情况下无需担心解码令牌。

实际上，尝试解码访问令牌是危险的，因为服务器不保证访问令牌将始终保持相同的格式。完全有可能在下次从服务获得访问令牌时，它将采用不同的格式。要记住的是，访问令牌对客户端是不透明的，并且应该仅用于发出API请求而不是自己解释。

如果您试图找出您的访问令牌是否已过期，您可以存储您第一次获得访问令牌时返回的过期时间，或者只是尝试发出请求查看当前令牌是否过期，并获取新的访问令牌。

如果您正在尝试查找有关登录用户的更多信息，您应该阅读特定服务的API文档以了解他们的建议。例如，Google的API使用OpenID Connect提供userinfo endpoint，该endpoint可以返回有关给定访问令牌的用户的信息。

### 刷新访问令牌

当您最初收到访问令牌时，它可能包含刷新令牌以及到期时间，如下例所示。

```json
{
  "access_token": "AYjcyMzY3ZDhiNmJkNTY",
  "refresh_token": "RjY2NjM5NzA2OWJjuE7c",
  "token_type": "bearer",
  "expires": 3600
}
```

刷新令牌的存在意味着访问令牌将过期，您将能够在没有用户交互的情况下获得新令牌。

“expires”值是访问令牌有效的秒数。这取决于令牌服务的提供商，并且可能取决于应用程序或组织自己的策略。您可以使用它来抢先刷新访问令牌，而不是等待带有过期令牌的请求失败。

如果您发出API请求并且令牌已经过期，您将收到一个指示响应。您可以检查此特定错误消息，然后刷新令牌并再次尝试请求。

如果您使用的是基于JSON的API，则可能会返回带有invalid_token错误的JSON错误响应。在一些情况下，WWW-Authenticate header也会出现invalid_token错误。

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer error="invalid_token"
  error_description="The access token expired"
Content-type: application/json
 
{
  "error": "invalid_token",
  "error_description": "The access token expired"
}
```

当您的代码识别出此特定错误时，它可以使用之前收到的刷新令牌向令牌endpoint发出请求，并返回可用于重试原始请求的新访问令牌。

要使用刷新令牌，请向服务的令牌endpoint发出POST请求grant_type=refresh_token，并包括刷新令牌和客户端凭据。

```http
POST /oauth/token HTTP/1.1
Host: authorization-server.com
 
grant_type=refresh_token
&amp;refresh_token=xxxxxxxxxxx
&amp;client_id=xxxxxxxxxx
&amp;client_secret=xxxxxxxxxx
```

响应将是新的访问令牌，也可能包含新的刷新令牌，就像您在交换访问令牌的授权代码时收到的那样。

```json
{
  "access_token": "BWjcyMzY3ZDhiNmJkNTY",
  "refresh_token": "Srq2NjM5NzA2OWJjuE7c",
  "token_type": "bearer",
  "expires": 3600
}
```

如果您没有获得新的刷新令牌，那么这意味着当新的访问令牌到期时，您现有的刷新令牌将继续工作。

请记住，用户可以在任何时候撤销应用程序 ，因此，当刷新访问令牌也失败时，您的应用程序需要能够处理这种情况。此时，您需要再次提示用户进行授权。
