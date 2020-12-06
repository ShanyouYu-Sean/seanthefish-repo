---
layout: post
title: tomcat classloader源码详解
date: 2020-11-15 12:14:47
tags: 
- jvm
categories:
- jvm
---

### 源码

```java
public void init() throws Exception {

        // 初始化，后文会说这个方法
        initClassLoaders();

        // 设置线程类加载器
        Thread.currentThread().setContextClassLoader(catalinaLoader);

        SecurityClassLoad.securityClassLoad(catalinaLoader);

        // Load our startup class and call its process() method
        // 通过之前初始化的classloader反射实例化Catalina类的实例
        Class<?> startupClass =
            catalinaLoader.loadClass
            ("org.apache.catalina.startup.Catalina");
        Object startupInstance = startupClass.newInstance();
        // Set the shared extensions class loader
        if (log.isDebugEnabled())
            log.debug("Setting startup class properties");
        String methodName = "setParentClassLoader";
        Class<?> paramTypes[] = new Class[1];
        paramTypes[0] = Class.forName("java.lang.ClassLoader");
        Object paramValues[] = new Object[1];
        paramValues[0] = sharedLoader;
        Method method =
            startupInstance.getClass().getMethod(methodName, paramTypes);

        // Catalina#setParentClassLoader为sharedLoader
        // 因为sharedLoader默认就是commonloader
        // 当然Catalina默认也是commonloader
        // 为啥不直接设置成commonclassloader呢？
        // 因为webappclassloader会把自己的parentclassloader设置成Catalina的ParentClassLoader
        // 注意是Catalina对象的爹是sharedLoader
        // catalinaLoader的爹仍旧是commonloader
        // 为啥这样？Catalina对象由catalinaLoader加载，Catalina对象的爹是sharedLoader，catalinaLoader的爹却是commonloader？ 求解
        method.invoke(startupInstance, paramValues);

        catalinaDaemon = startupInstance;

    }
```

initClassLoader()方法：

```java
private void initClassLoaders() {
        try {
            // 默认情况下common.loader是有值的
            // common.loader="${catalina.base}/lib","${catalina.base}/lib/*.jar","${catalina.home}/lib","${catalina.home}/lib/*.jar"
            commonLoader = createClassLoader("common", null);
            if( commonLoader == null ) {
                // no config file, default to this loader - we might be in a 'single' env.
                commonLoader = this.getClass().getClassLoader();
            }
            // catalinaLoader和sharedLoader实际上有共同的父类加载器commonLoader
            // 而如果server.loader, shared.loader为空
            // 那么此时的catalinaLoader,sharedLoader其实是同一个ClassLoader ———— commonclassloader
            catalinaLoader = createClassLoader("server", commonLoader);
            sharedLoader = createClassLoader("shared", commonLoader);
        } catch (Throwable t) {
            log.error("Class loader creation threw exception", t);
            System.exit(1);
        }
}
```

createClassLoader()方法：

```java
private ClassLoader createClassLoader(String name, ClassLoader parent) throws Exception {

    String value = CatalinaProperties.getProperty(name + ".loader");
    // 判断如果catalina.properties中没有配置对应的loader属性的话，直接返回父加载器
    // 而默认情况下，server.loader, shared.loader为空，那么此时的catalinaLoader,sharedLoader其实是同一个ClassLoader ————commonloader
    if ((value == null) || (value.equals("")))
        return parent;

    value = replace(value);

    List<Repository> repositories = new ArrayList<Repository>();

    StringTokenizer tokenizer = new StringTokenizer(value, ",");
    while (tokenizer.hasMoreElements()) {
        String repository = tokenizer.nextToken().trim();
        if (repository.length() == 0) {
            continue;
        }

        // Check for a JAR URL repository
        try {
            @SuppressWarnings("unused")
            URL url = new URL(repository);
            repositories.add(
                    new Repository(repository, RepositoryType.URL));
            continue;
        } catch (MalformedURLException e) {
            // Ignore
        }

        // Local repository
        if (repository.endsWith("*.jar")) {
            repository = repository.substring
                (0, repository.length() - "*.jar".length());
            repositories.add(
                    new Repository(repository, RepositoryType.GLOB));
        } else if (repository.endsWith(".jar")) {
            repositories.add(
                    new Repository(repository, RepositoryType.JAR));
        } else {
            repositories.add(
                    new Repository(repository, RepositoryType.DIR));
        }
    }
    //这里返回的是一个UrlClassLoader实例
    //最终调用org.apache.catalina.startup.ClassLoaderFactory#createClassLoader静态工厂方法创建了URLClassloader的实例
    //而具体的URL其实就是*.loader属性配置的内容
    ClassLoader classLoader = ClassLoaderFactory.createClassLoader
        (repositories, parent);


    return classLoader;

}
```

查看org.apache.catalina.startup.Catalina#getParentClassLoader调用栈，我们看到在StandardContext的startInternal方法中调用了它，那么我们查看一下它的代码，包含了如下代码片段：

```java
if (getLoader() == null) {
    // 初始化WebappLoader，把他的爹设置为sharedloader
            WebappLoader webappLoader = new WebappLoader(getParentClassLoader());
            webappLoader.setDelegate(getDelegate());
            setLoader(webappLoader);
}
try {

    if (ok) {

        // Start our subordinate components, if any
        if ((loader != null) && (loader instanceof Lifecycle))
            ((Lifecycle) loader).start();
        //other code    
    }
catch(Exception e){
}
```

因为WebappLoader符合Tomcat组件生命周期管理的模板方法模式，因此会调用到它的startInternal方法。我们接下来就来看看WebappLoader的startInternal，我们摘取一部分与本篇相关的代码片段如下：

```java
classLoader = createClassLoader();
classLoader.setResources(container.getResources());
// 设置是否配置委派模式
classLoader.setDelegate(this.delegate);
classLoader.setSearchExternalFirst(searchExternalFirst);
```

从上的代码可以看到调用了createClassLoader方法创建一个classLoader，那么我们再看来看看createClassLoader的代码：

```java
private WebappClassLoader createClassLoader()
    throws Exception {

// classLoader是WebappLoader的实例变量，其值为org.apache.catalina.loader.WebappClassLoader，其实就是通过反射调用了WebappClassLoader的构造函数，然后传递了sharedLoader作为其父亲加载器。
    Class<?> clazz = Class.forName(loaderClass);
    WebappClassLoader classLoader = null;

    if (parentClassLoader == null) {
        parentClassLoader = container.getParentClassLoader();
    }
    Class<?>[] argTypes = { ClassLoader.class };
    Object[] args = { parentClassLoader };
    Constructor<?> constr = clazz.getConstructor(argTypes);
    classLoader = (WebappClassLoader) constr.newInstance(args);

    return classLoader;

}
```

进一步来分析一下WebAppClassLoader的代码，在Java自定义类加载器，一般情况下，我们只需要重写findClass方法就好了，而对于WebAppClassLoader，通过查看源代码，我们发现loadClass和findClass方法都进行了重写，那么我们首先就来看看它的loadClass方法,它的代码如下：

```java
public synchronized Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException {

    if (log.isDebugEnabled())
        log.debug("loadClass(" + name + ", " + resolve + ")");
    Class<?> clazz = null;

    // Log access to stopped classloader
    if (!started) {
        try {
            throw new IllegalStateException();
        } catch (IllegalStateException e) {
            log.info(sm.getString("webappClassLoader.stopped", name), e);
        }
    }

    // (0) Check our previously loaded local class cache
    // 首先从当前ClassLoader的本地缓存中加载类，如果找到则返回。
    clazz = findLoadedClass0(name);
    if (clazz != null) {
        if (log.isDebugEnabled())
            log.debug("  Returning class from cache");
        if (resolve)
            resolveClass(clazz);
        return (clazz);
    }

    // (0.1) Check our previously loaded class cache
    // 在本地缓存没有的情况下，调用ClassLoader的findLoadedClass方法查看jvm是否已经加载过此类，如果已经加载则直接返回。
    clazz = findLoadedClass(name);
    if (clazz != null) {
        if (log.isDebugEnabled())
            log.debug("  Returning class from cache");
        if (resolve)
            resolveClass(clazz);
        return (clazz);
    }

    // (0.2) Try loading the class with the system class loader, to prevent
    //       the webapp from overriding J2SE classes
    // 通过系统的来加载器加载此类，这里防止应用写的类覆盖了J2SE的类,这句代码非常关键，如果不写的话，就会造成你自己写的类有可能会把J2SE的类给替换调，另外假如你写了一个javax.servlet.Servlet类，放在当前应用的WEB-INF/class中，如果没有此句代码的保证，那么你自己写的类就会替换到Tomcat容器Lib中包含的类。
    try {
        clazz = system.loadClass(name);
        if (clazz != null) {
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }
    } catch (ClassNotFoundException e) {
        // Ignore
    }

    // (0.5) Permission to access this class when using a SecurityManager
    if (securityManager != null) {
        int i = name.lastIndexOf('.');
        if (i >= 0) {
            try {
                securityManager.checkPackageAccess(name.substring(0,i));
            } catch (SecurityException se) {
                String error = "Security Violation, attempt to use " +
                    "Restricted Class: " + name;
                log.info(error, se);
                throw new ClassNotFoundException(error, se);
            }
        }
    }

    // 判断是否需要委托给父类加载器进行加载，delegate属性默认为false，那么delegatedLoad的值就取决于filter的返回值了，filter方法中根据包名来判断是否需要进行委托加载，默认情况下会返回false.因此delegatedLoad为false
    boolean delegateLoad = delegate || filter(name);

    // (1) Delegate to our parent if requested
    // 因为delegatedLoad为false,那么此时不会委托父加载器去加载，这里其实是没有遵循parent-first的加载机制。
    if (delegateLoad) {
        if (log.isDebugEnabled())
            log.debug("  Delegating to parent classloader1 " + parent);
        ClassLoader loader = parent;
        if (loader == null)
            loader = system;
        try {
            clazz = Class.forName(name, false, loader);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from parent");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }
    }

    // (2) Search local repositories
    if (log.isDebugEnabled())
        log.debug("  Searching local repositories");
    // 调用findClass方法在webapp级别进行加载
    try {
        clazz = findClass(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Loading class from local repository");
            if (resolve)
                resolveClass(clazz);
            return (clazz);
        }
    } catch (ClassNotFoundException e) {
        // Ignore
    }

    // (3) Delegate to parent unconditionally
    // 如果还是没有加载到类，并且不采用委托机制的话，则通过父类加载器去加载。
    if (!delegateLoad) {
        if (log.isDebugEnabled())
            log.debug("  Delegating to parent classloader at end: " + parent);
        ClassLoader loader = parent;
        if (loader == null)
            loader = system;
        try {
            clazz = Class.forName(name, false, loader);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from parent");
                if (resolve)
                    resolveClass(clazz);
                return (clazz);
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }
    }

    throw new ClassNotFoundException(name);

}
```

接着在来分析一下findClass，通过分析findClass的代码，最终会调用org.apache.catalina.loader.WebappClassLoader#findClassInternal方法，那我们就来分析一下它的代码：

```java
protected Class<?> findClassInternal(String name)
    throws ClassNotFoundException {

    //
    if (!validate(name))
        throw new ClassNotFoundException(name);

    String tempPath = name.replace('.', '/');
    String classPath = tempPath + ".class";

    ResourceEntry entry = null;

    if (securityManager != null) {
        PrivilegedAction<ResourceEntry> dp =
            new PrivilegedFindResourceByName(name, classPath);
        entry = AccessController.doPrivileged(dp);
    } else {
        // 通过名称去当前webappClassLoader的仓库中查找对应的类文件
        entry = findResourceInternal(name, classPath);
    }

    if (entry == null)
        throw new ClassNotFoundException(name);

    Class<?> clazz = entry.loadedClass;
    if (clazz != null)
        return clazz;

    synchronized (this) {
        clazz = entry.loadedClass;
        if (clazz != null)
            return clazz;

        if (entry.binaryContent == null)
            throw new ClassNotFoundException(name);

        try {
            // 将找到的类文件通过defineClass转变为Jvm可以识别的Class对象返回
            clazz = defineClass(name, entry.binaryContent, 0,
                    entry.binaryContent.length,
                    new CodeSource(entry.codeBase, entry.certificates));
        } catch (UnsupportedClassVersionError ucve) {
            throw new UnsupportedClassVersionError(
                    ucve.getLocalizedMessage() + " " +
                    sm.getString("webappClassLoader.wrongVersion",
                            name));
        }
        entry.loadedClass = clazz;
        entry.binaryContent = null;
        entry.source = null;
        entry.codeBase = null;
        entry.manifest = null;
        entry.certificates = null;
    }

    return clazz;

}
```