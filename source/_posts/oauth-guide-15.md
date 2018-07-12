---
layout: post
title: OAuth教程--安全考虑因素
date: 2018-07-10 22:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/authorization/security-considerations/)）**

以下是构建授权服务器时应考虑的一些已知问题。

除了此处列出的注意事项外，[OAuth 2.0线程模型和安全注意事项草案](https://tools.ietf.org/html/rfc6819)中还提供了更多信息。

### 网络钓鱼攻击

针对OAuth服务器的一种潜在攻击是网络钓鱼攻击。这是攻击者创建一个看起来与服务授权页面相同的网页的地方，该页面通常包含用户名和密码字段。然后，通过各种手段，攻击者可以诱骗用户访问该页面。除非用户可以检查浏览器的地址栏，否则该页面可能看起来与真正的授权页面相同，并且用户可以输入他们的用户名和密码。

攻击者可以尝试诱骗用户访问伪造服务器的一种方法是将此网络钓鱼页面嵌入到本机应用程序中的嵌入式Web视图中。由于嵌入式Web视图不显示地址栏，因此用户无法直观地确认它们位于合法站点上。遗憾的是，这在移动应用程序中很常见，并且开发人员通常希望通过整个登录过程将用户保留在应用程序中来提供更好的用户体验。某些OAuth提供商鼓励第三方应用程序打开Web浏览器或启动提供程序的本机应用程序，而不是允许它们在Web视图中嵌入授权页面。

#### 对策

确保通过https提供授权服务器以避免DNS欺骗。

授权服务器应该向开发人员介绍网络钓鱼攻击的风险，并且可以采取措施防止该页面嵌入到本机应用程序或iframe中。

应该让用户了解网络钓鱼攻击的危险，并且应该教导用户最佳实践，例如只访问他们信任的应用程序，并定期查看应用程序列表，对不再使用的应用程序授权撤销的访问权限。

该服务可能希望在允许其他用户使用该应用程序之前验证第三方应用程序。Instagram和Dropbox等服务目前都是这样做的，在初始创建应用程序时，该应用程序只能由开发人员或其他白名单用户帐户使用。应用程序提交审批并进行审核后，可以由该服务的整个用户群使用。这使服务有机会检查应用程序如何与服务交互。

### 点击劫持

在点击劫持攻击中，攻击者创建了一个恶意网站，在该网站中，攻击者网页上方的透明iframe中加载了授权服务器URL。攻击者的网页堆放在iframe下方，并且有一些看似无害的按钮或链接，非常小心地放在授权服务器的确认按钮下面。当用户单击误导性可见按钮时，他们实际上是单击授权页面上的隐藏按钮，从而授予对攻击者应用程序的访问权限。这允许攻击者欺骗用户在他们不知情的情况下授予访问权限。

#### 对策

通过确保授权URL始终直接在本机浏览器中加载，而不是嵌入在iframe中，可以防止这种攻击。较新的浏览器可以让授权服务器设置HTTP标头，X-Frame-Options旧版浏览器可以使用常见的Javascript“框架破坏”技术。

### 重定向URL操作

攻击者可以使用属于已知正常应用程序的客户端ID构建授权URL，将重定向URL设置为攻击者控制下的URL。如果授权服务器未验证重定向URL，并且攻击者使用“令牌”响应类型，则用户将使用URL中的访问令牌返回到攻击者的应用程序。如果客户端是公共客户端，并且攻击者拦截授权码，则攻击者还可以通过授权码交换访问令牌。

另一个类似的攻击是攻击者可以欺骗用户的DNS，并且注册的重定向不是https URL。这将允许攻击者伪装成有效的重定向URL，并以这种方式窃取访问令牌。

“打开重定向”攻击是指授权服务器不需要重定向URL的完全匹配，而是允许攻击者构建将重定向到攻击者网站的URL。无论这是否最终被用于窃取授权码或访问令牌，这也是一个危险，因为它可以用于发起其他无关的攻击。有关Open Redirect攻击的更多详细信息，请访问`https://oauth.net/advisories/2014-1-covert-redirect/`。

#### 对策

授权服务器必须要求应用程序注册一个或多个重定向URL，并且重定向到先前注册的URL必须完全匹配。

授权服务器还应要求所有重定向URL均为https。由于这有时会给开发人员带来负担，特别是在应用程序运行之前，在应用程序处于“开发阶段”时允许非https重定向URL并且只能由开发人员访问，然后要求在发布应用程序之前，重定向URL应更改为https URL。