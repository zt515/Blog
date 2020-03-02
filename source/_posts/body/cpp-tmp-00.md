---
title: C++ 模版元编程 第 0 节 --- 前言
date: 2019-12-26 17:43:18
tags: C++
---

> 我今天又看了 10 MB 的错误信息

<!-- more -->

## 为什么要写这个？
近日沉迷实现一个类型安全的 [EventEmitter](https://github.com/imkiwa/CppPlayground/blob/master/include/v9/kit/event.hpp)

![alt](/images/cpp-tmp-00-1.png)

某日晚上突然顿悟了一些东西，想要写下来。

什么？可能看不懂？~~0.5 分警告~~

前排提示：这套文章十分主观，绝大部分都来自于笔者自己的编码经验。所以，如果你不喜欢，~~那你来打我啊，反正我也不会改。~~ 

## 前言
本文假设读者已经有如下的 C++ 语言基础:

* class/struct/template 等关键字的拼写
* 知道啥是模版，啥是继承，啥是多态
* 有一个支持 C++11/14/17 的编译器（不用 clang 的都踢了）
* 爱思考，会谷歌，能面向 StackOverflow 编程（用百度的都踢了）
* 没了

## 计划的内容

* 模板基础知识
* 模板类型推导
* 模板匹配规则
* 模板元编程中的函数式思想
* 编译期计算
* 模板类型擦除
* 写出**优雅**的模板元编程代码（很重要！！！）
* 实战项目包括
  * 编译期斐波那契数列
  * 实现一个最 naive 的 std::any
  * 编译期快速排序
  * 类型列表
  * Event Emitter

按照预期设想：
这些内容会被~~均匀~~（可能？）分散在每个文章中，她们之中蕴含的思想将会贯穿本系列的所有文章。

## 作者联系方式
- 邮箱 [imkiva@icloud.com](mailto:imkiva@icloud.com)
- [**漂    流    瓶**](http://qqapp.qq.com/app/389.html)

## 致谢
- 感谢好友 [mikecovlee](https://github.com/mikecovlee) 不厌其烦回答我的问题
- 感谢好友 [newNcy](https://github.com/newNcy) 问了上面截图的那个问题

## 模版元编程是什么？有什么用？
断句：模版/元编程

</br>

所谓「元编程」，就是一种「抽象的编程」。我认为，对程序的抽象其实就是**找出程序之间的相似之处，然后用统一的逻辑将其表示出来，而对于程序之间的不同之处，我们用一种手段将他们隐藏起来**。在 C++ 中，这种手段就叫做「模版」，表示统一的逻辑的方式就叫「元编程」。

</br>

C++ 标准库是「泛型编程（Generic Programming）」的实践，这意味着数据和算法是分开的。算法最终需要的不过就是数据的比大小而已，所以我们需要用模版将算法的逻辑「抽象化」，比如我们知道标准库里有这样的算法:

```cpp
template <class T>
const T& min(const T& a, const T& b)
{
    return a < b ? a : b;
}
```

这里的 `T` 我们不用管是什么，我们只要知道，这个算法会返回二者中较小的一个，而至于怎么比较 `a` 和 `b`，那跟我 `min` 有什么关系？

</br>

所以在我的理解中，「模版」给了我们抽象算法的能力，算法是通用的，但仅仅只靠「模版」来抽象具体类型却不足以帮助我们写出通用的算法逻辑，我们还需要更强大的功能，来帮我们设计出更通用、更友好、更健壮、错误信息更可读的代码，于是就有了「元编程」。

## 花费大量时间学习模版元编程，值得吗？
不值得，你可以关掉这个页面了。

那我为什么要学？我只是觉得好玩罢了。

## 为什么歧视 MSVC？
我不是，我没有，你不要乱说啊。

## 模版参数命名约定
模版参数是指 `template <typename T>` 里的 `T`，其**定义域**为**所有类型**。
给这个参数取个**好名字**跟**禁用 Windows 自动更新**一样重要。

所以对于特定的一些模版参数，我的习惯是这样

- 普通的类型: `T`, `U`, `A`, `B`, `C`, `From`, `To`, ...
- [Callable 类型](https://en.cppreference.com/w/cpp/named_req/Callable): `F`, `Callable`, `Handler`, ...
- 函数参数类型: `Args`, `ArgsT`, ...
- 函数返回值类型: `R`, `RetType`, `ReturnType`, ...
- 变长参数: `Ts...`, `As...`, `Xs...`, ...
- ~~其他的看心情~~

## 暴论一：模版元编程是另一门编程语言
模版元编程是**借助了 C++ 语法**的另一门**运行在编译期**的编程语言，
它的作用是辅助 C++ 程序设计。

说 TMP (Template MetaProgramming, 模版元编程) 是另一门编程语言，是因为你可以在 TMP 中见到一些**好康的语法**，以及神似函数式语言的一些写法。

## 暴论二：类型即数值
作为一门编程语言，那么就有这门语言操作的数据。在 C++ 中我们操作的数据有各种类型，比如 `int`, `float`, `std::string`；类似的，在 TMP 中，我们操作的数据就是上面提到的那些类型，比如下面的代码：

```cpp
template <typename T>
using S = std::remove_reference_t<T>;
```

这里先解释一下，上面这段代码的意思是：
> `S` 是一个类型，`T` 也是一个类型，并且 `S` **取决于** `T`
> - 如果 `T` 是**任意一个类型 U** 的引用类型 `U &` (即 `T == U &`)，则 `S` 为 `U`
> - 如果不是，则 `S` 为 `T`。

举几个更直观的例子，

|        T       |       S       |
| :------------: | :-----------: |
|  int &         |    int        |
|  const int &   |    const int  |
|  ...           |    ...        |
|  U &           |    U          |

这个像不像我们小学二年级时学过的函数值表？

不妨更深入一步，之前已经说过 TMP 是一门编程语言，那么我们就可以认为上面那个 `S` 是一个函数，`S` 需要一个参数 `T`，其返回值记为 `S<T>`。

写成伪代码的形式就是这样

```cpp
S<T> {
    return std::remove_reference_t<T>;
}
```

或者写作

```cpp
S<T> {
    if (T == U &) {
        return U;
    } else {
        return T;
    }
}
```

~~有内味了，有内味了。~~

**（什么？你说要把尖括号换成圆括号你才看得出这是个函数？踢了！**

在一些资料书中，上面的模版 `S` 也被叫做**元函数 (metafunction)**，
可见我的暴论都没说错。

但是在后续的文章中，我依然会把类型和数值分开称呼（原因就在下面），但你得知道 TMP 是能**把类型当作数值一样来做运算的**。

</br>

那么 TMP 能不能操作 C++ 里的那些 `int`, `flaot`... 呢？当然可以！比如：

```cpp
// 接受一堆类型作为参数
template <typename ... Ts>
struct T {};

// 接受一堆 int 作为参数
template <int ... Is>
struct I {};

void foo() {
    using t1 = I<1, 2, 3, 4>;
    using t2 = T<int, char, short>;
}
```

怎么样？现在看出那个 `typename` 的意思了吗？
是不是相当于「**类型的类型**」呢？

</br>

怎么样？那个 `using` 是不是觉得很微妙？
像不像是在定义一个「常量」？
（`using` 定义出来的是类型，which is immutable）

## 暴论三：TMP 没有停机性检查

啥叫停机性检查？我们写个代码来看看：

```cpp
template <size_t A, size_t B>
struct Add {
    static constexpr size_t result = Add<A + 1, B - 1>::result;
};

template <size_t A>
struct Add<A, 0> {
    static constexpr size_t result = A;
};
```

`Add<A, B>` 可以在编译期计算 A + B，比如

```cpp
static_assert(Add<1, 2>  ::result == 3,   "You wrote a bug");
static_assert(Add<50, 99>::result == 149, "You wrote a bug");
```

但是如果你试着把终止条件去掉，让 `Add<A, B>` 成为一个死循环，让代码长这样：

```cpp
template <size_t A, size_t B>
struct Add {
    static constexpr size_t result = Add<A + 1, B - 1>::result;
};
```

我们再试着运行同样的代码，编译器会给出这样的错误：
```
fatal error: recursive template instantiation exceeded maximum depth of 1024
```

看到没？**C++ 编译器不会检查上面的代码是否能停机，而是限制模版展开次数。**

事实上，从错误信息可以看出，就算终止条件存在，当 `Add<A, B>` 展开次数超过 1024 次时，编译器一样会报错。

这也限制了一些特定的需求用 TMP 是无法完成的。（后文会详细讨论）
但这也让一些骚操作成为可能。

## 暴论四：TMP 是 duck-typing
啥是 duck-typing（鸭子类型）？~~你 tnd 不会谷歌吗？~~

简单的说，如果有这样的代码

```cpp
template <typename Duck>
void meow(Dock &&duck) {
    duck->meow();
}
```

相信大家都能看出来，不管这个 `Duck` 在实例化的时候被替换成了什么类型，只要这个类型提供了 `meow()` 的成员方法，这段代码就能通过编译。

</br>

这就是所谓的

> **当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子**

## 暴论五：TMP 是动态类型和静态类型的混合语言

举个例子就行啦:
```cpp
template <typename T>
struct happy {
    using type = int;
};

template <>
struct happy<float> {
    static constexpr int type = 1;
};
```

上面的例子里，我们为 `happy` **特化**（下一章会讲）了 `float` 类型，于是只有 `happy<float>::type` 本身是一个值，而不是一个类型，其余的 `happy<...>::type` 都是一个类型。

那么现在我们有一个元函数如下定义:

```cpp
template <typename T>
using get_happy = typename happy<T>::type;
```

乍一看没什么问题，但当你把 float 作为参数传递给 get_happy 的时候，编译器无疑会抛出一个编译错误。

```
error: no type named 'type' in 'struct happy<float>'
```

这个问题的原因我们会在后续的文章中慢慢解释...这里只能说是因为：**每个特化的分支中的定义是独立的**。

其次还能从此处需要手动指定 `typename` 看出来，如果编译器能从 `happy` 的泛化版本中推断出 `happy<...>::type` 是类型而不是值，那么在 `get_happy` 中就不需要手动指定了，因为泛化版本中的 `type` 和 特化版本中的 `type` 应该是同样的东西（都是类型或者都是值），但实际上并不是。所以我们可以知道：TMP 在对待类型参数上跟动态类型类似（实际上就是）。

那么静态类型表现在何处呢？

自然是直接将数值作为模板参数传递的时候啊！


## 总结
在与 TMP 打交道的过程中，请一定要记住这几点暴论，尤其是「**类型即数值**」，
这点**真的很！重！要！**

## 参考文献
- [1] wuye9036, *[CppTemplateTutorial](https://github.com/wuye9036/CppTemplateTutorial)*
- [2] Wikipedia, *[Duck Typing](https://en.wikipedia.org/wiki/Duck_typing)*
