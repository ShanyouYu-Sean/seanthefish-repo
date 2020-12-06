---
layout: post
title: classloader详解
date: 2020-11-14 12:14:47
tags: 
- jvm
categories:
- jvm
---
## 类加载的顺序

![DVLBkt.png](https://s3.ax1x.com/2020/11/17/DVLBkt.png)

六种情况必须立即对类进行“初始化”(而加载、验证、准备自然需要在此之 前开始):

- 遇到new、getstatic、putstatic或invokestatic这四条字节码指令时，如果类型没有进行过初始 化，则需要先触发其初始化阶段。能够生成这四条指令的典型Java代码场景有:
  - 使用new关键字实例化对象的时候。
  - 读取或设置一个类型的静态字段(被final修饰、已在编译期把结果放入常量池的静态字段除外)
  的时候。
  - 调用一个类型的静态方法的时候。
- 使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需 要先触发其初始化。
- 当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。
- 当虚拟机启动时，用户需要指定一个要执行的主类(包含main()方法的那个类)，虚拟机会先初始化这个主类。
- 当使用JDK 7新加入的动态语言支持时，如果一个java.lang.invoke.MethodHandle实例最后的解 析结果为REF_getStatic、REF_putStatic、REF_invokeStatic、REF_newInvokeSpecial四种类型的方法句柄，并且这个方法句柄对应的类没有进行过初始化，则需要先触发其初始化。
- 当一个接口中定义了JDK 8新加入的默认方法(被default关键字修饰的接口方法)时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。

`static{}`代码块会执行`<clinit>()`，并在方法构造器`<init>()`之前执行，注意`static{}`可以访问和赋值在他之前生命的静态变量，但对于其后生命的变量只能赋值，不能访问。父类永远先于子类执行。

## 类加载器

对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性。即，比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

### 双亲委派

站在Java虚拟机的角度来看，只存在两种不同的类加载器:

- 一种是启动类加载器(Bootstrap ClassLoader)，这个类加载器使用C++语言实现，是虚拟机自身的一部分;
- 另外一种就是其他所有 的类加载器，这些类加载器都由Java语言实现，独立存在于虚拟机外部，并且全都继承自抽象类 java.lang.ClassLoader。

站在Java开发者的角度来说，绝大多数Java程序都会使用到以下3个系统提供的类加载器来进行加载：

- 启动类加载器(Bootstrap Class Loader):前面已经介绍过，这个类加载器负责加载存放在`<JAVA_HOME>\lib`目录，或者被`-Xbootclasspath`参数所指定的路径中存放的，而且是Java虚拟机能够识别的(按照文件名识别，如rt.jar、tools.jar，名字不符合的类库即使放在lib目录中也不会被加载)类库加载到虚拟机的内存中。启动类加载器无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器去处理，那直接使用null代替即可。
- 扩展类加载器(Extension Class Loader):这个类加载器是在类`sun.misc.Launcher$ExtClassLoader`以Java代码的形式实现的。它负责加载`<JAVA_HOM E>\lib\ext`目录中，或者被`java.ext.dirs`系统变量所 指定的路径中所有的类库。根据“扩展类加载器”这个名称，就可以推断出这是一种Java系统类库的扩展机制，JDK的开发团队允许用户将具有通用性的类库放置在ext目录里以扩展Java SE的功能，在JDK 9之后，这种扩展机制被模块化带来的天然的扩展能力所取代。由于扩展类加载器是由Java代码实现的，开发者可以直接在程序中使用扩展类加载器来加载Class文件。
- 应用程序类加载器(Application Class Loader):这个类加载器由
`sun.misc.Launcher$AppClassLoader`来实现。由于应用程序类加载器是ClassLoader类中的`getSystemClassLoader()`方法的返回值，所以也称它为“系统类加载器”。它负责加载用户类路径(ClassPath)上所有的类库，开发者同样可以直接在代码中使用这个类加载器。如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。

![DZCFu6.png](https://s3.ax1x.com/2020/11/17/DZCFu6.png)

双亲委派模型的工作过程是:如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求(它的搜索范围中没有找到所需的类)时，子加载器才会尝试自己去完成加载。

使用双亲委派模型来组织类加载器之间的关系，一个显而易见的好处就是Java中的类随着它的类加载器一起具备了一种带有优先级的层次关系。相同包名的同名类只能在一个类加载器中加载，从而避免混乱。

```java
protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
    // 首先，检查请求的类是否已经被加载过了 
    Class c = findLoadedClass(name); 
    if (c == null) {
        try {
            if (parent != null) {
                c = parent.loadClass(name, false);
            } else {
            c = findBootstrapClassOrNull(name); }
        } catch (ClassNotFoundException e) {
        // 如果父类加载器抛出ClassNotFoundException 
        // 说明父类加载器无法完成加载请求
        }
        if (c == null) {
            // 在父类加载器无法加载时
            // 再调用本身的findClass方法来进行类加载 
            c = findClass(name);
        } 
    }
    if (resolve) { 
        resolveClass(c);
    }
    return c; 
}

```

### 破坏双亲委派

#### 自定义classloader

类加载器的概念和抽象类java.lang.ClassLoader无法避免loadClass()被子类覆盖，只能通过protected方法findClass()，并引导用写的类加载逻辑时尽可能去重写这个方法，而不是在loadClass()中编写代码。按照loadClass()方法的逻辑，如果父类加载失败，会自动调用自己的findClass()方法来完成加载，这样既不影响用户按照自己的意愿去加载类，又可以保证新写出来的类加载器是符合双亲委派规则的。

#### 线程上下文类加载器 (Thread Context ClassLoader)

以jndi为例：
JNDI存在的目的就是对资源进行查找和集中管理，它需要调用由其他厂商实现并部署在应用程 序的ClassPath下的JNDI服务提供者接口(Service Provider Interface，SPI)的代码，但启动类加载器是绝不可能认识、加载这些代码的。

为了解决这个困境，Java的设计团队只好引入了一个不太优雅的设计:线程上下文类加载器(Thread Context ClassLoader)。这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。

如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统上下文类加载器。在SPI接口的代码中使用线程上下文类加载器，就可以成功的加载到SPI实现的类。

JNDI服务使用这个线程上下文类加载器去加载所需的SPI服务代码，这是一种父类加载器去请求子类加载器完成类加载的行为，这种行为实际上是打通了双亲委派模型的层次结构来逆向使用类加载器，已经违背了双亲委派模型的一般性原则。

#### OSGi

OSGi实现模块化热部署的关键是它自定义的类加载器机制的实现，每一个程序模块(OSGi中称为 Bundle)都有一个自己的类加载器，当需要更换一个Bundle时，就把Bundle连同类加载器一起换掉以实 现代码的热替换。在OSGi环境下，类加载器不再双亲委派模型推荐的树状结构，而是进一步发展为更加复杂的网状结构。

#### Java9中的模块化

一篇详细讲解模块化的文章
http://seanthefish.com/2018/03/29/module-system/

jdk9提出了与“类路径”(ClassPath)相对应的“模块路径”(M odulePath)的概念。某个类库到底是模块还是传统的JAR包，只取决于它存放在哪种路径上。只要是放在类路径上的JAR文件，无论其中是否包含模块化信息(是否包含了module-info.class文件)，它都会被当作传统的JAR包来对待;相应地，只 要放在模块路径上的JAR文件，即使没有使用JMOD后缀，甚至说其中并不包含module-info.class文 件，它也仍然会被当作一个模块来对待。

- JAR文件在类路径的访问规则:所有类路径下的JAR文件及其他资源文件，都被视为自动打包在一个匿名模块(Unnamed Module)里，这个匿名模块几乎是没有任何隔离的，它可以看到和使用类路 径上所有的包、JDK系统模块中所有的导出包，以及模块路径上所有模块中导出的包。
- 模块在模块路径的访问规则:模块路径下的具名模块(Named Module)只能访问到它依赖定义中列明依赖的模块和包，匿名模块里所有的内容对具名模块来说都是不可见的，即具名模块看不见传统JAR包的内容。
- JAR文件在模块路径的访问规则:如果把一个传统的、不包含模块定义的JAR文件放置到模块路径中，它就会变成一个自动模块(Automatic Module)。尽管不包含module-info.class，但自动模块将默认依赖于整个模块路径中的所有模块，因此可以访问到所有模块导出的包，自动模块也默认导出自己所有的包。

在模块化的java9中：

- 扩展类加载器(Extension Class Loader)被平台类加载器(Platform Class Loader)取代。
- 平台类加载器和应用程序类加载器都不再派生自java.net.URLClassLoader，如果有程序直接 依赖了这种继承关系，或者依赖了URLClassLoader类的特定方法，那代码很可能会在JDK 9及更高版 本的JDK中崩溃。
- JDK 9中虽然仍然维持着三层类加载器和双亲委派的架构，但类加载的委派关系也发生了变动。当平台及应用程序类加载器收到类加载请求，在委派给父加载器加载前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要优先委派给负责那个模块的加载器完成加载。

![DZZ78S.png](https://s3.ax1x.com/2020/11/17/DZZ78S.png)

## tomcat

Tomcat的类加载机制是违反了双亲委托原则的，对于一些未加载的非基础类(Object,String等)，各个web应用自己的类加载器(WebAppClassLoader)会优先加载，加载不到时再交给commonClassLoader走双亲委托。

首先要了解的是，tomcat有两中类加载器模式：

默认情况下：

![DZJNi4.png](https://s3.ax1x.com/2020/11/17/DZJNi4.png)

高级模式：

![DZJRWd.png](https://s3.ax1x.com/2020/11/17/DZJRWd.png)

根据conf/catalina.properties文件中common.loader，server.loader，shared.loader的值来初始化commonLoader,catalinaLoader,sharedLoader，其中catalinaLoader,sharedLoader的父亲加载器是commonLoader，默认情况下，这3个ClassLoader是同一个实例变量。

tomcat容器不希望它下面的webapps之间能互相访问到，所以不能用jvm中的appClassLoarder去加载。所以tomcat新建一个sharedClassLoader（它的parent是commonClassLoader，commonClassLoader的parent是jvm中的appClassLoarder，默认情况下，sharedClassLoader和commonClassLoader是同一个UrlClassLoader实例），这是catalina容器使用的ClassLoader。对于每个webapp，为其新建一个webappClassLoader，用于加载webapp下面的类，这样webapp之间就不能相互访问了。tomcat的ClassLoader不完全遵循双亲委派，首先用webappClassLoader去加载某个类，如果找不到，再交给parent。而对于java核心库，不在tomcat的ClassLoader的加载范围。

- commonLoader：Tomcat最基本的类加载器，加载路径（/common/*）中的class可以被Tomcat容器本身以及各个Webapp访问；
- catalinaLoader：Tomcat容器私有的类加载器，加载路径（/server/*）中的class对于Webapp不可见；
- sharedLoader：各个Webapp共享的类加载器，加载路径（/shared/*（在tomcat 6之后已经合并到根目录下的lib目录下））中的class对于所有Webapp可见，但是对于Tomcat容器不可见；
- WebappClassLoader：各个Webapp私有的类加载器，加载路径（/WebApp/WEB-INF/*中的Java类库）中的class只对当前Webapp可见；

当应用需要到某个类时，默认会按照下面的顺序进行类加载（不使用委派模式）：

- 使用bootstrapclassloader加载器加载
- 使用webapppclassloader加载器在WEB-INF/classes中加载
- 使用webapppclassloader加载器在WEB-INF/lib中加载
- 使用appclassloader（systemclassloader）加载器加载
- 使用commonclassloader在CATALINA_HOME/lib中加载

使用委派模式的话（配置`<Loader delegate="true"/>`）:

- 使用bootstrapclassloader加载器加载
- 使用appclassloader（systemclassloader）加载器加载
- 使用commonclassloader在CATALINA_HOME/lib中加载
- 使用webapppclassloader加载器在WEB-INF/classes中加载
- 使用webapppclassloader加载器在WEB-INF/lib中加载

## was

简单说一下was，直接上图

![DZ0PDs.png](https://s3.ax1x.com/2020/11/17/DZ0PDs.png)

was里分为Parent_first和Parent_last模式

![DZ03Ux.png](https://s3.ax1x.com/2020/11/17/DZ03Ux.png)

![DZ0aKH.png](https://s3.ax1x.com/2020/11/17/DZ0aKH.png)

同时在was的appclassloader级别（即为ear）和webmoduleclassloader级别（即为war）都分别具备隔离和共享两种classloader模式。

![DZ0LM4.png](https://s3.ax1x.com/2020/11/17/DZ0LM4.png)

![DZ0vZR.png](https://s3.ax1x.com/2020/11/17/DZ0vZR.png)

Shared Library模式

![DZBALd.png](https://s3.ax1x.com/2020/11/17/DZBALd.png)

![DZBZdI.png](https://s3.ax1x.com/2020/11/17/DZBZdI.png)

独立的shared library模式（有自己的classloader，且必须为PARENT_LAST模式）

![DZBJwn.png](https://s3.ax1x.com/2020/11/17/DZBJwn.png)

常见问题：
（图片里说的很清楚，故不做多余解释，这些问题也可以引申到其他的服务器中）

![DZB0lF.png](https://s3.ax1x.com/2020/11/17/DZB0lF.png)
Solution:
 Create a shared library to hold the test1.jar file and associate the shared library with both EAR1 and EAR2.

![DZB46e.png](https://s3.ax1x.com/2020/11/17/DZB46e.png)
Solution:
 Place the dependent jars, test3.jar, in the same classloader as test1.jar so that both jar files are loaded by the same class loaderand visible to each other. 

![DZB7TI.png](https://s3.ax1x.com/2020/11/17/DZB7TI.png)
Solution:
 Set the application class loader mode to parent_last so the correct version from the EAR file to be picked up first.
 Or remove the duplicate class (wrong version) from the shared library.

又想起了曾经被was支配的恐惧。
打fatjar还是最简单有效的。