---
layout: post
title: OAuth教程--客户端ID和密钥
date: 2018-07-08 17:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/client-registration/client-id-secret/)）**

此时，您已构建了应用程序注册页面，您已准备好让开发人员注册该应用程序。当开发人员注册应用程序时，您需要生成客户端ID和可选的密钥。在生成这些字符串时，在安全性和美学方面需要考虑一些重要的事项。

### 客户ID

client_id是应用程序的公共标识符。即使它是公开的，最好是第三方无法猜测，因此许多实现使用类似32字符的十六进制字符串。它在授权服务器处理的所有客户端中也必须是唯一的。如果客户端ID是可猜测的，则可以更轻松地针对任意应用程序进行网络钓鱼攻击。

以下是来自支持OAuth 2.0的服务的客户端ID的一些示例：

- Foursquare： ZYDPLLBWSK3MVQJSIYHB1OR2JXCY0X2C5UJ2QAR2MAAIT5Q
- Github： 6779ef20e75817b79602
- Google： 292085223830.apps.googleusercontent.com
- Instagram： f2a1ed52710d4533bde25be6da03b6e3
- SoundCloud： 269d98e4922fb3895e9ae2108cbb5064
- Windows Live： 00000000400ECB04

如果开发人员正在创建“公共”应用程序（移动或单页应用程序），那么您根本不应该向应用程序发出client_secret。这是确保开发人员不会意外地将其包含在应用程序中的唯一方法。如果它不存在，它就不会泄露！

因此，您应该询问开发人员在启动时创建的应用程序类型。您可以向他们提供以下选项，并仅为“Web服务器”应用程序发出密钥。

- Web服务器应用程序
- 基于浏览器的应用
- 原生应用
- 移动应用

当然，没有什么可以阻止开发人员选择错误的选项，但是通过主动询问开发人员将使用哪种类型的应用程序，您可以帮助减少泄露秘密的可能性。了解正在创建哪种类型的应用程序的另一个原因是您需要注册公共客户端的重定向URL，但是对于私有客户端，重定向URL的注册在技术上是可选的。有关详细信息，请参阅[重定向URI注册](https://www.oauth.com/oauth2-servers/redirect-uris/redirect-uri-registration/)。

### 客户密钥

client_secret是只有应用程序和授权服务器知道的密钥。它必须足够随机以至于无法猜测，这意味着您应该避免使用常见的UUID库，这些库通常会考虑生成它的服务器的时间戳或MAC地址。生成安全密钥的一个好方法是使用加密安全库生成256位值并将其转换为十六进制表示。

在PHP中，您可以使用OpenSSL函数生成随机字节并转换为十六进制字符串：

```php
bin2hex(openssl_random_pseudo_bytes(32));
```

或者在PHP 7及更高版本中，random_bytes可以使用内置函数。

在Ruby中，您可以使用SecureRandom库生成十六进制字符串：

```ruby
require 'securerandom'
SecureRandom.hex(32)
```

开发人员永远不要将其client_secret公开（基于移动或基于浏览器的）应用程序包含在内是至关重 为了帮助开发人员避免意外地执行此操作，最好使客户端密钥在视觉上与ID不同。这种方式当开发人员复制并粘贴ID和密钥时，很容易识别哪个是哪个。通常使用较长的字符串来表示密钥是一种很好的方式来表明这一点，或者在密钥前加上“密钥”或“私密”。

### 存储和显示客户端ID和密钥

对于每个注册的应用程序，您需要存储公共client_id和私有client_secret。因为这些本质上等同于用户名和密码，所以不应以纯文本格式存储密钥，而应仅存储加密或散列版本，以帮助降低秘密泄露的可能性。

当您发出客户端ID和密钥时，您需要将它们显示给开发人员。大多数服务为开发人员提供了一种检索现有应用程序密钥的方法，尽管有些服务只显示一次密钥并要求开发人员立即自行存储。如果您只显示一次密钥，则可以存储它的散列版本以避免存储明文密码。

如果您选择稍后可以向开发人员显示的方式存储密钥，则在披露密钥时应采取额外的预防措施。保护密钥的常用方法是在开发人员尝试检索密钥时插入“重新授权”提示。

![GitHub重新授权提示](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_confirm_password.png)
GitHub在进行敏感更改时要求确认您的密码

该服务要求开发人员在泄密之前确认其密码。当您尝试查看或更新敏感信息时，这在Amazon或GitHub的网站中很常见。

![Dropbox'显示秘密'确认](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/dropbox_show_secret.png)
Dropbox会隐藏秘密，直到被点击为止

此外，在开发人员点击“显示”之前模糊应用程序详细信息页面上的密钥是防止意外泄露秘密的好方法。