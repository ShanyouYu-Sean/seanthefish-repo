---
layout: post
title: 用gradle构建Java 9模块化系统快速指南
date: 2018-04-11 16:28:00
tags: 
- Java 9
- Java module
- gradle
categories:
- Java 9
- Java module
- gradle
---

**本文是gradle官方的构建Java 9模块化系统的入门指南的翻译。（[原文地址](https://guides.gradle.org/building-java-9-modules/?_ga=2.174270880.30455902.1522735364-1287880600.1513842256)）**

Java 9最令人兴奋的特性之一是它支持开发和部署Java模块化系统。在本指南中，您将了解到如何用gradle实现模块化功能，你所要做的事情：

1. 为您的Java库生成Java 9模块。
2. 使用Java 9模块作为您的依赖。
3. 在Java 9模块中使用Java的ServiceLoader模式。
4. 使用Java 9模块运行应用程序。
5. 使用一个插件来更简单地完成以上功能。

虽然Gradle 4.6版尚未对Java 9模块提供一流的支持，本指南仍将向您介绍如何在支持完成之前对Java 9进行试验性的工作。

### 你需要什么

1. 大约41分钟
2. 一个文本编辑器
3. 一个命令提示符
4. Java开发工具包（JDK），版本1.9（版本174）+

### 了解示例项目

本指南逐步说明，如何将不使用任何Java 9功能的Java应用程序，转换为完全模块化的Java 9应用程序。原始版本的应用程序的源代码位于src/0-original目录中。它是由六个子项目组成的gradle多项目程序：

1. fairy - java应用程序storyteller的入口点。
2. tale - 公共Tale接口的库。
3. formula - 帮助改造Tale接口的库。
4. actors - fairy tale中所有characters的库。
5. pigs - 代表三个小猪的Tale实例的库。
6. bears - 代表金发姑娘和三只熊的Tale实例的库。

六个项目之间依赖关系的项目层次结构如下图所示：

![gradle-java9-1](https://raw.githubusercontent.com/ShanyouYu-Sean/blog-images/master/gradle_guide_java9/project-graph.png)

如果你对api和implementation不熟悉，请参阅在Gradle 3.5中加入的[Java Library Plugin](https://docs.gradle.org/current/userguide/java_library_plugin.html?_ga=2.204456536.934686470.1523424544-1287880600.1513842256)

你可以克隆源代码来查看原始项目的输出：

```bash
$ git clone https://github.com/gradle-guides/building-java-9-modules.git
$ cd building-java-9-modules/src/0-original
$ ./gradlew run

> Task :fairy:run
Once upon a time, there lived the big bad wolf, and the 3 little pigs.

<... elided ...>

Goldilocks ran out of the house at top speed to escape the 3 bears.
And they all lived happily ever after.


BUILD SUCCESSFUL
```

在开始修改此项目以使其使用Java 9模块之前，您需要了解项目结构的两个重要细节，就是，它使用 ServiceLoader API来在运行时加载fairy tale，并且它包含一个测试类来显示在使用Java 9之前，软件的模块封装是多么的脆弱。

#### ServiceLoader的用法

Java 1.6引入了一种简单的机制，用于在运行时将一些接口（“Service”）的一组实现绑定到一个消费类。有关该特性的[Oracle教程](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html)有点冗长，下面是它在示例应用程序中的使用方式：

```java
public static void main(String[] args) {
        ServiceLoader<Tale> loader = ServiceLoader.load(Tale.class);
        if (!loader.iterator().hasNext()) {
            System.out.println("Alas, I have no tales to tell!");
        }
        for (Tale tale : loader) {
            tale.tell();
        }
    }
```

JVM中的类加载器用ServiceLoader来找出，类路径上META-INF/services文件夹中的，名为org.gradle.fairy.tale.Tale的指定Tale类。

ears/src/main/resources/META-INF/services/org.gradle.fairy.tale.Tale:

```java
org.gradle.fairy.tale.bears.GoldilocksAndTheThreeBears
```

pigs/src/main/resources/META-INF/services/org.gradle.fairy.tale.Tale:

```java
org.gradle.fairy.tale.pigs.ThreeLittlePigs
```

在运行时加载这些实例,会使StoryTeller类以松耦合的方式连接到实现该Tale接口的两个库。你可以在应用程序的build.gradle文件的dependencies块中看到它。

fairy/build.gradle:

```gradle
dependencies {
    implementation project(':tale')

    runtimeOnly project(':pigs')
    runtimeOnly project(':bears')
}
```

注释掉以runtimeOnly开头的两行，并注意Gradle run任务的输出是如何改变的：

```bash
$ ./gradlew run

> Task :fairy:run
Alas, I have no tales to tell!


BUILD SUCCESSFUL
```

#### 模块化测试讨论

在最初的项目中，有一个测试类，展现了在Java 9之前的Java版本中，未实施模块化的一些问题。
formula/src/test/java/org/gradle/fairy/tale/formula/ModularityTest.java：

```java
    @Test
    public void canReachActor() {
        Actor actor = Imagination.createActor("Sean Connery");
        assertEquals("Sean Connery", actor.toString());
    }

    @Test
    public void canDynamicallyReachDefaultActor() throws Exception {
        Class clazz = ModularityTest
            .class.getClassLoader()
            .loadClass("org.gradle.actors.impl.DefaultActor");
        Actor actor = (Actor) clazz.getConstructor(String.class)
            .newInstance("Kevin Costner");
        assertEquals("Kevin Costner", actor.toString());
    }

    @Test
    public void canReachDefaultActor() {
        Actor actor = new org.gradle.actors.impl.DefaultActor("Kevin Costner");
        assertEquals("Kevin Costner", actor.toString());
    }

    /*
    @Test
    public void canReachGuavaClasses() {
        // This line would throw a compiler error because gradle has kept the implementation dependency "guava"
        // from leaking into the formula project.
        Set<String> strings = com.google.common.collect.ImmutableSet.of("Hello", "Goodbye");
        assertTrue(strings.contains("Hello"));
        assertTrue(strings.contains("Goodbye"));
    }
    */
```

这个类的四个测试有不同的目的：

1. canReachActor - 通过调用actors项目的公共api来表明formula项目的访问权限。
2. canDynamicallyReachDefaultActor - 尝试在运行时使用反射来加载actors子项目的私有类。这在Java 9之前是可能的，因为类路径会将应用程序的所有的实现细节暴露给其他所有的应用。
3. canReachDefaultActor - 尝试直接使用actors子项目的私有类。这只在Java 9之前可行，因为actors子项目的私有实现细节与该子项目的公共API构建在相同的位置。所以，它们在编译时和运行时都可用。
4. canReachGuavaClasses - 尝试使用actors子项目所依赖的类。需要注意的是，从Gradle 3.4开始，使用implementation关键字的依赖关系不包含在Java项目的消费者的编译类路径（compileClasspath）中。因此，这个测试被注释掉了，因为它不能用Gradle 3.4或更新的版本编译。

遵循本指南，你会看到Java 9将对于模块细节的访问权限变得更加紧密，并导致测试，canDynamicallyReachDefaultActor和canReachDefaultActor在运行时或编译时失败。

你可以运行Gradle的check任务，来认证这三个测试是否通过了0-original项目（尽管其中两个测试 打破了良好的模块化设计。）

```bash
$ ./gradlew check

BUILD SUCCESSFUL
```

您可以在[建构扫描](https://scans.gradle.com/s/l76lgbuizu4pm/tests/byProject?toggled=W1sxXSxbMSwwXSxbMSwwLDBdLFsxLDAsMCwwXV0)中查看此次调用gradle task任务的结果。

### 第1步 - 为单个子项目生成Java 9模块

如果您还不熟悉Java 9模块系统，请阅读：

1. [模块系统快速入门指南](http://openjdk.java.net/projects/jigsaw/quick-start) / [(原文翻译)](http://seanthefish.com/2018/03/29/java9-quick-guide/)
2. [模块系统综述](http://openjdk.java.net/projects/jigsaw/spec/sotms/) / [(原文翻译)](http://seanthefish.com/2018/03/29/module-system/)

本指南假定您已经熟悉以下概念：

1. 模块路径
2. 自动模块
3. module-info.java文件的基本语法

Java 9中模块系统的一个很好的功能就是可以以自下而上的方式将项目的所有代码库转换为Java 9模块。无论是从类路径还是模块路径中，我们都可以获取Java 9模块化jar包，所以我们可以在多项目构建中，转换单个叶节点以生成Java 9模块，但是在编译时使用该模块化的jar包或在类路径上使用该模块化jar包来运行该节点的输出。

将java-library项目转换为Java 9模块时，应该对项目进行五项更改:

1. 添加一个module-info.java文件来描述模块。
2. 修改compileJava任务以生成模块。
3. 修改compileTestJava任务以在本地修改模块。
4. 修改test任务以使用本地更改的模块。
5. （可选）在所有其他项目的清单条目（MANIFEST.MF）中添加Automatic-Module-Name属性。

我们建议为组成应用程序的所有项在META-INF/MANIFEST.MF文件中主动添加目Automatic-Module-Name清单条目 。提供Automatic-Module-Name允许库作者为未来预留模块名称，而不必将库转换为模块。这确保了库的消费者现在就可以知道模块名称将来会是什么。

在下面的小节中将介绍这些变化，并讨论为什么要进行变更。您还可以通过浏览src/1-single-module库中的示例项目来查看这些更改的结果。

我们做出以下五项改变的目标是让actors项目生成一个Java 9模块。前四项变更需要一起完成，第五项（可选）变更可以独立完成。

> 提醒一下，从这一点开始，所有的构建都需要在Java 9上运行

#### 添加一个module-info.java文件来描述模块。

将module-info.java文件添加到项目的actors/src/main/java目录。

actors/src/main/java/module-info.java:

```java
module org.gradle.actors {
    exports org.gradle.actors;
    requires guava;
}
```

该文件声明org.gradle.actors模块导出org.gradle.actors包（但不org.gradle.actors.impl包），并需要guava模块。Guava jar文件还不是Java 9模块，所以当你需要它们时，你必须使用JVM通过jar文件的文件名来推断生成的自动模块的名称。对于guava来说，jar文件的名称是[guava-22.0.jar](http://central.maven.org/maven2/com/google/guava/guava/22.0/)，因此根据[自动模块名称的规则](http://openjdk.java.net/projects/jigsaw/spec/sotms/#automatic-modules)，您需要的模块叫guava。

#### 修改compileJava任务以生成模块。

在actors子项目的build.gradle文件中添加以下内容。

actors/build.gradle：

```gradle
ext.moduleName = 'org.gradle.actors' //(1)

compileJava {
    inputs.property("moduleName", moduleName)
    doFirst {
        options.compilerArgs = [
            '--module-path', classpath.asPath,
        ]
        classpath = files()  //(2)
    }
}
```

1. 为模块名称定义一个变量，该变量允许您稍后为其他模块重复使用相同的代码，而无需对其进行更改。
2. 通过创建一个空文件集合来清除classpath属性。

编译Java 9模块时，您想使用--module-path而不是 --classpath读取您的依赖关系。因此，在该doFirst块中，您将清除该任务的classpath属性并添加一个编译器参数。

> --module-path被设置为原来的值classpath。这样做是因为，classpath已经有你所依赖的库的所有jar包和类输出目录。

> 在doFirst代码块内而不是在compileJava任务中修改options.compilerArgs参数的原因是，在执行这个任务时，你只需要重构compileClasspath（编译时的类路径）的配置。

#### 修改compileJava任务以生成模块。

Java 9模块系统的一个稍微混淆的方面是如何对Java 9模块内的代码运行单元测试。推荐的方法是在测试过程中“修补”模块。修补模块意味着向组成模块的包添加额外的类。在运行测试所需要的修补模块步骤中，您将使用相同的包来把测试类添加到模块中，以便测试类可以访问被测模块中的所有其他模块。

将以下内容添加到您build.gradle文件中，来实现在编译时对org.gradle.actors模块的修补。

actors/build.gradle：

```gradle
compileTestJava {
    inputs.property("moduleName", moduleName)
    doFirst {
        options.compilerArgs = [
            '--module-path', classpath.asPath, \\(1)
            '--add-modules', 'junit',  \\(2)
            '--add-reads', "$moduleName=junit", \\(3)
            '--patch-module', "$moduleName=" + files(sourceSets.test.java.srcDirs).asPath, \\(4)
        ]
        classpath = files()
    }
}
```

1. 用--module-path参数来作为classpath属性的默认值。
2. 显式地将junit自动模块添加为可观察模块。
3. 声明junit模块读取org.gradle.actors模块。
4. 将测试源文件添加到org.gradle.actors模块。

这些选项的添加将会导致测试源的输出目录中的生成类文件包含合适的元数据来修补org.gradle.actors模块，这些类文件会在接下来的更改中被使用。

#### 修改test任务以使用本地更改的模块。

运行测试时，我们必须配置运行测试的JVM使其发现我们的模块，并修补org.gradle.actors模块来引入测试类。

将以下内容添加到actors项目中的build.gradle文件中。

actors/build.gradle：

```gradle
test {
    inputs.property("moduleName", moduleName)
    doFirst {
        jvmArgs = [
            '--module-path', classpath.asPath, \\(1)
            '--add-modules', 'ALL-MODULE-PATH', \\(2)
            '--add-reads', "$moduleName=junit", \\(3)
            '--patch-module', "$moduleName=" + files(sourceSets.test.java.outputDir).asPath, \\(4)
        ]
        classpath = files()
    }
}
```

1. 这是测试运行时的classpath属性的默认值。
2. 使用特殊的ALL-MODULE-PATH，因为运行测试的JVM的main class不是Java 9模块的一部分。它是Gradle的测试运行器，因此它没有声明它需要使用的模块。该参数使模块路径中的所有模块都可以被测试类访问。
3. 声明junit读取org.gradle.actors模块。
4. 将测试类添加到org.gradle.actors模块。

#### （可选）在所有其他项目的清单条目（MANIFEST.MF）中添加Automatic-Module-Name属性。

为了向后兼容，Java 9的模块系统允许非模块化的jar文件出现在模块路径中。默认情况下，这些jar文件将被转换为自动模块，其名称基于jar文件的文件名。但是这会导致一些冗杂。许多jar文件名已经创建，但没有任何规则能保证这个jar的名字是唯一的。所以当负责维护一些常用jar的开发人员在将该jar转换为Java 9模块时，他们更希望会选择一个新的模块名称，而非由自动模块转换所自动生成的名称。

例如，你现在可以在module-info.java文件中通过requires guava子句指定模块，但稍后负责该项目的开发人员决定为其模块命名com.google.guava。现在，任何指定requires guava或任何依赖此模块的用户，都必须改变它们依赖的模块为requires com.google.guava，如此才能使用这些新模块，因为Java 9只允许模块路径上的模块包含特定的包。

因此，整个情况可能会变得非常混乱。这就是为什么[Stephen Colebourne认为](http://blog.joda.org/2017/05/java-se-9-jpms-automatic-modules.html)我们应该立即开始更新我们发布到公共存储库的所有jar,（至少要在jar的清单中指定Automatic-Module-Name属性），而且也不要发布任何未指定Automatic-Module-Name属性并且包含需要自动模块的模块的工件。

因此，在每个子项目的build.gradle文件中指定一个moduleName变量。例如：

fairy/build.gradle:

```gradle
ext.moduleName = 'org.gradle.fairy.app'
```

另外，在顶层build.gradle文件中的afterEvaluate代码块中的jar任务中添加manifest属性。

```gradle
jar {
        inputs.property("moduleName", moduleName)
        manifest {
            attributes('Automatic-Module-Name': moduleName)
        }
    }
```

现在，当您将jar文件发布到像Maven Central这样的工件存储库时，你发布的jar包的文件名就不再重要了; 你一定会（通过Automatic-Module-Name）得到你想要的Java 9模块名称。

> 在下一步中，您将摆脱这些清单属性，因为您已将每个子项目都转换为适当的Java 9模块。

#### 第1步 - 总结

这第一步是最复杂的，但现在您已经将第一个java-library项目转换为Java 9模块。所有其他子项目都在类路径上使用该模块。我们并没有真正解决在[模块化测试讨论](#模块化测试讨论)中演示的任何模块化违规问题 ，但是我们不必中断项目，将其逻辑体系结构转换为适当的Java 9模块。接下来，我们将根目录的build.gradle项目中来集中gradle更改，并将其应用于所有子项目中。

### 第2步 - 为所有子项目生成Java 9模块

这一步的目标是让我们的Gradle构建中的所有子项目都生成Java 9模块，并将它们的依赖作为Java 9模块使用。由于您已将actors子项目中的build.gradle文件中的moduleName变量声明与其他改变分离，因此只需将该文件中的moduleName声明以后的所有内容剪切并粘贴到根目录build.gradle文件中的afterEvaluate代码块中即可。

build.gradle：
```gradle
subprojects {
    afterEvaluate {
        repositories {
            jcenter()
        }

        compileJava {
            inputs.property("moduleName", moduleName)
            doFirst {
                options.compilerArgs = [
                    '--module-path', classpath.asPath,
                ]
                classpath = files()
            }
        }

        compileTestJava {
            inputs.property("moduleName", moduleName)
            doFirst {
                options.compilerArgs = [
                    '--module-path', classpath.asPath,
                    '--add-modules', 'junit',
                    '--add-reads', "$moduleName=junit",
                    '--patch-module', "$moduleName=" + files(sourceSets.test.java.srcDirs).asPath,
                ]
                classpath = files()
            }
        }

        test {
            inputs.property("moduleName", moduleName)
            doFirst {
                jvmArgs = [
                    '--module-path', classpath.asPath,
                    '--add-modules', 'ALL-MODULE-PATH',
                    '--add-reads', "$moduleName=junit",
                    '--patch-module', "$moduleName=" + files(sourceSets.test.java.outputDir).asPath,
                ]
                classpath = files()
            }
        }
    }
}
```

> 如果你做了步骤1的最后一步，则应在粘贴之前删除jar代码块。

如果您还没有为每个子项目添加moduleName变量声明，那么现在应该这样做。例如：

pigs/build.gradle：

```gradle
ext.moduleName = 'org.gradle.fairy.tale.pigs'
```

您还需要为每个子项目添加一个module-info.java文件。例如：

bears/src/main/java/module-info.java:

```gradle
module org.gradle.fairy.tale.bears {
    requires org.gradle.actors;
    requires transitive org.gradle.fairy.tale;
    requires org.gradle.fairy.tale.formula;

    exports org.gradle.fairy.tale.bears;
}
```

> 您还需要为pigs， formula，fairy，和tale子项目添加这些module-info.java文件。最终结果应该看起来像src/2-all-modules中的代码。

现在，Gradle的test任务将无法编译，除非您已将canReachDefaultActor测试注释掉。另外，canDynamicallyReachDefaultActor测试将在测试运行时失败，除非你添加@Ignore注释。

```bash
$ ./gradlew test

> Task :formula:test

org.gradle.fairy.tale.formula.ModularityTest > canDynamicallyReachDefaultActor FAILED
    java.lang.IllegalAccessException at ModularityTest.java:28

2 tests completed, 1 failed


FAILURE: Build failed with an exception.

* What went wrong:
Execution failed for task ':formula:test'.
> There were failing tests. See the report at: <link-to-report>

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.

BUILD FAILED
```

如果你注释掉canReachDefaultActor测试并为canDynamicallyReachDefaultActor添加@Ignore注解，剩下的测试应该通过，你可以在src/2-all-modules中得到完整的代码。

#### 第2步 - 总结

到目前为止，你已经在使用Java 9模块来编译和运行所有六个子项目和测试。这些子项目被适当地封装，并且没有一个包下的测试可以看到这个包所依赖的任何实现细节。但是，[Gradle的应用程序插件](https://docs.gradle.org/4.6/userguide/application_plugin.html?_ga=2.21328251.1136812865.1523857970-1287880600.1513842256)的一些特性依赖于类路径来加载和编译类，而不是模块路径。

此外，Java 9增加了一种更方便的方式来使用ServiceLoader功能。您将在第3步中了解如何处理这些问题。

### 第3步 - 在run和assemble任务中使用Java 9模块

现在所有子项目都已经转化为为Java 9模块，现在该学习fairy项目中的main class（org.gradle.fairy.app.StoryTeller）在运行时是如何使用这些模块的 。

运行本指南中介绍的应用程序有两种方式。首先是使用由[应用程序插件](https://docs.gradle.org/4.6/userguide/application_plugin.html?_ga=2.28841596.1136812865.1523857970-1287880600.1513842256)添加的Gradle run任务 。

```bash
$ ./gradlew run

> Task :fairy:run
Once upon a time, there lived the big bad wolf, and the 3 little pigs.

<... elided ...>

Goldilocks ran out of the house at top speed to escape the 3 bears.
And they all lived happily ever after.


BUILD SUCCESSFUL
```

另一种方法是使用Gradle的assemble任务来分别打包各个应用程序，然后提取到某个目录并在那里运行。

```bash
$ ./gradlew assemble

BUILD SUCCESSFUL
$ cp fairy/build/distributions/fair.tar /tmp
$ cd /tmp
$ tar xvf fairy.tar
x fairy/
x fairy/lib/
x fairy/lib/fairy.jar
x fairy/lib/pigs.jar
x fairy/lib/bears.jar
x fairy/lib/formula.jar
x fairy/lib/tale.jar
x fairy/lib/actors.jar
x fairy/lib/guava-22.0.jar
x fairy/lib/jsr305-1.3.9.jar
x fairy/lib/error_prone_annotations-2.0.18.jar
x fairy/lib/j2objc-annotations-1.1.jar
x fairy/lib/animal-sniffer-annotations-1.14.jar
x fairy/bin/
x fairy/bin/fairy
x fairy/bin/fairy.bat
$ ./bin/fairy
Once upon a time, there lived the big bad wolf, and the 3 little pigs.

<... elided ...>

Goldilocks ran out of the house at top speed to escape the 3 bears.
And they all lived happily ever after.
```

在步骤2之后，这两种机制都依赖于出现在类路径上的模块。这样也就会跳过了Java 9模块系统的模块化特性。在这一步中，您将：

1. 修改run任务以使用模块。
2. 修改startScript任务来使*nix和Windows系统使用模块。
3. 将ServiceLoader机制更新为Java 9语法。

一旦进行了更改1和2，运行该程序的两种机制都应该能直接运行，因此请随时再次运行这些命令以确认您已正确实施每项更改。

您依旧可以在资源代码库中的src/3-application目录中看到所有更改 。

### 修改run任务以使用模块

要在run任务中使用Java 9模块，你需要将以下内容添加到fairy项目中的build.gradle文件中。

fairy/build.gradle

```gradle
mainClassName = "$moduleName/org.gradle.fairy.app.StoryTeller" //(1)

run {
    inputs.property("moduleName", moduleName)
    doFirst {
        jvmArgs = [
            '--module-path', classpath.asPath,
            '--module', mainClassName //(2)
        ]
        classpath = files()
    }
}
```

1. 设置mainClassName属性包含moduleName。
2. 明确告诉Java 9使用该模块。

#### 修改startScript任务来使*nix和Windows系统使用模块。

在fairy/build/distributions目录中创建的tar和zip文件会包含启动脚本 ，这些脚本允许在所有支持的操作系统上，以可预测的方式启动JVM。

要修改已生成的startScripts，请将以下内容添加到您的fairy/build.gradle文件中：

fairy/build.gradle：

```gradle
startScripts {
    inputs.property("moduleName", moduleName)
    doFirst {
        classpath = files()
        defaultJvmOpts = [
            '--module-path', 'APP_HOME_LIBS',  \\(1)
            '--module', mainClassName
        ]
    }
    doLast{
        def bashFile = new File(outputDir, applicationName)
        String bashContent = bashFile.text
        bashFile.text = bashContent.replaceFirst('APP_HOME_LIBS', Matcher.quoteReplacement('$APP_HOME/lib'))

        def batFile = new File(outputDir, applicationName + ".bat")
        String batContent = batFile.text
        batFile.text = batContent.replaceFirst('APP_HOME_LIBS', Matcher.quoteReplacement('%APP_HOME%\\lib'))
    }
}
```

1. 将模块路径设置为独立于平台的占位符值，稍后将以特定于平台的方式替换*nix shell脚本和Windows .bat文件。

### 将ServiceLoader机制更新为Java 9语法。

Java 9模块系统引入了一种更好的方式来指定哪些模块为ServiceLoader机制提供服务的实现。首先，从两个目录bears/src/main和pigs/src/main中，删除resources文件夹，因为新机制不需要META-INF/services文件。

然后，调整每个项目的module-info.java文件。

fairy/src/main/java/module-info.java:

```java
module org.gradle.fairy.app {
    requires org.gradle.fairy.tale;
    uses org.gradle.fairy.tale.Tale;
}
```

bears/src/main/java/module-info.java:

```java
module org.gradle.fairy.tale.bears {
    requires org.gradle.actors;
    requires transitive org.gradle.fairy.tale;
    requires org.gradle.fairy.tale.formula;

    provides org.gradle.fairy.tale.Tale
        with org.gradle.fairy.tale.bears.GoldilocksAndTheThreeBears;
}
```

pigs/src/main/java/module-info.java:

```java
module org.gradle.fairy.tale.pigs {
    requires org.gradle.actors;
    requires transitive org.gradle.fairy.tale;
    requires org.gradle.fairy.tale.formula;

    provides org.gradle.fairy.tale.Tale
            with org.gradle.fairy.tale.pigs.ThreeLittlePigs;
}
```

由于fairy项目中的module-info.java声明它使用org.gradle.fairy.tale.Tale服务，所以该模块中的ServiceLoader实例将有权访问，所有由Java 9模块声明的在运行时提供的org.gradle.fairy.tale.Tale服务实现。

### 第4步 - 使用experimental-jigsaw插件来做与我们之间所做的同样的事情

虽然Gradle尚未将Java 9模块构建作为Java插件的一级特性加以支持，实验性插件也可让您在项目中尝试使用Java 9模块。

org.gradle.java.experimental-jigsaw插件只是一个简便的机制，可以在一个步骤中，提供本指南步骤1至3中的所有更改。它可能适用于您的项目，但您应该考虑到它是实验性的，不适合生产版本。

以下是如何使用插件：

actors/build.gradle：

```gradle
plugins {
    id 'java-library'
    id 'org.gradle.java.experimental-jigsaw' version '0.1.1'  \\(1)
}
```

1. 使用插件

actors/build.gradle：

```gradle
javaModule.name = 'org.gradle.actors'  \\(1)
```

1. 使用新javaModule.name设置来指定模块名称。

### 总结

此时，您的应用程序正在利用Java 9模块系统的大部分功能。本指南向您展示了如何修改常规插件java-library和application所添加的任务，来方便你使用Java 9模块进行工作。未来，Gradle团队将为模块系统添加一流的支持，但您现在就已经可以开始尝试！
