---
layout: post
title: OAuth教程--单页应用
date: 2018-07-02 10:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/oauth2-clients/single-page-apps/)）**

从网页加载Javascript和HTML源代码后，单页应用程序（或基于浏览器的应用程序）将会完全在浏览器中运行。由于浏览器可以使用所有源代码，因此无法保持客户端密钥的机密性，因此这些应用程序不会使用该密钥。该流程与授权代码流完全相同，但在最后一步，授权码在不使用客户端密钥的情况下交换访问令牌。

下图是用户与浏览器进行交互的示例，该浏览器直接向服务发出API请求。在首次从客户端下载Javascript和HTML源代码之后，浏览器会直接向服务发出API请求。在这种情况下，应用程序的服务器永远不会向服务发出API请求，因为所有事情都是直接在浏览器中处理的。

![用户的浏览器直接与API服务器通信](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/okta_oauth-diagrams.png)

### 授权

授权码是客户端交换访问令牌的临时代码。代码本身从授权服务器获得，用户可以查看客户端请求的信息，并批准或拒绝该请求。

Web流程的第一步是请求用户授权。这是通过为用户创建单击的授权请求链接来完成的。

授权URL通常采用以下格式：

```html
https://authorization-server.com/oauth/authorize
  ?client_id=a17c21ed
  &response_type=code
  &state=5ca75bd30
  &redirect_uri=https%3A%2F%2Fexample-app.com%2Fauth
```

用户访问授权页面后，服务会向用户显示请求的解释，包括应用程序名称，范围等。如果用户单击“批准”，服务器将重定向回网站，并附带授权码和URL查询字符串中的state值。

### 授权请求参数

以下参数用于发出授权请求。

>**client_id**
>这client_id是您的应用的标识符。首次向服务注册您的应用程序时，您将收到一个client_id。

>**response_type**
>response_type设置为code表示您希望以授权码作为响应。

>**redirect_uri （可选的）**
>该redirect_uri是在规范中可选的，但是有些服务需要它。这是您希望在授权完成后将用户重定向到的URL。这必须与您先前在服务中注册的重定向URL相匹配。

>**scope （可选的）**
>包括一个或多个范围值以请求其他访问级别。这些值将取决于特定的服务。

>**state （推荐的）**
>该state参数有两个功能。当用户被重定向回您的应用程序时，您在状态中包含的任何值都将包含在重定向中。这使您的应用程序有机会在被定向到授权服务器的用户和再次返回之间保留数据，例如使用state参数作为会话密钥。这可用于指示在授权完成后应用中要执行的操作，例如，指示在授权后要重定向到应用的哪个页面。这也可以作为CSRF保护机制。当用户重定向回您的应用程序时，请仔细检查状态值是否与您最初设置的值相匹配。这将确保攻击者无法拦截授权流程。

>请注意，缺少使用客户端密钥意味着使用state参数对单页应用程序更为重要。

### 举个栗子🌰

以下分步示例说明了对单页应用程序使用授权授予类型。

#### 步骤

##### 应用程序启动授权请求

应用程序通过制作包含ID的URL以及可选的scope和state来启动流程。该应用程序可以将其放入`<a href="">`标签中。

```html
<a href="https://authorization-server.com/authorize?response_type=code
     &client_id=mRkZGFjM&state=TY2OTZhZGFk">Connect Your Account</a>
```

##### 用户批准该请求

在被定向到auth服务器时，用户看到授权请求。

![示例授权请求](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/okta_oauth-diagrams-approve.png)

用户被带到服务并看到请求后，他们将允许或拒绝该请求。如果他们允许请求，他们将被重定向回重定向URL以及查询字符串中的授权码。然后，该应用程序需要交换授权码以获得访问令牌。

```http
https://example-app.com/cb?code=Yzk5ZDczMzRlNDEwY&state=TY2OTZhZGFk
```

如果您在初始授权网址中包含“state”参数，则该服务会在用户授权您的应用后将其返回给您。你的应用程序应该将state参数与它在初始请求中创建的state餐素进行比较。这有助于确保您只交换您请求的授权码，防止攻击者使用任意或被盗的授权码重定向到您的回调URL。

##### 交换访问令牌的授权码

要交换访问令牌的授权码，应用程序会向服务的令牌endpoint发出POST请求。请求将具有以下参数。

>**grant_type （需要）**
>该grant_type参数必须设置为“authorization_code”。

>**code （需要）**
>此参数用于从授权服务器接收的授权代码，该授权代码将位于此请求中的查询字符串参数“code”中。

>**redirect_uri （可能需要）**
>如果重定向URL包含在初始授权请求中，则它也必须包含在令牌请求中，并且必须相同。某些服务支持注册多个重定向URL，有些服务需要在每个请求上指定重定向URL。请查看服务的文档以了解具体信息。

>**客户端识别ID（必填）**
>尽管客户端密钥未在此流程中使用，但该请求需要发送客户端ID以识别发出请求的应用程序。这意味着客户端必须将客户端ID包含为POST主体参数，而不是像包含客户端密钥时那样使用HTTP基本认证。

```bash
POST /oauth/token HTTP/1.1
  Host: authorization-endpoint.com
  grant_type=code
  &code=Yzk5ZDczMzRlNDEwY
  &redirect_uri=https://example-app.com/cb
  &client_id=mRkZGFjM
```

#### 安全考虑

通过使用“state”参数并将重定向URL限制为可认证客户端，这是授权代码授予无客户端密钥的客户端的唯一安全方法。由于没有使用密钥，除了使用注册的重定向URL之外，没有办法验证客户的身份。这就是为什么您需要使用OAuth 2.0服务预先注册您的重定向网址。

尽管OAuth 2.0规范并不特别要求重定向URL使用TLS加密，但强烈建议您使用它。不需要的唯一原因是因为部署SSL网站对许多开发人员来说仍然是一个障碍，这将阻碍规范的广泛采用。有些API确实需要https作为重定向端点，但许多API仍然没有。