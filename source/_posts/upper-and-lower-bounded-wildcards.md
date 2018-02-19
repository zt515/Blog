---
title: 浅谈 Java 中的上届通配符和下届通配符
date: 2018-02-20 02:32:19
tags: Java
---

有这样的继承关系
```java
class Animal {} // 动物
class Dog extends Animal {} // 狗 是 动物
class Cat extends Animal {} // 猫科动物/猫 是 动物
class Tiger extends Cat {} // 老虎 是 猫科动物
```
<!-- more -->

## 协变和不变

首先要讲讲 `协变 (Covariant)` 和 `不变 (Invariant)` 的概念

众所周知，数组是协变的，看下面这段代码
```java
Cat[] cats = new Cat[10];
Animal[] animals = cats; // 可以, Cat 是 Animal 的子类
animals[0] = new Dog(0); // 可以, Dog 是 Animal 的子类
```
但是运行会出现 `java.lang.ArrayStoreException: Dog`

然而，容器是不变的，看下面这段代码
```java
List<Animal> list = new ArrayList<Cat>(); // 不可以，List<Cat> 不是 List<Animal> 的子类
```



## 上届通配符
```java
List<? extends Cat> list = new ArrayList<>(); // 保证存放猫科动物或猫科动物的子类的列表
list.add(new Tiger()); // 不可以, 列表中存放虽然是猫科动物，但猫科动物那么多，不一定是 Tiger
list.add(new Cat()); // 不可以, 列表中存放虽然是猫科动物，但猫科动物那么多，不一定是 Cat
list.add(new Animal()); // 不可以, 强行扩大范围，Animal 不一定是猫科动物
Cat cat = list.get(0); // 可以，列表中存放的都是猫科动物
```

## 下届通配符
```java
List<? super Cat> list = new ArrayList<>(); // 保证存放猫科动物或猫科动物的父类的列表
list.add(new Tiger()); // 可以, Tiger 一定是猫科动物
list.add(new Cat()); // 可以, Cat 一定是猫科动物
list.add(new Animal()); // 不可以, Animal 不一定是猫科动物
Cat cat = list.get(0); // 不可以, 不知道 list 中存放的到底是哪种猫科动物，不一定是 Cat
```

有人问："`List<? super Cat> list;` `list` 不是**存放猫科动物或猫科动物的父类**的列表吗？`Animal` 是 `Cat` 的父类，为什么不能 `list.add(new Animal())` 呢？"

这个问题问得好啊，回答："**Animal 不一定是 Cat 啊**，比如 `class Dog extends Animal {}`, 这里 `Dog` 是 `Animal`，但是 `list.add(new Dog())` 显然是不可能的"


## 个人理解
我认为，文档中对上届通配符和下届通配符的解释容易让新手产生疑惑，所以我是这样理解的：
* 对于上届通配符 `? extends T`，通配符 `?` 可以捕获的类型必须从 `T` 开始往继承链下方走，不一定要走到最底端
* 对于下届通配符 `? super T`，通配符 `?` 可以捕获的类型从继承链最底端开始往上走，最多走到 `T` 但不一定要走到 `T`


## PECS (Producer Extends Consumer Super) 原则
* 频繁读取的，适合用上界通配符 extends。
* 经常插入的，适合用下界通配符 super。
