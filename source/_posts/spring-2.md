---
layout: post
title: spring aop源码解析
date: 2020-11-27 15:00:00
tags: 
- spring
categories:
- spring
---

直接看 https://segmentfault.com/a/1190000022726755

## 总结

Spring如何实现AOP？

- AnnotationAwareAspectJAutoProxyCreator是AOP核心处理类
- AnnotationAwareAspectJAutoProxyCreator实现了BeanProcessor，
- 其中postProcesBeforeInitialization会扫描所有被@Aspect修饰的Bean，将其包装成AdvisorBeans，然后再在这些bean中，查找被@Around、@Before、@After、@AfterReturing、@AfterThrowing（如果一个方法同时被多个注解修饰，按顺序只会匹配第一个被找到的注解）修饰的方法，将注解的信息封装为AspectJExpressionPointcut，将方法的内部逻辑封装为不同类型的xxxAdvice，并最终将上述的Pointcut及Advice组合封装为InstantiationModelAwarePointcutAdvisorImpl提供出去，Advice实现了MethodInterceptor接口，在生成代理执行的时候调用。
- 其中postProcessAfterInitialization是核心方法。
- 核心实现分为2步
  - getAdvicesAndAdvisorsForBean获取当前bean匹配的增强器
    - 找所有Advisor
    - 找匹配的Advisor，也就是根据@Before，@After等注解上的表达式，与当前bean进行匹配，暴露匹配上的。通过Pointcut中的ClassFilter及MethodMatcher匹配目标Bean需要应用的Advisor。
    - 对匹配的Advisor进行扩展和排序，就是按照@Order或者PriorityOrdered的getOrder的数据值进行排序，越小的越靠前。
  - createProxy为当前bean创建代理,createProxy有2种创建方法，JDK代理或CGLIB
    - 如果设置了proxyTargetClass=true，一定是CGLIB代理
    - 如果proxyTargetClass=false，目标对象实现了接口，走JDK代理
    - 如果没有实现接口，走CGLIB代理
    - 对Cglib来讲，关键在于设置Enhancer的Callback，逻辑可以参考CglibAopProxy#getCallbacks
    - 对JDK Proxy来讲，关键在于定义InvocationHandler，逻辑可以参考JdkDynamicAopProxy#invoke
    - 在匹配的Advisor中找到当前方法匹配到Advices，将以上Advices封装为ReflectiveMethodInvocation，触发整条链
    - 如果一个Bean中的方法匹配上多个Advice，那多个Advice的invoke方法通过ReflectiveMethodInvocation进行链式调用

