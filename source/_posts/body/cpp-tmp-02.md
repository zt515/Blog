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

## 实战：TypeList
[戳这里](https://imkiva.com/blog/2020/03/body/cpp-tmp-practice-01/)

## 实战：Function Parser
<!-- [戳这里](https://imkiva.com/blog/2020/03/body/cpp-tmp-practice-02/) -->

我们要从一个 Callable 的类型中，解析出返回类型，和参数类型。
Callable 类型就是可以支持「函数调用语法」的类型，比如本身就是个函数，
或者重载了 `operator()`（即 Functor 类型）。

我们不妨先参照上面的伪代码，写出 Function Parser 的伪代码:
```cpp
/* 模式匹配 Callable 类型 */
match (Callable) {
    case R(Args...)             => /* 是函数，或者类静态函数 */
    case R(Class::*)(Args...)   => /* 是成员函数 */

    /* 不是函数/函数指针，那么是 Functor 吗？*/
    case _ => match (decltype(&Callable::operator())) {
                  case R(Class::*)(Args...) => /* 是 Functor */
                  case _                    => /* 不是，滚    */
              }
}
```

### 分析

TMP 的常用套路之一就是：先实现一个通用版本，然后在这个通用版本的基础上进行特化。
```cpp
template <typename Callable>
struct function_parser {
    /* 什么都不做 */
};
```

接着看上面的第一个匹配式子: `case R(Args...)`，我们这样匹配它
```cpp
template <typename R, typename ...Args>
struct function_parser<R(Args...)> {
    using return_type = R;
    using arg_types = TypeList::List<Args...>;
    using function_type = std::function<R(Args...)>;
};
```

再强调一次，特化处的 `template <typename R, typename ...Args>` 是用于声明需要用到哪些参数的，
也就是声明模式匹配表达式 `case R(Args...)` 里的 `R` 和 `Args`。

通用版本里的模板参数声明的意义与特化中的**完全不同**，千万不要弄混淆了！！

<br/>

然后是第二个匹配表达式: `case R(Class::*)(Args...)`
```cpp
template <typename Class, typename R, typename ...Args>
struct function_parser<R(Class::*)(Args...)> {
    using return_type = R;
    using arg_types = TypeList::List<Class&, Args...>;
    using class_type = Class;
    using function_type = std::function<R(Class&, Args...)>;
};
```

注意这里的 `arg_types` 和 `function_type`，他们的第一个参数都多了一个 `Class &`，
这是因为，现在匹配的是成员函数，而成员函数都需要一个 `this`，这个 `this` 自然就是 `Class &`.

<br/>

现在我们已经完成了前两个模式匹配了，但第三个是一个通配符（即前两个都没匹配到），
那么在 TMP 里应该怎么对应呢？

没错，就是我们一开始写的那个通用版本，我们再搬出来看看:
```cpp
template <typename Callable>
struct function_parser {
};
```

我们需要在这个地方对 `decltype(&Callable::operator())` 再进行一次模式匹配，
所以现在我就要祭出 TMP 中常用的第二个法宝：**继承**。

<br/>

这里我们应该进行的模式匹配中，也会包含 `case R(Class::*)(Args...)` 这个模式，
而这个模式的处理我们已经实现过了，所以我们让 `function_parser` 的通用版本继承自己的某个特化版本，
这样，在所有模式都匹配不到的时候，重新匹配第二次，这样不就行了嘛？

```cpp
template <typename Callable>
struct function_parser : public function_parser<decltype(&Callable::operator())> {
};
```

这样，就完成了整个模式匹配的过程。但值得一提的是，
上面这样的写法存在不足之处：**匹配到 lambda 表达式** 的具体类型无法转换。
比如:
```cpp
auto lambda = []() { printf("yes"); };
using FT = typename function_parser<decltype(lambda)>::function_type;
FT f = lambda; /* error! */
```

为啥呢？

<br/>

这就要讲一下 Functor 和一般的成员函数的不同了。
Functor 是重载了 `operator()` 的对象，而这个函数正好是个成员函数，
所以我们直接匹配 **`Callable::operator()`** 这个函数的类型的时候，
我们一定会进入到 `R(Class::*)(Args...)` 这个分支中，但是这个分支中，
我们必须让参数列表的第一个参数是 `this`，**但 `Functor` 是不需要的**，
这便是冲突的原因。

<br/>

那怎么解决呢？

很简单，我们单独为 Functor 做一个 Functor Parser 就行了。

```cpp
template <typename F>
struct functor_parser : public functor_parser<decltype(&F::operator())> {
};

template <typename Class, typename R, typename ...Args>
struct functor_parser<R(Class::*)(Args...)> {
    using function_type = FunctionAlias<R(Args...)>;
    using return_type = R;
    using class_type = Class;
};
```

然后我们让 Function Parser 的通用版本继承自 Functor Parser:
```cpp
template <typename Callable>
struct function_parser : public functor_parser<Callable> {
};
```

大功告成！


### 工业化版本
实际的版本中，需要考虑的情况还有 const 和函数指针，由于本篇文章只是讲解核心部分，
所以这些不那么重要的就忽略了。

<br/>
这里是一份跑在生产环境的 Function Parser：[libmozart/function.hpp](https://github.com/libmozart/core/blob/bd432a29d9e56cecb4b89e7a8cdb7280981814fc/mozart%2B%2B/mpp_core/function.hpp#L58)

也许会有帮助！

