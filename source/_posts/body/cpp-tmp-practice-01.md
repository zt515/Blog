---
title: C++ 模版元编程实践 --- TypeList
date: 2020-03-02 02:32:54
tags: C++
---

> 我今天又看了 10 MB 的错误信息

<!-- more -->

## TypeList 是什么？

List 是什么？自然是列表。
TypeList，顾名思义，就是类型列表。
只不过列表里的元素不是数据，而是一个个的类型。
所以我们需要将类型视为一种数据。

而在 C++ 中，能将类型作为数据处理的只有元编程。
所以这部分涉及到大量元编程的前置知识：

1. [前言](https://imkiva.com/blog/2019/12/body/cpp-tmp-00/)
2. [「泛化」和「特化」](https://imkiva.com/blog/2019/12/body/cpp-tmp-01/)
3. [模板匹配规则](https://imkiva.com/blog/2020/01/body/cpp-tmp-02/)

## 可变参数模板
一个列表，里面存放的元素个数可以是任意的，所以我们需要一种「容器」
能包装「任意个数」的类型。

这种东西，叫做「可变参数模板」(variadic templates):

```cpp
template <typename ...Ts>
struct Fuck {};
```

那么这个 `Ts...` 就叫做「参数包」(parameter pack)，
我们可以用 `sizeof...(Ts)` 得到参数包中元素的个数。

这里需要提一下参数包的解包规则，当 `...` 出现在一个语法单元的末尾时，
代表这个表达式中的**参数包**将按照这个**语法单元**的格式进行展开。

比如：
```cpp
template <typename ...Ts>
void fuck(Ts &&...ts) {
    use(std::forward<Ts>(ts)...);
}
```

所以，当你这样使用 `fuck` 函数的时候：
```cpp
T t;
U u;
fuck(t, u);
```

里面的 `use` 函数调用处包含一个 `std::forward<Ts>(ts)` 这样的语法单元，
所以这里就会展开成：

```cpp
use(std::forward<T>(t), std::forward<U>(u));
```

又比如：
```cpp
template <typename ...Ts>
class son : public Ts... {};
```

所以当你这样使用 `son` 的时候:
```cpp
son<A, B, C, D>
```

由于这里包含一个 `public Ts` 的语法单元，
所以这里会展开成

```cpp
class son : public A, public B, public C, public D {};
```

Wow, awesome!

## 先来一个容器！

看看上面的可变参数模板，是不是跟我们要的容器很相似？
我们要的是这样的：

```cpp
List<int, bool, char, List<int>, std::string>
```

是吧？

于是我们有了容器了:
```cpp
template <typename ...Ts>
struct List {};
```

但是为了代码的整洁，我们可以考虑把这部分放到一个结构体里去（具体原因后面会体现）:
```cpp
struct TypeList {
    /* 列表元素的容器 */
    template <typename ...Ts>
    struct List {};

    /* 为了少打几个字，为空列表 List<> 取个别名 */
    /* 这不是显而易见的嘛 ？ */
    using Nil = List<>;
};
```

## 容器有了，然后呢？

> 所以你最后的成果就是纯理论的，就像欧氏几何一样，
> 先设定几条简单的不证自明的公理，
> 再在这些公理的基础上推导出整个理论体系。

我们不妨思考一个列表的「最小行为集合」，在这个集合里的操作我们认为是「公理」。
然后其他的操作都可以使用这些「公理」实现。

对于一个列表，首先有的必须是往容器中添加内容，所以这个集合里必定包含「cons」。

然后我们需要能对这个列表进行遍历，不然，我要个列表只能存，不能读，那还玩个🔨。

所以，这个集合里还需要包含「head」和「tail」。

那么，足够了吗？足够了。

## 我们先来实现 cons
这里需要用到特化，因为特化可以给我们使用「模式匹配」的能力。
```cpp
// I 后缀代表 Implementation（实现）
template <typename T, typename L>
struct consI;

template <typename T, typename ...Ts>
struct consI<T, List<Ts...>> {
    using type = List<T, Ts...>;
};
```

请注意这里的特化形式 `consI<T, List<Ts...>>`，这里要求我们传给 cons 的第二个参数必须是
列表容器，否则这个特化就无法匹配。

同时，还需要注意特化内部的 `using type = List<T, Ts...>`，这里就完成了添加操作：
把 T 添加到了原有列表的头部。

现在我们需要往列表里添加元素，就要这样用：
```cpp
using my_list = List<int, char>;
using new_list = typename consI<bool, my_list>::type;
```

淦！`typename consI<bool, my_list>::type` 这一点都不像在调用一个函数！
我们可以多写一步，把后面那个 type 省略掉。

```cpp
template <typename T, typename L>
using cons = typename consI<T, L>::type;
```

于是我们就能写:
```cpp
using new_list = cons<bool, my_list>;
```

Wow, awesome!

## 然后是 head 和 tail
有了 `cons` 的实现经验，我们实现 `head` 和 `tail` 就很简单了。

先来实现 `head`:
```cpp
template <typename L>
struct headI;

template <typename T, typename ...Ts>
struct headI<List<T, Ts...>> {
    using type = T;
};

template <typename L>
using head = typename headI<L>::type;
```

然后是 `tail`，这不过就是 `head` 的另一种情况罢了:
```cpp
template <typename L>
struct tailI;

template <typename T, typename ...Ts>
struct tailI<List<T, Ts...>> {
    using type = List<Ts...>;
};

template <typename L>
using tail = typename tailI<L>::type;
```

## 其他操作？
其他操作都可以通过 `cons`, `head`, `tail` 三种基本操作组合出来，不信你试试？

## 参考资料
* [Functional Streams](https://www.codewars.com/kata/5512ec4bbe2074421d00028c)
* [libmozart/typelist](https://github.com/libmozart/core/blob/bd432a29d9e56cecb4b89e7a8cdb7280981814fc/mozart%2B%2B/mpp_core/type_traits.hpp#L107)
