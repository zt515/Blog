---
title: C++ 模版元编程 第 2 节 --- 模板匹配规则
date: 2020-01-12 20:46:11
tags: C++
---

> 我今天又看了 10 MB 的错误信息

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

所以，我们尝试用 P \* 去匹配 `double *`，如果匹配成功，那么 P 就变成了 `double`。
有没有觉得像什么？模式匹配！

没错，模板匹配规则，就是模式匹配。

## 元编程中的模式匹配

上面已经有了一个很生动的例子，用 P \* 匹配 `double *`，所以 P 是 `double`。
那么能不能说的更笼统一点呢？

~~当然可以，不然我还能自己把自己问死了？~~

> Pattern Matching：compare something with its construction form
> 模式匹配：将一个东西，按照其「构造的方式」进行对比

当然上面这个版本需要进一步阐述：

### 1. 构造？
比如你有一个基本类型 `int`，我们就可以这个类型的基础上构造出 `int *` 类型。

这里可以说 \* 是一个构造器，它接受一个类型，返回一个新类型。
不妨用 `Star` 来代替 \*，于是就可以说：**`Star` 接受一个类型 `T`，返回新的类型 `Star(T)`**。
用伪代码表示如下：
```cpp
Type Star(T) {
    return T *;
}
```

同理，用 `Const` 代替 `const` 关键字，我们说：**`Const` 接受一个类型 `T`，返回新的类型 `Const(T)`**。
用伪代码表示如下：
```cpp
Type Const(T) {
    return const T;
}
```

这就是构造。~~我真的只能解释成这样了，呜哇啊啊啊啊啊啊啊啊~~

### 2. 匹配？

我们有一个值/类型 T，我们可以尝试用一个「表达式」(表达式中可以包含类型) 去跟 T 的「构造形式」做对比，这就叫「匹配」。
匹配过程中，**尽可能地**让 T 的「构造形式」中**没有被被匹配的部分最少**。


于是，我们就可以用 P * 尝试去匹配尽可能符合条件的 P。

|       类型      |       P       |
| :------------: | :-----------: |
|  int \*        |    int        |
|  int \*\*      |    int \*     |
|  int \*\*\*\*  |    int \*\*\* |
|  int(*)(int)   |    int(int)   |

用上面的 `Const` 和 `Star` 举例子，如果写成伪代码：
```cpp
match (T) { /* 对 T 进行匹配 */
    case Star(U)  => /* T 是 U * */
    case Const(U) => /* T 是 const U */
    case _        => /* T 两种都不是 */
}
```

这就是匹配。~~我真的只能解释成这样了，呜哇啊啊啊啊啊啊啊啊~~


## 具体例子？

光说概念可能不好理解，我们直接上代码来看。

我们知道，C++ 中的模板类并不是类型，而是类型构造器（即：模板类本身不能作为对象的类型，只有在实例化以后才可以）。
就是这样:
```cpp
/* 错误，std::vector 不是类型，而是类型构造器 */
std::vector v1;

/* 正确，std::vector 接受一个类型参数 int，构造出(实例化)了类型 std::vector<int> */
std::vector<int> v2;
```

所以我们这里就可以看出，std::vector 是类似于上面的 `Star`, `Const` 之类的构造器。
于是我们就可以进行模式匹配:
```cpp
template <typename V>
struct vector_value_type;

template <typename ValueType>
struct vector_value_type<std::vector<ValueType>> {
/*                       ^^^^^^^^^^^^^^^^^^^^^^   */
    using type = ValueType;
};
```

注意看代码中标出来的地方，这里我们用跟上面一样的伪代码写出来就是这样:
```cpp
match (V) {
    case std::vector<ValueType> => return ValueType;
    case _                      => fuck;
}
```

快！自己理解！~~我真的只能解释成这样了，呜哇啊啊啊啊啊啊啊啊~~

## 来吧！解决问题！

问题是啥来着？

> 现在问题又来了，如果我想细化指针类型的 id 怎么办呢？
> 比如现在我们需要这样操作，让 `id<P *> = id<P> + 100`，
> 即下面这样的断言恒真（伪代码）:
>
> ```cpp
> static_assert(id<P *>::value == id<P>::value + 100, "FUCK");
> ```

很简单啊：
```cpp
template <typename P>
struct id<P *> {
    static constexpr size_t value = 100 + id<P>::value;
};
```

~~你说是吧？~~
