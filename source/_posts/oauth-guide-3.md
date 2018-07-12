---
layout: post
title: OAuth教程--服务器端应用程序
date: 2018-06-29 16:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/oauth2-clients/server-side-apps/)）**

服务器端应用程序是处理OAuth 2服务器时遇到的最常见的应用程序类型。这些应用程序在Web服务器上运行，其中应用程序的源代码不可供公众使用，因此他们可以保持其客户端密钥的机密性。
下图说明了用户与正在与客户端通信的浏览器进行交互的典型示例。客户端和API服务器之间具有单独的安全通信通道。用户的浏览器从不直接向API服务器发出请求，所有内容都先通过客户端。

![应用程序的服务器与API通信](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/okta_oauth-flow.png)

服务器端应用程序使用authorization_code授权类型。在此流程中，在用户授权应用程序后，应用程序会收到一个“授权码”，然后它可以交换访问令牌。

### 授权码授予

授权码是客户端将为访问令牌交换的临时代码。代码本身从授权服务器获得，其中用户有机会查看客户端请求的信息，并批准或拒绝该请求。

授权码与其他授权类型相比具有一些优势。当用户授权应用程序时，会使用URL中的临时代码将它们重定向回应用程序。应用程序交换访问令牌的代码。当应用程序发出访问令牌请求时，该请求将使用客户端密钥进行身份验证，从而降低攻击者拦截授权代码并自行使用它的风险。这也意味着访问令牌永远不会被用户看到，所以它是将令牌传递回应用程序的最安全的方式，降低了令牌泄漏给其他人的风险。

Web流程的第一步是请求用户授权。这是通过创建用户单击的授权请求链接来完成的。

#### 与OAuth1的不同

在OAuth 1中，您首先必须向API发出请求以获取用于请求的唯一URL。在OAuth 2.0中，可以直接创建授权URL，因为它仅包含客户端自身创建的信息。

授权URL通常采用以下格式：

```html
https://authorization-server.com/oauth/authorize
  ?client_id=a17c21ed
  &response_type=code
  &state=5ca75bd30
  &redirect_uri=https%3A%2F%2Foauth2client.com%2Fauth
```

确切的URL端点将由您要连接的服务指定，但参数名称将始终相同。

请注意，在接受服务之前，您很可能首先需要在服务中注册重定向URL。这也意味着您无法根据请求更改重定向网址。然而，您可以使用该state参数来自定义请求。请参阅下面的详细信息。

用户访问授权页面后，该服务会向用户显示请求的解释，包括应用程序名称，范围等（请参阅下面的“批准请求”示例的屏幕快照。）如果用户单击“批准”，则服务器将带着您在查询字符串参数中提供的“code”和“state”参数，重定向回应用程序。请务必注意，这不是访问令牌。您可以使用授权码完成的唯一一件事就是发出请求以获取访问令牌。

### 授权授予参数

以下参数用于进行授权授予。您应该使用以下参数构建一个查询字符串，并将其附加到从其文档中获取的应用程序的授权端点。

>**client_id**
>client_id是您的应用的标识符。首次向服务注册您的应用程序时，您将收到一个client_id。

>**response_type**
>response_type设置为code表示您希望授权码作为响应。

>**redirect_uri （可选的）**
>redirect_uri对于API来说是可选的，但强烈建议您添加此参数。这是您希望在授权完成后将用户重定向到的URL。这必须与您先前在服务中注册的重定向URL相匹配。

>**scope （可选的）**
>包括一个或多个范围值（以空格分隔）以请求其他访问级别。该值取决于特定服务。

>**state （推荐的）**
>该state参数有两个功能。当用户重定向回您的应用程序时，state包含的任何值都将包含在重定向中。这使您的应用程序有机会在被定向到授权服务器的用户和再次返回之间保留数据，例如使用state参数作为会话密钥。这可用于指示在授权完成后应用中要执行的操作，例如，指示在授权后要将重定向到应用的哪个页面。这也可以作为CSRF保护机制。当用户重定向回您的应用程序时，请仔细检查状态值是否与您最初设置的值相匹配。这将确保攻击者无法拦截授权流程。

将所有这些查询字符串参数组合到登录URL中，来指引用户的浏览器进行重定向。通常，应用会将这些参数放入登录按钮，或者将该应用自己的登录URL作为HTTP重定向发送。

### 用户批准该请求

在用户进入服务并查看请求后，他们将允许或拒绝该请求。如果他们允许请求，他们将携带着查询字符串中的授权码，重定向回重定向URL。然后，该应用程序需要交换授权码以获得访问令牌。

### 交换访问令牌的授权码

为了交换访问令牌的授权码，应用程序向服务的令牌端点发出POST请求。该请求将具有以下参数。

>**grant_type （必须）**
>grant_type参数必须设置为“authorization_code”。

>**code （必须）**
>此参数用于设置从授权服务器接收的授权代码，该授权代码将位于此请求中的查询字符串参数“code”中。

>**redirect_uri （可能需要）**
>如果重定向URL包含在初始授权请求中，则它也必须包含在令牌请求中，并且必须相同。某些服务支持注册多个重定向URL，有些服务需要在每个请求上指定重定向URL。请查看服务的文档以了解具体信息。

>**客户端验证（必填）**
>该服务将要求客户端在提出访问令牌请求时进行身份验证。通常，服务通过HTTP Basic Auth，用客户端的client_id和client_secret来支持客户端身份验证。但是，一些服务通过接受client_id和client_secret作为POST主体参数来支持身份验证。检查服务的文档以找出服务期望的内容，OAuth 2.0规范将此决定留给服务。

### 例子

以下实例逐步说明如何使用授权代码授予类型。

#### 步骤

从上层来看是这样的：

- 创建一个包含应用程序客户端ID，重定向URL和state参数的登录链接
- 用户看到授权提示并批准请求
- 用户使用身份验证代码将用户重定向回应用程序的服务器
- 应用程序通过身份验证码来交换访问令牌

#### 应用程序启动授权请求

应用程序通过制作包含ID，scope和state的URL来启动流程。该应用程序可以将其放入`<a href="">`标签中。

```html
<a href="https://authorization-server.com/oauth/authorize?response_type=code
     &client_id=mRkZGFjM&state=5ca75bd30">Connect Your Account</a>
```

#### 用户批准该请求

在被定向到auth服务器后，用户看到如下所示的授权请求。如果用户批准该请求，他们将带着身份验证码和state参数重定向回应用程序。

![示例授权请求](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/okta_oauth-allow-auth.png)

#### 该服务将用户重定向回应用程序

服务发送重定向的header，将用户的浏览器重定向回发出请求的应用程序。重定向将在URL中包含“code”参数。

>https://example-app.com/cb?code=Yzk5ZDczMzRlNDEwY

#### 应用程序用身份验证码来交换访问令牌

应用程序使用身份验证码通过向授权服务器发出POST请求来获取访问令牌。

```bash
POST /oauth/token HTTP/1.1
Host: authorization-server.com
 
code=Yzk5ZDczMzRlNDEwY
&grant_type=code
&redirect_uri=https://example-app.com/cb
&client_id=mRkZGFjM
&client_secret=ZGVmMjMz
```

auth服务器验证请求，返回访问令牌和过期时使用的刷新令牌。

响应：

```json
{
  "access_token": "AYjcyMzY3ZDhiNmJkNTY",
  "refresh_token": "RjY2NjM5NzA2OWJjuE7c",
  "token_type": "bearer",
  "expires": 3600
}
```

### 可能的错误

有几种情况下，您可能会在授权期间收到错误响应。

错误会通过在返回的重定向URL的查询字符串提示。重定向URL总会有一个错误参数，也可能包含error_description和error_uri。

例如:
`https://example-app.com/cb?error=invalid_scope`

尽管服务器返回error_description密钥，但错误描述并不打算显示给用户。相反，您应该向用户显示您自己的错误消息。这允许您告诉用户采取适当的措施来纠正问题，并且如果您正在构建多语言网站，还可以让您有机会本地化错误消息。

#### 无效的重定向网址

如果提供的重定向URL无效，则auth服务器不会重定向到它。相反，它可以向用户显示描述问题的信息。

#### 无法识别 client_id

如果无法识别客户端ID，则auth服务器不会重定向用户。相反，它可能会显示一条描述问题的信息。

#### 用户拒绝该请求

如果用户拒绝授权请求，则服务器会将用户重定向包含error=access_denied查询字符串的重定向URL ，并且不会出现code字段。由应用程序决定此时向用户显示的内容。

#### 无效的参数

如果一个或多个参数无效，例如缺少必需值，或者response_type参数错误，服务器将重定向到重定向URL并包含描述问题的查询字符串参数。

error参数的其他可能值为：

- invalid_request：请求缺少必需参数，包含无效参数值，或者格式错误。
- unauthorized_client：客户端无权使用此方法请求授权码。
- unsupported_response_type：授权服务器不支持使用此方法获取授权码。
- invalid_scope：请求的范围无效，未知或格式错误。
- server_error：授权服务器遇到意外情况，导致无法完成请求。
- temporarily_unavailable：由于服务器临时过载或维护，授权服务器当前无法处理请求。

此外，服务器可以包括参数error_description和error_uri关于错误的附加信息。

### 示例代码

让我们通过Github了解授权代码流的工作示例。示例代码是用PHP编写的，不需要外部包，也不需要框架。如果需要的话，这可以很容易翻译成其他语言。

我们来定义一个方法，`apiRequest()`它是一个围绕`cURL`的简单包装。此函数包含`application / json`header，并自动解码JSON响应。如果我们在会话中有访问令牌，它会使用访问令牌发送正确的OAuth header。

```php
function apiRequest($url, $post=FALSE, $headers=array()) {
  $ch = curl_init($url);
  curl_setopt($ch, CURLOPT_RETURNTRANSFER, TRUE);

  if($post)
    curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($post));

  $headers[] = 'Accept: application/json';

  if(isset($_SESSION['access_token']))
    $headers[] = 'Authorization: Bearer ' . $_SESSION['access_token'];

  curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);

  $response = curl_exec($ch);
  return json_decode($response);
}
```

现在让我们设置一些OAuth流程所需的变量。

```php
// Fill these out with the values you got from Github
$githubClientID = '';
$githubClientSecret = '';
 
// This is the URL we'll send the user to first to get their authorization
$authorizeURL = 'https://github.com/login/oauth/authorize';
 
// This is the endpoint our server will request an access token from
$tokenURL = 'https://github.com/login/oauth/access_token';
 
// This is the Github base URL we can use to make authenticated API requests
$apiURLBase = 'https://api.github.com/';
 
// Start a session so we have a place to store things between redirects
session_start();
```

首先，我们设置“登录”和“注销”视图。

```php
// If there is an access token in the session the user is logged in
if(isset($_SESSION['access_token'])) {
  // Make an API request to Github to fetch basic profile information
  $user = apiRequest($apiURLBase . 'user');
 
  echo '<h3>Logged In</h3>';
  echo '<h4>' . $user->name . '</h4>';
  echo '<pre>';
  print_r($user);
  echo '</pre>';
 
} else {
  echo '<h3>Not logged in</h3>';
  echo '<p><a href="?login">Log In</a></p>';
}
```

已注销的视图包含指向我们的登录URL的链接，该URL启动OAuth流程。

当用户登录时，我们向Github发出API请求以检索基本配置文件信息并显示其名称。

现在我们已经设置了必要的变量，让我们开始OAuth流程。

我们要让人们做的第一件事是通过?action=login的查询字符串来访问此页面以启动该过程。

```php
// Start the login process by sending the user to Github's authorization page
if(isset($_GET['login'])) {
  // Generate a random hash and store in the session for security
  $_SESSION['state'] = bin2hex(random_bytes(16));
  unset($_SESSION['access_token']);
 
  $redirectURI = 'http://' . $_SERVER['SERVER_NAME'] . $_SERVER['PHP_SELF']
  $params = array(
    'client_id' => $githubClientID,
    'redirect_uri' => $redirectURI,
    'scope' => 'user',
    'state' => $_SESSION['state']
  );
 
  // Redirect the user to Github's authorization page
  header('Location: ' . $authorizeURL . '?' . http_build_query($params));
  die();
}
```

我们建立一个授权URL，然后将用户发送到那里。该URL包含我们的公共客户端ID，我们之前在Github上注册的重定向URL，我们要求的范围以及“state”参数。

我们使用state参数作为额外的安全检查，这样在Github将用户返回的查询字符的state变量中，我们可以验证我们确实发起了这个请求，而不是其他人劫持会话。

此时，用户将被定向到Github，他们将看到标准OAuth授权提示，如下所示。

![示例授权请求](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_oauth_prompt.png)

当用户批准请求时，他们将带着code和state参数重定向回我们的页面。下面的代码处理此请求。

```php
// When Github redirects the user back here, there will be a "code" and "state" 
// parameter in the query string
if(isset($_GET['code'])) {
  // Verify the state matches our stored state
  if(!isset($_GET['state'])  || $_SESSION['state'] != $_GET['state']) {
    header('Location: ' . $_SERVER['PHP_SELF'] . '?error=invalid_state');
    die();
  }
 
  // Exchange the auth code for a token
  $redirectURI = 'http://' . $_SERVER['SERVER_NAME'] . $_SERVER['PHP_SELF'];
  $token = apiRequest($tokenURL, array(
    'client_id' => $githubClientID,
    'client_secret' => $githubClientSecret,
    'redirect_uri' => $redirectURI,
    'code' => $_GET['code']
  ));
  $_SESSION['access_token'] = $token->access_token;
 
  header('Location: ' . $_SERVER['PHP_SELF']);
  die();
}
```

这里的关键是向Github的令牌端点发送请求。该请求包含我们的公共客户端ID以及私人客户端密钥。我们还发送与之前相同的重定向URL以及授权码。如果一切都被验证，Github会生成一个访问令牌并在响应中返回它。我们将访问令牌存储在会话中并重定向到主页，并且用户已登录。

为简单起见，我们在此示例中未包含任何错误处理代码。实际上，您需要检查从Github返回的错误并向用户显示相应的消息。

### 下载示例代码

您可以在GitHub上[下载完整的示例代码](https://github.com/aaronpk/sample-oauth2-client)。

### 用户体验考虑事项

为了使授权代码授权生效，授权页面必须出现在用户熟悉的Web浏览器中，并且不得嵌入到iframe弹出窗口或移动到应用程序中的嵌入式浏览器中。因此，对于传统的“Web应用程序”来说，用户已经在Web浏览器中并且重定向到服务器的授权页面的这种操作，是最适用的。

### 安全考虑

授权码授权是为可以保护其客户端ID和密钥的客户端设计的。因此，最适合不提供其源代码的在服务器上运行的网络应用程序。

如果应用程序想要使用授权代码授权但无法保护其密钥（即本机移动应用程序），则在请求交换访问令牌的授权代码时就不需要客户端密钥。但是，某些服务不接受没有客户端密钥的授权码交换，因此本机应用程序可能需要为这些服务使用备用方法。

尽管OAuth 2.0规范并不特别要求重定向URL使用TLS加密，但我强烈建议您使用它。不需要的唯一原因是因为部署SSL网站对许多开发人员来说仍然是一个障碍，这将阻碍规范的广泛采用。有些API确实需要https作为重定向端点，但许多API仍然没有这样要求。