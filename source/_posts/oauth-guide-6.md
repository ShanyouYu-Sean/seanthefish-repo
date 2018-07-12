---
layout: post
title: OAuth教程--进行身份验证请求
date: 2018-07-08 15:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/oauth2-clients/making-authenticated-requests/)）**

无论您使用哪种授权类型，或者您是否使用了客户端密钥，您现在都可以使用API​​来使用OAuth 2.0 Bearer Token。

API服务器可以通过两种方式接受Bearer Token。一个在HTTP Authorization标头中，另一个在post body参数中。它取决于它支持的服务，因此您需要检查文档以确定。

在HTTP标头中传入访问令牌时，您应该发出如下请求：

```bash
POST /resource/1/update HTTP/1.1
Authorization: Bearer RsT5OjbzRn430zqMLgV3Ia"
Host: api.authorization-server.com
 
description=Hello+World
```

如果服务接受帖子正文中的访问令牌，那么您可以发出如下请求：

```bash
POST /resource/1/ HTTP/1.1
Host: api.authorization-server.com
 
access_token=RsT5OjbzRn430zqMLgV3Ia
&description=Hello+World
```

请注意，由于OAuth 2.0规范实际上并不需要上面的任何一个选项，因此您必须阅读与之交互的特定服务的API文档，以了解它们是否支持post body参数或HTTP标头。