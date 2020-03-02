---
title: Threaded Interpreting
date: 2017-09-30 23:33:03
tags: virtual-machine
---

### 前言
从广义上来说，虚拟机的种类繁多，但我们这里特指跨平台的，用于实现语言功能的虚拟机。

假设语言都实现到了生成中间语言这一步，那么虚拟机的实现可以有
* Interpreting
  也就是通过类似 `while() { switch() {} }` 的循环，分析中间语言的每条指令，动态解释执行  
* Binary Translation
  顾名思义，就是虚拟机实现了从中间语言，到可执行文件的转换的功能，在运行时，将中间语言转换成了可执行文件，最终执行

<!--more-->

### 发现
一般我们在实现一个虚拟机的时候，解释器部分第一反应肯定是使用传统的`decode-dispatch`模式，说白了就是类似下面这一段代码：
```c++
switch (*pc++) {
    case OP_MOV: break;
    case OP_ADD: break;
    case OP_SUB: break;
    case OP_MUL: break;
    case OP_DIV: break;
    default:
        printf("Unknown instrution\n");
        exit(1);
}
```

### 分析
在很久以前，编译器的优化功能还不完善，甚至完全不存在的时候，上一张中的那个巨大的`switch-case`块会被编译成类似这样的代码：
```c++
opcode = *pc++;
if (opcode == OP_MOV) { 
    // mov
} else if (opcode = OP_ADD) {
    // add
}
...
```
假设虚拟机一共有 `X` 条指令，每种指令在代码中是均匀分布的（即频率相等）。那么在指令解码阶段，平均需要 `X/2` 个判断才能完成解码，这种算法的复杂度就是 `O(n)`。
根据上述，如果能将指令解码阶段的时间复杂度降低到 `O(1)` 级别，那么虚拟机的执行性能将会有显著的提高。

### 解决
要将时间复杂度降低到 `O(1)`，那么就一定要去除这个巨大的`switch`，而我们的指令集总共只有 `X` 条指令，而这些指令的 `opcode` 又正好是从 `0` 开始，一直到 `X - 1`为止。

那么可以设想，**把每条指令都使用 `label` 标示出来，然后将他们的地址存到一个数组中，当执行的时候，可以直接使用 `opcode` 作为下标访问数组，得到对应指令标签的地址，再跳转过去。**

这种方法在专业领域被称为 `Threaded Interpreting`，然而根据上文所述，在标准的C语法中，**并不存在获取标签地址的功能**，**也不支持跳转到保存在变量中的标签地址**。要实现这样的功能，就必须借助汇编语言实现。而现代编译器一般情况下如果源代码内嵌了汇编，就会自动采用完全不优化的方式编译（相当于使用 `-O0` 编译），这显然是得不偿失的。 

经多方查找，发现我们的方法已经有前人想到了，`gcc` 中直接就拓展了`对标签取地址`的写法:
```c++
void *ptr = &&label;
```
跳转写法:
```c++
goto *ptr;
```

那么解释器可以写成这样:
```c++
void exec(int *pc) {
    void *opcodes_labels[OP_COUNT] = { &&op_mov, &&op_add, ... };
    goto *opcodes_labels[*pc++];

    op_mov:
        // mov
        ...
        goto *opcodes_labels[*pc++];

    op_add:
        // add
        ...
        goto *opcodes_labels[*pc++];
        
    ...
}
```
像这样，`exec()` 函数一被执行，就创建一个数组存放跳转表，然后直接跳转到第一条指令对于的地方，执行完毕后再跳转到下一条指令的地方。

### 后话
本文章只是做一个粗略的理论分析，具体效果可以查看 [KiVM](https://github.com/imkiva/KiVM) 的实现，`DragonVM` 对支持此拓展的编译器使用 `Threaded Interpreting`，而对不支持的则使用 `switch-dispatch` 的方式来进行指令解码和分派。

<div class="tip">
特别注意：`goto *opcodes_labels[*pc++]` 这行代码是非常不安全的（具体原因因为篇幅问题不再赘述），解决办法之一就是在运行前先检查一次代码是否合法。
</dic>
