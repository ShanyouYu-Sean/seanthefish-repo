---
layout: post
title: OAuth教程--使用Google登录
date: 2018-07-13 3:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

尽管OAuth是**授权协议**而不是**身份验证协议**，但它通常用作身份验证工作流程的基础。许多常见OAuth API的典型用途是在登录第三方应用程序时识别当前用户。

身份验证和授权经常相互混淆，但如果从应用程序的角度考虑它们，则可以更容易理解。正在验证用户的应用只是验证用户是谁。授权用户的应用程序正在尝试访问或修改属于该用户的内容。

OAuth被设计为授权协议，因此每个OAuth流程的最终结果是应用程序获取访问令牌，以便能够访问或修改有关用户帐户的内容。访问令牌本身并未说明用户是谁。

有几种方法可以让不同的服务为应用程序提供一种查找用户身份的方法。一种简单的方法是API提供“用户信息”的endpoint，当使用访问令牌访问API时，该endpoint将返回经过身份验证的用户的名称和其他配置文件信息。虽然这不是OAuth标准的一部分，但它是许多服务采用的常见方法。更高级和标准化的方法是使用OAID 2.0扩展的OpenID Connect。OpenID Connect在后面有更详细的介绍。

本章将使用简化的OpenID Connect工作流程和Google API来识别用户并登录到您的应用程序。

### 创建一个应用程序

在我们开始之前，我们需要在Google API控制台中创建一个应用程序，以获取客户端ID和客户端秘钥，并注册重定向URL。

访问`https://console.developers.google.com/`并创建一个新项目。您还需要为项目创建OAuth 2.0凭据，因为Google不会自动执行此操作。在侧栏中，单击“Credentials”选项卡，然后单击“Create credentials”并从下拉列表中选择“OAuth client ID”。

![创建一个应用程序](https://ws3.sinaimg.cn/large/006tKfTcly1ftdy5lw1czj30sg0jwn09.jpg)

Google控制台会提示您提供有关您的应用程序的一些信息，例如产品名称，主页和logo。在下一页上，选择"Web application"类型，然后输入重定向URL。这样，您就会收到客户端ID和秘钥。

### 建立环境

本章节示例代码是用PHP编写的，不需要外部包，也不需要框架，这可以很容易地翻译成其他语言。要模仿此示例代码，您应该将它全部放在一个PHP文件中。

创建一个新文件夹并在该文件夹中创建一个空文件`index.php`。从该文件夹内部运行的命令行输入`php -S localhost:8000`，您将能够在浏览器中访问`http://localhost:8000`来运行您的代码。以下示例中的所有代码都应添加到此index.php文件中。

让我们为OAuth流程设置一些变量，添加我们在创建应用程序时从Google获得的客户端ID和秘钥。

```php
// Fill these out with the values you got from Google
$googleClientID = '';
$googleClientSecret = '';
 
// This is the URL we'll send the user to first
// to get their authorization
$authorizeURL = 'https://accounts.google.com/o/oauth2/v2/auth';
 
// This is Google's OpenID Connect token endpoint
$tokenURL = 'https://www.googleapis.com/oauth2/v4/token';
 
// The URL for this script, used as the redirect URL
$baseURL = 'https://' . $_SERVER['SERVER_NAME']
    . $_SERVER['PHP_SELF'];
 
// Start a session so we have a place
// to store things between redirects
session_start();
```

定义了这些变量，并开始会话，让我们设置登录和注销的页面。我们将显示一个超级简单的页面，它只是指示用户是否已登录，并且具有登录或注销的链接。

```php
// If there is a user ID in the session
// the user is already logged in
if(!isset($_GET['action'])) {
  if(!empty($_SESSION['user_id'])) {
    echo '<h3>Logged In</h3>';
    echo '<p>User ID: '.$_SESSION['user_id'].'</p>';
    echo '<p>Email: '.$_SESSION['email'].'</p>';
    echo '<p><a href="?action=logout">Log Out</a></p>';
 
    // Fetch user info from Google's userinfo endpoint
    echo '<h3>User Info</h3>';
    echo '<pre>';
    $ch = curl_init('https://www.googleapis.com/oauth2/v3/userinfo');
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
      'Authorization: Bearer '.$_SESSION['access_token']
    ]);
    curl_exec($ch);
    echo '</pre>';
 
  } else {
    echo '<h3>Not logged in</h3>';
    echo '<p><a href="?action=login">Log In</a></p>';
  }
  die();
}
```

已注销的页面包含指向我们的登录的URL链接，该URL用于启动OAuth流程。

### 授权请求

现在我们已经设置了必要的变量，让我们开始OAuth流程。

我们要让人们做的第一件事是,在查询字符串为`?action=login`时，访问此页面以启动该流程。

请注意，此请求中的scope现在是OpenID Connect的scope，“openid email”，表示我们不是要求访问用户的Google数据，只是想知道他们是谁。

另请注意，我们使用`response_type=code`参数来指示我们希望Google返回用来交换`id_token`的授权码。

```php
// Start the login process by sending the user
// to Google's authorization page
if(isset($_GET['action']) && $_GET['action'] == 'login') {
  unset($_SESSION['user_id']);
 
  // Generate a random hash and store in the session
  $_SESSION['state'] = bin2hex(random_bytes(16));
 
  $params = array(
    'response_type' => 'code',
    'client_id' => $googleClientID,
    'redirect_uri' => $baseURL,
    'scope' => 'openid email',
    'state' => $_SESSION['state']
  );
 
  // Redirect the user to Google's authorization page
  header('Location: '.$authorizeURL.'?'.http_build_query($params));
  die();
}
```

生成用于保护客户端的“state”参数非常重要。这是客户端生成并存储在会话中的随机字符串。当Google将用户发送回应用时，该应用使用state参数来验证是否是它发出的请求。

我们建立一个授权URL，然后将用户发送到那里。该网址包含我们的公共客户ID，我们之前在Google注册的重定向网址，我们要求的scope以及state参数。

![Google的授权请求](https://ws4.sinaimg.cn/large/006tKfTcly1ftdz43ass6j30se0x2wgz.jpg)

如果用户已登录Google，他们会看到如上所示的帐户选择页面，要求他们选择现有帐户或使用其他帐户。请注意，此屏幕看起来不像典型的OAuth屏幕，因为用户没有授予应用程序任何权限，而只是尝试识别它们。

当用户选择一个帐户时，他们将被重定向回我们的页面，并在请求中返回code和state参数。下一步是使用Google API验证授权码。

### 获取ID令牌

当用户被重定向回我们的应用程序时，查询字符串中将有一个code和state参数。该state参数将与我们在初始授权请求中设置的参数相同，并且在用于我们的应用程序之前，应该检查它是否匹配。这可确保我们的应用不会欺骗Google向攻击者发送授权码。

```php
// When Google redirects the user back here, there will
// be a "code" and "state" parameter in the query string
if(isset($_GET['code'])) {
  // Verify the state matches our stored state
  if(!isset($_GET['state']) || $_SESSION['state'] != $_GET['state']) {
    header('Location: ' . $baseURL . '?error=invalid_state');
    die();
  }
 
  // Verify the authorization code
  $ch = curl_init($tokenURL);
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
  curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query([
    'grant_type' => 'authorization_code',
    'client_id' => $googleClientID,
    'client_secret' => $googleClientSecret,
    'redirect_uri' => $baseURL,
    'code' => $_GET['code']
  ]));
  $response = json_decode(curl_exec($ch), true);
 
  // ... fill in from the code in the next section
}
```

此代码首先检查从Google返回的“状态”是否与我们在会话中存储的状态匹配。

我们向Google的令牌endpoint建立了一个POST请求，其中包含我们应用的客户端ID和密码，以及Google在查询字符串中发回给我们的授权码。

Google将验证我们的请求，然后使用访问令牌和ID令牌进行回复。响应将如下所示。

```json
{
  "access_token": "ya29.Glins-oLtuljNVfthQU2bpJVJPTu",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6ImFmZmM2MjkwN
  2E0NDYxODJhZGMxZmE0ZTgxZmRiYTYzMTBkY2U2M2YifQ.eyJhenAi
  OiIyNzIxOTYwNjkxNzMtZm81ZWI0MXQzbmR1cTZ1ZXRkc2pkdWdzZX
  V0ZnBtc3QuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJhdWQi
  OiIyNzIxOTYwNjkxNzMtZm81ZWI0MXQzbmR1cTZ1ZXRkc2pkdWdzZX
  V0ZnBtc3QuYXBwcy5nb29nbGV1c2VyY29udGVudC5jb20iLCJzdWIi
  OiIxMTc4NDc5MTI4NzU5MTM5MDU0OTMiLCJlbWFpbCI6ImFhcm9uLn
  BhcmVja2lAZ21haWwuY29tIiwiZW1haWxfdmVyaWZpZWQiOnRydWUs
  ImF0X2hhc2giOiJpRVljNDBUR0luUkhoVEJidWRncEpRIiwiZXhwIj
  oxNTI0NTk5MDU2LCJpc3MiOiJodHRwczovL2FjY291bnRzLmdvb2ds
  ZS5jb20iLCJpYXQiOjE1MjQ1OTU0NTZ9.ho2czp_1JWsglJ9jN8gCg
  WfxDi2gY4X5-QcT56RUGkgh5BJaaWdlrRhhN_eNuJyN3HRPhvVA_KJ
  Vy1tMltTVd2OQ6VkxgBNfBsThG_zLPZriw7a1lANblarwxLZID4fXD
  YG-O8U-gw4xb-NIsOzx6xsxRBdfKKniavuEg56Sd3eKYyqrMA0DWnI
  agqLiKE6kpZkaGImIpLcIxJPF0-yeJTMt_p1NoJF7uguHHLYr6752h
  qppnBpMjFL2YMDVeg3jl1y5DeSKNPh6cZ8H2p4Xb2UIrJguGbQHVIJ
  vtm_AspRjrmaTUQKrzXDRCfDROSUU-h7XKIWRrEd2-W9UkV5oCg"
}
```

应将访问令牌视为不透明字符串。除了能够使用它来发出API请求之外，它对您的应用程序没有重要意义。

ID令牌具有您的应用可以解析的特定结构，以找出登录者的用户数据。ID令牌是JWT，在OpenID Connect中有更详细的解释。您可以将Google的JWT粘贴到jsonwebtoken.io等网站，以快速向您显示内容。

### 验证用户信息

通常，在信任ID令牌中的任何信息之前验证ID令牌至关重要。这是因为通常您的应用会通过不受信任的渠道（例如浏览器重定向）获取ID令牌。

在这种情况下，您使用客户端密钥通过HTTPS连接向Google获取ID令牌，以便向Google进行身份验证，因此您可以确信您获得的ID令牌实际上来自服务商而非攻击者。考虑到这一点，我们可以不加验证的解码ID令牌。谷歌就是这样做的：`https://developers.google.com/identity/protocols/OpenIDConnect#obtainuserinfo`。

看看上面的JWT。它由三个部分组成，每个部分用一个句点分隔。我们可以在点上分割字符串，然后取出中间部分。中间部分是base64编码的JSON字符串，包含ID令牌数据。以下是JWT中的数据示例。

```json
{
  "azp": "272196069173.apps.googleusercontent.com",
  "aud": "272196069173.apps.googleusercontent.com",
  "sub": "110248495921238986420",
  "hd": "okta.com",
  "email": "aaron.parecki@okta.com",
  "email_verified": true,
  "at_hash": "0bzSP5g7IfV3HXoLwYS3Lg",
  "exp": 1524601669,
  "iss": "https://accounts.google.com",
  "iat": 1524598069
}
```

我们真正关心的这个演示是两个属性sub和email。sub(subject)属性包含了登录用户的唯一用户标识符。我们会提取它，并将其存储在会话中，这将向我们的应用程序表明用户已经登录。

我们还会在会话中存储ID令牌和访问令牌，以便我们以后可以使用它们，这也是我们获取并显示用户信息的另一种方法。

```php
  // ... continuing from the previous code sample, insert this
 
  // Split the JWT string into three parts
  $jwt = explode('.', $data['id_token']);
 
  // Extract the middle part, base64 decode, then json_decode it
  $userinfo = json_decode(base64_decode($jwt[1]), true);
 
  $_SESSION['user_id'] = $userinfo['sub'];
  $_SESSION['email'] = $userinfo['email'];
 
  // While we're at it, let's store the access token and id token
  // so we can use them later
  $_SESSION['access_token'] = $data['access_token'];
  $_SESSION['id_token'] = $data['id_token'];
 
  header('Location: ' . $baseURL);
  die();
}
```

现在，您将被重定向回应用程序的主页，我们将使用我们在先前创建的代码向您显示用户ID和电子邮件。

```javascript
echo '<p>User ID: '.$_SESSION['user_id'].'</p>';
echo '<p>Email: '.$_SESSION['email'].'</p>';
```

#### 使用ID令牌检索用户信息

Google提供了一个额外的API endpoint，称为tokeninfo endpoint，您可以使用它来查找ID令牌详细信息，而不是自己解析它。这不建议用于生产应用程序，因为它需要额外的HTTP往返，但可用于测试和故障排除。

Google的tokeninfo endpoint`https://www.googleapis.com/oauth2/v3/tokeninfo`位于其OpenID Connect发现文档中:`https://accounts.google.com/.well-known/openid-configuration`。要查找我们收到的ID令牌的信息，请使用查询字符串中的ID令牌向tokeninfo endpoint发出GET请求。

`https://www.googleapis.com/oauth2/v3/tokeninfo?id_token=eyJ`

响应将是一个JSON对象，其中包含JWT本身包含的类似属性列表。

```json
{
 "azp": "272196069173.apps.googleusercontent.com",
 "aud": "272196069173.apps.googleusercontent.com",
 "sub": "110248495921238986420",
 "hd": "okta.com",
 "email": "aaron.parecki@okta.com",
 "email_verified": "true",
 "at_hash": "NUuq_yggZYi_2-13hJSOXw",
 "exp": "1524681857",
 "iss": "https://accounts.google.com",
 "iat": "1524678257",
 "alg": "RS256",
 "kid": "affc62907a446182adc1fa4e81fdba6310dce63f"
}
```

#### 使用访问令牌来检索用户信息

如前所述，许多OAuth 2.0服务还提供endpoint来检索登录用户的用户信息。这是OpenID Connect标准的一部分，endpoint将成为服务的OpenID Connect Discovery文档的一部分。

谷歌的userinfo endpoint是`https://www.googleapis.com/oauth2/v3/userinfo`。在这种情况下，您使用访问令牌而不是ID令牌来查找用户信息。向该 endpoint发出GET请求，并像执行OAuth 2.0 API请求时那样,在HTTP Header中添加Authorization传递访问令牌。

```http
GET /oauth2/v3/userinfo
Host: www.googleapis.com
Authorization: Bearer ya29.Gl-oBRPLiI9IrSRA70...
```

响应将是一个JSON对象，其中包含有关用户的若干属性。响应将始终包含sub密钥，该密钥是用户的唯一标识符。Google还会返回用户的个人资料信息，例如姓名，个人资料照片网址，性别，区域设置，个人资料网址和电子邮件。服务器还可以添加自己的声明，例如Google hd在使用G Suite帐户时显示帐户的“托管域”。

```json
{
 "sub": "110248495921238986420",
 "name": "Aaron Parecki",
 "given_name": "Aaron",
 "family_name": "Parecki",
 "picture": "https://lh4.googleusercontent.com/-kw-iMgD
   _j34/AAAAAAAAAAI/AAAAAAAAAAc/P1YY91tzesU/photo.jpg",
 "email": "aaron.parecki@okta.com",
 "email_verified": true,
 "locale": "en",
 "hd": "okta.com"
}
```

#### 下载示例代码
您可以从GitHub下载此示例中使用的完整示例代码，网址为`https://github.com/aaronpk/sample-oauth2-client`。

在用户登录后，您已经看到了三种不同的方式来获取用户的个人资料信息。那么您应该使用哪个以及何时使用？

对于性能敏感的应用程序，您可能在每个请求上读取ID令牌或使用它们来维护会话，您绝对应该在本地验证ID令牌而不是发出网络请求。Google的API文档提供了有关离线验证ID令牌的详细信息的指南。

如果您所做的只是在登录后尝试查找用户的姓名和电子邮件，那么向userinfo endpoint发出API请求是最简单，最直接的选择。