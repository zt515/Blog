---
title: 在 macOS 上编译 OpenJDK 8
date: 2018-02-24 17:24:47
tags: JVM
---

> Java 的水平并不是看了几本虚拟机与并发的书籍就可以搞定的。
> 而是应该一头扎进~~痛苦~~的 JVM 源码海洋......

<!-- more -->


## ChangeLog
* 2018-02-24: 初次完成
* 2018-05-17: 新增 freetype 错误解决方案（评论区）
* 2019-10-21: 在 macOS Catalina 上重新测试


## 环境
* macOS Catalina 10.15

## 准备工作
* [Xcode 11](https://itunes.apple.com/us/app/xcode/id497799835?mt=12)

* xcode-select
    ```shell
    $ xcode-select -install
    ```

* [Homebrew](https://brew.sh/)

* Mercurial
    ```shell
    $ brew install mercurial
    ```

* [XQuartz](https://www.xquartz.org/)

* JDK 8 
    ```shell
    $ java -version
    java version "1.8.0_162"
    Java(TM) SE Runtime Environment (build 1.8.0_162-b12)
    Java HotSpot(TM) 64-Bit Server VM (build 25.162-b12, mixed mode)
    ```

* freetype
    ```shell
    $ brew install freetype
    ```

## 下载源码
```shell
$ hg clone http://hg.openjdk.java.net/jdk8/jdk8 JDK8
$ cd JDK8
$ bash ./get_source.sh
```

## 编译相关环境变量
* 先创建一个 `envsetup.sh`，写入以下内容
    ```shell
    # 设定语言选项，必须设置
    export LANG=C
    # Mac平台，C编译器不再是GCC，是clang
    export CC=gcc
    # 跳过clang的一些严格的语法检查，不然会将N多的警告作为Error
    export COMPILER_WARNINGS_FATAL=false
    # 链接时使用的参数
    export LFLAGS='-Xlinker -lstdc++'
    # 是否使用clang
    export USE_CLANG=true
    # 使用64位数据模型
    export LP64=1
    # 告诉编译平台是64位，不然会按32位来编译
    export ARCH_DATA_MODEL=64
    # 允许自动下载依赖
    export ALLOW_DOWNLOADS=true
    # 并行编译的线程数，编译时间长，为了不影响其他工作，我选择为2
    export HOTSPOT_BUILD_JOBS=2
    # 是否跳过与先前版本的比较
    export SKIP_COMPARE_IMAGES=true
    # 是否使用预编译头文件，加快编译速度
    export USE_PRECOMPILED_HEADER=true
    # 是否使用增量编译
    export INCREMENTAL_BUILD=true
    # 编译内容
    export BUILD_LANGTOOLS=true
    export BUILD_JAXP=false
    export BUILD_JAXWS=false
    export BUILD_CORBA=false
    export BUILD_HOTSPOT=true
    export BUILD_JDK=true
    # 编译版本
    export SKIP_DEBUG_BUILD=true
    export SKIP_FASTDEBUG_BUILD=false
    export DEBUG_NAME=debug
    # 避开javaws和浏览器Java插件之类的部分的build
    export BUILD_DEPLOY=false
    export BUILD_INSTALL=false
    # 加上产生调试信息时需要的 objcopy
    export OBJCOPY=gobjcopy
    ```
    

然后 `source envsetup.sh`

## configure
执行以下命令
```shell
$ bash ./configure \
    --with-target-bits=64 \
    --with-debug-level=slowdebug \
    --enable-debug-symbols \
    ZIP_DEBUGINFO_FILES=0
```

### 错误 1
```shell
configure: error: GCC compiler is required. Try setting --with-tools-dir.
```
编辑 `common/autoconf/generated-configure.sh`
搜索 `GCC compiler is required`，注释掉所有搜索出来的结果

执行完毕以后会看到类似下面的结果
```shell
Configuration summary:
* Debug level:    slowdebug
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: macosx, CPU architecture: x86, address length: 64

Tools summary:
* Boot JDK:       java version "1.8.0_151"
* C Compiler:      version  (at /usr/bin/gcc)
* C++ Compiler:    version  (at /usr/bin/g++)

Build performance summary:
* Cores to use:   2
* Memory limit:   8192 MB
* ccache status:  not installed (consider installing)
```

## make
执行以下命令
```shell
$ make all
```

### 错误 1
```shell
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/src/share/vm/code/relocInfo.hpp:367:27: error: friend declaration specifying a default argument must be a definition
inline friend relocInfo prefix_relocInfo(int datalen = 0);
                        ^
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/src/share/vm/code/relocInfo.hpp:462:18: error: friend declaration specifying a default argument must be the only declaration
inline relocInfo prefix_relocInfo(int datalen) {
                ^
```

解决办法：
编辑 `hotspot/src/share/vm/code/relocInfo.hpp`
搜索 `datalen = 0`, 改为 `datalen`
![Error1Step1](/images/building-openjdk8-make-all-error-1-step-1.png)
搜索 `int datalen`, 第 462 行，改为 `int datalen = 0`
![Error1Step2](/images/building-openjdk8-make-all-error-1-step-2.png)

### 错误 2
```shell
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/src/share/vm/opto/loopPredicate.cpp:775:73: error: ordered comparison between pointer and zero ('const TypeInt *' and 'int')
    assert(rng->Opcode() == Op_LoadRange || _igvn.type(rng)->is_int() >= 0, "must be");
```

解决办法：
`_igvn.type(rng)->is_int() >= 0` 改成 `_igvn.type(rng)->is_int()->_lo >= 0`

### 错误 3
```shell
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/src/share/vm/runtime/virtualspace.cpp:331:14: error: ordered comparison between pointer and zero ('char *' and 'int')
if (base() > 0) {
```

解决办法：
`base() > 0` 改成 `base() != 0`

### 错误 4
```shell
Running nasgen
Exception in thread "main" java.lang.VerifyError: class jdk.nashorn.internal.objects.ScriptFunctionImpl overrides final method setPrototype.(Ljava/lang/Object;)V
```

解决办法：
修改 `nashorn/make/BuildNashorn.gmk`
第 80 行 `-cp` 修改为 `-Xbootclasspath/p:` 注意这里有个 `:`
修改完成以后是这样 `-Xbootclasspath/p:$(NASHORN_OUTPUTDIR)/...`

### 错误 5
```shell
compiling /users/kiva/documents/openjdk8/jdk8/hotspot/src/share/vm/adlc/dfa.cpp warning: include path for stdlibc++ headers not found; pass '-stdlib=libc++' on the command line to use the libc++ standard library instead [-wstdlibcxx-not-found]
```

原因：Xcode 10 以后移除了 libstdc++ 的支持，而 hotspot 这个辣鸡居然还一直用着这个，于是编译器找不到 libstdc++ 的头文件就罢工了
解决办法：
打开 [这个链接](https://github.com/imkiwa/xcode-missing-libstdc-), clone 到本地，参考 `install.sh` 将文件链接或者复制到对应位置（**慎重直接执行，请一定事先核对路径是否正确！**）

### 错误 6
```objc
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/agent/src/os/bsd/MacosxDebuggerLocal.m:27:9: fatal error: 'JavaNativeFoundation/JavaNativeFoundation.h' file not found
#import <JavaNativeFoundation/JavaNativeFoundation.h>
```

原因：因为 Xcode 之前会安装类似 xxx-for-java-command-lines-tools 的框架包到 /System/Library/Frameworks, 而自从 macOS 10.14 开始，这些框架包全部都被安装到了 /Library/Developer/CommandLineTools/SDKs/MacOSX10.1x.sdk

解决方法：
先在终端下执行

```shell
$ find / -name "*JavaNativeFoundation.h*"
...
/Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk/System/Library/Frameworks/JavaVM.framework/Versions/A/Frameworks/JavaNativeFoundation.framework/Versions/A/Headers/JavaNativeFoundation.h
/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk/System/Library/Frameworks/JavaVM.framework/Versions/A/Frameworks/JavaNativeFoundation.framework/Versions/A/Headers/JavaNativeFoundation.h
...
```

然后可以得出真正的 `JavaVM.framework` 是被安装到了 `/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk/System/Library/Frameworks/JavaVM.framework/` 下，于是我们对 `hotspot/make/bsd/makefiles.saproc.make` 这个文件做如下修改

将
![error610](/images/building-openjdk-make-all-error6-from1.png)
修改为
![error611](/images/building-openjdk-make-all-error6-to1.png)

将
![error620](/images/building-openjdk-make-all-error6-from2.png)
修改为
![error621](/images/building-openjdk-make-all-error6-to2.png)

### 错误 7
```shell
fatal error: 'CoreGraphics/CGBase.h' file not found
#import <CoreGraphics/CGBase.h>
```

解决办法：这个问题的原因跟上面的一样。涉及到两个文件：
* `jdk/make/lib/PlatformLibraries.gmk`
* `jdk/make/lib/Awt2dLibraries.gmk`

将包含 `ApplicationServices.framework` 的路径替换成 `/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk/System/Library/Frameworks/CoreGraphics.framework`

将包含 `JavaVM.framwork` 的路径替换成 `/Library/Developer/CommandLineTools/SDKs/MacOSX10.15.sdk/System/Library/Frameworks/JavaVM.framework/Frameworks`

上面的路径都可以通过 `find / -name ...` 找到

## 完成
最后我们看到了这样的输出
```shell
----- Build times -------
Start 2018-02-24 21:33:13
End   2018-02-24 21:37:59
00:00:00 corba
00:00:00 demos
00:03:01 docs
00:00:00 hotspot
00:01:40 images
00:00:00 jaxp
00:00:00 jaxws
00:00:02 jdk
00:00:00 langtools
00:00:02 nashorn
00:04:46 TOTAL
-------------------------
Finished building OpenJDK for target 'all'
```
恭喜！编译完成了！

让我们测试一下
```shell
$ cd build/macosx-x86_64-normal-server-slowdebug/jdk/bin
$ ./java -version
openjdk version "1.8.0-internal-debug"
OpenJDK Runtime Environment (build 1.8.0-internal-debug-kiva_2018_02_24_20_52-b00)
OpenJDK 64-Bit Server VM (build 25.0-b70-debug, mixed mode)
```
哈哈哈哈哈哈红红火火恍恍惚惚

## 参考文献
>[Mac 环境使用 clang 编译 OpenJDK8](https://wind2412.github.io/2017/10/20/Mac-%E7%8E%AF%E5%A2%83%E4%BD%BF%E7%94%A8-clang-%E7%BC%96%E8%AF%91-OpenJDK8/)
>[MAC 编译 OpenJDK8](https://github.com/ydcun/Java/blob/master/java/src/main/java/com/ydcun/openjdk/jdk8/MAC%E7%BC%96%E8%AF%91OpenJDK8.md)
>[mac 编译 openjdk8 记录](http://blog.csdn.net/yuankundong/article/details/78876523)
>[编译 openjdk 过程中遇到的错误](https://xiexianbin.cn/java/2017/03/15/OpenJDK-compile-error)
>[How to elegantly compile OpenJDK (Mac version)](http://www.programmersought.com/article/6281426636/)
>[mac 10.13.x 编译 openjdk8](https://www.jianshu.com/p/d9a1e1072f37)
