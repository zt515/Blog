---
title: C++ 模版元编程 第 2 章 --- 模板匹配规则
date: 2020-01-12 20:46:11
tags: C++
---

> 我今天又看了 10 MB 的错误信息

<!-- more -->

## 导入

上篇里讲到了特化，那么这里我们就从特化的更多应用开始吧~

比如我们需要实现这样的一个功能，我们自己实现一个 `sizeof`.
这种时候有观众就会问了：“sizeof 不是平台相关的吗？”
这有啥事，~~这里写代码就全部按直觉来！~~

看下面的代码:
```cpp
template <typename T>
struct size;
```

这是我们简单的 size 函数，现在它还没有实现。
但别担心，下一步我们就要为我们支持的类型进行特化了。

比如，我们知道，int 通常是 4 个字节（这里不考虑别的情况）:
```cpp
template <>
struct size<int> {
    static constexpr size_t value = 4;
};
```

以此类推，我们能特化更多:

```cpp
template <>
struct size<float> {
    static constexpr size_t value = 4;
};

template <>
struct size<double> {
    static constexpr size_t value = 8;
};
```

这时候，你可能就问了，如果我要计算 `int *` 的大小怎么办？
好说！特它！

我们知道 int * 在 64 位系统上是 8 个字节。于是我们据此特化。

```cpp
template <>
struct size<int *> {
    static constexpr size_t value = 8;
};
```

问题似乎解决了。

但你有一天忘记了你的具体实现，你只记得你曾经写过一个元函数，可以计算类型的大小，于是你写下了 `size<double *>::value` 这样的代码。最后你得到了这样的错误

```
error: no member named 'value' in 'size<double *>'
```

你一看~原来是没有为 double * 特化实现，于是你又写下了这样的代码

```cpp
template <>
struct size<double *> {
    static constexpr size_t value = 8;
};
```

然后你仔细一想，发现事情不简单：所有指针类型的大小都是 8 呀！难道我要为所有指针都手动特化吗？

当然不用！还记得之前我们说过 **所有类型** 这一概念吗？没错，这可以用一个模板参数来表达！此处不妨命名为 `P` 吧（P 表示 pointer），于是你拿起键盘，手起刀落，写下这样的代码:

```cpp
template <>
struct size<P *> {
    static constexpr size_t value = 8;
};
```

嗯！我们为所有类型 P 的指针类型 P * 特化了 size 函数，让 size 函数对所有指针类型的参数都返回 8.

但是...编译器可不这么认为:
```
error: 'P' was not declared in this scope
```

嗯？P 没有定义是什么回事？难道编译器不知道这个 P 表示所有类型吗？

你想的没错，编译器还真不知道！回想一下，我们说过，模板参数可以是类型，也可以是某个常数值，那么这里的 P 我们没有告诉编译器是个类型，所以编译器就不知道应该认为这个 P 是什么东西。编译器也想偷懒，所以她就直接报错了......

那么我们怎么告诉编译器 P 是个类型呢？很简单，就像这样:

```cpp
template <typename P>
struct size<P *> {
    static constexpr size_t value = 8;
};
```

在上面的 `template` 里面加上一条 `typename P` 就行了，这就像在做一个声明：**我这里需要用到一个参数 P，这个 P 是一个类型名。**

然后我们再把代码放一起来看:

```cpp
template <typename T>
struct size;

template <>
struct size<int *> {
    static constexpr size_t value = 8;
};

template <typename P>
struct size<P *> {
    static constexpr size_t value = 8;
};
```

// TODO


现在我们可以自然而然地引出模板匹配规则了

## 模板匹配规则

注意看上面的那个对指针类型的特化，我们说 P 是一个类型名，然而诸如像 `int *`, `double *` 之类的都是类型名。那么为什么还会匹配到 `P *` 呢？或者说，用 `P *` 匹配了 `double *` 后，`P` 是什么呢？

我们不妨试一试

```cpp
template <typename T>
struct what;

template <typename P>
struct what<P *> {
    using type = P;
};
```
