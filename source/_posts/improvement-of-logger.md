---
title: Java 输出 Log 的一种优化方案
date: 2018-02-19 16:49:58
tags: Java
---

我们在开发中，经常会有需要打印 log 的地方，但是在正式发行的版本中，我们又不希望这些 log 被打印出来。于是乎我们会这样写代码:
```java
public class Log {
    public static final boolean DEBUG = true;

    public static void debug(String message) {
        if (DEBUG) {
            System.err.println(message);
        }
    }
}
```

<!-- more -->

看起来没毛病对吧？确实，没毛病！
但在正式发布的版本里，调用 `Log.debug()` 的代码依然还是执行了一次，如下面这行代码:
```java
Log.debug("someVar = " + someVar);
```

即使日志不会被打印出来，但字符串拼接还是实实在在地走了一次。于是我们可以这样:
```java
if (Log.DEBUG) {
    Log.debug("someVar = " + someVar);
}
```

嗯～这样做问题就解决了。但是....这样很丑啊！！

试想，一段逻辑里都是这样的代码，不管你疯不疯，反正我是疯了。


正文开始
=======
那么有什么办法既好看，又快速呢？现在就要请我们的主角登场了：`Supplier`

首先我们看看 `Supplier` 是个什么东西
```java
@FunctionalInterface
public interface Supplier<T> {
    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

以下是引用[官方文档](https://docs.oracle.com/javase/8/docs/api/java/util/function/Supplier.html)里的说法
> **Type Parameters**:
> T - the type of results supplied by this supplier
> **Functional Interface**:
> This is a functional interface and can therefore be used as the assignment target for a lambda expression or method reference.
>```java
@FunctionalInterface
public interface Supplier<T>
```
> Represents a supplier of results.
There is no requirement that a new or distinct result be returned each time the supplier is invoked.
> 
> This is a functional interface whose functional method is get().
> 
> **Since**:
> 1.8

简单说，就是 Supplier 接口返回一个任意类型的值，类似与工厂方法。

利用 Java8 这个特性，我们可以把上文的 `Log` 改成这样:
```java
import java.util.function.Supplier;

public class Log {
    private static final boolean DEBUG = true;

    public static void debug(Supplier<String> message) {
        if (DEBUG) {
            System.err.println(message.get());
        }
    }
}
```

这样做不仅可以让 `DEBUG` 不暴露出去，而且在调用的时候，利用 `lambda 表达式` 还可以写的非常优雅:
```java
Log.debug(() -> "someVar = " + someVar);
```

要注意的是，Java 中的 `lambda 表达式` 中捕获的其实是 **值** 而不是 **变量**
联系此处，即 `someVar` 应该是 **final** 或 **即成事实的 final (effectively final)**
比如下面这行代码:
```java
String someVar = getSomeVar();

Log.debug(() -> "someVar = " + someVar);
```
虽然我们没有给 `someVar` 显式地声明 `final`，但从语义上来说，`someVar` 是 `final` 的，这种情况被成为 `即成事实的 final`


但是像下面这样的代码是无法通过编译的:
```java
String someVar;
if (condition1) { someVar = var1; }
else { someVar = var2; }

Log.debug(() -> "someVar = " + someVar);
```

有时候不可避免地会遇到这种情况，毕竟我们用的是 Java 而不是 Kotlin ~~(不要脸地打一波广告)~~
此时我们可以~~巧妙地脏一波~~：
```java
String someVar;
if (condition1) { someVar = var1; }
else { someVar = var2; }

final String finalSomeVar = someVar;
Log.debug(() -> "someVar = " + finalSomeVar);
```
