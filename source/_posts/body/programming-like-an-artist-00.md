---
title: Haskell —— 像艺术家一样编程(0)
date: 2018-11-15 23:33:16
tags: Haskell
---

#### newtype 和 data 的区别 （一）

<!--more-->
现在我们有下面的代码

```haskell
data D = D Int
newtype N = N Int

d (D i) = 10086
n (N i) = 10086
```

然后我们尝试
```haskell
Prelude> d undefined
*** Exception: Prelude.undefined

Prelude> n undefined
10086
```

### 原因
#### 导入：⊥
`⊥` 在 `λ 演算` 中表示**不可能计算完成的结果**
在 `Haskell` 中，我们可以用**异常， undefined** 等等来表示 `⊥`

#### 结论
初步探究发现，是因为 `newtype` 定义的类型，其类型中并不包含一个 `⊥` 的值
而 `data` 定义的类型，其类型中包含 `⊥` 这样一个值
所以在计算 `n ⊥` 的时候，由于 `N` 中不包含 `⊥`，所以对于所有参数，结果都是 `10086`
但是 `D` 中包含了 `⊥`，且 `d ⊥ = ⊥`，所以我们就得到了一个异常。
