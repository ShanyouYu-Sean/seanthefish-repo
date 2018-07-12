---
layout: post
title: OAuth教程--删除应用程序和撤消密钥
date: 2018-07-08 18:00:00
tags: 
- OAuth
categories:
- OAuth
---
**本文是oauth.com上的教程的翻译。（[原文地址](https://www.oauth.com/oauth2-servers/client-registration/deleting-applications-revoking-secrets/)）**

开发人员需要一种方法来删除（或至少停用）他们的应用程序。为开发人员提供一种方法来撤销并为其应用程序生成新的客户端密钥也是一个好主意。

### 删除应用程序

当开发人员删除应用程序时，该服务应通知开发人员删除应用程序的后果。例如，GitHub告诉开发人员将撤销所有访问令牌，以及将受影响的用户数量。

![GitHub删除应用程序提示](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_delete_application.png)
GitHub要求确认删除申请

删除应用程序应立即撤消颁发给应用程序的所有访问令牌和其他凭据，例如待处理的授权码和刷新令牌。

### 撤销密钥

该服务应该为开发人员提供重置客户端密钥的方法。在密钥被意外暴露的情况下，开发人员需要一种方法来确保可以撤销旧密钥。撤销密钥不一定会使用户的访问令牌无效，因为如果开发人员想要使所有用户令牌无效，他们也可以随时删除该应用程序。

![GitHub重置客户端密钥提示](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/oauth-guide/github_reset_client_secret.png)
GitHub要求确认重置应用程序的密钥

重置密钥应该保持所有现有的访问令牌都处于活动状态。但是，这确实意味着使用旧密钥的任何已部署应用程序将无法使用旧密钥刷新访问令牌。部署的应用程序需要在能够使用刷新令牌之前更新其密钥。