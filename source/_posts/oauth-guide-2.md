---
layout: post
title: OAuth教程--OAuth 2.0客户端
date: 2018-06-29 15:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/oauth2-clients/)）**

本章简要介绍如何使用典型的OAuth 2.0 API。虽然OAuth 2.0的每个实现可能略有不同，但实际上，它们中的大多数共享很多共同点。本章介绍了OAuth 2.0对于新手的介绍，并举例说明了三种主要类型的应用程序如何与OAuth 2.0 API进行交互。

### 创建应用程序

在与OAuth 2.0 API进行交互之前，必须先向服务注册应用程序。注册过程通常涉及对服务的网站创建一个帐户，然后输入有关应用程序的基本信息，如名称，网站，标识等注册申请后，你会得到一个client_id和client_secret，将在授权过程中使用。

创建应用程序时最重要的事情之一，是注册应用程序将使用的一个或多个重定向URL。重定向网址是OAuth 2.0服务在授权应用程序后将用户返回的位置。注册这些是至关重要的，否则很容易创建可以窃取用户数据的恶意应用程序。本章稍后将对此进行更详细的介绍。

每个服务都以稍微不同的方式实现注册过程，因此我们将介绍在GitHub上创建应用程序的示例。

### GitHub

在您的“Account Settings”页面中，点击边栏中的“Applications”链接。您将前往[https://github.com/settings/applications](https://github.com/settings/applications)。从那里，单击“注册新应用程序”，您将看到一个简短的表单，如下所示。

![在GitHub上注册一个新的应用程序](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_create_new_application.png)

填写所需信息，包括回调URL。如果您在本地开发应用程序，则必须使用本地地址作为回调URL。由于GitHub每个应用程序只允许一个注册的回调URL，因此创建两个应用程序非常有用，一个用于开发，另一个用于生产。

完成此表单后，您将进入一个页面，您可以在其中看到发给您应用程序的客户ID和密钥，如下所示。

![GitHub应用程序已创建](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_application_created.png)

客户端ID被视为公共信息，用于构建登录URL，或者可以包含在网页的Javascript源代码中。客户端密钥必须保密。如果部署的应用程序无法保密，例如Javascript或本机应用程序，则在授权期间不会使用该密钥。

### 重定向网址和state

OAuth 2.0 API仅将用户重定向到已注册的URL，以防止攻击者可以获取授权代码或访问令牌的重定向攻击。某些服务可能允许您注册多个重定向URL，这可以在为Web应用程序和移动应用程序使用相同的客户端ID时，或者在开发和生产服务中使用相同的客户端ID时提供帮助。

为了安全起见，重定向URL必须是https端点，以防止在授权过程中拦截令牌。如果您的重定向URL不是https，则攻击者可能能够拦截授权代码并使用它来劫持会话。如果服务允许使用非https重定向，则必须采取额外的预防措施以确保无法进行此类攻击。

大多数服务将重定向URL验证视为完全匹配。这意味着重定向网址 https://example.com/auth 不匹配 https://example.com/auth?destination=account 。最佳做法是避免在重定向URL中使用查询字符串参数，并使其仅包含路径。

某些应用程序可能有多个他们想要启动OAuth进程的位置，例如主页上的登录链接以及查看某些公共项目时的登录链接。对于这些应用程序，尝试注册多个重定向URL可能很诱人，或者您可能认为需要能够根据请求更改重定向URL。相反，OAuth 2.0为此提供了一种机制，即“state”参数。

“state”参数可以用于任何你想要的地方，它是一个对OAuth 2.0服务不透明的字符串。您在初始授权请求期间的任何state值都会在用户授权应用程序后返回。其中一个常见的应用是包括一个随机字符串来防止CSRF攻击。您还可以用JWT之类的方法中对重定向URL进行编码，并在用户重定向回应用程序后对其进行解析，以便您在登录后将用户带回适当的位置。