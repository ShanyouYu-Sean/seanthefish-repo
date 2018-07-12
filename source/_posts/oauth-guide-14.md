---
layout: post
title: OAuth教程--授权响应
date: 2018-07-10 21:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/authorization/the-authorization-response/)）**

根据授权类型，授权服务器将使用授权码或访问令牌进行响应。

### 授权码响应

如果请求有效并且用户授予授权请求，则授权服务器生成授权码并将用户重定向回应用程序，将代码和先前的“state”值添加到重定向URL。

### 生成授权码

授权码必须在颁发后不久到期。OAuth 2.0规范建议最长生命周期为10分钟，但实际上，大多数服务设置的到期时间要短得多，大约30-60秒。授权代码本身可以是任意长度，但应记录代码的长度。

因为授权代码是短暂的并且是单次使用的，所以它们是实现自编码的很好的候选者。使用此技术，您可以避免将授权代码存储在数据库中，而是将所有必要的信息编码到代码本身中。您可以使用服务器端环境的内置加密库，也可以使用JSON Web Signature（JWS）等标准。由于此字符串只需要您的授权服务器可以理解，因此不需要使用JWS等标准来实现此字符串。也就是说，如果您没有可以轻松访问的已经可用的加密库，JWS是一个很好的候选者，因为有许多语言的库可用。

需要与授权代码关联的信息如下。

>**client_id**
>请求此代码的客户端ID（或其他客户端标识符）

>**redirect_uri**
>使用的重定向网址。需要存储，因为访问令牌请求必须包含相同的重定向URL，以便在发出访问令牌时进行验证。

>**用户信息(User info)**
>用于标识此授权代码所针对的用户的某种方式，例如用户ID。

>**到期日期(Expiration Date)**
>代码需要包含到期日期，以便使它能只持续很短的时间。

>**唯一ID(Unique ID)**
>代码需要其自己的某种ID，以便能够检查代码之前是否已被使用过。数据库ID或随机字符串就足够了。

通过创建JWS令牌或生成随机字符串并将关联信息存储在数据库中生成授权码后，您需要将用户重定向到指定的应用程序重定向URL。要添加到重定向URL的查询字符串的参数如下：

>**code**
>此参数包含客户端稍后将为访问令牌交换的授权码。

>**state**
>如果初始请求包含state参数，则响应还必须包含请求中的确切值。客户端将使用此将此响应与初始请求相关联。

例如，授权服务器通过发送以下HTTP响应来重定向用户。

```http
HTTP/1.1 302 Found
Location: https://oauth2client.com/redirect?code=g0ZGZmNjVmOWI&state=dkZmYxMzE2
```

### 隐式授权类型响应

使用隐式授权，授权服务器立即生成访问令牌，并使用令牌和其他参数重定向到回调URL。有关生成访问令牌的详细信息以及响应中所需参数的详细信息，请参阅访问[令牌响应](https://www.oauth.com/oauth2-servers/access-tokens/access-token-response/)。

例如，授权服务器通过发送以下HTTP响应来重定向用户（出于显示目的，需要额外的换行符）。

```http
HTTP/1.1 302 Found
Location: https://example-app.com/redirect#access_token=MyMzFjNTk2NTk4ZTYyZGI3
 &state=dkZmYxMzE2
 &token_type=bearer
 &expires_in=86400
```

### 错误响应

在两种情况下，授权服务器应直接显示错误消息，而不是将用户重定向到应用程序：如果client_id无效，或者redirect_uri无效。在所有其他情况下，可以将用户重定向到应用程序的重定向URL以及描述错误的查询字符串参数。

重定向到应用程序时，服务器会将以下参数添加到重定向URL：

>**error**
>通常是以下列表中的一个ASCII错误代码：
>- *invalid_request* - 请求缺少参数，包含无效参数，多次包含参数，或者无效。
>- *access_denied* - 用户或授权服务器拒绝该请求
>- *unauthorized_client* - 不允许客户端使用此方法请求授权码，例如，如果机密客户端尝试使用隐式授权类型。
>- *unsupported_response_type8* - 服务器不支持使用此方法获取授权代码，例如，如果授权服务器从未实现隐式授权类型。
>- *invalid_scope* - 请求的范围无效或未知。
>- *server_error* - 服务器可以使用此错误代码重定向，而不是向用户显示500内部服务器错误页面。
>- *temporarily_unavailable* - 如果服务器正在进行维护或不可用，则可以返回此错误代码，而不是使用503 Service Unavailable状态代码进行响应。

>**error_description**
>授权服务器可以可选地包括错误的描述。此参数旨在供开发人员理解错误，而不是要向最终用户显示。除双引号和反斜杠外，此参数的有效字符是ASCII字符集，特别是十六进制代码20-21,23-5B和5D-7E。

>**error_uri**
>服务器还可以将URL返回到人类可读的网页，其中包含有关错误的信息。这是为了让开发人员获得有关错误的更多信息，而不是要向最终用户显示。

>**state**
>如果请求包含状态参数，则错误响应还必须包含请求中的确切值。客户端可以使用它将此响应与初始请求相关联。

例如，如果用户拒绝授权请求，服务器将构造以下URL并发送HTTP重定向响应（如下所示）（URL中的换行符用于说明目的）。

```http
HTTP/1.1 302 Found
Location: https://example-app.com/redirect?error=access_denied
 &error_description=The+user+denied+the+request
 &error_uri=https%3A%2F%2Foauth2server.com%2Ferror%2Faccess_denied
 &state=wxyz1234
```