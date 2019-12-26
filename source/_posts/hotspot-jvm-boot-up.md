---
title: HotSpot 0x00 - JVM Boot
date: 2019-10-21 23:05:03
tags: JVM
---

> Java 的水平并不是看了几本虚拟机与并发的书籍就可以搞定的。
> 而是应该一头扎进~~痛苦~~的 JVM 源码之海......

<!-- more -->

## Intro
This is not my first time reading HotSpot source code, but my first time writing blog posts
to record my reading process and thoughts. I am going to write in English as I always wanted 
to have a **hardcore** blog like [ice1000's](https://ice1000.org).

Recently my friends and I chatted in a QQ group. I felt like they all had **hardcore** things in addition to the GitHub profile page. Wondering what else could I take out when asked **contributions** I've ever made to the open-source community, I decided to note down my
thoughts and share them with the world. One the one hand, this fulfills my sense of
achievement. On the other hand, this might become my "brand" someday.

Apart from what I have learned, I am still lacking the ability to structure my expressions.
I hope I can get improved through this long hard process of trying to say things clearly and 
vividly, which is always referred to as "monkey-oriented-writing" (by myself).

## Intro for this topic
And recall from the previous post, we successfully debugged HotSpot in Xcode.
And after almost one year of ~~research~~(forgetting), I made the same thing in CLion. So in the next many many posts related to HotSpot source analysis I would like to choose CLion as my
tool.

First of all, I'd like to say that, CLion is really a **memory and cpu killer**.

## Overview
* JavaVM Boot
* The main() Method Execution 

## Conventions
My environment: `macOS Catalina 10.15` with `CLion 2019.2` and `Xcode 11`.
I only care about the implementation on macOS, so code clipping may happen in the code posted below and future.
And one more thing, to make the post beautiful, empty line insertion may also happen.

My test program:
```java
package com.imkiva.kivm;

public class Main {
    public static void main(String[] args) {
        System.out.println("hello world, hello KiVM.");
    }
}
```

So my launch command is `java -cp /path/to/out com.imkiva.kivm.Main`

## JavaVM Boot
This part focues on how JavaVM is created and initialized.

### main()
Source: `jdk/src/share/bin/main.c`

Just like many other C/C++ programs, Java has its `main()` function. Our first aim is to find that entrance.

```c
/*
 * Entry point.
 */
int
main(int argc, char **argv)
{
    int margc;
    char** margv;
    const jboolean const_javaw = JNI_FALSE;

    margc = argc;
    margv = argv;

    return JLI_Launch(margc, margv,
                   sizeof(const_jargs) / sizeof(char *), const_jargs,
                   sizeof(const_appclasspath) / sizeof(char *), const_appclasspath,
                   FULL_VERSION,
                   DOT_VERSION,
                   (const_progname != NULL) ? const_progname : *margv,
                   (const_launcher != NULL) ? const_launcher : *margv,
                   (const_jargs != NULL) ? JNI_TRUE : JNI_FALSE,
                   const_cpwildcard, const_javaw, const_ergo_class);
}
```

Here, the main() finally calls the `JLI_Launch()` with commandline arguments and lots of others.
The `margc` and `margv` are used to support both `main()` and `WinMain()`. On macOS, it is just a copy of `argc` and `argv`.

The `FULL_VERSION` and `DOT_VERSION` and all `const_*` are defined by Makefile defining macros (see `jdk/src/share/bin/defines.h`).


### JLI_Launch()
Source: `jdk/src/share/bin/java.c`

Let's have a look at `JLI_Launch()`

```c
int
JLI_Launch(int argc, char ** argv,              /* main argc, argc */
        int jargc, const char** jargv,          /* java args */
        int appclassc, const char** appclassv,  /* app classpath */
        const char* fullversion,                /* full version defined */
        const char* dotversion,                 /* dot version defined */
        const char* pname,                      /* program name */
        const char* lname,                      /* launcher name */
        jboolean javaargs,                      /* JAVA_ARGS */
        jboolean cpwildcard,                    /* classpath wildcard*/
        jboolean javaw,                         /* windows-only javaw */
        jint ergo                               /* ergonomics class policy */
)
{
    // local variables
    // ...

    // InitLauncher() will initialize platform specific settings
    InitLauncher(javaw);

    // ...

    // Make sure the specified version of the JRE is running.
    SelectVersion(argc, argv, &main_class);

    // check platform specific flags 
    // create posix_spawnattr_t and create a "main" thread
    // to make the Cocoa event loop to be run on that
    // see: jdk/src/macosx/bin/java_md_macosx.c
    CreateExecutionEnvironment(&argc, &argv,
                               jrepath, sizeof(jrepath),
                               jvmpath, sizeof(jvmpath),
                               jvmcfg,  sizeof(jvmcfg));

```

Here, the `CreateExecutionEnvironment()` (which executes in `Thread-1`) launchs a macOS application thread in `Thread-2`. Afterwards, `Thread-1` waits.


### apple_main()
Source: `jdk/src/macosx/bin/java_md_macosx.c`

`Thread-2` begins in `apple_main()` and re-calls our `main()`.
This process is like registering a **native** application to macOS.

```c
/*
 * Unwrap the arguments and re-run main()
 */
static void *apple_main (void *arg)
{
    objc_registerThreadWithCollector();

    if (main_fptr == NULL) {
        main_fptr = (int (*)())dlsym(RTLD_DEFAULT, "main");
        if (main_fptr == NULL) {
            JLI_ReportErrorMessageSys("error locating main entrypoint\n");
            exit(1);
        }
    }

    struct NSAppArgs *args = (struct NSAppArgs *) arg;
    exit(main_fptr(args->argc, args->argv));
}
```

And the `main()` calls the `JLI_Launch()` and the initialization process continues in `Thread-2`.


### JLI_Launch()
Source: `jdk/src/share/bin/java.c`

```c
int
JLI_Launch(...)
{
    // ...

    // ifn is the InvocationFunctions which contains 3 essential
    // JNI invocation APIs: 
    // CreateJavaVM, GetDefaultJavaVMInitArgs and GetCreatedJavaVMs
    ifn.CreateJavaVM = 0;
    ifn.GetDefaultJavaVMInitArgs = 0;

    // ...

    // Initialize these 3 APIs in ifn
    if (!LoadJavaVM(jvmpath, &ifn)) {
        return(6);
    }

    // ...

    // Aha, the most common trick 
    ++argv;
    --argc;

    // IsJavaArgs() = javaargs (value here is false)
    if (IsJavaArgs()) {
        /* Preprocess wrapper arguments */
        TranslateApplicationArgs(jargc, jargv, &argc, &argv);
        if (!AddApplicationOptions(appclassc, appclassv)) {
            return(1);
        }
    } else {
        /* Set default CLASSPATH */
        cpath = getenv("CLASSPATH");
        if (cpath == NULL) {
            cpath = ".";
        }
        SetClassPath(cpath);
    }

    /* Parse command line options; if the return value of
     * ParseArguments is false, the program should exit.
     */
    if (!ParseArguments(&argc, &argv, &mode, &what, &ret, jrepath))
    {
        return(ret);
    }

    // set pseudo property
    // set platform properties
    // ...

    return JVMInit(&ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

Let's focus on the `LoadJavaVM()` function. It takes an `InvocationFunctions *`.

```c
// jdk/src/macosx/bin/java_md_macosx.c
jboolean
LoadJavaVM(const char *jvmpath, InvocationFunctions *ifn)
{
    // ...
    libjvm = dlopen(jvmpath, RTLD_NOW + RTLD_GLOBAL);
    
    // ...
    ifn->CreateJavaVM = (CreateJavaVM_t)
        dlsym(libjvm, "JNI_CreateJavaVM");
    ifn->GetDefaultJavaVMInitArgs = (GetDefaultJavaVMInitArgs_t)
        dlsym(libjvm, "JNI_GetDefaultJavaVMInitArgs");
    ifn->GetCreatedJavaVMs = (GetCreatedJavaVMs_t)
        dlsym(libjvm, "JNI_GetCreatedJavaVMs");
    // ...

    return JNI_TRUE;
}
```

This function fills invocation APIs with symbols from `dlopen("/path/to/libjvm.dylib")` and `dlsym()`.

#### Summary
The `JLI_Launch()` does the following things in two difference threads:
* Initialize and parse platform specified objects and flags. (`Thread-1`)
* Register to macOS and re-run JLI_Launch() in new thread. (`Thread-1`)
* Fill JNI Invocation APIs(ifn) from `libjvm.dylib` (`Thread-2`)
* Parse envrionment variables and commandline arguments (`Thread-2`)
* Call JVMInit() (`Thread-2`)

### JVMInit()
Source: `jdk/src/macosx/bin/java_md_macosx.c`

After `JLI_Launch()`, the function `JVMInit()` is called. 
We only care about one line in this function:
```c
int
JVMInit(InvocationFunctions* ifn, jlong threadStackSize,
                 int argc, char **argv,
                 int mode, char *what, int ret) {
    // ...
    return ContinueInNewThread(ifn, threadStackSize, argc, argv, mode, what, ret);
}
```

`ContinueInNewThread()` was defined in `java.c` and it is just a wrapper function:
```c
int
ContinueInNewThread(...)
{
    // ...
    rslt = ContinueInNewThread0(JavaMain, threadStackSize, (void*)&args);
    // ...
}
```

The `ContinueInNewThread0()` does what it sounds like it should do. It blocks current thread
and continues execution in a new thread. 

`ContinueInNewThread0()` was defined in `java_md_macosx.c`:
```c
int
ContinueInNewThread0(int (JNICALL *continuation)(void *), ...) {
    // ...
    if (pthread_create(&tid, &attr, (void *(*)(void*))continuation, (void*)args) == 0) {
      void * tmp;
      pthread_join(tid, &tmp);
      rslt = (int)tmp;
    }
    // ...
}
```

The argument `continuation` is `JavaMain()` defined in `java.c`.
As a result, `Thread-2` waits by calling `pthread_join()`, and `Thread-3` starts to play.
Now our Java VM is going to **RUA**!


### JavaMain()
Source: `jdk/src/share/bin/java.c`

Show the code:
```c
int JNICALL
JavaMain(void * _args)
{
    JavaMainArgs *args = (JavaMainArgs *)_args;
    int argc = args->argc;
    char **argv = args->argv;
    int mode = args->mode;
    char *what = args->what;
    InvocationFunctions ifn = args->ifn;

    JavaVM *vm = 0;
    JNIEnv *env = 0;
    jclass mainClass = NULL;
    jclass appClass = NULL; // actual application class being launched
    jmethodID mainID;
    jobjectArray mainArgs;
    int ret = 0;
    jlong start, end;

    // Call objc_registerThreadWithCollector()
    RegisterThread();

    /* Initialize the virtual machine */
    start = CounterGet();
    if (!InitializeJVM(&vm, &env, &ifn)) {
        JLI_ReportErrorMessage(JVM_ERROR1);
        exit(1);
    }

    // ...
}
```

`int mode`: The value of `enum LaunchMode`. Here the value is 1 (`LM_ClASS`)
```c
// jdk/src/share/bin/java.h
enum LaunchMode {               // cf. sun.launcher.LauncherHelper
    LM_UNKNOWN = 0,
    LM_CLASS,
    LM_JAR
};
```

`char *what`: The main class name. Here the value is `com.imkiva.kivm.Main`

The `InitializeJVM()` will call `ifn->CreateJavaVM()` to create the Java VM.
```c
static jboolean
InitializeJVM(JavaVM **pvm, JNIEnv **penv, InvocationFunctions *ifn)
{
    JavaVMInitArgs args;
    jint r;

    // ...

    r = ifn->CreateJavaVM(pvm, (void **)penv, &args);
    JLI_MemFree(options);
    return r == JNI_OK;
}
```

In fact, `ifn->CreateJavaVM` is a function pointer to `JNI_CreateJavaVM` (see above)


### JNI_CreateJavaVM()
Source: `hotspot/src/share/vm/prims/jni.cpp`

```cpp
jint JNICALL JNI_CreateJavaVM(JavaVM **vm, void **penv, void *args) {
    // ...

    result = Threads::create_vm((JavaVMInitArgs*) args, &can_try_again);
    if (result == JNI_OK) {
        JavaThread *thread = JavaThread::current();
        /* thread is thread_in_vm here */
        *vm = (JavaVM *)(&main_vm);
        *(JNIEnv**)penv = thread->jni_environment();

        // ...
    }
}
```

Yeah, we see the **famous** `Threads::create_vm()` now.


### Threads::create_vm()
Source: `hotspot/src/share/vm/runtime/thread.cpp`

// TODO

#### Summary
// TODO


## The main() Method Execution 

### LoadMainClass()
Source: `jdk/src/share/bin/java.c`

We have to continue with `JavaMain()` function.
Now we say that after `InitializeJVM()`, the `JavaVM *` and `JNIEnv *` are set.
So it's time to run our Java's `main()` method.

```c
int JNICALL
JavaMain(void * _args)
{
    // listed above
    // ...

    mainClass = LoadMainClass(env, mode, what);
    CHECK_EXCEPTION_NULL_LEAVE(mainClass);
    /*
     * In some cases when launching an application that needs a helper, e.g., a
     * JavaFX application with no main method, the mainClass will not be the
     * applications own main class but rather a helper class. To keep things
     * consistent in the UI we need to track and report the application main class.
     */
    appClass = GetApplicationClass(env);
    NULL_CHECK_RETURN_VALUE(appClass, -1);
    /*
     * PostJVMInit uses the class name as the application name for GUI purposes,
     * for example, on OSX this sets the application name in the menu bar for
     * both SWT and JavaFX. So we'll pass the actual application class here
     * instead of mainClass as that may be a launcher or helper class instead
     * of the application class.
     */
    PostJVMInit(env, appClass, vm);
    /*
     * The LoadMainClass not only loads the main class, it will also ensure
     * that the main method's signature is correct, therefore further checking
     * is not required. The main method is invoked here so that extraneous java
     * stacks are not in the application stack trace.
     */
    mainID = (*env)->GetStaticMethodID(env, mainClass, "main",
                                       "([Ljava/lang/String;)V");
    CHECK_EXCEPTION_NULL_LEAVE(mainID);

    /* Build platform specific argument array */
    mainArgs = CreateApplicationArgs(env, argv, argc);
    CHECK_EXCEPTION_NULL_LEAVE(mainArgs);

    /* Invoke main method. */
    (*env)->CallStaticVoidMethod(env, mainClass, mainID, mainArgs);

    /*
     * The launcher's exit code (in the absence of calls to
     * System.exit) will be non-zero if main threw an exception.
     */
    ret = (*env)->ExceptionOccurred(env) == NULL ? 0 : 1;
    LEAVE();
}
```

