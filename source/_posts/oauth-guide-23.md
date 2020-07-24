---
layout: post
title: OAuth教程--IndieAuth
date: 2018-07-13 23:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

IndieAuth是一个基于OAuth 2.0的分散式身份协议，它使用URL来识别用户和应用程序。它允许人们在使用该身份登录和授权应用程序时使用其控制下的域作为其身份。该规范可以在`https://www.w3.org/TR/indieauth/`找到。

所有用户ID都是URL，并且应用程序也通过其URL而不是预先注册的客户端ID进行标识。这使得它非常适用于您不希望开发人员在每个授权服务器上注册帐户的情况，例如在WordPress安装中对用户进行身份验证的应用程序。

IndieAuth分离授权服务器和发布访问令牌的角色，以便可以为流程的每个部分使用完全独立的实现和服务。

当应用程序只需要识别用户进行登录时，IndieAuth可以用作身份验证机制，或者应用程序可以使用IndieAuth来获取用户使用的访问令牌。

例如，Micropub客户端使用IndieAuth获取访问令牌，然后用于在用户的网站上创建内容。

IndieAuth建立在OAuth 2.0框架之上，如下所示：

- 指定用于标识用户的机制和格式（可解析的URL）
- 指定在给定profile URL的情况下发现授权和令牌endpoint的方法
- 指定客户端ID的格式（也作为可解析的URL）
- 所有客户都是公共客户，因为不使用客户秘钥
- 客户端注册不是必需的，因为所有客户端都必须使用可解析的URL作为其客户端ID
- 重定向URI注册是由应用程序在其网站上公布其有效的重定向URL来完成的
- 指定令牌endpoint和授权endpoint进行通信的机制，类似于令牌自解析这是针对授权码的
  
更多信息和规范可以在[indieauth.net](IndieAuth)找到。下面是两个工作流程的简要概述。

### 发现

在应用程序可以重定向到授权服务器之前，应用程序需要知道将用户引导到哪个授权服务器！这是因为每个用户都由URL标识，并且用户的URL指示其授权服务器所在的位置。

应用程序首先需要提示用户输入其URL，或以其他方式获取其URL。通常，应用程序将包含一个URL字段，供用户输入其URL。

该应用程序将向用户的URL发出HTTP GET请求，查找HTTP Link header或HTML `<link>`的标记的`authorization_endpoint`的`rel`值。在客户端也试图为用户获取访问令牌的情况下，它还将查找`token_endpoint`的`rel`值。

例如，GET请求`https://aaronparecki.com/`可能会返回以下内容，显示为缩写的HTTP请求。

```http
HTTP/2 200
content-type: text/html; charset=UTF-8
link: <https://aaronparecki.com/auth>; rel="authorization_endpoint"
link: <https://aaronparecki.com/token>; rel="token_endpoint"
link: <https://aaronparecki.com/micropub>; rel="micropub"
 
<!doctype html>
<meta charset="utf-8">
<title>Aaron Parecki</title>
<link rel="authorization_endpoint" href="/auth">
<link rel="token_endpoint" href="/token">
<link rel="micropub" href="/micropub">
```

请注意，endpoint URL可以是相对URL或绝对URL，并且可以位于与用户endpoint相同的域或不同的域上。这允许用户为任何组件使用托管服务。

有关发现的更多详细信息，请访问
`https://www.w3.org/TR/indieauth/#discovery-by-clients`。

### 登录工作流程

用户登录应用程序的基本流程如下。

- 用户以应用程序的登录形式输入其个人URL。
- 发现：应用程序获取URL并查找用户的授权endpoint。
- 授权请求：应用程序将用户的浏览器定向到发现的授权endpoint，作为标准OAuth 2.0授权以及在第一步中输入的用户URL。
- 身份验证/批准：用户在其授endpoint进行身份验证并批准登录请求。授权服务器生成授权代码并重定向回应用程序的重定向URL。
- 验证：应用程序检查授权endpoint处的代码，类似于交换访问令牌的代码，但不返回访问令牌，因为这只是对身份验证的检查。授权endpoint使用经过身份验证的用户的完整URL进行响应。

#### 认证请求

当应用程序构建URL以对用户进行身份验证时，该请求看起来与OAuth授权请求非常相似，除了不需要预先注册客户端，并且该请求还将包括用户的profile URL。URL将如下所示。

```http
https://user.example.net/auth?
    me=https://user.example.net/
    &amp;redirect_uri=https://example-app.com/redirect
    &amp;client_id=https://example-app.com/
    &amp;state=1234567890
```

然后授权服务器会要求用户登录，就像OAuth流程一样，然后询问用户是否要继续登录应用程序，如下所示。

![auth](https://ws1.sinaimg.cn/large/006tKfTcly1ftoifez6zej311g0fywm8.jpg)

如果用户批准，将使用查询字符串中的授权码（和应用程序的状态值）将它们重定向回应用程序。

然后，应用程序将获取授权码并使用授权endpoint对其进行验证，以确认登录用户的身份。应用程序向授权endpoint发出POST请求code，`client_id`并且`redirect_uri`就像正常的授权码一样交换。

```http
POST /auth
Host: user.example.net
Content-type: application/x-www-form-urlencoded
 
code=xxxxxxxx
&amp;client_id=https://example-app.com/
&amp;redirect_uri=https://example-app.com/redirect
```

响应将是一个简单的JSON对象，其中包含用户的完整profile URL。

```http
HTTP/1.1 200 OK
Content-Type: application/json
 
{
  "me": "https://user.example.net/"
}
```

有关处理请求和响应的更多详细信息，请参阅
`https://www.w3.org/TR/indieauth/#authorization-code-verification`。

### 授权工作流程

当应用程序尝试获取用户的访问令牌以修改或访问用户的数据时，将使用授权工作流程。这类似于访问数据中描述的OAuth 2.0授权码工作流程，但不使用预先注册客户端，因为使用了URL。

下面是授权应用程序的用户的基本流程。

- 用户以应用程序的登录形式输入其个人URL。
- 发现：应用程序获取URL并查找用户的授权和令牌endpoint。
- 授权请求：应用程序将用户的浏览器定向到发现的授权endpoint，作为标准OAuth 2.0授权授予和请求的范围，以及在第一步中输入的用户URL。
- 身份验证/批准：用户在其授权endpoint进行身份验证，查看请求的范围并批准请求。授权服务器生成授权代码并重定向回应用程序的重定向URL。
- 令牌交换：应用程序向令牌端点发出请求用授权码交换访问令牌。令牌端点使用访问令牌和经过身份验证的用户的完整URL进行响应。

#### 授权请求

当应用程序构建URL以对用户进行身份验证时，该请求看起来与OAuth授权请求非常相似，除了不需要预先注册客户端，并且该请求还将包括用户的profile URL。URL将如下所示。

```http
https://user.example.net/auth?
    me=https://user.example.net/
    &amp;response_type=code
    &amp;redirect_uri=https://example-app.com/redirect
    &amp;client_id=https://example-app.com/
    &amp;state=1234567890
    &amp;scope=create+update
```

请注意，与上述身份验证请求不同，此请求包括`response_type=code`和应用程序请求的请求范围列表。

授权服务器将要求用户登录，然后向他们提供授权提示。

不同的IndieAuth服务器可能会以不同的方式显示此提示，如我网站的授权服务器和下面显示的WordPress IndieAuth插件的屏幕截图所示。

![me](https://ws3.sinaimg.cn/large/006tKfTcly1ftoininhyaj31020y47fr.jpg)

![wordpress](https://ws1.sinaimg.cn/large/006tKfTcly1ftoio589r3j30bd0hrt9i.jpg)

当用户批准该请求时，服务器将用户重定向回具有查询字符串中的授权码的应用程序。

为了获得访问令牌，应用程序使用授权码和其他所需数据向用户的令牌endpoint（在第一个发现步骤中发现的endpoint）发出POST请求。

```http
POST /token
Host: user.example.net
Content-type: application/x-www-form-urlencoded</pre>
 
<pre class="break-before">grant_type=authorization_code
&amp;code=xxxxxxxx
&amp;client_id=https://example-app.com/
&amp;redirect_uri=https://example-app.com/redirect
&amp;me=https://user.example.net/
```

令牌endpoint将为用户生成访问令牌，并使用正常的OAuth 2.0令牌响应进行响应，并添加授权应用程序的用户的profile URL。

```http
HTTP/1.1 200 OK
Content-Type: application/json
 
{
  "me": "https://user.example.net/",
  "token_type": "Bearer",
  "access_token": "XXXXXX",
  "scope": "create update"
}
```