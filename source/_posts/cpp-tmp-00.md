---
title: C++ 模版元编程 第 0 章 --- 前言
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

## 作者联系方式
- 邮箱 [imkiva@icloud.com](mailto:imkiva@icloud.com)
- [**漂    流    瓶**](http://qqapp.qq.com/app/389.html)

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

## 总结
在与 TMP 打交道的过程中，请一定要记住这几点暴论，尤其是「**类型即数值**」，
这点**真的很！重！要！**

