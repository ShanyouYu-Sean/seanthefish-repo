---
layout: post
title: CompletableFuture
date: 2020-12-18 10:00:00
tags: 
- 并发
categories:
- 并发
---

转自 https://hhbbz.github.io/2018/08/04/CompletableFuture%E7%BB%88%E6%9E%81%E6%8C%87%E5%8D%97/

## CompletableFuture主要Api详述

### 声明

如果CompletableFuture的方法没有参数Executor并且以…Async结尾，它将会使用 ForkJoinPool.commonPool() (在JDK8中的通用线程池，基于Fork/join线程池和任务窃取实现），这适用于CompletableFuture类中的大多数的方法。所以下面分析的时候可能会直接跳过命名为…Async的方法。

### 创建和获取CompletableFuture

使用 new 关键字新建CompletableFuture实例并不是唯一的选择，CompletableFuture提供了静态工厂方法用于创建自身的实例：

```java
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor);
static CompletableFuture<Void> runAsync(Runnable runnable);
static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);
```

runAsync()易于理解，注意它需要Runnable，因此它返回`CompletableFuture<Void>`作为Runnable不返回任何值。如果你需要处理异步操作并返回结果，使用`Supplier<U>`，它是一个函数式接口, 可以这样使用`Supplier<U>`：

```java
final CompletableFuture<String> future = CompletableFuture.supplyAsync(new Supplier<String>() {
    @Override
    public String get() {
        //...long running...
        return "42";
    }
}, executor);
```

### 转换–CompletableFuture.thenApply()

方法不以Async结尾，意味着Action使用相同的线程执行，而Async可能会使用其它的线程去执行(如果使用相同的线程池，也可能会被同一个线程选中执行)。

apply一般翻译为’作用于’，但是在CompletableFuture中，`***thenApply()***`起到转换结果的作用，总结来说就是叠加多个CompletableFuture的功能，把多个CompletableFuture组合在一起，跨线程池进行异步调用，调用的过程就是结果转换的过程。先看下这些方法：

```java
<U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn);
<U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor);
```

使用例子：

```java
CompletableFuture<String> f1 =  = CompletableFuture.supplyAsync(() -> {
    return "42";
}, executor)；
CompletableFuture<Integer> f2 = f1.thenApply(Integer::parseInt);
CompletableFuture<Double> f3 = f2.thenApply(r -> r * r * Math.PI);
```

或者直接使用流式编程：

```java
CompletableFuture<Double> f3 = CompletableFuture.supplyAsync(() -> {
    return "42";
}, executor).thenApply(Integer::parseInt).thenApply(r -> r * r * Math.PI);
```

### 终端运行(消费)–CompletableFuture.thenRun()/CompletableFuture.thenAccept()

CompletableFuture有两种典型的”最终”阶段方法，其实就是Lambda的终端方法，使用的是Consumer接口(消费操作的接口)。当CompletableFuture的结果已经准备好，thenAccept()执行最终消费操作，thenRun()执行Runnable，没有返回值（或者说返回结果为Void）。

```java
CompletableFuture<Void> thenAccept(Consumer<? super T> block);
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> block);
CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> block,Executor executor);
CompletableFuture<Void> thenRun(Runnable action);
CompletionStage<Void> thenRunAsync(Runnable action);
CompletionStage<Void> thenRunAsync(Runnable action,Executor executor);
```

下面是一个例子：

```java
future.thenAcceptAsync(dbl -> log.debug("Result: {}", dbl), executor);
log.debug("Continuing");
```

thenAccept( )/thenRun( )方法并没有发生阻塞（即使没有明确的executor)。它们像事件侦听器。上例中”Continuing”消息将立即出现，但是这个时候thenAcceptAsync()有可能尚未执行完。thenAccept()和thenRun()的区别是：thenAccept()是针对结果进行消费，因为入参是Consumer函数式接口，有入参无返回值，而thenRun()它的入参是一个Runnable的实例，表示当得到上一步的结果时的操作，也就是当得到上一步的结果则异步执行Runnable。

### 异常处理

```java
CompletableFuture<String> future = new CompletableFuture<>();
        try {
            throw new RuntimeException("test exception");
        }catch (Exception e){
            future.completeExceptionally(e);
            future.complete("test success");
        }
System.out.println(future.get());
//结果（触发了completeExceptionally后，complete将会失效）：
Exception in thread "main" java.util.concurrent.ExecutionException: java.lang.RuntimeException: test exception
	at java.util.concurrent.CompletableFuture.reportGet(CompletableFuture.java:357)
	at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1895)
	at org.throwable.TestGitA.main(TestGitA.java:22)
Caused by: java.lang.RuntimeException: test exception
	at org.throwable.TestGitA.main(TestGitA.java:17)
...
```

补偿型的例子：

```java
String result = CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         if (1 == 1) {
             throw new RuntimeException("测试一下异常情况");
         }
         return "s1";
     }).exceptionally(e -> {
         System.out.println(e.getMessage());
         return "hello world";
     }).join();
System.out.println(result);
//结果
java.lang.RuntimeException: 测试一下异常情况
hello world
```

全方位型的例子：

```java
//注意这里OK为String类型
CompletableFuture<Integer> safe = future.handle((ok, ex) -> {
    if (ok != null) {
        return Integer.parseInt(ok);
    } else {
        log.warn("Problem", ex);
        return -1;
    }
});
```

### CompletableFuture的”串联”–CompletableFuture.thenCompose()

thenCompose()方法允许你对两个异步操作进行流水线，第一个操作完成时，将其结果作为参数传递给第二个操作。你可以创建两个CompletableFuture对象，对第一个CompletableFuture对象调用thenCompose() ，并向其传递一个函数。当第一个CompletableFuture执行完毕后，它的结果将作为该函数的参数，这个函数的返回值是以第一个CompletableFuture的返回做输入计算出的第二个CompletableFuture对象。

```java
<U> CompletableFuture<U> thenCompose(Function<? super T,CompletableFuture<U>> fn);
<U> CompletableFuture<U> thenComposeAsync(Function<? super T,CompletableFuture<U>> fn);
<U> CompletableFuture<U> thenComposeAsync(Function<? super T,CompletableFuture<U>> fn,Executor executor);
```

thenCompose()是一个重要的方法，它允许构建健壮的和异步的管道，没有阻塞和等待的中间步骤。在下面的事例中，仔细观察thenApply()(map)和thenCompose()（flatMap）的类型和差异，calculateRelevance()方法返回CompletableFuture实例：

```java
CompletableFuture<Document> docFuture = //...
CompletableFuture<CompletableFuture<Double>> f =  docFuture.thenApply(this::calculateRelevance);
CompletableFuture<Double> relevanceFuture = docFuture.thenCompose(this::calculateRelevance);
//...
private CompletableFuture<Double> calculateRelevance(Document doc)  //...
```

thenapply（）是接受一个`Function<? super T,? extends U>`参数用来转换CompletableFuture，相当于流的map操作，返回的是非CompletableFuture类型，它的功能相当于将`CompletableFuture<T>`转换成`CompletableFuture<U>`。

thenCompose（）在异步操作完成的时候对异步操作的结果进行一些操作，并且仍然返回CompletableFuture类型，相当于flatMap，用来连接两个CompletableFuture。

```java
// thenapply（）
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
      return 100;
});
CompletableFuture<String> f = future.thenApplyAsync(i -> i * 10).thenApply(i -> i.toString());
System.out.println(f.get()); //"1000"

// thenCompose（）
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<String> f = future.thenCompose( i -> {
    return CompletableFuture.supplyAsync(() -> {
        return (i * 10) + "";
    });
});
System.out.println(f.get()); //1000
```

### CompletableFuture的”并联”–CompletableFuture.thenCombine()

thenCombine()用于连接两个独立的CompletableFuture，它接收名为 BiFunction 的第二参数，这个参数定义了当两个CompletableFuture 对象完成计算后结果如何合并，返回携带计算合并结果的一个新的CompletableFuture。

```java
<U,V> CompletableFuture<V> thenCombine(CompletableFuture<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);
<U,V> CompletableFuture<V> thenCombineAsync(CompletableFuture<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);
<U,V> CompletableFuture<V> thenCombineAsync(CompletableFuture<? extends U> other, BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

假设你有两个CompletableFuture，一个加载Customer另一个加载最近的Shop。他们彼此完全独立，但是当他们完成时，您想要使用它们的值来计算Route。下面是一个例子：

```java
CompletableFuture<Customer> customerFuture = loadCustomerDetails(123);  //省略loadCustomerDetails方法代码
CompletableFuture<Shop> shopFuture = closestShop();  //省略closestShop方法代码
CompletableFuture<Route> routeFuture = customerFuture.thenCombine(shopFuture, (cust, shop) -> findRoute(cust, shop));
   
//...
private Route findRoute(Customer customer, Shop shop) //...
```

新建customerFuture和shopFuture。那么routeFuture包装它们然后“等待”它们完成。当它们的结果准备好了，它会运行我们提供的函数来结合所有的结果(findRoute())。当两个基本的CompletableFuture实例完成并且findRoute()也完成时，这样routeFuture将会完成。

另一个例子

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    return 100;
});
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    return "abc";
});
CompletableFuture<String> f =  future.thenCombine(future2, (x,y) -> y + "-" + x);
System.out.println(f.get()); //abc-100
```

### 结果记录–CompletableFuture.whenComplete()

`***CompletableFuture.whenComplete()***`的作用是CompletableFuture运行完成时，对结果的记录操作，记录的操作由函数 `BiConsumer<? super T, ? super Throwable>`完成，一般BiConsumer这种消费操作应该是终端操作，但是whenComplete返回的是CompletableFuture的接口的实例，这个实例就是调用whenComplete的原始CompletableFuture对象。

```java
CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action);
CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action,Executor executor);
```

一个使用例子如下：

```java
String result = CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         if (1 == 1) {
             throw new RuntimeException("测试一下异常情况");
         }
         return "s1";
     }).whenComplete((s, t) -> {
         System.out.println(s);
         System.out.println(t.getMessage());
     }).exceptionally(e -> {
         System.out.println(e.getMessage());
         return "hello world";
     }).join();
System.out.println(result);
//控制台输出
null
java.lang.RuntimeException: 测试一下异常情况
java.lang.RuntimeException: 测试一下异常情况
hello world
```

这里也可以看出，如果使用了exceptionally，就会对最终的结果产生影响，也就证明了whenComplete返回的是原始的CompletableFuture对象。

### 结果处理–CompletableFuture.handle()

CompletableFuture.handle() 的作用是CompletableFuture运行完成时，对结果的处理。这里的完成时有两种情况，一种是正常执行，返回预期的值。另外一种是遇到异常抛出造成程序的中断。

```java
<U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
<U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
<U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

一个出现异常时的例子：

```java
String result = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //出现异常
        if (1 == 1) {
            throw new RuntimeException("测试一下异常情况");
        }
        return "s1";
    }).handle((s, t) -> {   //这里t的参数类型为Throwable。
        if (t != null) {
            return "hello world"; //这里是异常不为null时候的逻辑，可以选择补偿，也可以直接抛出异常t，一旦抛出异常，调用join（）的时候异常会外抛。
        }
        return s;
    }).join();
System.out.println(result);
//控制台输出
hello world
```

一个未出现异常时的例子：

```java
String result = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "s1";
    }).handle((s, t) -> {
        if (t != null) {
            return "hello world";  
        }
        return s;   //未出现异常，实际上走到这一步
    }).join();
System.out.println(result);
//控制台输出
s1
```

### 合并消费–CompletableFuture.thenAcceptBoth()

CompletableFuture.thenAcceptBoth() 用于连接两个独立的CompletableFuture，它接收名为BiConsumer的第二参数，这个参数定义了当两个CompletableFuture对象完成计算后，结果如何消费，有点像thenCombine，但是对于两个CompletableFuture的计算操作是终端操作，没有返回值(或者说返回结果为Void类型)。

```java
<U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
<U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
<U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action, Executor executor);
```

一个例子如下，5000毫秒后控制台输出”hello world”：

```java
CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(2000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "hello";
     }).thenAcceptBoth(CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "world";
     }), (s1, s2) -> System.out.println(s1 + " " + s2));
```

### 合并执行–CompletableFuture.runAfterBoth()

CompletableFuture.runAfterBoth() 用于连接两个独立的CompletableFuture，不关心两个CompletableFuture的计算结果，当两个CompletableFuture执行完成后，执行Runnable。

```java
CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

一个例子如下，5000毫秒后控制台输出”hello world”：

```java
CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(2000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "s1";
     }).runAfterBothAsync(CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "s2";
     }), () -> System.out.println("hello world"));  
```

### 时间优先转换–CompletableFuture.applyToEither()

CompletableFuture.applyToEither() 用于连接两个独立的CompletableFuture，选择计算（返回结果）最快的一个CompletableFuture，进行转换计算操作(`Function<? super T, U>`)并返回结果。

```java
<U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
<U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
<U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);
```

我们现实开发场景中，总会碰到有两种渠道完成同一个事情，所以就可以调用这个方法，找一个最快的结果进行处理。一个例子如下：

```java
String result = CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "s1";
     }).applyToEither(CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(2000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "hello world";
     }), s -> s).join();  //2000毫秒后返回"hello world"
```

### 时间优先消费–CompletableFuture.acceptEither()

CompletableFuture.acceptEither() 用于连接两个独立的CompletableFuture，选择计算（返回结果）最快的一个CompletableFuture，进行消费操作(Consumer<? super T> action)，无返回值。

```java
public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action,Executor executor);
```

一个例子如下：

```java
CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "s1";
     }).acceptEither(CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(2000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "hello world";
     }), System.out::println); //2000毫秒后控制台打印 "hello world"
```

### 时间优先执行–CompletableFuture.runAfterEither()

CompletableFuture.runAfterEither() 用于连接两个独立的CompletableFuture，不关心任何CompletableFuture的返回值，任何一个CompletableFuture执行完毕得到了结果后会马上执行Runable。

```java
CompletionStage<Void> runAfterEither(CompletionStage<?> other,Runnable action);
CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action);
CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

一个例子如下：

```java
CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(3000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "s1";
     }).runAfterEither(CompletableFuture.supplyAsync(() -> {
         try {
             Thread.sleep(2000);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
         return "s2";
     }), () -> System.out.println("hello world")); //() -> System.out.println("hello world")是Runable的Lambda实现。 2000毫秒后控制台打印 "hello world"
```

### 结果赋值

CompletableFuture的完计算结果直接赋值方法主要有以下几个：

- boolean complete(T value)，通过CAS赋值计算结果，内部会发送完成状态，再次调用无效。
- boolean completeExceptionally(Throwable ex)，通过CAS赋值计算异常，内部会发送完成状态，再次调用无效。
- void obtrudeValue(T value)，直接赋值计算结果，内部会发送完成状态，再次调用无效。
- obtrudeException(Throwable ex)，直接赋值计算异常，内部会发送完成状态，再次调用无效。

只要上面四个方法之一被调用，CompletableFuture就会标记为’完结状态’，再次调用其他方法将不会起效，另外，obtrudeXXX方法属于强制赋值，不建议使用，因为它们会直接覆盖当前的值。

一个例子如下：

```java
CompletableFuture<String> future = new CompletableFuture<>();
        try {
            future.complete("test success");
        }catch (Exception e){
            future.completeExceptionally(e);
        }
System.out.println(future.get());

//输出 test success
```

### 结果获取

- `T get() throws InterruptedException, ExecutionException` ，永久阻塞，直到返回结果值，允许中断，计算过程中所有的异常会包裹为新的ExecutionException实例再抛出。
- `T get(long timeout, TimeUnit unit)throws InterruptedException, ExecutionException, TimeoutException` ，添加超时时间设定，如果超时会抛出TimeoutException，如果获取到结果则释放并返回，允许中断，计算过程中所有的异常会包裹为新的ExecutionException实例再抛出。
- `T join()` ，永久阻塞，直到返回结果值，不允许中断，计算过程中所有的异常会直接抛出。
- `T getNow(T valueIfAbsent)` ，如果当前的计算结果为null，马上返回valueIfAbsent，否则调用join()的逻辑。 
  
结果的获取不做举例，因为这个实在太常用，强烈建议使用`T get(long timeout, TimeUnit unit)`，其他三个方法看场景选择使用。

### 其它

#### 取消–cancel()

调用CompletableFuture实例的 cancel() 方法可以取消当前的CompletableFuture，此时该CompletableFuture实例会进入’完结状态’，其结果会传入一个新的CancellationException实例，此时通过上一节的’结果获取’中的Api调用就会按各自的处理模式抛出异常。

#### 所有完成

调用CompletableFuture的静态方法 `CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)` ，当所有的传入的CompletableFuture实例都完成的时候，会返回一个新建的CompletableFuture，也就是程序将会阻塞在此方法调用，直到所有传入CompletableFuture都完成，这个时候返回值CompletableFuture实例也完成。举个例子：`CompletableFuture.allOf(cf1,cf2).join() ;`，其中cf1、cf2是两个独立的CompletableFuture实例。如果你的程序有这么一段代码，那么执行的时候会阻塞在此，直到cf1和cf2都完成了，才会释放。

#### 任一完成

调用CompletableFuture的静态方法 `CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)`， 这个方法和上面的’所有完成’是相对的。当所有的传入的CompletableFuture实例中只要有一个实例完成的时候，会返回一个新建的CompletableFuture，也就是程序将会阻塞在此方法调用，直到有一个传入的CompletableFuture完成，这个时候返回值CompletableFuture实例也完成。举个例子： `CompletableFuture.anyOf(cf1,cf2).join();` ，其中cf1、cf2是两个独立的CompletableFuture实例。如果你的程序有这么一段代码，那么执行的时候会阻塞在此，直到cf1或cf2其中一个完成了，才会释放。

## 实战例子

现在假设有一个API网关，在调用查询用户某个订单详情的时候，需要分别从订单服务的订单信息接口、用户服务的用户信息接口两个接口拉取数据，一般来说，低效的伪代码大概如下：

```java
//这两个参数从外部获得
Long userId = 10006L;
String orderId = "XXXXXXXXXXXXXXXXXXXXXX";
//从用户服务获取用户信息
UserInfo userInfo = userService.getUserInfo(userId);
//从用订单务获取订单信息
OrderInfo orderInfo = orderService.getOrderInfo(orderId);
//返回两者的聚合DTO
return new OrderDetailDTO(userInfo,orderInfo);
```

其实如果微服务设计得足够好，下面三个外部接口的信息一定是不相关联的，也就是可以并行获取，三个接口的结果都获取完毕之后做一次数据聚合到DTO即可，也就是聚合的耗时大致是这三个接口中耗时最长的接口的响应时间。修改后的代码如下：

```java
@Service
public class OrderDetailService {
	/**
	 * 建立一个线程池专门交给CompletableFuture使用
	 */
	private final ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 20, 0, TimeUnit.SECONDS,
			new ArrayBlockingQueue<>(100));
	@Autowired
	private UserService userService;
	@Autowired
	private OrderService orderService;
	public OrderDetailDTO getOrderDetail(Long userId, String orderId) throws Exception {
		CompletableFuture<UserInfo> userInfoCompletableFuture = CompletableFuture.supplyAsync(() -> userService.getUserInfo(userId), executor);
		CompletableFuture<OrderInfo> orderInfoCompletableFuture = CompletableFuture.supplyAsync(() -> orderService.getOrderInfo(orderId), executor);
		CompletableFuture<OrderDetailDTO> result
				= userInfoCompletableFuture.thenCombineAsync(orderInfoCompletableFuture, OrderDetailDTO::new, executor);
		return result.get();
	}
}
```

上面的代码还没有考虑到外部的微服务异常的情况，但是相对串行的拉取外部信息的接口的操作方式，这种并行的方式显然是更加高效的，而且CompletableFuture的supplyAsync方法可以传入Supplier接口实例，也就是允许任何参数类型的表达式。当然，其实用ExecutorService的invokeAll方法也可以达到相同的效果.