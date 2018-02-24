---
title: 在 macOS 上编译 OpenJDK 8
date: 2018-02-24 17:24:47
tags: JVM
---

<!-- more -->

## 准备工作
* [Xcode](https://itunes.apple.com/us/app/xcode/id497799835?mt=12)

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
    export ALT_PARALLEL_COMPILE_JOBS=2
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
    --with-debug-level=slowdebug--enable-debug-symbols \
    ZIP_DEBUGINFO_FILES=0
```

* 错误 1
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

* 错误 1
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

* 错误 2
```shell
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/src/share/vm/opto/loopPredicate.cpp:775:73: error: ordered comparison between pointer and zero ('const TypeInt *' and 'int')
    assert(rng->Opcode() == Op_LoadRange || _igvn.type(rng)->is_int() >= 0, "must be");
```
    解决办法：
    `_igvn.type(rng)->is_int() >= 0` 改成 `_igvn.type(rng)->is_int()->_lo >= 0`

* 错误 3
```shell
/Users/kiva/Documents/OpenJDK8/jdk8/hotspot/src/share/vm/runtime/virtualspace.cpp:331:14: error: ordered comparison between pointer and zero ('char *' and 'int')
if (base() > 0) {
```
    解决办法：
    `base() > 0` 改成 `base() != 0`

* 错误 4
```shell
Running nasgen
Exception in thread "main" java.lang.VerifyError: class jdk.nashorn.internal.objects.ScriptFunctionImpl overrides final method setPrototype.(Ljava/lang/Object;)V
```
    解决办法：
    修改 `nashorn/make/BuildNashorn.gmk`
    第 80 行 `-cp` 修改为 `-Xbootclasspath/p:` 注意这里有个 `:`
    修改完成以后是这样 `-Xbootclasspath/p:$(NASHORN_OUTPUTDIR)/...`

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
