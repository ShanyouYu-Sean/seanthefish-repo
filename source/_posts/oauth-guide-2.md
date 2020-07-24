---
layout: post
title: OAuth教程--访问OAuth服务器中的数据
date: 2018-07-13 2:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

在本章中，我们将介绍如何在现有OAuth 2.0服务器上访问您的数据。对于此示例，我们将使用GitHub API，并构建一个简单的应用程序，该应用程序将展示出该GitHub登录用户创建的所有库。

### 创建一个应用程序

在我们开始之前，我们需要在GitHub上创建一个应用程序，以获取客户端ID和客户端密钥。

在GitHub.com上，从“设置”页面，单击侧栏中的“开发人员设置”链接。您最终将访问`https://github.com/settings/developers`。从那里，单击“新OAuth应用程序”，您将看到一个简短的表单，如下所示。

填写所需信息，包括回调URL。如果您在本地开发应用程序，则必须使用本地地址作为回调URL。由于GitHub每个应用程序只允许一个注册的回调URL，因此创建两个应用程序非常有用，一个用于开发，另一个用于生产。

![在GitHub上注册一个新的OAuth应用程序](https://ws1.sinaimg.cn/large/006tNc79ly1ft8d4v1w2oj318g0v40wo.jpg)

完成此表单后，您将进入一个页面，您可以在其中看到发布到您的应用程序的客户端ID和秘钥，如下所示。

客户端ID被视为公共信息，用于构建登录URL，或者可以包含在网页的Javascript源代码中。客户密钥必须保密。不要将它提交到您的git存储库！

![GitHub应用程序已创建](https://ws3.sinaimg.cn/large/006tNc79ly1ft8dayhs69j316k1vdqcv.jpg)

### 建立环境

此示例代码是用PHP编写的，不需要外部包，也不需要框架。这可以很容易地翻译成其他语言。您可以将它全部放在一个PHP文件中。

创建一个新文件夹并在该文件夹中创建一个空文件`index.php`。从该文件夹内部运行命令行，`php -S localhost:8000`，您将能够在浏览器中访问`http://localhost:8000`来运行您的代码。以下示例中的所有代码都应添加到此index.php文件中。

为了让我们更轻松，让我们定义一个方法，`apiRequest()`它是一个简单的cURL包装器。此函数将包含GitHub API所需的`Accept`和`User-Agent Header`，并自动解码JSON响应。如果我们在会话中有访问令牌，它也会发送带有访问令牌的正确OAuth Header，以便进行经过身份验证的请求。

```php
function apiRequest($url, $post=FALSE, $headers=array()) {
  $ch = curl_init($url);
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);
 
  if($post)
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($post));
 
  $headers = [
    'Accept: application/vnd.github.v3+json, application/json',
    'User-Agent: https://example-app.com/'
  ];
 
  if(isset($_SESSION['access_token']))
    $headers[] = 'Authorization: Bearer '.$_SESSION['access_token'];
 
  curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
 
  $response = curl_exec($ch);
  return json_decode($response, true);
}
```

现在让我们设置一些OAuth流程所需的变量。

```php
// Fill these out with the values from Github
$githubClientID = '';
$githubClientSecret = '';
 
// This is the URL we'll send the user to first
// to get their authorization
$authorizeURL = 'https://github.com/login/oauth/authorize';
 
// This is the endpoint we'll request an access token from
$tokenURL = 'https://github.com/login/oauth/access_token';
 
// This is the Github base URL for API requests
$apiURLBase = 'https://api.github.com/';
 
// The URL for this script, used as the redirect URL
$baseURL = 'https://' . $_SERVER['SERVER_NAME']
    . $_SERVER['PHP_SELF'];
 
// Start a session so we have a place to
// store things between redirects
session_start();
```

首先，让我们设置“登录”和“注销”视图。这将显示一条简单的消息，指示用户是登录还是注销。

```php
// If there is an access token in the session
// the user is already logged in
if(!isset($_GET['action'])) {
  if(!empty($_SESSION['access_token'])) {
    echo '<h3>Logged In</h3>';
    echo '<p><a href="?action=repos">View Repos</a></p>';
    echo '<p><a href="?action=logout">Log Out</a></p>';
  } else {
    echo '<h3>Not logged in</h3>';
    echo '<p><a href="?action=login">Log In</a></p>';
  }
  die();
}
```

已注销的视图包含指向我们的登录URL的链接，该URL启动OAuth流程。

### 授权请求

现在我们已经设置了必要的变量，让我们开始OAuth流程。

我们要让人们做的第一件事是,在查询字符串`?action=login`中访问此页面以启动该过程。

请注意，我们在此请求中要求的范围包括`user`和`public_repo`。这意味着该应用程序将能够读取用户配置文件信息以及访问公共存储库。

```php
// Start the login process by sending the user
// to Github's authorization page
if(isset($_GET['action']) && $_GET['action'] == 'login') {
  unset($_SESSION['access_token']);
 
  // Generate a random hash and store in the session
  $_SESSION['state'] = bin2hex(random_bytes(16));
 
  $params = array(
    'response_type' => 'code',
    'client_id' => $githubClientID,
    'redirect_uri' => $baseURL,
    'scope' => 'user public_repo',
    'state' => $_SESSION['state']
  );
 
  // Redirect the user to Github's authorization page
  header('Location: '.$authorizeURL.'?'.http_build_query($params));
  die();
}
```

生成用于保护客户端的“state”参数非常重要。这是客户端生成并存储在会话中的随机字符串。我们使用state参数作为额外的安全检查，以便当Github将用户发送回查询字符串中的状态时，我们可以验证我们确实发起了此请求，并且它不是发出该请求的攻击者。

我们建立授权URL，然后将用户发送到那里。URL包含我们的公共客户端ID，我们之前在Github上注册的重定向URL，我们要求的范围以及“state”参数。

![GitHub的授权请求](https://ws3.sinaimg.cn/large/006tNc79ly1ft8difjyjmj30xg1dsjx4.jpg)

此时，用户将看到Github的OAuth授权提示，如上所示。

当用户批准该请求时，他们将被重定向回我们的页面并在请求中包含code和state参数。下一步是用授权码交换访问令牌。

### 获取访问令牌

当用户被重定向回我们的应用程序时，查询字符串中将有一个code和state参数。该state参数将与我们在初始授权请求中设置的参数相同，在我们继续使用应用程序之前应该检查它是否匹配。这可以确保我们的应用程序不会被欺骗向GitHub发送攻击者的授权代码。

```php
// When Github redirects the user back here,
// there will be a "code" and "state" parameter in the query string
if(isset($_GET['code'])) {
  // Verify the state matches our stored state
  if(!isset($_GET['state'])
    || $_SESSION['state'] != $_GET['state']) {
 
    header('Location: ' . $baseURL . '?error=invalid_state');
    die();
  }
 
  // Exchange the auth code for an access token
  $token = apiRequest($tokenURL, array(
    'grant_type' => 'authorization_code',
    'client_id' => $githubClientID,
    'client_secret' => $githubClientSecret,
    'redirect_uri' => $baseURL,
    'code' => $_GET['code']
  ));
  $_SESSION['access_token'] = $token['access_token'];
 
  header('Location: ' . $baseURL);
  die();
}
```

在这里，我们向Github的令牌endpoint发送请求，用授权码来交换访问令牌。该请求包含我们的公共客户端ID以及私密客户端密钥。我们还发送与之前相同的重定向URL以及授权码。

如果一切都检出，Github会生成一个访问令牌并在响应中返回它。我们将访问令牌存储在会话中并重定向到主页，并且用户已登录。

GitHub的回复如下所示。

```json
{
  "access_token": "e2f8c8e136c73b1e909bb1021b3b4c29",
  "token_type": "bearer",
  "scope": "public_repo,user"
}
```

我们的代码已经提取了访问令牌并将其保存在会话中。下次访问该页面时，它会识别出已存在访问令牌并显示我们之前创建的登录页面。

注意：为简单起见，我们在此示例中未包含任何错误处理代码。实际上，您将检查从GitHub返回的错误并向用户显示相应的消息。