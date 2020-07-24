---
layout: post
title: OAuth教程--移动和原生应用
date: 2018-07-13 6:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

与单页应用程序一样，移动应用程序也无法保持客户机密的机密性。因此，移动应用还必须使用不需要客户端密钥的OAuth流程。当前的最佳做法是使用授权流程以及启动外部浏览器，以确保本机应用程序无法修改浏览器窗口或检查内容。

许多网站都提供移动SDK，为您处理授权过程。对于这些网站，你最好直接使用他们的SDK，因为他们可能用非标准的方式添加增加了他们的API。Google提供了一个名为AppAuth的开源库，它可以处理下面描述的流程的实现细节。它旨在能够与任何实现规范的OAuth 2.0服务器一起使用。如果服务不提供自己的抽象，并且您必须直接使用其OAuth 2.0的endpoint，你就可以参照本节介绍来了解如何使用授权与API进行交互。

### 授权

创建一个“登录”按钮，该按钮将打开SFSafariViewController或启动本机浏览器。您将使用与服务器端应用程序中所述相同的授权请求参数。

对于本机应用程序的重定向URL，在iOS上，应用程序可以注册像`org.example.app://`这样的自定义URL scheme，只要访问具有该scheme的URL，就会启动应用程序。在Android上，应用程序可以注册URL匹配模式，如果访问了与模式匹配的URL，则会启动本机应用程序。

### 举个栗子🌰

在这个例子中，我们将创建一个简单的iPhone应用程序，获取访问虚构API的授权。

#### 应用程序启动授权请求

要开始授权过程，应用程序应该有一个“登录”按钮。该链接应构建为服务授权endpoint的完整URL。

授权URL通常采用以下格式：

```http
https://authorization-server.com/authorize
?client_id=eKNjzFFjH9A1ysYd
&response_type=code
&redirect_uri=exampleapp://auth
&state=1234zyx
```

在这种情况下请注意重定向URL的自定义方案。iOS提供了应用程序注册自定义URL方案的功能。在Android上，应用可以改为匹配特定的网址模式，以便应用在访问特定网址时在应用列表中显示并进行处理。在iOS上，您应该在应用程序的.plist文件中注册您将使用的自定义方案。这将导致设备在访问以您的自定义方案开头的URL时启动您的应用，包括移动版Safari或其他iOS应用。

当用户点击“登录”按钮时，应用程序应用SFSafariViewController打开共享系统cookie的嵌入式浏览器来打开登录URL。WebView在应用程序中使用嵌入式窗口被认为是非常危险的，因为这使用户无法保证他们正在查看服务自己的网站，并且是网络钓鱼攻击的简单来源。通过使用SFSafariViewController共享Safari cookie的API，您可以知道用户是否已经登录该服务。

#### 用户批准该请求

在被定向到auth服务器时，用户看到如下所示的授权请求。

![内置浏览器](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/sfsafariviewcontroller-example.png)

嵌入式SFSafariViewController。右上角的“完成”按钮折叠视图并将用户返回到应用程序。

#### 该服务将用户重定向回应用程序

当用户完成登录后，该服务将重定向回您的应用程序的重定向URL，在这种情况下，该URL具有一个自定义方案，该方案将触发您的应用程序委托的application:openURL:options:方法。Location重定向的标题将类似于以下内容，它将作为url参数传递给您的方法。

```html
org.example.app://auth?state=1234zyx
&code=lS0KgilpRsT07qT_iMOg9bBSaWqODC1g061nSLsa8gV2GYtyynB6A
```

然后，您的应用应该从URL解析授权码，交换代码以获取访问令牌，并关闭SFSafariViewController。除了不使用客户端密钥之外，交换访问令牌的代码与授权代码流中的代码相同。

### 安全考虑因素

#### 始终打开本机浏览器或使用 SFSafariViewController

您永远不应该使用OAuth提示打开嵌入式Web视图，因为它无法让用户验证他们正在查看的网页的来源。攻击者会创建一个看起来就像授权网页并将其嵌入到自己的恶意应用程序中的网页，让他们能够窃取用户名和密码。

#### PKCE

如果您使用的服务支持PKCE扩展（[RFC 7636](https://tools.ietf.org/html/rfc7636)），那么您应该利用它提供的额外安全性。通常，例如在使用Google OAuth API的情况下，服务提供的本机SDK将透明地处理此问题，因此您无需担心详细信息，并且无需任何额外工作即可从额外的安全性中受益。
