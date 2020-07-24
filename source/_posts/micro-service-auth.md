---
layout: post
title: 微服务架构下的认证鉴权系统设计
date: 2018-11-25 10:12:00
tags: 
- spring
- micro service
categories:
- spring
- micro service
---

单体应用下的权限设计主要是[RBAC](https://en.wikipedia.org/wiki/Role-based_access_control), 要接触微服务下的权限架构首先要熟悉这个，目前在Java的技术栈之下主要是spring security和shiro，本文主要是spring security技术栈，并在spring security上做了相应的定制化设计。

先上架构图：

![架构图](https://ws3.sinaimg.cn/large/006tNbRwly1fxnw84eo38j318c0fw45a.jpg)

微服务的架构下，服务与服务拆分，就会产生服务与服务之间的鉴权，网关到服务的鉴权，前端到网关的验证的诸多问题，网上对于这些问题有许多看起来很靠谱的解决方式，但是却没一个讲的很清楚的，本文在这里将会以多个角度系统的对微服务下的权限管理做一个综述。

### 认证还是授权

我发现网上大多数的文章都没有对这两点分的很清楚，网上大多数文章中，多数都讲的是认证，而没有权限管理，属于挂羊头卖狗肉，忽略的最本质的问题。

首先声明，本文所指的鉴权系统，认证和授权两部分都有包涵。

先说认证：

先梳理一下在一个微服务系统中，一个request走了多少步。前端发出request，到达网关，网关进行检查过滤，通过后就可以转发到对应的service中了，在没有加入认证授权系统的时候，我们可以简单的认为一个request在微服务架构中只走了三部，即客户端->网关->service。

zhizhi