---
title: Haskell —— 像艺术家一样编程(1)
date: 2018-11-16 23:08:25
tags: Haskell
---

#### 初识 Type Inference

一接触 `Haskell`，就被它强大的类型系统吸引了，因为在我看来，`Haskell` 的类型系统比我之前学过的任何一门语言都要强大
在《Learn You A Haskell For Great Good》上，我看到这样一句话
> 如果不确定正在编写的函数的类型，那就先写好这个函数，然后让 GHC 去推导它的类型，最后再把它抄上去

<!--more-->

于是我就对 GHC 推导函数类型的过程产生了极大的兴趣，然后就有了这篇~~探索性质的文章~~

### 好了，现在是正文
大部分静态类型语言都有`类型推导 (Type Inference)`，引入类型推导不仅极大地提高了静态类型语言的开发效率，同时又能保证静态类型的严谨性，真是一举两得。

虽然一直使用着一些语言里的类型推导，但是却从来没有深入分析过类型推导的过程，最初我以为~~类型推导只要推导出表达式的类型就可以了~~，但学习 `Haskell` 之后，我才发现，**几乎**一切的类型都可以被编译器自动推导，让我眼界大开。

#### 类型变量
先看这样一个函数
```haskell
head' :: [a] -> a
head (x:_) = x
```

可以看到，`head'` 函数的类型为 `[a] -> a`
这里的 `a` 是`类型变量`，它可以根据参数的类型发生一些变化，比如当我传入的参数是一个`[Int]` 的时候，这个 `a` 就是 `Int`

#### 推导过程
我们用一个例子来演示这个过程
假定有函数 `f :: a -> b`，我们需要推导函数 `twice f x = f (f x)` 的类型 

##### f x
上面提到函数 `f :: a -> b`，那么对于 `f x`，既然 `x` 可以作为 `f` 的参数，那么 `x` 一定可以作为 `a` 来使用，这就是一个最基本的类型推导。我们将这一过程记为：

$$\frac{f :: a \to b \quad x :: a}{f\ \ x :: b}$$

##### f (f x)
我们先按照上面的定义写出 `f (f x)` 的类型

$$\frac{f :: a \to b \quad f\ \ x :: a}{f\ \ (f \ \ x) :: b}$$

然后再将 `f x` 展开，得到

$$\frac{f\ ::\ a \to b \quad \frac{f\ ::\ c \to a \quad x\ ::\ c}{f\ x\ ::\ a}}{f\ \ (f \ x)\ ::\ b}$$

而又由于**`f`都是同一个函数**，所以**上面提到的类型 \\(a \to b\\) 和 \\(c \to a\\) 中，箭头左右两边对应的类型应该被推导为相同类型**，记为 \\(a \equiv c\\) 和 \\(b \equiv a\\) (或者 \\(a \sim c\\) 和 \\(b \sim a\\))

然后我们做一组替换，把所有的 `c` 换成 `a`，把所有的 `b` 也换成 `a`，记为 \\(c \mapsto a\\) 和 \\(b \mapsto a\\)，于是我们得到了

$$\frac{f\ ::\ a \to a \quad \frac{f\ ::\ a \to a \quad x\ ::\ a}{f\ \ x\ ::\ a}}{f\ \ (f \ x)\ ::\ a}$$

再由上面的定义可以得到，`f :: a -> a`, `x :: a`, `f (f x) :: a`
所以 twice 即为 `twice :: (a -> a) -> a -> a`

##### const id
我们有 `const :: x -> y -> x`  和 `id :: a -> a` ，现在我们需要推导 `const id` 的类型

仿照上例不难得出：

$$\frac{const\ ::\ a \to b \quad id\ ::\ a}{const\ id\ ::\ b}$$

先分析 `const :: x -> y -> x` ，因为有 \\(a \to b \equiv  x \to (y \to x)\\)

所以做替换：\\(a \mapsto x\\) 和 \\(b \mapsto y \to x\\)

然后我们得到了

$$\frac{const\ ::\ x \to (y \to x) \quad id\ ::\ x}{const\ id\ ::\ y \to x}$$

接着再分析 `id :: a -> a` ，因为有  \\(x \equiv  a \to a\\)

所以做替换 \\(x \mapsto a \to a\\) 

得到

$$\frac{const\ ::\ x \to (y \to a \to a) \quad id\ ::\ a \to a}{const\ id\ ::\ y \to a \to a}$$

看看分母！是不是就是 `const id` 的类型吗！

因为类型变量可以更改名字，所以我们得到结论： `const id :: a -> b -> b`

#### 最后引出相关概念
- **`Unification`** (化一)
    把箭头两端的类型听过替换转化为相同类型的过程成为化一
- **`Unifier`** (化一器)
    如果一组替换能把类型 a 和 b 转化为相同的类型，那么这一组替换称为 a 和 b 的化一器
