---
layout: post
title: 什么是OAuth？
date: 2018-06-19 14:00:00
tags: 
- OAuth
- OIDC
categories:
- OAuth
- OIDC
---
**本文是Matt Raible的What the Heck is OAuth?的翻译。（[原文地址](https://developer.okta.com/blog/2017/06/21/what-the-heck-is-oauth)）**

>围绕OAuth的实际情况存在很多混淆。
>
>有些人认为OAuth是一种登录流程（就像当您使用Google登录登录应用程序时一样），有些人不假思索的认为OAuth是一种“安全事物”。
>
>我将向您展示OAuth是什么，解释它的工作原理，并希望让您了解OAuth是如何使您的应用受益的。

### 什么是OAuth？

我们先从最顶层的描述说起，OAuth 不是 API或服务：它是授权的开放标准，任何人都可以实施。

更具体地说，OAuth是应用程序可以用来为客户端提供“安全授权访问”的标准。OAuth通过HTTPS工作，并使用access tokens为设备，API，服务器和应用程序授权，而非credentials。

有两种版本的OAuth：[OAuth 1.0a](https://tools.ietf.org/html/rfc5849)和[OAuth 2.0](https://tools.ietf.org/html/rfc6749)。这些规范是完全不同的，不能一起使用：它们之间没有向后兼容性。

哪一个更受欢迎？如今，OAuth 2.0是最广泛使用的OAuth形式。所以从现在开始，每当我说“OAuth”时，我都在谈论OAuth 2.0--因为它很可能是您将要使用的。

### 为什么使用OAuth？

OAuth是对直接身份验证模式的响应。这种模式因HTTP基本认证（Basic Authentication）而闻名，用户会被提示输入用户名和密码。基本身份验证仍然是服务器端应用程序的API身份验证的基本形式：用户发送API密钥ID和密钥，而不是通过每个请求向服务器发送用户名和密码。在OAuth之前，网站会提示您直接在表单中输入用户名和密码，他们会像您一样登录您的数据（例如您的Gmail帐户）。这通常被称为[反密码模式(the password anti-pattern)](https://arstechnica.com/information-technology/2010/01/oauth-and-oauth-wrap-defeating-the-password-anti-pattern/)。

为了为网络创建更好的系统，联合身份为单点登录（SSO）的场景所创建。在这种情况下，终端用户与他们的身份提供者交谈，并且身份提供者生成一个加密签名的token，交给应用程序对用户进行身份验证。应用程序信任身份提供者。只要这种信任关系与登录的断言一起工作，你就登陆成功了。下图显示了这是如何工作的。

![oauth工作流程](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/browser_spa_implicit_flow-9f0d10069f4363030e4283679bd4914f9aa47e192b32a166d1e186bdb929e1d2.png)

### 浏览器隐式流程

联合身份因为由2005年3月15日发布的OASIS标准SAML 2.0而变得知名。它是一个很大的规范，但主要的两个组件是它的身份验证请求协议（又名Web SSO）以及它打包身份属性和签名的方式，称为SAML断言。Okta用SSO chiclets做到这一点。我们发送一条消息，我们在断言中签名，在断言中记录着用户是谁，来自Okta。再加上一个数字签名，你就登录成功了。

### SAML
SAML基本上是浏览器中的会话cookie，可让您访问webapps。如果您想要在Web浏览器之外的各种设备和场景中使用，它将会受到限制。

当SAML 2.0于2005年推出时，它是有道理的。不过，自那以后发生了很多变化。现在我们拥有现代化的网页和原生应用程序开发平台，有单页面应用程序（SPA），如Gmail / Google收件箱，Facebook和Twitter。它们与传统的Web应用程序有不同的行为，因为它们会对API进行AJAX（后台HTTP调用）。移动电话也进行API调用，电视，游戏控制台和物联网设备也一样。SAML SSO在这方面并不是特别优秀。

### OAuth和API
我们构建API的方式也发生了很大变化。在2005年，人们投资于WS- *来构建Web服务。现在，大多数开发人员已经转移到REST和无状态API。简而言之，REST是通过网络推送JSON包的HTTP命令。

开发人员构建了很多API。API经济是您今天可能在会议室听到的常见流行词。公司需要保护其REST API，以允许许多设备访问它们。在过去，您需要输入您的用户名/密码，该应用程序将直接以您的身份登录。这引起了委托授权问题。

“我怎样才能允许应用程序访问我的数据，而不必给我的密码？”

如果您见过以下对话框之一，那就是我们正在谈论的内容。这是一个应用程序，询问您是否可以代表您访问数据。

![Facebook OAuth](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/biketoworkday-fb-login-f00e39aabbf3e44bc3570333643cbf5d966fc27367dbffd2623ff4a3694831c3.png)

这是OAuth。

OAuth是REST / API的委托授权框架。它使应用程序可以在不泄露用户密码的情况下获取用户数据的有限访问权限（范围）。它将认证(authentication)与授权(authorization)分离开来，并支持多种用例来满足不同的设备功能。它支持服务器到服务器应用程序，基于浏览器的应用程序，移动/本机应用程序和控制台/电视。

对于应用程序来说，你可以把它想成酒店钥匙卡。如果你有酒店钥匙卡，你可以进入你的房间。你如何获得酒店钥匙卡？您必须在前台进行身份验证才能获得。在认证并获得钥匙卡后，您可以访问整个酒店的资源。

简单地说，OAuth就是：

- 应用程序向用户请求授权
- 用户授权应用程序并提供证据
- 应用程序向服务器提供授权证明以获取令牌（token）
- 令牌（token）仅限于访问用户为特定应用程序授权的内容

### OAuth中央组件
OAuth建立在以下中心组件之上：

- 作用域和同意书（Scopes and Consent）
- 扮演者（Actors）
- 客户端（Clients）
- 令牌（Tokens）
- 授权服务器（Authorization Server）
- 流程（Flows）
 
### OAuth作用域
作用域是您在应用程序请求权限时在授权屏幕上看到的内容。它们是客户在请求令牌时所需求的权限包。这些由应用程序开发人员在编写应用程序时进行编码。

![OAuth作用域](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/auth-scope.png)

作用域将授权策略的决策与执行分离。这是OAuth的第一个关键方面。权限是中心。它们不会隐藏在应用程序层后面。它们经常在API文档中列出：以下是这个应用程序需要的范围。

你必须获得这一同意。这被称为首次使用的信任。这是网络上非常重要的用户体验变化。OAuth之前的大多数人只是用id和密码对话框。现在你有了这个新的屏幕，你必须训练用户使用。重新调整互联网人口是困难的。从熟悉技术的年轻人到祖父母都有各种各样的用户，他们不熟悉这种流程。这是网络上的一个新概念，现在是前沿和中心。现在你必须授权并征得同意。

同意书可以根据应用程序而有所不同。它可以是时间敏感的作用域范围（日，周，月），但并非所有平台都允许您选择持续时间。当您同意时，需要注意的一点是，该应用程序可以代表您执行某些操作 - 例如，LinkedIn会将您网络中的每个人都发送出去。

OAuth是互联网规模的解决方案，因为它应用于每个应用程序。您经常可以登录到仪表板，查看您授予访问权的应用程序并撤消同意。

### OAuth扮演者
OAuth流程中的角色如下所示：

- 资源所有者：拥有资源服务器中的数据。例如，我是我的Facebook个人资料的资源所有者。
- 资源服务器：存储应用程序想要访问的数据的API
- 客户端：想要访问您的数据的应用程序
- 授权服务器：OAuth的主要引擎
![OAuth演员](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/oauth-actor.png)

资源所有者是一个可以使用不同凭据更改的角色。它可以是最终用户，也可以是公司。

客户端可以公开和保密。OAuth命名法中两者之间存在显着的区别。受信任的客户端可以信任存储秘钥。它们不是在桌面上运行，或通过应用商店分发。人们无法对其进行逆向工程并获得密钥。他们在最终用户无法访问的受保护区域运行。

公共客户端是浏览器，移动应用程序和物联网设备。

![OAuth客户端](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/oauth-client.png)

客户端注册也是OAuth的关键组件。这就像OAuth的DMV（车辆管理局）。您需要获取应用程序的牌照。这是您的应用在授权对话框中显示的方式。

### OAuth令牌
访问令牌是客户用来访问资源服务器（API）的令牌。他们的目的是短暂的。想象他们在几个小时和几分钟内，而不是几个月和几个月。你不需要一个机密的客户端来获得访问令牌。您可以使用公共客户端来获取访问令牌。它们旨在优化互联网规模问题。因为这些令牌可以短暂存在并扩展出来，所以它们不能被撤销，你只能等待它们超时。

另一个令牌是刷新令牌。这种寿命要长得多。几天，几个月，几年。这可以用来获得新的令牌。要获得刷新令牌，应用程序通常需要受信任客户端进行身份验证。

刷新令牌可以被撤消。当在仪表板中撤销应用程序的访问时，您正在刷新它的刷新令牌。这使您能够强制客户更新秘钥。您使用刷新令牌获取新的访问令牌，访问令牌通过线路打所有的API资源。当你每次刷新访问令牌时，都会得到一个新的加密签名令牌。更新的开关内置于系统中。

OAuth规范并未定义令牌的含义。它可以以任何你想要的格式。通常情况下，您希望这些令牌是JSON Web Token（一种标准）。简而言之，JWT（发音为“jot”）是令牌认证的安全可靠标准。JWT允许您使用签名对信息进行数字签名(claims)，并可以在稍后使用一个秘密的签名密钥进行验证。要了解关于JWT的更多信息，请参阅[Java中的JWT初学者指南](https://stormpath.com/blog/beginners-guide-jwts-in-java)。

令牌从授权服务器上的endpoint上取回。两个主要endpoint是授权endpoint和令牌endpoint。它们分开用于不同的用例。授权endpoint是您去哪里获得用户同意和授权的地方。这将返回一个授权，表示用户已同意。然后授权被传递给令牌端点。令牌端点处理授权并说“很好，这是您的刷新令牌和您的访问令牌”。

![授权服务器](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/oauth-server.png)

您可以使用访问令牌访问API。一旦到期，您将必须返回带有刷新令牌的令牌端点以获取新的访问令牌。

缺点是这导致了很多开发者的反对。开发人员对OAuth最大的难点之一就是你必须管理刷新令牌。你将状态管理推给每个客户端开发者。你获得了关键轮换的好处，但是你为开发者创造了很多痛苦。这就是开发人员喜欢API密钥的原因。他们可以复制/粘贴它们，将它们放在文本文件中，然后用它们完成。API密钥对开发人员非常方便，但对安全性非常不利。

这是一个代价问题。让开发人员执行OAuth流程可提高安全性，但存在更多的摩擦。工具包和平台有机会简化事情并帮助进行令牌管理。幸运的是，现在OAuth已经非常成熟了，您最喜欢的语言或框架都可能用用于简化OAuth的工具。

我们已经谈了一些关于客户端类型，令牌类型和授权服务器的endpoint以及我们如何将其传递给资源服务器的内容。我提到了两种不同的流程：获得授权和获取令牌。这些不必在同一个通道上发生。前向通道是浏览器的内容。浏览器将用户重定向到授权服务器，用户表示同意。这发生在用户的浏览器上。一旦用户获得授权许可并将其交给应用程序，客户端应用程序就不再需要使用浏览器来完成OAuth流程以获取令牌。

令牌旨在被客户端应用程序使用，以便它可以代表您访问资源。我们称之为后向通道。后向的通道是直接从客户端应用程序到资源服务器的HTTP调用，用于交换令牌的授权许可。这些通道用于不同的流程，具体取决于您拥有的设备功能。

![流动通道](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/oauth-channel.png)

例如，您通过用户代理进行授权的前向通道可能如下所示：

- 资源所有者启动流程来委派对受保护资源的访问
- 客户端通过浏览器重定向到授权服务器上的授权端点，向所需范围发送授权请求
- 授权服务器返回一个同意对话框，指出“您是否允许此应用程序访问这些范围？”当然，您需要向应用程序进行身份验证，所以如果您未通过资源服务器的身份验证，它会询问你登录。如果您已经有了一个缓存的会话cookie，您只会看到同意对话框。查看同意对话框，并同意。
- 授权许可通过浏览器重定向传递回应用程序。这一切都发生在前向通道。

![前向通道](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/front-channel.png)

这种流程中也存在一种称为隐式流程的变化。我们稍后再讨论。

这就是它在请求中的样子。

**Request**
> GET https://accounts.google.com/o/oauth2/auth?scope=gmail.insert gmail.send
&redirect_uri=https://app.example.com/oauth2/callback
&response_type=code&client_id=812741506391
&state=af0ifjsldkj

这是一个带有一堆查询参数的GET请求（不是出于示例目的的URL编码）。范围来自Gmail的API。redirect_uri是当授权完成时，所返回的客户端应用程序的URL。这应该与客户注册过程（在DMV处）的值相匹配。您不希望授权被退回到外部应用程序。响应类型会改变OAuth流向。客户端ID也来自注册过程。国家是一个安全标志，类似于XRSF。要了解有关XRSF的更多信息，请参阅[DZone的“跨站请求伪造解释”](https://dzone.com/articles/cross-site-request-forgery)。

**Response**
>HTTP/1.1 302 Found
Location: https://app.example.com/oauth2/callback?
code=MsCeLvIaQm6bTrgtp7&state=af0ifjsldkj

>*code*返回授权认证，*state*确保它不是伪造的，即它是由同一个请求发出。

前向通道完成后，会进行前向通道，并将授权码交换为访问令牌。

客户端应用程序使用受信任的客户端凭证和客户端ID访问授权服务器上的令牌endpoint，并发送访问令牌请求。该过程交换访问令牌和（可选）刷新令牌的授权代码授权。客户端使用访问令牌访问受保护的资源。

![后向通道流](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/back-channel.png)

下面是其在HTTP中的样子。

**Request**
> POST /oauth2/v3/token HTTP/1.1
Host: www.googleapis.com
Content-Type: application/x-www-form-urlencoded
> 
>code=MsCeLvIaQm6bTrgtp7&client_id=812741506391&client_secret={client_secret}&redirect_uri=https://app.example.com/oauth2/callback&grant_type=authorization_code

grant_type是OAuth的可扩展性部分。这是来自预先获得的授权代码。它开辟了用不同方式来描述这些授权的灵活性。这是最常见的OAuth流程类型。

**Response**
>{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA"
}

响应是JSON。您可以在使用令牌时具有反应性或主动性。主动性是在你的客户端有一个计时器。反应性是捕捉一个错误，然后尝试获取新的令牌。

一旦获得访问令牌，就可以在验证头中使用访问令牌（使用token_type前缀）来提出受保护的资源请求。

>curl -H "Authorization: Bearer 2YotnFZFEjr1zCsicMWpAA" \
  https://www.googleapis.com/gmail/v1/users/1444587525/messages
  
所以现在你有一个前向通道，一个后向通道，不同的终端和不同的客户端。你必须为不同的用例进行混合和匹配。这增加了OAuth的复杂性，并且可能会引起混淆。

### OAuth流程
第一个流程就是我们所说的**隐式流程（Implicit Flow）**。它被称为隐式流的原因是因为所有的通信都是通过浏览器进行的。没有后端服务器为访问令牌兑换授权许可。SPA是这个流程用例的一个很好的例子。该流程也称为2 Legged OAuth。

隐式流程针对仅限于浏览器的公共客户端进行了优化。访问令牌直接从授权请求中返回（仅限于前向通道）。它通常不支持刷新标记。它假定资源所有者和公共客户端位于同一设备上。由于所有事情都发生在浏览器上，它最容易受到安全威胁的影响。

黄金标准是**授权代码流程（Authorization Code Flow）**，又名3 Legged，它同时使用前向通道和后向通道。这就是我们在本文中讨论最多的内容。客户端应用程序使用前向通道流获取授权码授权。客户端应用程序使用后向通道交换访问令牌（以及可选的刷新令牌）的授权代码授权。它假定资源所有者和客户端应用程序位于不同的设备上。这是最安全的流程，因为您可以验证客户端以兑换授权授权，令牌永远不会通过用户代理。不仅有隐式流程和授权码流程，您还可以使用OAuth执行额外的流程。因此，OAuth更像是一个框架。

对于服务器到服务器方案，您可能需要使用**客户端凭证流程（Client Credential Flow）**。在这种情况下，客户端应用程序是一个受信任的客户端，它自己独立运行，而不是代表用户。它更像是一种服务帐户类型的场景。您所需要的只是客户的凭证来完成整个流程。这是一个仅使用客户凭证获取访问令牌的后向通道。它支持共享秘钥或断言，作为使用对称或非对称密钥签名的客户端凭证。

对称密钥算法是加密算法，只要您有密码，就可以解密任何内容。这通常在保护PDF或.zip文件时发现。

公钥加密或非对称加密是使用密钥对的任何加密系统：公钥和私钥。公钥可以被任何人读取，私钥对于所有者来说是神圣的。这使得数据安全无需分享密码。

还有一种传统模式称为**资源所有者密码流程（Resource Owner Password Flow）**。这与使用用户名和密码方案的直接身份验证非常相似，不推荐使用。它是原生用户名/密码应用程序（如桌面应用程序）的传统授权类型。在此流程中，您向客户端应用程序发送用户名和密码，并从授权服务器返回访问令牌。它通常不支持刷新令牌，并且假定资源所有者和公共客户端位于同一设备上。

对OAuth的最新补充是**断言流程（Assertion Flow）**，它与客户端证书流程类似。这是为了打开联合认证的想法。该流程允许授权服务器信任来自第三方的授权许可，如SAML IdP。授权服务器信任身份提供商。该断言用于从令牌endpoint获取访问令牌。这对于那些投资SAML或SAML相关技术并允许他们与OAuth集成的公司来说非常有用。由于SAML断言是短暂的，因此此流程中不存在刷新标记，并且每次断言到期时都必须继续检索访问标记。

不在OAuth规范中的，是**设备流程（Device Flow）**。没有网络浏览器，只有一个像电视一样的控制器。用户code是从授权请求返回的，必须通过浏览器上的URL访问授权请求才能进行授权。客户端应用程序使用后向通道流程来轮询访问令牌和可选刷新令牌的授权许可。这种方式也在CLI客户端中很受欢迎。

我们已经使用不同的角色和标记类型介绍了六种不同的流程。由于客户端的能力，我们需要获得客户端的同意书，以及了解谁正在征求同意，这些都是必要的，并且为OAuth增加了很多复杂性。

当人们问你是否支持OAuth时，你必须澄清他们要求的东西。他们问你是支持全部六个流程，还是只支持主流程？在所有不同的流程之间有很多可用的粒度。

### 安全和企业
OAuth有很大的覆盖面。有了隐式流程，就有很多重定向和很多错误空间。有很多人试图在应用程序之间利用OAuth，如果你不遵循推荐的网络安全101准则，就很容易做到。例如：

- 始终使用带state参数的CSRF令牌来确保流程完整性
- 始终将重定向URI列入白名单，以确保正确的URI验证
- 将同一客户端绑定到具有客户端ID的授权许可和令牌请求
- 对于受信任的客户，确保客户机密不泄露。不要把你的应用程序中的客户端秘密通过App Store分发！

关于OAuth的最大抱怨一般来自安全人员。这是关于Bearer tokens，并且他们可以像会话cookie一样传递。您可以将它传递出去，而且您可以很好地进行操作，而不是以加密方式绑定到用户。使用JWT有助于避免被篡改。但是，最终，JWT只是一串字符，因此它们可以轻松复制并用于Authorization标题中。

### 企业OAuth 2.0使用案例
OAuth将授权策略决策与身份验证分离开来。它可以正确地混合细粒度和粗粒度的授权。它可以取代传统的Web访问管理（WAM）策略。在构建可访问特定API的应用程序时，限制和撤销权限也很好。它确保只有托管或兼容的设备才能访问特定的API。它与身份取消配置工作流程深度集成，以撤消用户或设备的所有令牌。最后，它支持与身份提供者的联合。

### OAuth不是身份验证协议
总结一下OAuth 2.0的一些误解：它不与OAuth 1.0向后兼容。它用HTTPS替代所有通信的签名。今天人们谈论OAuth时，他们正在谈论OAuth 2.0。

由于OAuth是授权框架而不是协议，因此您可能会遇到互操作性问题。团队如何实施OAuth有很多差异，您可能需要自定义代码才能与供应商进行集成。

OAuth 2.0不是身份验证协议。它甚至在[文档](https://oauth.net/articles/authentication/)中都这么说。

![OAuth 2.0不是身份验证协议](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/oauth-not-authentication.png)

我们一直在谈论授权。这不是关于验证用户，这是关键。仅对于OAuth 2.0来说，用户是不存在的。您只需拥有一个令牌即可访问资源。

在过去的几年中，OAuth发生了大量的增加。这些增加了OAuth之上的复杂性，以完成各种企业方案。例如，JWT可以用作可签名和加密的互操作令牌。

### 使用OAuth 2.0进行伪身份验证
Facebook Connect和Twitter让着名的OAuth登录成为了热门话题。在此流程中，客户端使用/me的endpoint访问获取令牌。它所说的是，客户端可以使用令牌访问资源。人们发明了这个假端点，作为用访问令牌取回用户配置文件的一种方式。这是获取用户信息的非标准方式。标准中没有任何人说每个人都必须实现这个端点。访问令牌意味着不透明。它们是为了API而设计的，它们不是为了包含用户信息而设计的。

你真正尝试验证与回答的问题应该是：这是谁的用户，有没有对用户进行认证，以及如何做的用户进行身份验证。您通常可以通过SAML断言来回答这些问题，而不是使用访问令牌和授权许可。这就是我们称之为伪认证的原因。

### 输入OpenID Connect
为解决伪身份验证问题，将OAuth 2.0，Facebook Connect和SAML 2.0的最佳部分组合在一起，创建OpenID Connect。OpenID Connect（OIDC）扩展了OAuth 2.0，id_token为客户端和UserInfo endpoint提供了新的用户属性签名。与SAML不同，OIDC为身份提供了一套标准范围和声明。例子包括：profile，email，address，和phone。

OIDC的创建是为了使网络具有完全动态的可扩展性。不再需要像SAML那样下载元数据和联合认证。内置的动态联合认证可以注册，发现元数据。您可以输入您的电子邮件地址，然后它动态地发现您的OIDC提供商，动态下载元数据，动态地知道它将使用的证书，并允许BYOI(Bring Your Own Identity)。它支持企业的高保证级别和关键SAML使用案例。

![OpenID连接协议套件](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/openid-connect-protocol.png)

OIDC因谷歌和微软两大早期使用者而闻名。Okta也在OIDC上投入了大量资金。

初始请求中的变化就是它包含标准作用域（如openid和email）：

**Request**
> GET https://accounts.google.com/o/oauth2/auth?
scope=openid email&
redirect_uri=https://app.example.com/oauth2/callback&
response_type=code&
client_id=812741506391&
state=af0ifjsldkj

**Response**
>HTTP/1.1 302 Found
Location: https://app.example.com/oauth2/callback?
code=MsCeLvIaQm6bTrgtp7&state=af0ifjsldkj

>*code*返回授权认证，*state*确保它不是伪造的，即它是由同一个请求发出。

而授权获取令牌的响应包含一个ID令牌。

**Request**	
>POST /oauth2/v3/token HTTP/1.1
Host: www.googleapis.com
Content-Type: application/x-www-form-urlencoded
>
>code=MsCeLvIaQm6bTrgtp7&client_id=812741506391&
  client_secret={client_secret}&
  redirect_uri=https://app.example.com/oauth2/callback&
  grant_type=authorization_code

**Response**
>{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjFlOWdkazcifQ..."
}

您可以看到，在OAuth之上很好地分层，将ID令牌作为结构化令牌返回。一个ID令牌是一个JSON Web令牌（JWT）。JWT（aka“jot”）比基于XML的巨大SAML断言小得多，可以在不同设备之间高效传递。JWT有三部分：标题(header)，正文(body)和签名(signature)。标题说明使用什么算法对它进行签名，声明(claims)在正文中，并且在签名中签名。

Open ID Connect流程涉及以下步骤：

- 发现OIDC元数据
- 执行OAuth流程以获取id令牌和访问令牌
- 获取JWT签名密钥并可选择动态注册客户端应用程序
- 根据内置日期和签名在本地验证JWT ID令牌
- 根据需要使用访问令牌获取其他用户属性

![OIDC流程](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/what-is-oauth/oidc-flow.png)


### 总结
OAuth 2.0是委托访问API的授权框架。它涉及请求资源所有者授权/同意的范围的客户端。授权授予交换访问令牌和刷新令牌（取决于流程）。有多种流程来解决不同的客户端和授权方案。JWT可用于授权服务器和资源服务器之间的结构化令牌。

OAuth具有非常大的安全覆盖面。确保使用安全工具包并验证所有输入！

OAuth不是身份验证协议。OpenID Connect针对身份验证方案扩展了OAuth 2.0，通常称为“带花括号的SAML”。如果您希望进一步深入了解OAuth 2.0，我建议您查看[OAuth.com](https://www.oauth.com/)，使用Okta的Auth SDK进行测试，然后尝试自己的OAuth流程。