---
layout: post
title: OAuth教程--用户登录
date: 2018-07-10 19:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/authorization/requiring-user-login/)）**

用户在单击应用程序的“登录”或“连接”按钮后将看到的第一件事是您的授权服务器UI。由授权服务器决定是否在每次访问授权屏幕时要求用户登录，或者让用户登录一段时间。如果授权服务器要在请求之间记住用户，那么它将需要请求用户在将来访问时授权该应用程序。

通常像Twitter或Facebook这样的网站希望他们的用户在大多数时间都是签名的，因此他们为他们的授权屏幕提供了一种方式，通过不要求他们每次登录来为用户提供简化的体验。但是，根据服务以及第三方应用程序的安全要求，可能需要或允许开发人员选择要求用户在每次访问授权屏幕时登录。

在Google的API中，应用程序可以添加prompt=login到授权请求，这会导致授权服务器在显示授权提示之前强制用户再次登录。

在任何情况下，如果用户已退出，或者在您的服务上还没有帐户，则需要为他们提供在此屏幕上登录或创建帐户的方法。

![Twitter的授权屏幕的退出视图](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/twitter_logged_out_auth_screen.png)

因为OAuth 2.0规范中未指定，您可以按照您希望的方式对用户进行身份验证。大多数服务使用传统的用户名/密码登录来验证其用户，但这绝不是解决问题的唯一方法。在企业环境中，常见的技术是使用SAML（一种基于XML的身份验证标准）来利用组织中的现有身份验证机制，同时避免创建另一个用户名/密码数据库。

一旦用户使用授权服务器进行身份验证，它就可以继续处理授权请求并将用户重定向回应用程序。通常，服务器将认为成功登录也意味着用户授权该应用程序。在这种情况下，具有登录提示的授权页面将需要包括描述以下事实的文本：通过登录，用户正在批准该授权请求。这将导致以下用户流程。

![登录和未登录的用户流程](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_create_new_application_1.png)

如果授权服务器需要通过SAML或其他内部系统对用户进行身份验证，则用户流程将如下所示

![用于单独验证服务器的用户流](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/okta_oauth-diagrams_2.png)

在此流程中，用户在登录后被定向回授权服务器，在那里他们看到授权请求，就像他们已经登录一样。