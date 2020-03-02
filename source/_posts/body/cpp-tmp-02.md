---
title: C++ 模版元编程 第 2 节 --- 模板匹配规则
date: 2020-01-12 20:46:11
tags: C++
---

> 我今天又看了 10 MB 的错误信息

**本文还未完成**

<!-- more -->

## 导入

上篇里讲到了特化，那么这里我们就从特化的更多应用开始吧~

比如我们需要实现这样的一个功能，我们需要为类型编号.
即将类型映射到一个整数上，
比如 `int` 类型编号为 `0`，`float` 类型编号为 `1`，
`double` 类型编号为 `2`，指针类型编号为 `3`
...

看下面的代码:
```cpp
template <typename T>
struct id;
```

这是我们简单的 `id` 函数，现在它还没有实现。
但别担心，下一步我们就要为我们支持的类型进行特化了。

比如，将 `int` 类型编号为 `0`
```cpp
template <>
struct id<int> {
    static constexpr size_t value = 0;
};
```

以此类推，我们能特化更多:

```cpp
template <>
struct id<float> {
    static constexpr size_t value = 1;
};

template <>
struct id<double> {
    static constexpr size_t value = 2;
};
```

这时候，你可能就问了，如果我要映射 `int *` 怎么办？
好说！特它！

```cpp
template <>
struct id<int *> {
    static constexpr size_t value = 3;
};
```

问题似乎解决了。

但你有一天忘记了你的具体实现，你只记得你曾经写过一个元函数，可以将类型映射为整数，于是你写下了 `id<double *>::value` 这样的代码。最后你得到了这样的错误

```
error: no member named 'value' in 'id<double *>'
```

你一看~原来是没有为 `double *` 特化实现，于是你又写下了这样的代码

```cpp
template <>
struct id<double *> {
    static constexpr size_t value = 3;
};
```

然后你仔细一想，发现事情不简单：难道我要为所有指针类型都手动特化吗？

当然不用！还记得之前我们说过 **所有类型** 这一概念吗？没错，这可以用一个模板参数来表达！此处不妨命名为 `P` 吧（P 表示 pointer），于是你拿起键盘，手起刀落，写下这样的代码:

```cpp
template <>
struct id<P *> {
    static constexpr size_t value = 3;
};
```

嗯！我们为所有类型 `P` 的指针类型 `P *` 特化了 `id` 函数，让 `id` 函数对所有指针类型的参数都返回 `3`.

但是...编译器可不这么认为:
```
error: 'P' was not declared in this scope
```

嗯？`P` 没有定义是什么回事？难道编译器不知道这个 `P` 表示所有类型吗？

你想的没错，编译器还真不知道！

在这里，编译器认为这个 `P` 应该是某个**实际存在**的类型，比如 `struct P {}` 这样的。但是我们的意思是让 `P` 表示**所有类型**，所以我们需要**这里的 P 是类型参数**，which 我们必须手动告诉编译器，就像这样:

```cpp
template <typename P>
struct id<P *> {
    static constexpr size_t value = 3;
};
```

在上面的 `template` 里面加上一条 `typename P` 就行了，这就像在做一个声明：**我这里需要用到一个参数 P，这个 P 是一个类型名。**

然后我们再把代码放一起来看:

```cpp
template <typename T>
struct id;

template <>
struct id<int *> {
    static constexpr size_t value = 3;
};

template <typename P>
struct id<P *> {
    static constexpr size_t value = 3;
};
```

然后我们再写一点测试代码

```cpp
static_assert(id<int *>::value == 3);      // ok
static_assert(id<char *>::value == 3);     // ok
static_assert(id<double *>::value == 3);   // ok
```

嗯，都正确。

现在问题又来了，如果我想细化指针类型的 id 怎么办呢？
比如现在我们需要这样操作，让 `id<P *> = id<P> + 100`，
即下面这样的断言恒真（伪代码）:

```cpp
static_assert(id<P *>::value == id<P>::value + 100, "FUCK");
```

要解决这个问题，我们需要先明白模板匹配规则。

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

template <typename T>
using what_t = typename what<T>::type;
```

然后再写一个辅助函数用来打印类型的名字:

```cpp
template <typename T>
struct show {
    show() = delete;
};
```

然后我们来看看 P 到底是个什么鬼:

```cpp
show<what_t<double *>>();
```

编译这段代码，我们能从编译错误信息中看到这样一句:

```
[with T = 'double']
```

