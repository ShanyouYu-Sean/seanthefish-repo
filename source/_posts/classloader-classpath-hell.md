---
layout: post
title: 也来谈谈classloader和classpath hell问题
date: 2018-01-11 12:14:47
tags: 
- classLoader
categories:
- classLoader
---
classpath hell, 也叫 [jar hell](https://en.wikipedia.org/wiki/Java_Classloader#JAR_hell)。
先来简单翻译一下wiki：
Java Classloader 是 Java Runtime Environment 的一部分，它动态地将 Java 类加载到 jvm 中。类通常是按需加载的（即只有在需要实例化这个类时才会被加载）。因为有了 classloader 的存在，java runtime 不再需要知道有哪些文件。一个 java 类通常只会被加载一次。
当jvm启动时，会用到三个 classloader：
1. Bootstrap class loader
2. Extensions class loader
3. System class loader

bootstrap class loader是 jvm 的一部分，会加载 <JAVA_HOME>/jre/lib 下的所有类。
