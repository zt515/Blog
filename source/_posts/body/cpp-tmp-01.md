---
title: C++ 模版元编程 第 1 节 --- 「泛化」和「特化」
date: 2019-12-28 10:57:34
tags: C++
---

> 我今天又看了 10 MB 的错误信息

<!-- more -->

## 前言
[上一篇文章](https://imkiva.com/blog/2019/12/body/cpp-tmp-00/)讲了一些基础前提以后，这里开始进入 TMP 真正有趣的地方。

## 泛化
我们有一个做除法的静态成员函数(~~别问我为什么是静态成员函数而不是全剧函数~~)：

```cpp
class Div {
    static double doit(double lhs, double rhs) {
        return lhs / rhs;
    }
}
```

但是这个函数只能对 `double` 类型做除法，现在我们需要将其中的算法逻辑抽象出来，让它能对所有类型都做除法，于是我们引入模版参数 `T`，来代表一个**任意类型**。

```cpp
template <typename T>
class Div {
    static T doit(T lhs, T rhs) {
        return lhs / rhs;
    }
}
```

好了，现在我们的 `Div::doit` 就能应用到**任意支持「除法」运算的类型**上了。（这里强调**支持「除法」运算**是因为如果 `T` 不是`原始数据类型`并且也没有重载正确的 `operator/` 时，编译器会报错）。

</br>

如你所见，上面的过程就叫做「泛化」。我们说泛化后的模版类 `Div` 是「原型」，也叫 `Div` 的「泛化版本」。

</br>

~~（其实我非常不想扯名词的，但后文需要，没办法~~

## 特化

在上面的例子中，当 `T` 是 `int` 的时候，而 `rhs` 又恰好为 0，那么上面的代码就会出现运行时异常。

所以我们可以**为 `int` 类型特化 `Div`** 来处理这种特殊情况：

```cpp
template <>
class Div<int> {        /* 为 int 类型特化 */
    static int doit(int lhs, int rhs) { 
        if (rhs == 0) {
            printf("Divide by zero\n");
            return 0;
        }
        return lhs / rhs;
    }
}

void foo() {
    Div<int>::doit(1, 0);    /* Divide by zero */
    Div<double>::doit(1, 2); /* 0.5            */
}
```

等等！那个 `template <>` 是什么玩意？！
~~你哪来这么多问题！~~，在模版匹配规则的地方会详细讲这个东西。

现在可以理解为 **「Div 在为 int 特化后，不再需要额外的模版参数」**，
那既然不需要模版参数了，那能不能去掉这一行呢？

答案是不行，因为这个特化依赖于原型。关键字 `template` 相当于告诉编译器：这是一个模版类的特化版本，不是独立存在的一个类。

</br>

让「原型」在遇到不同的类型时执行不同的代码，从而做到处理特殊情况。这种方法就叫做「特化」，更准确的说，

> **特化（specialization）是根据一个或多个特殊的整数或类型，给出模板实例化时的一个指定内容。**

特化，就是**处理特例**。我们在数学中学过「斐波那契数列」，这个数列满足下面的关系：

```
F(0) = 1
F(1) = 1
F(n) = F(n - 1) + F(n - 2), n >= 2
```

它有两个特例，当 n == 0 或 n == 1 时，F(n) 的值恒为 1。

~~显然~~, F(0) 和 F(1) 就是在处理斐波那契数列中的特例，这就是「特化」。

## 实战：编译期斐波那契数列

~~真就说什么来什么~~，我们来整一个利用 TMP 特性实现的编译期斐波那契数列。

我们先把一般情况写出来，也就是「原型」：

```cpp
/* 这里有疑惑的自觉看上一章 */
template <int N>
struct Fib {
    /* 递归调用元函数 Fib 计算第 N 项的值 */
    /* constexpr 会「请求」编译器在编译期试着对这个值求值 */
    static constexpr int value = 
        Fib<N - 1>::value + Fib<N - 2>::value;
};
```

然后再处理 N 为 1 和 2 时的特殊情况，注意！！！我要特化了！！！

处理 $F_{0} = 1$

```cpp
template <>
struct Fib<0> {
    static constexpr int value = 1;
};

```

处理 $F_{1} = 1$

```cpp
template <>
struct Fib<1> {
    static constexpr int value = 1;
};
```

然后我们来测试一下：

```cpp
static_assert(Fib<20>::value == 10946, "You wrote a bug");
```

嗯！我没写出 bug！

## 偏特化
后面讲到模版匹配规则的时候会再详细来说。
可以先看参考文献里的资料。

## 参考文献
- [1] wuye9036, *[CppTemplateTutorial](https://github.com/wuye9036/CppTemplateTutorial)*
- [2] en.cppreference.com, *[constexpr specifier (since C++11)](https://en.cppreference.com/w/cpp/language/constexpr)*
- [3] en.cppreference.com, *[partial template specialization](https://en.cppreference.com/w/cpp/language/partial_specialization)*
