---
layout: post
title: OAuth教程--访问令牌
date: 2018-07-13 12:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com)）**

访问令牌是应用程序用于代表用户发出API请求的内容。访问令牌表示特定应用程序访问用户数据的特定部分的授权。

访问令牌必须在运输和存储过程中保密。应该看到访问令牌的唯一方是应用程序本身，授权服务器和资源服务器。应用程序应确保同一设备上的其他应用程序无法访问访问令牌的存储。访问令牌只能在https连接上使用，因为通过非加密通道传递它会使第三方拦截变得非常容易。

令牌endpoint是应用程序发出请求以获取用户访问令牌的位置。本节介绍如何验证令牌请求以及如何返回相应的响应和错误。

### 授权码请求

当应用试图交换访问令牌时，将使用授权码交换授予。用户通过重定向URL返回应用程序后，应用程序将从URL获取授权码并使用它来请求访问令牌。此请求将发送到令牌endpoint。

#### 请求参数

访问令牌请求将包含以下参数。

>**grant_type （需要）**
>grant_type参数必须设置为“authorization_code”。

>**code （需要）**
>code参数是客户端先前从授权服务器接收的授权码。

>**redirect_uri （可能需要）**
>如果重定向URI包含在初始授权请求中，则服务也必须在令牌请求中要求它。令牌请求中的重定向URI必须与生成授权代码时使用的重定向URI完全匹配。 否则服务必须拒绝该请求。

>**client_id （如果不存在其他客户端身份验证，则需要）**
>如果客户端通过HTTP Basic Auth或其他方法进行身份验证，则不需要此参数。否则，此参数是必需的。

如果客户端发出了客户端密钥，则服务器必须对客户端进行身份验证。验证客户端的一种方法是接受此请求中的另一个参数client_secret。或者，授权服务器可以使用HTTP Basic Auth。从技术上讲，规范允许授权服务器支持任何形式的客户端身份验证，甚至提到公钥/私钥对作为选项。但实际上，大多数服务器都支持使用此处提到的方法之一或两者共同来验证客户端的简单方法。

#### 验证授权码

在检查所有必需参数并在客户端被发出秘钥时验证客户端之后，授权服务器可以继续验证请求的其他部分。

然后，服务器检查授权码是否有效，并且尚未过期。然后，该服务必须验证请求中提供的授权码是否已发送给所标识的客户端。最后，服务必须确保存的重定向URI参数与用于请求授权代码的重定向URI匹配。

如果所有内容都检出，该服务可以生成访问令牌并进行响应。

例
以下示例显示了私密客户端的授权授予请求。

```http
POST /oauth/token HTTP/1.1
Host: authorization-server.com
 
grant_type=authorization_code
&code=xxxxxxxxxxx
&redirect_uri=https://example-app.com/redirect
&client_id=xxxxxxxxxx
&client_secret=xxxxxxxxxx
```

#### 安全考虑因素

##### 防止重复攻击

如果多次使用授权码，授权服务器必须拒绝后续请求。授权码存储在数据库中是很容易实现，因为它们可以简单地标记为已使用。

如果要实现自编码授权码（如我们的示例代码中所示），则需要跟踪已在令牌生命周期中使用的令牌。一种方法是通过生命周期之内，将授权码缓存在缓存中来实现此目的（具有过期时间的缓存）。这样在验证时，我们可以通过检查授权码的缓存来检查它们是否已被使用。一旦授权码到达其到期日期，它将不再在缓存中，但我们可以根据到期日期拒绝它。

如果授权码被多次使用，则应将其视为攻击。如果可能，服务应撤销此授权码发出的先前访问令牌。

### 密码授予

当应用程序交换用户的访问令牌的用户名和密码时，将使用密码授予。这正是首先要创建的OAuth，因此您绝不允许第三方应用使用此授权。

此授权类型的常见用途是用您的服务为应用启用密码登录。用户使用他们的用户名和密码登录来登录网站或原生应用程序，但绝不允许第三方应用程序直接询问用户的密码。

#### 请求参数

访问令牌请求将包含以下参数。

>**grant_type（必填）** - grant_type参数必须设置为“password”。
>**username （必填）** - 用户的用户名。
>**password （必填）** - 用户的密码。
>**scope （可选）** - 应用程序请求的范围。
>**客户端身份验证**（如果客户端被授予秘钥，则需要）

如果客户端被授予秘钥，则必须验证此请求。通常，服务将允许其他请求参数如client_id和client_secret，或接受在HTTP Basic auth header中的客户端ID和秘钥。

例
以下是服务将接收的示例密码授予。

```http
POST /oauth/token HTTP/1.1
Host: authorization-server.com
 
grant_type=password
&username=user@example.com
&password=1234luggage
&client_id=xxxxxxxxxx
&client_secret=xxxxxxxxxx
```

### 客户端凭据

当应用程序请求访问令牌来访问其自己的资源，而不是代表用户访问他们的资源时，将使用客户端凭据授予。

#### 请求参数

>**grant_type （需要）**
>grant_type参数必须设置为client_credentials。

>**scope （可选的）**
>您的服务可以支持客户端凭据授予的不同范围。实际上，实际上并没有多少服务支持这一点。

>**客户端验证（必填）**
>客户端需要为此请求进行身份验证。通常，服务将接受请求参数client_id和client_secret，或接受在HTTP Basic auth header中的客户端ID和秘钥。

例
以下是客户端凭据的请求例子。

```http
POST /token HTTP/1.1
Host: authorization-server.com
 
grant_type=client_credentials
&client_id=xxxxxxxxxx
&client_secret=xxxxxxxxxx
```

### 访问令牌响应

#### 响应成功

如果访问令牌的请求有效，则授权服务器需要生成访问令牌（和可选的刷新令牌）并将这些令牌返回给客户端，通常还有一些有关授权的其他属性。

具有访问令牌的响应应包含以下属性：

>**access_token （必需）** 授权服务器发出的访问令牌字符串。
>**token_type （必需）** 这是令牌的类型，通常只是字符串“bearer”。
>**expires_in （推荐）** 如果访问令牌会过期，服务器应回复授予访问令牌的持续时间。
>**refresh_token（可选）** 如果访问令牌会过期，则应该返回刷新令牌，应用程序可以使用该令牌来获取新的访问令牌。但是，使用隐式授权发出的令牌无法发出刷新令牌。
>**scope（可选）** 如果用户授予的范围与应用程序请求的范围相同，则此参数是可选的。如果授予的范围与请求的范围不同，例如，如果用户修改了范围，则此参数是必需的。

使用访问令牌进行响应时，服务器还必须包含附加`Cache-Control: no-store`和`Pragma: no-cache`的HTTP header，以确保客户端不会缓存此请求。

例如，成功的令牌响应可能如下所示：

```json
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
 
{
  "access_token":"MTQ0NjJkZmQ5OTM2NDE1ZTZjNGZmZjI3",
  "token_type":"bearer",
  "expires_in":3600,
  "refresh_token":"IwOGYzYTlmM2YxOTQ5MGE3YmNmMDFkNTVk",
  "scope":"create"
}
```

####访问令牌

OAuth 2.0 Bearer Token的格式实际上是在单独的规范[RFC 6750](https://tools.ietf.org/html/rfc6750)中描述的。规范要求的令牌没有已定义的结构，因此您可以生成任意字符串并实现所需的令牌。令牌中的有效字符是字母数字，和标点字符：-._~+/

通常，服务将生成随机字符串并将其与相关的用户和范围信息一起存储在数据库中，或者将使用自编码令牌，其中令牌字符串本身包含所有必要的信息。

#### 响应失败

如果访问令牌请求无效，例如重定向URL与授权期间使用的URL不匹配，则服务器需要返回错误响应。

使用HTTP 400状态代码（除非另有说明）返回错误响应，带有`error`和`error_description`参数。该`error`参数将始终是下面列出的值之一。

>**invalid_request** - 请求缺少参数，因此服务器无法继续执行请求。如果请求包含不受支持的参数或重复参数，也可能会返回此信息。
>**invalid_client** - 客户端身份验证失败，例如请求包含无效的客户端ID或秘钥。在这种情况下发送HTTP 401响应。
>**invalid_grant** - 授权码（或密码授予类型的用户密码）无效或已过期。或者授权中给出的重定向URL与此访问令牌请求中提供的URL不匹配，也将返回这个的错误。
>**invalid_scope** - 对于包含范围（密码或client_credentials授权）的访问令牌请求，此错误表示请求中的范围值无效。
>**unauthorized_client** - 此客户端无权使用请求的授权类型。例如，如果限制哪些应用程序可以使用隐式授权，则会为其他应用程序返回此错误。
>**unsupported_grant_type** - 如果请求授权服务器无法识别的授权类型，请使用此代码。请注意，未知的授权类型也使用此特定错误代码而不是使用invalid_request。

返回错误响应时有两个可选参数，`error_description`和`error_uri`。这些旨在为开发人员提供有关错误的更多信息，而不是向最终用户显示。但是，请记住，无论您多么警告，许多开发人员都会将此错误文本直接传递给最终用户，因此最好确保它对最终用户至少有所帮助。

该`error_description`参数只能包含ASCII字符，最多应该是一两句话来描述错误的情况。这`error_uri`是链接到API文档的好地方，可以获取有关如何更正遇到的特定错误的信息。

整个错误响应作为JSON字符串返回，类似于成功响应。以下是错误响应的示例。

```json
HTTP/1.1 400 Bad Request
Content-Type: application/json;charset=UTF-8
Cache-Control: no-store
Pragma: no-cache
 
{
  "error": "invalid_request",
  "error_description": "Request was missing the 'redirect_uri' parameter.",
  "error_uri": "See the full API docs at https://authorization-server.com/docs/access_token"
}
```

### 自编码访问令牌

自编码令牌提供了一种通过编码令牌字符串中的所有必要信息来避免在数据库中存储令牌的方法。这样做的主要好处是API服务器能够在不对每个API请求进行数据库查找的情况下验证访问令牌，从而使API更容易扩展。

OAuth 2.0 Bearer Tokens的好处是应用程序无需了解服务中如何实现访问令牌。这意味着可以在不影响客户端的情况下更改令牌的实现规则。

如果您已经拥有可水平扩展的分布式数据库系统，那么使用自编码令牌可能无法获得任何好处。实际上，如果已经解决了分布式数据库问题，使用自编码令牌只会引入新问题，因为使自编码令牌失效会成为额外的障碍。

有许多方法可以自编码令牌。您选择的实际方法仅对您的实现很重要，因为令牌信息不会暴露给外部开发人员。

创建自编码标记的一种方法是创建序列化的json串来包含在token中的所有数据，并使用仅为您的服务器知道的密钥对结果字符串进行签名。

一种常见的技术是使用[JSON Web Signature（JWS）](https://tools.ietf.org/html/rfc7515)标准来处理令牌的编码，解码和验证。 [JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)规范定义了一些字段，你可以在JWS中使用，或定义一些时间戳字段来确定令牌是否有效。我们将在此示例中使用JWT库，因为它提供了时间到期的内置处理。

#### 编码

下面的代码是用PHP编写的，使用Firebase PHP-JWT库来编码和验证令牌。您需要包含该库才能运行示例代码

```php
<?php
use \Firebase\JWT\JWT;
 
# Define the secret key used to create and verify the signature
$jwt_key = 'secret';
 
# Set the user ID of the user this token is for
$user_id = 1000;
 
# Set the client ID of the app that is generating this token
$client_id = 'https://example-app.com';
 
# Provide the list of scopes this token is valid for
$scope = 'read write';
 
$token_data = array(
 
  # Subject (The user ID)
  'sub' => $user_id,
 
  # Issuer (the token endpoint)
  'iss' => 'https://' . $_SERVER['PHP_SELF'],
 
  # Client ID (this is a non-standard claim)
  'cid' => $client_id,
 
  # Issued At
  'iat' => time(),
 
  # Expires At
  'exp' => time()+7200, // Valid for 2 hours
 
  # The list of OAuth scopes this token includes
  'scope' => $scope
);
$token_string = JWT::encode($token_data, $jwt_key);
```

这将产生一个字符串，如：

```jwt
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOjEwMDAsI
mlzcyI6Imh0dHBzOi8vYXV0aG9yaXphdGlvbi1zZXJ2ZXIuY29tIiw
iY2lkIjoiaHR0cHM6Ly9leGFtcGxlLWFwcC5jb20iLCJpYXQiOjE0N
zAwMDI3MDMsImV4cCI6MTUyOTE3NDg1MSwic2NvcGUiOiJyZWFkIHd
yaXRlIn0.QiIrnmaC4VrbAYAsu0YPeuJ992p20fSxrXWPLw-gkFA
```

此令牌由三部分组成，以句点分隔。第一部分描述了使用的签名方法。第二部分包含令牌数据。第三部分是签名。

例如，此令牌的第一个部分是JSON对象：

```json
{
   "typ":"JWT",
   "alg":"HS256”
}
```

第二个部分包含API端点处理请求所需的实际数据，例如用户标识和范围访问。

```json
{
  "sub": 1000,
  "iss": "https://authorization-server.com",
  "cid": "https://example-app.com",
  "iat": 1470002703,
  "exp": 1470009903,
  "scope": "read write"
}
```

Base64编码前两个部分产生以下两个字符串：

```jwt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

```jwt
eyJzdWIiOjEwMDAsImlzcyI6Imh0dHBzOi8vYXV0aG9yaXphdGlvbi1z
ZXJ2ZXIuY29tIiwiY2lkIjoiaHR0cHM6Ly9leGFtcGxlLWFwcC5jb20i
LCJpYXQiOjE0NzAwMDI3MDMsImV4cCI6MTUyOTE3NDg1MSwic2NvcGUi
OiJyZWFkIHdyaXRlIn0
```

然后我们计算两个字符串和秘钥的散列，从而产生另一个字符串：

```jwt
QiIrnmaC4VrbAYAsu0YPeuJ992p20fSxrXWPLw-gkFA
```

最后，将所有三个字符串连接在一起，用句点分隔。

```jwt
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOjEwMDAsI
mlzcyI6Imh0dHBzOi8vYXV0aG9yaXphdGlvbi1zZXJ2ZXIuY29tIiw
iY2lkIjoiaHR0cHM6Ly9leGFtcGxlLWFwcC5jb20iLCJpYXQiOjE0N
zAwMDI3MDMsImV4cCI6MTUyOTE3NDg1MSwic2NvcGUiOiJyZWFkIHd
yaXRlIn0.QiIrnmaC4VrbAYAsu0YPeuJ992p20fSxrXWPLw-gkFA
```

#### 解码

可以使用相同的JWT库来验证访问令牌。库将同时解码和验证签名，如果签名无效，或者令牌的过期日期已经过去，则会抛出异常。

注意：任何人都可以通过base64解码令牌信息来解码令牌字符串的中间部分。因此，您不要存储您不希望用户或开发人员在令牌中看到的私人信息或信息，这一点很重要。如果要隐藏令牌信息，可以使用JSON Web加密规范加密令牌中的数据。

```php
try {
  # Note: You must provide the list of supported algorithms in order to prevent 
  # an attacker from bypassing the signature verification. See:
  # https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
  $token = JWT::decode($token_string, $jwt_key, ['HS256']);
  $error = false;
} catch(\Firebase\JWT\ExpiredException $e) {
  $token = false;
  $error = 'expired';
  $error_description = 'The token has expired';
} catch(\Firebase\JWT\SignatureInvalidException $e) {
  $token = false;
  $error = 'invalid';
  $error_description = 'The token provided was malformed';
} catch(Exception $e) {
  $token = false;
  $error = 'unauthorized';
  $error_description = $e->getMessage();
}
 
if($error) {
  header('HTTP/1.1 401 Unauthorized');
  echo json_encode(array(
    'error'=>$error, 
    'error_description'=>$error_description
  ));
  die();
} else {
  // Now $token has all the data that we encoded in it originally
  print_r($token);
}
```

此时，该服务具有所需的所有信息，例如可用的用户ID，范围等，并且不必进行数据库查找。接下来，它可以检查以确保访问令牌未过期，可以验证范围是否足以执行所请求的操作，然后可以处理该请求。

因为可以在不进行数据库查找的情况下验证令牌，所以在令牌过期之前无法使令牌无效。您需要采取其他步骤来使自编码的令牌无效。

### 访问令牌生命周期

当您的服务发出访问令牌时，您需要做出一些关于您希望令牌持续多久的决定。不幸的是，每项服务都没有全面的解决方案。不同的选项会有各种权衡，因此您应该选择最适合您应用需求的选项（或选项组合）。

#### 短期访问令牌和长期刷新令牌

授予令牌的常用方法是使用访问令牌和刷新令牌的组合，以获得最大的安全性和灵活性。OAuth 2.0规范建议使用此选项，并且一些实现已采用此方法。

通常，使用此方法的服务将发出持续时间从几小时到几周的访问令牌。当服务发出访问令牌时，它还会生成一个永不过期的刷新令牌，并在响应中返回该令牌。（请注意，无法使用隐式授权发布刷新令牌。）

当访问令牌到期时，应用程序可以使用刷新令牌来获取新的访问令牌。它可以在幕后进行，无需用户参与，因此对用户来说是一个无缝的过程。

这种方法的主要好处是该服务可以使用自编码访问令牌，无需数据库查找即可对其进行验证。但是，这意味着没有办法直接使这些令牌过期，因此，令牌的发布时间很短，因此应用程序被迫不断刷新它们，从而使服务有机会在需要时撤销应用程序的访问权限。

从第三方开发人员的角度来看，处理刷新令牌往往令人沮丧。开发人员非常喜欢不会过期的访问令牌，因为处理的代码要少得多。为了帮助缓解这些问题，服务通常会在其SDK中构建令牌刷新逻辑，以便该流程对开发人员而言是透明的。

总之，在以下情况下使用短期访问令牌和长期刷新令牌：

- 您想使用自编码访问令牌
- 您希望限制泄露访问令牌的风险
- 您将提供给开发人员处理刷新逻辑的SDK

#### 短期访问令牌，没有刷新令牌

如果要确保用户了解正在访问其帐户的应用程序，则该服务可以在不刷新令牌的情况下发出相对短期的访问令牌。访问令牌可以持续从当前应用程序会话到几周。当访问令牌到期时，应用程序将被强制使用户再次登录，以便您作为服务知道用户不断参与重新授权应用程序。

通常，此选项由服务使用，如果第三方应用程序意外或恶意泄漏访问令牌，则存在高风险。通过要求用户不断重新授权应用程序，如果攻击者从服务中窃取访问权限，则该服务可以确保潜在的损害受到限制。

通过不发布刷新令牌，这使得应用程序无法在用户不在屏幕前的情况下持续使用访问令牌。这使得连续同步数据的应用程序将无法在此方法下执行此操作。

从用户的角度来看，这是最有可能让人感到沮丧的选项，因为用户必须不断重新授权应用程序。

总之，在以下情况下使用没有刷新令牌的短期访问令牌：

- 您希望最大程度地防范泄露访问令牌的风险
- 您希望强制用户了解他们授予的第三方访问权限
- 您不希望第三方应用程序对用户数据进行脱机访问

#### 非过期访问令牌

非过期访问令牌是开发人员最简单的方法。如果您选择此选项，请务必考虑您所做的权衡。

如果你想能够任意撤销它们，那么使用自编码令牌是不切实际的。因此，您需要将这些令牌存储在某种数据库中，因此可以根据需要删除它们或将其标记为无效。

请注意，即使服务打算发布非过期访问令牌以供正常使用，您仍需要提供一种机制，以便在特殊情况下使其过期，例如，如果用户明确要撤消应用程序的访问权限，或者用户帐户已删除。

对于测试自己的应用程序的开发人员来说，非过期访问令牌就更容易。您甚至可以为开发人员预生成一个或多个非过期访问令牌，并在应用程序详细信息页面上显示给他们。通过这种方式，他们可以立即开始使用令牌发出API请求，而不必担心设置OAuth流程以开始测试API。

总之，在以下情况下使用非过期访问令牌：

- 你有一个机制来随时撤销访问令牌
- 如果令牌泄露，你没有巨大的风险
- 您希望为开发人员提供简单的身份验证机制
- 您希望第三方应用程序可以脱机访问用户的数据

### 刷新访问令牌

本节介绍如何允许开发人员使用刷新令牌获取新的访问令牌。如果您的服务与访问令牌一起发布刷新令牌，那么您需要实现此处描述的刷新授权类型。

#### 请求参数

访问令牌请求将包含以下参数。

>**grant_type （需要）**
grant_type参数必须设置为“refresh_token”。

>**refresh_token （需要）**
>先前发布给客户端的刷新令牌。

>**scope （可选的）**
>请求的范围不得包含原始访问令牌中未发布的其他范围。通常，这不会包含在请求中，如果省略，服务应发出一个访问令牌，其范围与先前发布的相同。

>**客户端身份验证（如果客户端被授予秘钥，则需要）**
>通常，刷新令牌仅用于私密客户端。但是，由于可以在没有客户端秘钥的情况下使用授权码流程，因此刷新授权也可以给没有秘钥的客户端使用。如果客户端被授予秘钥，则客户端必须验证此请求。通常，该服务将允许额外的查询参数client_id和client_secret，或接受在HTTP Basic auth header中的客户端ID和秘钥。如果客户端没有秘钥，则此请求中不会出现客户端身份验证。

#### 验证刷新令牌授权

在检查所有必需参数并在客户端被授予秘钥时验证客户端之后，授权服务器可以继续验证请求的其他部分。

然后，服务器检查刷新令牌是否有效，并且尚未过期。如果向私密客户端发出刷新令牌，则服务必须确保将请求中的刷新令牌发送给经过身份验证的客户端。

如果所有内容都检出，该服务可以生成访问令牌并进行响应。服务器可以在响应中发出新的刷新令牌，但是如果响应不包括新的刷新令牌，则客户端假定现有的刷新令牌仍然有效。

例
以下是服务将接收的示例刷新授权。

```http
POST /oauth/token HTTP/1.1
Host: authorization-server.com
 
grant_type=refresh_token
&refresh_token=xxxxxxxxxxx
&client_id=xxxxxxxxxx
&client_secret=xxxxxxxxxx
```