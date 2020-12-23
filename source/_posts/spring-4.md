---
layout: post
title: spring 事务
date: 2020-11-27 17:00:00
tags: 
- spring
categories:
- spring
---

源码直接看 https://segmentfault.com/a/1190000018001752

## Spring事务源码梳理

- 通过注解@EnableTransactionManagement中的@Import(TransactionManagementConfigurationSelector.class)给容器中导入了两个组件，分别是：AutoProxyRegistrar和ProxyTransactionManagementConfiguration
- AutoProxyRegistrar：它是一个后置处理器，给容器中注册一个InfrastructureAdvisorAutoProxyCreator，InfrastructureAdvisorAutoProxyCreator利用后置处理器机制在对象创建以后，对对象进行包装，返回一个代理对象(增强器)，代理对象执行方法，利用拦截器链进行调用。
- ProxyTransactionManagementConfiguration：给容器中注册事务增强器
- 事务增强器要用事务注解信息：AnnotationTransactionAttributeSource来解析事务注解
- 事务拦截器中：transactionInterceptor()，它是一个TransactionInterceptor(保存了事务属性信息和事务管理器)，而TransactionInterceptor是一个MethodInterceptor，在目标方法执行的时候，执行拦截器链，事务拦截器(首先获取事务相关的属性，再获取PlatformTransactionManager，如果没有指定任何transactionManager，最终会从容器中按照类型获取一个PlatformTransactionManager，最后执行目标方法，如果异常，便获取到事务管理器进行回滚，如果正常，同样拿到事务管理器提交事务。


## 传播机制类型

下面的类型都是针对于被调用方法来说的，理解起来要想象成两个 service 方法的调用才可以。

- PROPAGATION_REQUIRED (默认)
  支持当前事务，如果当前没有事务，则新建事务
  如果当前存在事务，则加入当前事务，合并成一个事务
- REQUIRES_NEW
  新建事务，如果当前存在事务，则把当前事务挂起
  这个方法会独立提交事务，不受调用者的事务影响，父级异常，它也是正常提交
- NESTED
  如果当前存在事务，它将会成为父级事务的一个子事务，方法结束后并没有提交，只有等父事务结束才提交
  如果当前没有事务，则新建事务
  如果它异常，父级可以捕获它的异常而不进行回滚，正常提交
  但如果父级异常，它必然回滚，这就是和 REQUIRES_NEW 的区别
- SUPPORTS
  如果当前存在事务，则加入事务
  如果当前不存在事务，则以非事务方式运行，这个和不写没区别
- NOT_SUPPORTED
  以非事务方式运行
  如果当前存在事务，则把当前事务挂起
- MANDATORY
  如果当前存在事务，则运行在当前事务中
  如果当前无事务，则抛出异常，也即父级方法必须有事务
- NEVER
  以非事务方式运行，如果当前存在事务，则抛出异常，即父级方法必须无事务

一点小说明
一般用得比较多的是 PROPAGATION_REQUIRED ， REQUIRES_NEW；REQUIRES_NEW 一般用在子方法需要单独事务，