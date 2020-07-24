---
layout: post
title: OAuth教程--无浏览器和输入约束设备的OAuth
date: 2018-07-13 16:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

OAuth 2.0“设备流”扩展程序可在具有Internet连接但没有浏览器或简单方法输入文本的设备上启用OAuth。如果您曾在Apple TV等设备上登录过自己的YouTube帐户，那么您已经遇到过此工作流程。谷歌参与了这个扩展的开发，并且也是它在生产中的早期实现者。

此流程也可在智能电视，媒体控制台，相框，打印机或硬件视频编码器等设备上看到。在该流程中，设备指示用户在诸如智能电话或计算机的辅助设备上打开URL以完成授权。用户的两个设备之间不需要通信通道。

### 用户流程

当您开始登录设备（例如此硬件视频编码器）时，设备会与Google通话以获取设备代码，如下所示。

![设备发出API请求以获取设备代码](https://ws2.sinaimg.cn/large/006tKfTcly1fto9emscc3j30sg07nwf7.jpg)

接下来，我们看到设备会显示代码以及URL。

![设备显示设备代码和URL](https://ws2.sinaimg.cn/large/006tKfTcly1fto9f6gvy7j30sg07nt9h.jpg)

登录Google帐户后访问该网址会显示一个界面，提示您输入设备上显示的代码。

![Google会提示用户输入代码](https://ws3.sinaimg.cn/large/006tKfTcly1fto9frx7oxj30me0jdgm9.jpg)

输入代码并单击“下一步”后，您将看到标准OAuth授权提示，该提示描述了应用程序请求的范围，如下所示。

![Google会显示应用程序请求的范围](https://ws1.sinaimg.cn/large/006tKfTcly1fto9g4x5oxj30ok0dhaay.jpg)

一旦您允许该请求，Google就会显示一条消息，说明要返回您的设备，如下所示。

![Google会指示用户返回设备](https://ws2.sinaimg.cn/large/006tKfTcly1fto9gir6bij30lp0izmxt.jpg)

几秒钟后，设备完成并且您已登录。

总的来说，这是一次非常轻松的体验。由于您可以使用任何想要打开URL的设备，因此可以使用您可能已登录的授权服务器的主计算机或电话。这也使您在设备上无需输入数据！

让我们逐步了解设备所需的功能。

### 授权请求

首先，客户端向授权服务器发出请求以请求设备代码的请求。

```http
POST /token HTTP/1.1
Host: authorization-server.com
Content-type: application/x-www-form-urlencoded
 
client_id=a17c21ed
```

请注意，某些授权服务器将允许设备在此请求中指定范围，稍后将在授权界面上向用户显示该范围。

授权服务器使用JSON进行响应，该json包含设备代码，用户将输入的代码，用户应访问的URL以及轮询间隔。

```json
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
{
    "device_code": "NGU5OWFiNjQ5YmQwNGY3YTdmZTEyNzQ3YzQ1YSA",
    "user_code": "BDWP-HQPK",
    "verification_uri": "https://authorization-server.com/device",
    "interval": 5,
    "expires_in": 1800
}
```

设备在其显示屏上向用户显示`verification_uri`和`user_code`，指示用户输入该URL处的代码。

### 令牌请求

当设备等待用户在他们自己的计算机或电话上完成授权流程时，设备同时开始轮询令牌端点以请求访问令牌。

设备以`device_code`指定的速率发出POST请求轮询。设备应继续请求访问令牌，直到`authorization_pending`返回，显示用户授予或拒绝请求或设备代码到期。

```http
POST /token HTTP/1.1
Host: authorization-server.com
Content-type: application/x-www-form-urlencoded
 
grant_type=urn:ietf:params:oauth:grant-type:device_code&amp;
client_id=a17c21ed&amp;
device_code=NGU5OWFiNjQ5YmQwNGY3YTdmZTEyNzQ3YzQ1YSA
```

授权服务器将回复错误或访问令牌。设备流程规范定义了两个额外的错误代码，这超越了OAuth 2.0用户核心定义，即`authorization_pending`和`slow_down`。

如果设备过于频繁地轮询，授权服务器将返回`slow_down`错误。

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
 
{
  "error": "slow_down"
}
```

如果用户尚未允许或拒绝该请求，则授权服务器将返回`authorization_pending`错误。

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
 
{
  "error": "authorization_pending"
}
```

如果用户拒绝请求，授权服务器将返回`access_denied`错误。

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
 
{
  "error": "access_denied"
}
```

如果设备代码已过期，授权服务器将返回`expired_token`错误。设备可以立即请求新的设备代码。

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
Cache-Control: no-store
 
{
  "error": "expired_token"
}
```

最后，如果用户允许请求，则授权服务器像正常一样发出访问令牌并返回标准访问令牌响应。

```json
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
 
{
  "access_token": "AYjcyMzY3ZDhiNmJkNTY",
  "refresh_token": "RjY2NjM5NzA2OWJjuE7c",
  "token_type": "bearer",
  "expires": 3600
}
```

### 授权服务器要求

支持设备流并不是授权服务器的大量额外工作。在向现有授权服务器添加对设备流的支持时，请记住以下几点。

#### 设备代码请求

设备将向授权服务器发出请求以获取该流程所需的验证码集合。以下参数是请求的一部分。

> **client_id** - 必需，客户端注册中描述的客户端标识符。
> **scope** - 可选，范围中描述的请求范围。

验证客户端ID和范围后，授权服务器返回带有验证码的URL，设备代码和用户代码的响应。除了上面给出的示例之外，授权服务器还可以返回一些可选参数。

> **device_code** - 必需，授权服务器生成的验证码。
> **user_code** - 必需的，用户将在设备屏幕上输入的代码应该相对较短。通常使用6-8个数字和字母。
> **verification_uri** - 必需，用户应访问的授权服务器上的URL以开始授权。用户需要在他们的计算机或移动电话上手动输入此URL。
> **expires_in** - 可选，设备代码和用户代码的生命周期（以秒为单位）。
> **interval** - 可选，客户端轮询令牌端点的请求之间应等待的最短时间（以秒为单位）。

#### 用户代码（user code）

在许多情况下，用户的设备将是他们的移动电话。通常，这些界面比完整的计算机键盘更受限制，例如iPhone需要额外的选项来切换到数字输入。为了帮助减少数据输入错误并加快代码输入，用户代码的字符集应考虑到这些限制，例如仅使用大写字母。

用于设备代码的合理字符集是不区分大小写的AZ字符的，而且没有元音，以避免意外拼写单词。最好使用base-20字符集BCDFGHJKLMNPQRSTVWXZ。比较输入的代码时，最好忽略字符集中没有的任何字符，如标点符号。例如用户码为BDWP-HQPK，授权服务器应该不区分大小写忽略标点符号来比较输入的字符串，所以应该允许以下匹配：bdwphqpk。

#### 验证网址

设备将显示的验证URL应尽可能短，并且易于记忆。它将显示在可能非常小的屏幕上，用户必须在他们的计算机或手机上手动输入。

请注意，服务器应返回包含URL方案的完整URL，但某些设备可能会在显示URL时选择对其进行修剪。因此，服务器应配置为将http重定向到https，并在普通域和www前缀上提供服务，以防用户输入错误或设备省略该部分URL。

Google的授权服务器是一个很容易输入的短网址的很好的例子。代码请求的响应是`https://www.google.com/device`, 但所有显示`google.com/device`的设备，Google都会相应地重定向。

#### 非文本接口的优化

没有显示或具有非文本显示的客户端显然无法向用户显示URL。因此，可以使用一些其他方法将验证URL和用户代码传达给用户。

该设备可能能够通过NFC或蓝牙或甚至通过显示QR码来广播验证URL。在这些情况下，设备可以使用参数将用户代码包括为验证URL的一部分。例如：
`https://authorization-server.com/device?user_code=BDWP-HQPK`
这样，当用户启动URL时，可以在验证界面中预填充用户代码。建议授权服务器仍然要求用户确认代码而不是自动进行。

如果设备能够显示代码，即使它无法显示URL，也可以通过提示用户确认验证界面上的代码与其设备上显示的代码相匹配来获得额外的安全性。如果这不是一个必需项，那么授权服务器至少可以要求用户确认他们刚刚请求授权的设备。

### 安全考虑因素

#### 用户代码暴力攻击

由于用户有可能手动将用户代码输入到尚未了解的等待授权设备的界面中，因此应采取预防措施以避免对用户代码进行暴力攻击的可能性。

通常使用具有比授权码所使用的熵少得多的短代码，以便容易地手动输入。因此，建议授权服务器对用于验证用户代码的端点进行速率限制。

速率限制应基于用户代码的熵，以使暴力攻击变得不可行。例如，在上述20个字符集中有8个字符，它提供大约34位的熵。在选择可接受的速率限制时，您可以使用此公式计算熵的位数。log2(208) = 34.57

#### 远程网络钓鱼

设备流程可能在攻击者拥有的设备上启动，以欺骗用户授权攻击者的设备。例如，攻击者可能发送短信指示用户访问URL并输入用户代码。

为了降低此风险，除了用户界面中描述的授权界面中包含的标准信息之外，建议授权界面要让用户非常清楚他们正在授权物理设备访问其帐户。