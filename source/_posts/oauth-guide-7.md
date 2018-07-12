---
layout: post
title: OAuth教程--注册新的应用程序
date: 2018-07-08 16:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/client-registration/registering-new-application/)）**

当开发人员访问您的网站时，他们需要一种方法来创建新的应用程序并获取凭据。通常，您可以让他们在创建应用程序之前创建开发人员帐户，或代表其组织创建帐户。

虽然OAuth 2.0规范不要求您在授予凭据之前特别收集任何应用程序信息，但大多数服务会在发出client_id和client_secret之前收集有关应用程序的基本信息，例如应用程序名称和图标。但是，为了安全起见，您需要开发人员为应用程序注册一个或多个重定向URL，这一点非常重要。重定向URL中对此进行了更详细的说明。

通常，服务收集有关应用程序的信息，例如：

- 应用名称
- 应用程序的图标
- 应用程序主页的URL
- 应用程序的简短描述
- 应用程序隐私策略的链接
- 重定向网址列表

下面是GitHub用于注册应用程序的界面。在其中，它们收集应用程序名称，主页URL，回调URL和可选描述。

![在Github上创建一个新的应用程序](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_create_new_application_1.png)
在GitHub上创建一个新的应用程序

最好向开发人员展示您从中收集的信息是显示给最终用户，还是仅供内部使用。

Foursquare的应用程序注册页面要求提供类似的信息，但他们还要求提供简短的标语和隐私政策URL。这些在授权提示中显示给用户。

![在Foursquare上创建一个新的应用程序](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/foursquare_create_new_application.png)
在Foursquare上创建一个新的应用程序

由于使用隐式授权类型的安全性考虑因素，某些服务（例如Instagram）默认情况下会禁用新应用程序的此授权类型，并要求开发人员在应用程序的设置中明确启用它，如下所示。或者，该服务可以使开发人员选择他们正在创建的应用程序类型（公共或私有），并仅向私有应用程序发出密钥。

![在Instagram上创建一个新的应用程序](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/instagram_create_new_application.png)
在Instagram上创建一个新的应用程序

Instagram还提供了一个说明，指示开发人员不要用可能使应用程序看起来来自Instagram的单词命名他们的应用程序。这也是包含API使用条款链接的优势。