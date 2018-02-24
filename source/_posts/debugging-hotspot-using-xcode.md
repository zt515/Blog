---
title: 在 Xcode 中调试 Hotspot
date: 2018-02-24 22:25:14
tags: JVM
---

上一篇文章中我们已经成功编译了 OpenJDK 8，光编译是没有用的。我们得搞清楚它是怎么运行的。
网上许多文章都在说怎么用 eclipse 调试，但我从心底里抵触 eclipse (也许是 Android Studio 的原因)
由于我们使用的是 macOS，不禁想到 Xcode 这个东西。(Xcode 似乎安装过了我就没打开过，这次就宠幸它一下吧！)

<!-- more -->

## 第一步：新建工程
打开 Xcode，选择 `New Xcode Project`，选择 `Command Line Tools`
![NewProject](/images/debugging-hotspot-using-xcode-1.png)

然后按要求填好信息，项目名我填了 `hotspot`

## 第二步：添加 Hotspot 代码
注意：我们需要把整个 OpenJDK 的源码导入进来
先在 Xcode 自动生成的文件上右键选择 `删除`
都删完以后右键 `hotspot`，选择 `Add Files to hotspot`
选中编译目录的全部文件，添加

这里会耗费比较多的时间，因为 Xcode 会把文件都复制进来，还会生成索引
待到一切完成以后，Xcode 界面应该看起来像这样
![SourcesAdded](/images/debugging-hotspot-using-xcode-2.png)

## 第三步：配置调试参数
打开 `Product -> Scheme -> Edit Scheme`

因为我们已经编译好了，所以在 Build 选项卡中删除所有的 Target
![DeleteTargets](/images/debugging-hotspot-using-xcode-3.png)

然后来到 Run 选项卡，选择 Executable 为编译输出的 java
一般位于 `build/.../jdk/bin/java`
![SelectExecutable](/images/debugging-hotspot-using-xcode-4.png)

再点上方的 Arguments，设置运行的 Java 程序，我这里随便写了个 hello world
![SetArguments](/images/debugging-hotspot-using-xcode-5.png)

## 第四步：开始调试
打开 `hotspot/src/share/vm/prims/jni.cpp`，在 `JNI_CreateJavaVM` 函数上下断点
然后点运行开始调试
![Debugging](/images/debugging-hotspot-using-xcode-6.png)

