---
layout: post
title: Java模块系统综述
date: 2018-03-29 16:28:00
tags: 
- Java 9
categories:
- Java 9
---

**本文是Mark Reinhold的The State of the Module System最新版的翻译。（[原文地址](http://openjdk.java.net/projects/jigsaw/spec/sotms/)）**

> 这份文档略有过时。其基本概念没有任何改变，但requires public关键字已被重新命名为requires transitive，并增加了几项附加功能。更新正在准备中，准备就绪后会在这里发布。

本文是对[Jigsaw项目](http://openjdk.java.net/projects/jigsaw/)中对Java SE平台所做的增强的一个非正式的概述，并针对[JSR 376：Java平台模块系统](http://openjdk.java.net/projects/jigsaw/spec/)所提出。有[相关的文档](http://openjdk.java.net/jeps/261)描述了对特定于JDK工具和API的增强，这些超出了JSR的范围。

正如JSR所述，模块系统是为了提供可靠的配置，使程序组件相互显式的声明依赖，配合其强大的封装能力，使组件允许声明其中哪些公共类型可供其他组件访问，哪些不可以，并以此来替换脆弱，容易出错的类路径机制。

这些功能将直接对Java SE平台本身、Java应用程序开发人员，Java类库开发人员有利，而且也会间接地实现可伸缩平台、更高的平台完整性和更高的性能。

### 目录
- [目录](#目录)
- [定义模块](#定义模块)
  - [模块声明](#模块声明)
  - [模块打包](#模块打包)
  - [模块描述符](#模块描述符)
- [平台模块](#平台模块)
- [使用模块](#使用模块)
  - [模块路径](#模块路径)
  - [解决依赖](#解决依赖)
  - [可读性](#可读性)
  - [可访问性](#可访问性)
  - [隐含的可读性](#隐含的可读性)
- [兼容性和迁移](#兼容性和迁移)
  - [未命名模块](#未命名模块)
  - [自下而上的迁移](#自下而上的迁移)
  - [自动模块](#自动模块)
  - [与类路径建立联系](#与类路径建立联系)
- [服务](#服务)
- [高级特性](#高级特性)
  - [反射](#反射)
  - [反射的可读性](#反射的可读性)
  - [类加载器](#类加载器)
  - [未命名模块与类加载器](#未命名模块与类加载器)
  - [层](#层)
  - [限制性导出](#限制性导出)
- [总结](#总结)

这是本文档的第二版。相对于[最初版本](http://openjdk.java.net/projects/jigsaw/spec/sotms/2015-09-08)，本版引入了兼容性和迁移的解释，修改了反射可读性的描述，进行了重新排序以改善叙述的流程，并且组织了更容易定位的目录。

文中仍然存在许多未解决的问题，其解决方案将反映在本文档的未来版本中。

### 定义模块
为了提供可靠的配置和强大的封装，使其既能接近开发人员，又能被现有工具链支持，我们将模块视为一种基本的新型Java程序组件。一个模块是一个命名的，能自我描述的代码和数据的集合。其代码被组织为一组包（package），包中包含Java类和接口。
#### 模块声明
模块的自我描述体现在它的声明中，这是一种Java编程语言的新构造。最简单的模块声明只是指定其模块的名称：
```java
module com.foo.bar { }
```
可以添加一个或多个require子句来声明该模块在编译时和运行时依赖于其他名称的某些模块：
```java
module com.foo.bar {
    requires org.baz.qux;
}
```
最后，可以添加exports子句来声明该模块中仅有特定包中的公共类型可供其他模块使用：
```java
module com.foo.bar {
    requires org.baz.qux;
    exports com.foo.bar.alpha;
    exports com.foo.bar.beta;
}
```
如果一个模块的声明不包含任何exports子句，那么它将不会导出任何类型到其他模块。

按照惯例，模块声明的源代码放置在名为module-info.java的文件中，该文件位于模块的源文件层次结构的根目录下。 com.foo.bar模块的源文件可能包括：
```java
module-info.java
com/foo/bar/alpha/AlphaFactory.java
com/foo/bar/alpha/Alpha.java
...
```
按照惯例，模块声明被编译成名为module-info.class的文件，并放置在.class文件输出目录中。

模块名称跟包名称一样，不得相互冲突。命名模块的推荐方法是使用长期用于命名软件包的反向域名模式。因此，模块的名称通常就是其导出包名称的前缀，但这种关系不是强制性的。

模块的声明不包括其版本号，也不包括它所依赖的子模块的版本号。这样做是[故意为之](http://openjdk.java.net/projects/jigsaw/spec/reqs/02#version-selection)的：模块系统的目标不是解决版本选择问题，这最好留给构建工具和容器应用程序来做。

模块声明是Java编程语言的一部分，这其中的原因有几个。其中最重要的一点是，模块必须在编译时和运行时都可用，以实现各个阶段的确定性，即确保模块系统在编译时和运行时都以相同的方式工作。这反过来又能防止多种错误的发生，或者至少在编译时更早地报告错误使其更容易诊断和修复。

源文件中的模块声明文件和模块中的其他源文件，将会一起编译为.class文件供Java虚拟机使用，这是建立确定性的自然方式。这种方法将立即为开发人员所熟悉，并且IDE和构建工具也会很容易支持。尤其是IDE，可以依照依赖需要为现有组件提供模块声明的提示。

#### 模块打包
现有工具已经可以创建，操作和使用JAR文件，因此为了便于使用和迁移，我们定义了模块化JAR文件。除了根目录中还包含了一个module-info.class文件之外，模块化的JAR文件就像普通的JAR文件一样。上述com.foo.bar模块的模块化JAR文件可能具有以下内容：
```
META-INF/
META-INF/MANIFEST.MF
module-info.class
com/foo/bar/alpha/AlphaFactory.class
com/foo/bar/alpha/Alpha.class
...
```
模块化的JAR文件可以被当作模块，在这种情况下，它的module-info.class文件被用来作为模块的声明。它也可以被放在普通的类路径上，在这种情况下，它的module-info.class文件将被忽略。模块化JAR文件允许库的维护者在所有版本上发布工件（artifacts），该工件既可作为Java SE 9及更高版本的模块，也可作为类路径上的常规JAR文件。我们期望包含jar工具的Java SE 9的实现将增强该工具，以便轻松创建模块化JAR文件。

为了模块化Java SE平台的JDK，我们将引入一种新的打包机制（artifact format），它将超越JAR文件来容纳原生代码、配置文件和其他类型的数据（如果这种数据真的存在）。这种机制利用了在源文件中的模块声明并将它们编译成.class文件，此.class文件与其他的打包方式都不同。这种被临时命名为“JMOD”的新格式是否应该成为标准化仍旧是一个悬而未决的问题。（已成为Java9的标准格式之一，请查看[oracle官方文档](https://docs.oracle.com/javase/9/tools/jmod.htm)）

#### 模块描述符
将模块声明编译到.class文件中的最后一个优点是.class文件已经具有精确定义和可扩展的格式。因此，我们可以将module-info.class文件视为更通用的模块描述符，其中包括源代码级模块声明的编译形式，还包括在声明最初编译之后插入的.class文件中的附加信息。

例如，IDE、或者记录打包时间的工具，可以插入包含文档信息的属性，例如模块的版本、标题、说明和许可证。这些信息可以在编译时和运行时通过模块系统的反射来读取，以用于写文档，程序诊断和调试。它也可以被下游工具用于构建跨操作系统的程序包。特定的属性将被标准化，但由于Java类文件格式是可扩展的，所以其他工具和框架将能够根据需要来定义附加属性。非标准的属性不会影响模块系统本身的行为。

### 平台模块
Java SE 9的平台规范，使用模块系统将平台划分为一组模块。Java SE 9平台的实现可能包含所有的平台模块，或者可能仅包含其中的一部分。

在任何情况下，模块系统专用的唯一模块是已命名的基础模块java.base。基本模块定义并导出所有平台的核心软件包，包括模块系统本身：
```java
module java.base {
    exports java.io;
    exports java.lang;
    exports java.lang.annotation;
    exports java.lang.invoke;
    exports java.lang.module;
    exports java.lang.ref;
    exports java.lang.reflect;
    exports java.math;
    exports java.net;
    ...
}
```
基本模块始终存在。每个其他模块都隐式的建立在基本模块之上，而基本模块则不依赖于其他模块。

其余的平台模块将共享“ java.”名称前缀，并有可能包括，例如，模块java.sql用于数据库连接， 模块java.xml用于XML处理，模块java.logging进行记录日志。按照惯例，尽管没有在Java SE 9平台规范中定义，但是专用于JDK的模块将共享“ jdk.”名称前缀。

### 使用模块
单个模块可以在模块工件（artifacts）中定义，或者内嵌于编译时或运行时环境。要在任一阶段使用它们，模块系统必须定位它们，然后确定它们如何相互关联的，并以此提供可靠的配置和强大的封装。

#### 模块路径
为了定位包中定义的模块，模块系统搜索由系统定义的模块路径（module path）。模块路径是一个序列，其中的每个元素都是模块工件或包含模块工件的目录。系统会按顺序搜索模块路径的元素，以找到定义合适的第一个模块工件。

模块路径（module path）与类路径（class path）有着很大的不同，它更加健壮。类路径的固有脆弱性是基于这样一个事实：即它是一种在所有包中通过路径来定位各个类型的工作方式，它不会在不同的包文件本身之间进行区分。这使得它无法预先知道程序什么时候缺少了某个包。它还允许不同的程序包（artifacts）在相同的包（package）中定义类型，即使这些程序包是只是版本不同，或者就是完全不同的组件（jar hell）。

相反，模块路径是定位整个模块、而不是某个类型的一种手段。如果模块系统无法满足来自模块路径的模块工件的特定依赖性，或者如果在同一目录中遇到定义相同名称模块的两个模块工件，则编译器或虚拟机将报告错误并退出。

内置于编译时或运行时环境的模块以及模块路径中的模块工件定义的模块统称为可观察模块的范围。

#### 解决依赖
假设我们有一个使用上述com.foo.bar模块和平台java.sql模块的应用程序。包含应用程序核心的模块声明如下：
```java
module com.foo.app {
    requires com.foo.bar;
    requires java.sql;
}
```
鉴于这种初始应用程序模块，该模块系统可通过表达依赖性的requires来定位额外观察到的模块，以满足这些依赖关系，然后解决这些模块的依赖关系，并依此类推，直到每个模块的每一个的依赖都被满足。这个传递闭包计算的结果是一个模块图，对于每个依赖其他模块的模块，它包含从第一个模块到第二个模块的有向边。

要为模块com.foo.app构建模块图，模块系统将检查模块的声明java.sql，即：
```java
module java.sql {
    requires java.logging;
    requires java.xml;
    exports java.sql;
    exports javax.sql;
    exports javax.transaction.xa;
}
```
它还会循环检查其声明的com.foo.bar模块（上面的模块定义中已经示出），包括org.baz.qux模块，java.logging模块和 java.xml模块; 为简洁起见，最后三个这里没有显示，因为它们没有声明对任何其他模块的依赖。

根据所有这些模块声明，为com.foo.app模块画出的模块图，包含以下节点和边：

![module-pic-1](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-1.png)

在该图中，深蓝色线条表示显式依赖关系，如requires，而浅蓝色线条表示每个模块对基本模块的隐式依赖关系。

#### 可读性
当一个模块直接依赖于模块图中的另一个模块时，则第一个模块中的代码将能够引用第二个模块中的类型。因此，我们说第一个模块读取第二个模块，或者等同地，第二个模块可以被第一个模块读取。因此，在上述的曲线图中，com.foo.app模块读取com.foo.bar和 java.sql模块，但并不读取org.baz.qux模块，java.xml模块或 java.logging模块。java.logging模块可由java.sql模块读取，但不能被其他模块读取。（根据定义，每个模块都会自行读取自己本身。）

在模块图中定义的可读性关系是可靠配置的基础 ：模块系统能确保每个依赖是由另一个模块完成的，模块图是非循环的，每个模块最多只能读取一个包含制定包的模块，这样定义相同名称包的模块就不会相互干扰。

这样的配置不仅更可靠，也可以更快。当模块中的代码引用包中的某个类型时，那么该包将保证在该模块中定义，或者只在该模块读取的模块中定义一个。因此，在寻找特定类型的定义时，不需要在多个模块中搜索它，或者更糟糕的，沿着整个路径搜索它。

#### 可访问性
在模块图中定义的可读性关系与exports模块声明中的子句相结合，是强封装的基础：只有当某个模块被另一个模块读取时，Java编译器和虚拟机才会将这个模块中的包中的公共类型视为只能被另一个模块所访问，同时还需要这个模块导出该包。即，如果两种类型的S和T在不同的模块中定义的，并且T是 public，则如果代码S 可以存取 T，必须满足以下条件：

1. S的模块读取T的模块，
2. T的模块导出T包。

跨越模块边界的类型引用以及私有的方法和字段在这种情况下都是不可用的：任何尝试使用它的操作都将导致编译器报告错误，或者由Java虚拟机报出的IllegalAccessError，或者由反射运行时API引发的IllegalAccessException。因此，即使声明了一个类型public，如果它的包没有在其模块的声明中导出，那么它将只能被该模块中的代码访问。

如果模块中的封闭类型是可访问的，并且其成员本身也被声明成允许访问，那么跨模块也可以访问并引用到其方法或字段。

要了解上述模块图的封装是如何工作的，我们标记出每个模块导出的包：

![module-pic-2](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-2.png)

模块com.foo.app中的代码可以访问com.foo.bar.alpha包中声明的公共类型， 因为模块com.foo.app依赖于模块com.foo.bar，并且因为模块com.foo.bar导出com.foo.bar.alpha包。如果com.foo.bar包含内部程序包（internal package），即com.foo.bar.internal包，则模块com.foo.app不能访问该com.foo.bar.internal包中的任何类型，因为com.foo.bar并没有导出这个内部包。模块com.foo.app中的代码也不能引用org.baz.qux包中的类型，因为模块com.foo.app不依赖于模块org.baz.qux，因此不会读取它（在这个例子中，模块的依赖并不能传递）。

#### 隐含的可读性
如果一个模块读取另一个模块，则在某些情况下，它也能符合逻辑地读取其他一些模块。

例如，平台的java.sql模块依赖于java.logging模块和java.xml模块，不仅因为它包含了这些模块中的类型的代码实现，还因为它直接声明使用了这些模块中的类型。java.sql.Driver接口声明了一个公共的方法：
```java
public Logger getParentLogger();
```
其中Logger是在java.logging模块所导出的包java.util.logging中声明的类型。

假设，例如，com.foo.app模块中的代码调用此方法来获取日志，然后记录一条消息：
```java
String url = ...;
Properties props = ...;
Driver d = DriverManager.getDriver(url);
Connection c = d.connect(url, props);
d.getParentLogger().info("Connection acquired");
```
如果com.foo.app模块像为如上所述声明，那么这样的代码将不起作用：该getParentLogger方法返回另一个模块java.logging中所声明的Logger类型，而模块com.foo.app并没有读取模块java.logging ，因此调用java.logging模块中Logger类的info方法将会失败，因为该类以及该方法无法访问。

解决这个问题的一个方法寄希望于每一位开发者在依赖java.sql模块并使用getParentLogger方法Logger类的同时，还必须记得声明对java.logging模块的依赖。当然，这样的方式是不可靠的，因为它违反了最小意外原则（principle of least surprise）：如果一个模块依赖于第二个模块，那么很自然的我们会期望去使用第一个模块中的所有属性，包括在第二的模块中声明的属性，也会在我们依赖第一个模块是变得立即可见（即模块依赖的传递性）。

因此，我们扩展了模块声明，以便一个模块可以将附加模块的可读性授予依赖它的任何模块。这种隐含的可读性通过requires public子句来表达（在正式版的jdk中已经被更新为requires transitive）。java.sql模块的声明实际上是这样的：
```java
module java.sql {
    requires public java.logging;
    requires public java.xml;
    exports java.sql;
    exports javax.sql;
    exports javax.transaction.xa;
}
```
该public关键字是指，任何依赖于模块java.sql的模块，不仅仅会读取java.sql模块，也会读取java.logging模块和java.xml模块。因此，上述com.foo.app模块的模块图，包含两个额外的深蓝色边缘，通过绿色边缘链接到java.sql模块，因为java.logging模块和java.xml模块被该模块隐性的依赖：

![module-pic-3](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-moudle-3.png)

com.foo.app模块现在可以包含访问java.logging模块和java.xml模块的导出包中的所有公共类型的代码，即使其声明中没有提及这些模块。

一般而言，如果一个模块的导出包引用了了另一个模块中的包的类型，则第一个模块应该使用requires public子句来声明对第二个模块的依赖。这将确保依赖于第一个模块的其他模块将自动读取第二个模块，从而访问该模块的导出包中的所有类型。

### 兼容性和迁移
到目前为止，我们已经看到如何从头开始定义模块，将它们打包到模块工件中，并将它们与其他模块一起使用，这些模块既可以嵌入到平台中，也可以直接在包中定义。

当然，大多数Java代码是在引入模块系统之前编写的，并且必须继续像现在一样继续工作，而不用更改（向下兼容）。因此，即使平台本身由模块组成，模块系统也应该可以在类路径上编译和运行由JAR文件组成的应用程序。它还允许将现有应用程序以灵活和渐进的方式迁移到模块化形式。

#### 未命名模块
如果我们的需求时在加载一个其所在的包未再任何已知的模块中声明的类型，则模块系统将尝试从类路径加载它。如果成功，那么该类型会被认为成为一个，被称为未命名模块的特殊模块中的成员，以确保每个类型都与某个模块相关联。未命名模块在高层次上类似于现有概念中的未命名包。当然，所有其他模块都有名称，所以我们今后将把它们称为命名模块。

未命名模块读取所有的其他模块。因此，从类路径加载的任何类型的代码，都将能够访问所有其他可读模块的导出类型，默认情况下，该模块将包含所有内置的已命名平台模块。因此，编译并在Java SE 8上运行的现有类路径应用程序将在Java SE 9上以完全相同的方式进行编译和运行，只要它使用的是标准的，未弃用的Java SE API即可。

未命名的模块默认导出其所有软件包。这可以实现灵活的迁移。但是，它并不意味着命名模块中的代码可以访问未命名模块中的类型。实际上，命名模块甚至不能声明对未命名模块的依赖。这种限制是故意的，因为如果允许命名模块依赖于类路径的任意内容，就不可能实现可靠的配置。

如果在命名模块和未命名模块中都定义了同样名字的包，那么未命名模块中的包将被忽略。即使类路径十分混乱，这种可靠的配置，仍能确保每个模块最多只能读取一个模块来提供你所需要的包。如果在上面的示例中，类路径上的JAR文件，包含一个名为，com/foo/bar/alpha/AlphaFactory.class的.class文件，那么该文件将永远不会被加载，因为包com.foo.bar.alpha 是由模块com.foo.bar导出的。

#### 自下而上的迁移
从类路径加载的类型作为未命名模块中的成员，这种处理将允许我们自下而上的，将现有的应用程序从JAR文件形式迁移到模块化的形式。

例如，上面显示的应用程序最初是为Java SE 8构建的，因为它是放置在类路径上的一组JAR文件。如果我们在Java SE 9上按原样运行它，那么JAR文件中的类型将在未命名的模块中定义。该模块将读取所有其他模块，包括所有内置平台模块; 为简单起见，假设那些被读取的模块被限制为java.sql模块，java.xml模块， java.logging模块和java.base模块。因此我们获得如下的模块图：

![module-pic-4](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-4.png)

我们可以立即将org-baz-qux.jar转换为命名模块，因为我们知道它不会引用其他两个JAR文件中的任何类型，因此作为命名模块，它也不会引用未命名模块中的任何类型。（这是因为我们刚刚从最初的例子中知道了这一点。如果我们不知道它时候引用未命名模块的话，我们可以借助诸如类似jdeps的工具来发现它。）

我们编写一个模块声明为org.baz.qux，将其添加到模块的源代码中，编译并将结果打包为模块化JAR包。如果我们将该JAR文件放在模块路径上，并将其他类放在类路径上，我们将获得改进的模块图：

![module-pic-5](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-5.png)

com-foo-bar.jar和com-foo-app.jar中的代码会继续工作，因为未命名的模块会读取每个已命名的模块，这个未命名模块现在包含新模块org.baz.qux。

我们可以类似地进行模块化com-foo-bar.jar，然后接着模块化com-foo-app.jar最终结束预期的模块图，如前所示：

![module-pic-6](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-6.png)

如果我们了解原始JAR文件中的类型所做的工作，我们当然可以在一个步骤中将它们全部三个模块化。然而，如果 org-baz-qux.jar是独立维护的，或许由完全不同的团队或组织来维护，那么它可以在其他两个组件之前模块化，并且在com-foo-app.jar模块化之前也可以模块化com-foo-bar.jar。

#### 自动模块
自下而上的迁移很简单，但并非总是可行的。即使org-baz-qux.jar的维护者尚未将其转换为适当的模块，或者可能永远不会，我们仍然可能想要将模块化com-foo-app.jar和com-foo-bar.jar。

我们已经知道代码是com-foo-bar.jar依赖org-baz-qux.jar。但是，如果我们转换com-foo-bar.jar为命名模块com.foo.bar，但留org-baz-qux.jar在类路径中，那么该代码将不再起作用：org-baz-qux.jar将继续在未命名模块中定义，但com.foo.bar是一个命名模块，它不能声明依赖于未命名模块。

那么，我们必须以某种方式安排org-baz-qux.jar作为一个命名模块出现，以便com.foo.bar可以依赖它。我们可以fork org.baz.qux的源代码并将其模块化，但是如果维护人员不愿意将该更改合并到上游存储库中，那么只要我们需要它，我们就必须维护这个分支。

因此，我们将把org-baz-qux.jar作为一个自动模块，不加修改地放在模块路径上，而不是类路径上。这将定义一个可观察模块，其模块名称，org.baz.qux来源于JAR文件的名称，以便其他非自动模块可以通常的方式依赖它：

![module-pic-7](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-7.png)

自动模块是一个隐式定义的命名模块，因为它没有模块声明。相比之下，普通的命名模块是通过模块声明明确定义的; 我们今后将把它们称为显式模块。

没有方法可以预先告诉自动模块可能依赖哪些其他模块。因此，在模块图解析完成后，自动模块将读取每个其他命名模块，无论是自动模块还是显式模块：

![module-pic-8](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-8.png)

（这些新的可读性边在模块图中创建了循环，这使得推理起来有些困难，但我们认为这些是可以容忍的，而且通常是为了实现更灵活迁移所导致的临时结果。）

类似地，没有方法可以告诉自动模块中的哪些包打算供其他模块使用，或者仍旧是通过类路径上的类继续使用。因此，即使自动模块中的每个软件包只用于内部使用，也会被视为导出：

![module-pic-9](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-9.png)

现在com.foo.app中的代码可以访问org.baz.qux中的类型，尽管我们知道它实际上并没有这样做。

自动模块提供了混乱的类路径和显式模块规则之间的中间地带。它们允许将由JAR文件组成的现有应用程序从上到下迁移到模块，如上所示，或者以自上而下和自下而上的方法组合来迁移。通常，我们可以从类路径上的任意一组JAR文件开始，使用一个工具jdeps来分析它们之间的依赖关系，将我们控制的源代码组件转换为显式模块，然后将剩余的JAR文件按原样放在模块路径中。我们不控制源代码的JAR文件将被视为自动模块，直到它们也转换为显式模块为止。

#### 与类路径建立联系
许多现有的JAR文件可以用作自动模块，但有些不能。如果类路径上的两个或多个JAR文件包含同一个包中的类型，那么最多可以有其中的一个来用作自动模块，因为模块系统仍然保证每个命名模块至多读取一个包含了需要的包的命名模块，以保证定义了相同名称包的命名模块不会相互干扰。在这种情况下，我们经常会发现实际上，我们只需要其中一个JAR文件。如果其他的JAR文件重复或接近重复，并以某种方式错误地放在类路径上，则可以将其中一个用作自动模块，其他的JAR文件就会被舍弃。但是，如果类路径上的多个JAR文件有意包含在同一个包中的类型，那么它们必须都保留在类路径中（即作为一个在类路径上的未命名模块而非在模块路径上的自动模块）。

因为这些JAR文件不能用作自动模块，为了启用迁移，我们会将自动模块，视为建立在显式模块和仍然处于类路径上的代码（未命名模块）之间的桥梁：自动模块除了读取其他所有命名模块之外，还将读取未命名的模块。如果我们的应用程序的原始类路径中，包含了JAR文件org-baz-fiz.jar和org-baz-fuz.jar，那么我们将有图：

![module-pic-10](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-10.png)

如前所述，未命名模块导出其所有软件包，因此自动模块中的代码将能够访问所有从类路径加载的公用类型（即未命名模块中的所用公共类型）。

使用类路径中的类型的自动模块，不能将这些类型暴露给其他依赖于它的显式模块，因为显式模块无法声明对未命名模块的依赖关系。如果显式模块com.foo.app中的代码引用了一个自动模块com.foo.bar中的公共类型，并且自动模块com.foo.bar明确声明使用了仍在类路径上的一个JAR文件中的类型，则com.foo.app中的代码将无法访问该类路径上的类型，因为命名模块com.foo.bar不能依赖于未命名的模块。这可以通过将模块com.foo.app暂时视为自动模块来解决，以便其代码可以访问类路径中的类型，直到类路径上的相关JAR文件（未命名模块）可以被视为自动模块或转换为显式模块为止。

### 服务
利用服务接口和服务提供者的松散耦合是构建大型软件系统的强大工具。Java通过[java.util.ServiceLoader](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)类来支持服务，该类在运行时通过搜索类路径来定位服务提供者。对于在模块中定义的服务提供者，我们必须考虑如何在一组可观察模块中找到这些模块，解决它们的依赖性，并使提供者可以使用使用相应服务的代码。

假设，例如，我们的com.foo.app模块使用MySQL数据库，并且在具有声明的可观察模块中提供MySQL JDBC驱动程序
```java
module com.mysql.jdbc {
    requires java.sql;
    requires org.slf4j;
    exports com.mysql.jdbc;
}
```
其中org.slf4j是驱动程序（jdbc driver）使用的日志记录库，并且com.mysql.jdbc是包含java.sql.Driver这一服务接口的具体实现的包。（实际上并不需要导出驱动程序包，我们这样做是为了使代码清晰可见。）

为了让java.sql模块使用这个驱动程序， ServiceLoader类必须能够通过反射来实例化驱动程序类; 为了实现这一点，模块系统必须将驱动模块添加到模块图中并解决其依赖性，因此：

![module-pic-11](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-11.png)

为了实现这一点，模块系统必须能够识别所有使用服务的模块，然后从一组可观察模块中找到服务的提供者。

模块系统可以通过，扫描模包中的类文件并调用ServiceLoader::load方法，来识别对服务的使用，但是这样做不仅会很慢，而且并不可靠。模块使用特定服务应该是是模块的一个基本定义，所以为了效率和清晰度，我们在模块的声明中用一个uses子句来表示对服务的使用：
```java
module java.sql {
    requires public java.logging;
    requires public java.xml;
    exports java.sql;
    exports javax.sql;
    exports javax.transaction.xa;
    uses java.sql.Driver;
}
```
模块系统可以通过扫描META-INF/services资源条目来识别服务提供者，就像现在ServiceLoader类所做的那样。但是，模块提供特定服务借口的具体实现同样很重要，所以我们在模块的声明中用一个provides子句表示模块的提供者：
```java
module com.mysql.jdbc {
    requires java.sql;
    requires org.slf4j;
    exports com.mysql.jdbc;
    provides java.sql.Driver with com.mysql.jdbc.Driver;
}
```
现在，只要阅读这些模块的声明，就很容易看出来，其中一个m模块使用另一个提供的服务。

在模块声明中声明服务提供者和服务使用者的关系具有提高效率和代码清晰度的优势。这两种服务声明都可以在编译时进行解释，以确保服务提供者和服务使用者都可以访问服务接口（例如，java.sql.Driver）。服务提供者声明可以进一步解释，以确保提供者（例如，com.mysql.jdbc.Driver）确定实现其宣称的服务接口。服务使用的声明可以通过工具来提前编译，以确保在运行之前服务提供者能恰当的被编译。

出于迁移目的，如果定义自动模块的JAR文件包含META-INF/services资源条目，则将每个这样的条目视为该模块中的provides关键字的对应子句。自动模块被认为可以使用每一种可用的服务。

### 高级特性
本文档的其余部分讨论了高级特性，这些特性虽然很重要，但大多数开发人员可能并不感兴趣。

#### 反射
为了使模块图在运行时反射也可用，我们在java.lang.reflect包中定义了一个叫Module类，并在一个新的包java.lang.module中定义一些相关类型。Module类的一个实例在运行时代表一个单独的模块。每个类型都在一个模块中，因此每个Class对象都有一个关联的Module对象，该对象由新Class::getModule方法返回。

Module对象的基本操作是：
```java
package java.lang.reflect;

public final class Module {
    public String getName();
    public ModuleDescriptor getDescriptor();
    public ClassLoader getClassLoader();
    public boolean canRead(Module target);
    public boolean isExported(String packageName);
}
```
其中ModuleDescriptor是java.lang.module包中的类，它的实例表示模块描述符; getClassLoader方法返回模块的类加载器; canRead方法告诉模块是否可以读取目标模块; isExported方法告诉模块是否导出给定的包。

java.lang.reflect包并不是平台上唯一的反射工具。相似的工具也被添加到javax.lang.model模块，为了支持编译时的注释处理和文档生成工具。

#### 反射的可读性
框架是使用反射来加载，检查，并在运行时实例化的其他类的工具。Java SE平台本身的框架示例是服务加载器，资源包，动态代理和序列化，当然还有许多流行的外部框架库，用于数据库持久性，依赖注入和测试等多种用途。

鉴于需要在运行时发现类，框架必须能够访问其构造函数之一以实例化它。但事实表明，情况通常不会如此。

Java平台的[XML解析器](https://docs.oracle.com/javase/8/docs/api/javax/xml/stream/package-summary.html)，如果[加载和实例化](https://docs.oracle.com/javase/8/docs/api/javax/xml/stream/XMLInputFactory.html#newFactory--)由系统配置命名为javax.xml.stream.XMLInputFactory的[XMLInputFactory](https://docs.oracle.com/javase/8/docs/api/javax/xml/stream/XMLInputFactory.html)的服务实现，它就将通过ServiceLoader类，优先于所有服务提供者发现被发现。忽略异常处理和安全检查的代码大致如下所示：
```java
String providerName
    = System.getProperty("javax.xml.stream.XMLInputFactory");
if (providerName != null) {
    Class providerClass = Class.forName(providerName, false,
                                        Thread.getContextClassLoader());
    Object ob = providerClass.newInstance();
    return (XMLInputFactory)ob;
}
// Otherwise use ServiceLoader
...
```
在模块化设置中，只要包含的服务提供者的类为上下文类加载器所知，Class::forName就仍将继续工作。但是，通过反射的方式调用服务提供者的newInstance方法将不起作用：服务提供者可能会从类路径加载，在这种情况下，它将位于未命名的模块中，或者它可能位于某个已命名的模块中，无论是哪种情况，我们的框架本身都在java.xml模块中。该模块仅依赖于基本模块，因此也只读取基本模块，此框架将无法访问任何其他模块中的服务提供者类。

为了使框架可以访问服务提供者类，我们需要使框架的模块可以读取服务提供者的模块。我们可以要求每个框架在运行时都明确地，将必要的可读性边缘添加到模块图中，就像本文档的早期版本一样，但是经验表明这种方法很麻烦并且会导致迁移的障碍。

因此，我们需要修改反射API，我们假设任何反射其他类型的代码，都位于一个可以访问到被反射类型所在模块的模块中。这使得上面的例子和其他类似的代码可以毫无改变地工作。这种方法不会削弱强封装：公开类型仍然必须位于导出的包中，以便从其所在模块外部进行访问，无论是通过编译代码还是通过反射。

> 事实上，在这种需要发射其他模块的情况下，如果我们只想要反射共有类型，那只要在模块中导出相应的包就可以；但如果我们想要通过setAccessible(true)方法来反射私有类型是，必须在模块声明时添加open关键字或者opens子句，来使模块成为一个开放模块或者开放模块中的软件包，使反射对私有类型可见，否则就会在运行报出Accessing Error。这一点原作者并未提及，我个人认为这种设计很好的保证了模块的强封装的特性

#### 类加载器
每个类型都在一个模块中，并且在运行时每个模块都有一个类加载器，但是类加载器是否只加载一个模块呢？事实上，模块系统对模块和类加载器之间的关系几乎没有限制。一个类加载器可以从一个模块或多个模块加载类型，只要这些模块不相互干扰，并且只有一个加载器加载了特定模块中的所有类型。

这种灵活性对于兼容性至关重要，因为它允许我们保留平台现有的内置类加载器的层次结构。引导类加载器（bootstrap classloader）和扩展类加载器（extension classloader）仍然存在，并用于从平台模块加载类型。应用程序类加载器（application classloader）仍然存在，用于从工件的模块路径中找到加载类型。

这种灵活性还会使模块化现有应用程序变得更加容易，这些应用程序已经构建了复杂的层次结构甚至自定义类加载器，因次我们可以将这些加载器升级到模块中的加载类型，而无需更改其委托模式。

> 事实上，Java9的classloader是有改变的，这一部分我以后会单独写一篇文章来总结。如现在想了解请参考[此链接](http://www.cnblogs.com/IcanFixIt/p/7131676.html)中的扩展机制部分

#### 未命名模块与类加载器
我们之前了解到，如果某个类型未在命名模块中定义，那么它将被视为未命名模块的成员，但与未命名模块相关的是哪个类加载器呢？

事实证明，每个类加载器都有自己独特的未命名模块，它是由新ClassLoader::getUnnamedModule方法返回的。如果一个类加载器加载了一个没有在命名模块中定义的类型，那么该类型就被认为是在该加载器的未命名模块中。例如，Class类中的getModule方法将返回其加载器的未命名模块。应用加载器（application classloader）的未命名模块，会从类路径中加载不在任何模块下定义的包中的类型。

#### 层
模块系统并不指定模块和类加载器之间的关系，但为了加载特定的类型，它必须以某种方式能够找到合适的加载器。因此，在运行时，模块图的实例化会生成一个层（layer），这个层将图中的每个模块映射到负责加载该模块中的类型的唯一类加载器上。与可被发现的模块相反，引导层（boot layer）是由Java虚拟机在启动时通过解析应用程序的初始模块所创建的。

大多数应用程序以及几乎当前所有的应用程序都不会使用引导层以外的层。然而，多个层可用于带有插件的复杂应用程序或容器体系结构（如应用程序服务器，IDE和测试框架）。这样的应用程序可以使用动态的类加载和模块系统的反射API，来加载和运行由一个或多个模块组成的应用程序。然而，这通常需要添加两种额外的灵活性：

1. 使用模块的应用程序可能需要不同版本的已存在的模块。例如，Java EE Web应用程序可能需要java.xml.ws模块中的不同版本的JAX-WS，而不是内置于运行时环境的版本。

2. 使用模块的应用程序可能需要已经被发现的服务提供者以外的服务提供者。应用程序甚至可能嵌入自己的首选服务提供者。Web应用程序，可能包含一个它所期望的[Woodstox XML解析器](https://github.com/FasterXML/woodstox)版本，在这种情况下，ServiceLoader类应优先返回它需要的服务提供者而不是任何其他的服务提供者。

与可被发现的模块的环境相反，一个容器应用程序可以为一个使用模块的应用程序的初始模块，在其已有的层上创建一个新的层。这样的环境可以包含可升级平台模块的替代版本以及其他已存在于较低层的非平台模块，解析器优先解析这些备用模块。这种环境也可以在那些已经在较低层被发现的服务提供者之外发现不同的服务提供者; ServiceLoader类将在较低层返回服务提供者之前去加载这些服务提供者。

层可以堆叠：我们可以在引导层之上构建新层，然后再在其上创建另一个层。作为正常解析过程的结果，所指定的层中的模块可以读取该层中或下层中的模块。因此，层的模块图可以通过引用包括其下的每个层的模块图来表示。

> 上面这一节翻译的不太好，事实上我也不太理解这一节的内容。以后会仔细研究一下，目前只知道现在JDK中已经有了ModuleLayer类，可以通过Module.getLayer()获得。

#### 限制性导出
偶尔有必要安排某些类型在一组模块中可访问，但其他模块无法访问。

在标准JDK的java.sql模块和 java.xml模块的代码实现中，使用了java.base模块中的sun.reflect包下定义的类型 。为了让代码访问sun.reflect包中的类型，我们可以简单地从java.base模块中导出该包：
```java
module java.base {
    ...
    exports sun.reflect;
}
```
然而，这将使得每个sun.reflect包中的类型对所有模块都可见（因为每个模块都读取java.base），而这是不合理的，因为该包中的一些类定义了有特权的，安全敏感的方法。

因此，我们扩展了模块声明以允许将包导出到一个或多个特定命名的模块，而不被其他模块可见。java.base模块的声明实际上只将sun.reflect包导出到特定的一组JDK模块：
```java
module java.base {
    ...
    exports sun.reflect to
        java.corba,
        java.logging,
        java.sql,
        java.sql.rowset,
        jdk.scripting.nashorn;
}
```
通过在模块图中添加另一种类型的边缘（此处为彩色金色），可以将这些限制性的导出显示在模块图中：

![module-pic-12](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/java%20module%20system/example-module-12.png)

前面提到的可访问性规则如下所述：如果两种类型S并且T在不同模块中定义，并且T是public，则代码S可以在以下情况下访问T：

1. S的模块读取T的模块，和
2. T的模块直接导出T的包到S的模块，亦或是导出到所有模块。

我们为用于反射的Module类提供了一种方法，来确定是否将包导出到特定模块，而非所有模块：
```java
public final class Module {
    ...
    public boolean isExported(String packageName, Module target);
}
```
限制性的导出可能会意外地将内部的类型到处到预期之外的模块，因此我们必须小心使用它们。例如命名一个名为java.corba的模块以访问sun.reflect包中的类型。为了防止这种情况，我们可以在构建时分析相关模块，并在每个模块的描述符中记录允许依赖它的模块内容的哈希值，并使用限制性导出。在分析期间，我们需要验证，那些使用限制性导出到其他命名模块的模块，其模块内容的哈希值，与引用该模块的模块中记录的该模块的哈希值匹配。只要声明和使用限制性的导出的模块以这种方式绑定在一起，限制性的导出就可以安全地在不受信任的环境中使用。

### 总结
这里描述了模块系统的很多方面，但大多数开发人员只需要使用其中的一部分。我们期望在未来几年内，大多数Java开发人员都会熟悉模块声明，模块化JAR文件，模块路径，可读性，可访问性，未命名模块，自动模块和模块化服务等基本概念。相比之下，反射可读性，层和限制性的导出等更高级功能可能被使用的可能性比较小。