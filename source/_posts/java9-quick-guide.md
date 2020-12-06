---
layout: post
title: Project Jigsaw:Java模块系统快速入门指南
date: 2018-03-29 10:12:00
tags: 
- Java 9
categories:
- Java 9
---

**本文为openjdk官方java模块入门指南的翻译。([原文地址](http://openjdk.java.net/projects/jigsaw/quick-start))**

本文档提供了一些让开发者快速上手Java模块系统的简单例子。
例子中的文件路径用/划分，文件分隔符是：，windows开发者请用\和；替换以上分隔符。

### Greetings
第一个例子展示了一个打印"Greetings!"字符串的简单模块。这个模块由两个源文件组成（module-info.java和Main.java）。按照惯例，模块的源文件应该在一个以模块名字为命名的目录中。
```java
src/com.greetings/com/greetings/Main.java
src/com.greetings/module-info.java

$ cat src/com.greetings/module-info.java
module com.greetings { }

$ cat src/com.greetings/com/greetings/Main.java
package com.greetings;
public class Main {
    public static void main(String[] args) {
        System.out.println("Greetings!");
    }
}
```
以下命令会将源文件编译到mods/com.greetings目录中：
```bash
$ mkdir -p mods/com.greetings

$ javac -d mods/com.greetings \
    src/com.greetings/module-info.java \
    src/com.greetings/com/greetings/Main.java
```
现在我们用以下命令来运行这个例子：
```bash
$ java --module-path mods -m com.greetings/com.greetings.Main
```
--module-path 是模块的路径，他的值是包含模块的一个或多个目录。
-m 指定了主模块，/后面的值是模块里包含main方法的全类名。

### Greetings world
第二个例子更新了第一个例子中的模块声明，它声明了需要一个名为org.astro的模块的依赖。
模块org.astro导出了名为org.astro的api包。
```java
src/org.astro/module-info.java
src/org.astro/org/astro/World.java
src/com.greetings/com/greetings/Main.java
src/com.greetings/module-info.java

$ cat src/org.astro/module-info.java
module org.astro {
    exports org.astro;
}

$ cat src/org.astro/org/astro/World.java
package org.astro;
public class World {
    public static String name() {
        return "world";
    }
}

$ cat src/com.greetings/module-info.java
module com.greetings {
    requires org.astro;
}

$ cat src/com.greetings/com/greetings/Main.java
package com.greetings;
import org.astro.World;
public class Main {
    public static void main(String[] args) {
        System.out.format("Greetings %s!%n", World.name());
    }
}
```
依次编译这两个模块，用javac命令在编译com.greetings模块时指定模块的路径，使得org.astro模块中的引用和其导出包中的类型可以被引用。
```bash
$ mkdir -p mods/org.astro mods/com.greetings

$ javac -d mods/org.astro \
    src/org.astro/module-info.java src/org.astro/org/astro/World.java

$ javac --module-path mods -d mods/com.greetings \
    src/com.greetings/module-info.java src/com.greetings/com/greetings/Main.java
```
使用与第一个例子相同的方式来运行这个例子：
```bash
$ java --module-path mods -m com.greetings/com.greetings.Main
Greetings world!
```

### 多模块编译
在上面的例子中，模块com.greetings和模块org.astro是分开编译的，我们也可以用一条javac命令来编译多个模块
```bash
$ mkdir mods

$ javac -d mods --module-source-path src $(find src -name "*.java")

$ find mods -type f
mods/com.greetings/com/greetings/Main.class
mods/com.greetings/module-info.class
mods/org.astro/module-info.class
mods/org.astro/org/astro/World.class
```

### 打包
在目前的实例中，编译好的模块都分散在文件系统中，为了调度和部署，我们通常会把模块打包成模块化jar包，模块化jar包就是在jar包最顶层目录包含module-info.class的普通jar包。

以下命令将会在mlib目录中创建org.astro@1.0.jar和com.greetings.jar
```bash
$ mkdir mlib

$ jar --create --file=mlib/org.astro@1.0.jar \
    --module-version=1.0 -C mods/org.astro .

$ jar --create --file=mlib/com.greetings.jar \
    --main-class=com.greetings.Main -C mods/com.greetings .

$ ls mlib
com.greetings.jar   org.astro@1.0.jar
```
在这个例子中，模块org.astro在打包时被指定了其版本号1.0，模块com.greetings在打包时被制定了其main方法主类com.greetings.Main。

现在，我们可以直接运行模块com.greetings而无需制定其main class
```bash
$ java -p mlib -m com.greetings
Greetings world!
```
命令可以用-p来替代--module-path。

jar命令拥有许多新选项，其中一个就是打印模块化jar包中声明的模块。
```bash
$ jar --describe-module --file=mlib/org.astro@1.0.jar
org.astro@1.0 jar:file:///d/mlib/org.astro@1.0.jar/!module-info.class
exports org.astro
requires java.base mandated
```

### 缺少requires或者exports关键字
如果我们在com.greetings模块中遗漏了requires关键字：
```java
$ cat src/com.greetings/module-info.java
module com.greetings {
    // requires org.astro;
}

$ javac --module-path mods -d mods/com.greetings \
    src/com.greetings/module-info.java src/com.greetings/com/greetings/Main.java
src/com.greetings/com/greetings/Main.java:2: error: package org.astro is not visible
    import org.astro.World;
                ^
    (package org.astro is declared in module org.astro, but module com.greetings does not read it)
1 error
```
如果我们在org.astro模块中遗漏了exports关键字：
```java
$ cat src/com.greetings/module-info.java
module com.greetings {
    requires org.astro;
}
$ cat src/org.astro/module-info.java
module org.astro {
    // exports org.astro;
}

$ javac --module-path mods -d mods/com.greetings \
    src/com.greetings/module-info.java src/com.greetings/com/greetings/Main.java
$ javac --module-path mods -d mods/com.greetings \
    src/com.greetings/module-info.java src/com.greetings/com/greetings/Main.java
src/com.greetings/com/greetings/Main.java:2: error: package org.astro is not visible
    import org.astro.World;
                ^
    (package org.astro is declared in module org.astro, which does not export it)
1 error
```

### 服务
服务允许服务提供者和服务消费者中建立松散的耦合结构。在这个例子中，存在这一个服务提供者和一个服务消费者：模块com.socket提供了API用来做network socket；模块org.fastsocket是一个服务提供模块，它提供了com.socket.spi.NetworkSocketProvider的实现，并且不导出任何包。

下面是模块com.socket的代码：
```java
$ cat src/com.socket/module-info.java
module com.socket {
    exports com.socket;
    exports com.socket.spi;
    uses com.socket.spi.NetworkSocketProvider;
}

$ cat src/com.socket/com/socket/NetworkSocket.java
package com.socket;

import java.io.Closeable;
import java.util.Iterator;
import java.util.ServiceLoader;

import com.socket.spi.NetworkSocketProvider;

public abstract class NetworkSocket implements Closeable {
    protected NetworkSocket() { }

    public static NetworkSocket open() {
        ServiceLoader<NetworkSocketProvider> sl
            = ServiceLoader.load(NetworkSocketProvider.class);
        Iterator<NetworkSocketProvider> iter = sl.iterator();
        if (!iter.hasNext())
            throw new RuntimeException("No service providers found!");
        NetworkSocketProvider provider = iter.next();
        return provider.openNetworkSocket();
    }
}


$ cat src/com.socket/com/socket/spi/NetworkSocketProvider.java
package com.socket.spi;

import com.socket.NetworkSocket;

public abstract class NetworkSocketProvider {
    protected NetworkSocketProvider() { }

    public abstract NetworkSocket openNetworkSocket();
}
```
以下是模块org.fastsocket的代码
```java
$ cat src/org.fastsocket/module-info.java
module org.fastsocket {
    requires com.socket;
    provides com.socket.spi.NetworkSocketProvider
        with org.fastsocket.FastNetworkSocketProvider;
}

$ cat src/org.fastsocket/org/fastsocket/FastNetworkSocketProvider.java
package org.fastsocket;

import com.socket.NetworkSocket;
import com.socket.spi.NetworkSocketProvider;

public class FastNetworkSocketProvider extends NetworkSocketProvider {
    public FastNetworkSocketProvider() { }

    @Override
    public NetworkSocket openNetworkSocket() {
        return new FastNetworkSocket();
    }
}

$ cat src/org.fastsocket/org/fastsocket/FastNetworkSocket.java
package org.fastsocket;

import com.socket.NetworkSocket;

class FastNetworkSocket extends NetworkSocket {
    FastNetworkSocket() { }
    public void close() { }
}
```
为了简单化，我们同时编译两个模块，但事实上，服务提供者和服务消费者几乎总是分开编译的
```bash
$ mkdir mods
$ javac -d mods --module-source-path src $(find src -name "*.java")
```
最后我们对模块com.greetings用新的模块的api做更改
```java
$ cat src/com.greetings/module-info.java
module com.greetings {
    requires com.socket;
}

$ cat src/com.greetings/com/greetings/Main.java
package com.greetings;

import com.socket.NetworkSocket;

public class Main {
    public static void main(String[] args) {
        NetworkSocket s = NetworkSocket.open();
        System.out.println(s.getClass());
    }
}


$ javac -d mods/com.greetings/ -p mods $(find src/com.greetings/ -name "*.java")
```
最后，我们运行com.greetings模块
```bash
$ java -p mods -m com.greetings/com.greetings.Main
class org.fastsocket.FastNetworkSocket
```
输出结果确认服务提供者已经被定位成功。

### The linker
jlink是一个用来在一组拥有传递性依赖的模块之中，建立一个自定义的模块化可运行镜像的工具。

此工具目前需要制定模块路径的模块化jar包或者jmod格式。jdk会将标准的或jdk指定的模块以jmod格式打包。

以下命令会创建一个包含模块com.greetings和其传递性依赖的可运行镜像。
```bash
jlink --module-path $JAVA_HOME/jmods:mlib --add-modules com.greetings --output greetingsapp
```
--module-path的值是包含打包后的模块的路径。在windows下需要将':'替换成';'。
$JAVA_HOME/jmods是包含java.base.jmod和其他标准化jdk模块的路径。mlib路径包含模块com.greetings的artifact。

jlink工具也包含许多高级的选项来自定义镜像，详见jlink --help

### --patch-module
开发者常会从Doug Lea的CVS中checkout出java.util.concurrent下的类并用-Xbootclasspath/p来替换源文件编译。(我都不知道Doug Lea还在更新juc的代码，膜拜大神)

现在-Xbootclasspath/p已经被舍弃，它在模块化系统中替代是--patch-module，用来替换模块中的类，它也可以被用来增大模块的内容。

javac命令同样也支持--patch-module选项用来编译模块中的as if部分。

以下是用新版本的java.util.concurrent.ConcurrentHashMap来编译并用其运行的例子
```bash
javac --patch-module java.base=src -d mypatches/java.base \
    src/java.base/java/util/concurrent/ConcurrentHashMap.java

java --patch-module java.base=mypatches/java.base ...
```

### 更多链接
- [The State of the Module System](http://openjdk.java.net/projects/jigsaw/spec/sotms/)
- [JEP 261: Module System](http://openjdk.java.net/jeps/261)
- [Project Jigsaw](http://openjdk.java.net/projects/jigsaw/)