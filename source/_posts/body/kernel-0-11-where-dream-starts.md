---
title: Linux 0.11 - 梦开始的地方
date: 2020-02-19 02:45:27
tags: linux
---

这一切还要从一只蝙蝠说起...

<!-- more -->

## 前排提醒
编译一个 1991 年发布的版本是一个很折腾的过程，如果你不想折腾，可以直接到页面底部查看结论。

## 需要的环境
1. 有耐心看错误信息的脑子
2. 没了

## 我的测试环境
* Ubuntu 18.04,     x86_64
* WSL Ubuntu 18.04, x86_64

## 编译过程

#### 0x00 
```
make: as86: Command not found
```

解决办法：

```shell
$ sudo apt install bin86
```

#### 0x01
```
make: gas: Command not found
```

修改所有 `Makefile`

```
gas -> as
gar -> ar
gld -> ld
```

#### 0x02
```
as -c -o boot/head.o boot/head.s
boot/head.s: Assembler messages:
boot/head.s:43: Error: unsupported instruction `mov'
boot/head.s:47: Error: unsupported instruction `mov'
boot/head.s:59: Error: unsupported instruction `mov'
boot/head.s:61: Error: unsupported instruction `mov'
boot/head.s:136: Error: invalid instruction suffix for `push'
boot/head.s:137: Error: invalid instruction suffix for `push'
boot/head.s:138: Error: invalid instruction suffix for `push'
boot/head.s:139: Error: invalid instruction suffix for `push'
boot/head.s:140: Error: invalid instruction suffix for `push'
boot/head.s:151: Error: invalid instruction suffix for `push'
boot/head.s:152: Error: invalid instruction suffix for `push'
boot/head.s:153: Error: invalid instruction suffix for `push'
boot/head.s:154: Error: operand type mismatch for `push'
boot/head.s:155: Error: operand type mismatch for `push'
boot/head.s:161: Error: invalid instruction suffix for `push'
boot/head.s:163: Error: invalid instruction suffix for `pop'
boot/head.s:165: Error: operand type mismatch for `pop'
boot/head.s:166: Error: operand type mismatch for `pop'
boot/head.s:167: Error: invalid instruction suffix for `pop'
boot/head.s:168: Error: invalid instruction suffix for `pop'
boot/head.s:169: Error: invalid instruction suffix for `pop'
boot/head.s:214: Error: unsupported instruction `mov'
boot/head.s:215: Error: unsupported instruction `mov'
boot/head.s:217: Error: unsupported instruction `mov'
Makefile:34: recipe for target 'boot/head.o' failed
make: *** [boot/head.o] Error 1
```

这是由于在 64 位机器上编译的原因，需要告诉编译器，我们要编译成 32 位的目标，
在所有 `Makefile` 的 `AS` 后面添加 `--32`，`CFLAGS` 中加 `-m32 -fno-stack-protector`

#### 0x03
```
boot/head.s:231: Error: alignment not a power of 2
```

把 `align n` 改成 `align 2^n` (`2^n` 自己算出来)

```
.align 2 应该改为 .align 4
.align 3 应该改为 .align 8
```

新版本的 AS 和老版本的 AS 处理 .align 的方式不一样


#### 0x04
```
gcc: error: unrecognized command line option ‘-fcombine-regs’; did you mean ‘-Wcompare-reals’?
gcc: error: unrecognized command line option ‘-mstring-insns’; did you mean ‘-fstrict-enums’?
```

从所有 `Makefile` 中删除这两个编译参数


#### 0x05
```
In file included from init/main.c:8:0:
init/main.c:23:29: error: static declaration of ‘fork’ follows non-static declaration
 static inline _syscall0(int,fork)
                             ^
include/unistd.h:134:6: note: in definition of macro ‘_syscall0’
 type name(void) \
      ^~~~
include/unistd.h:210:5: note: previous declaration of ‘fork’ was here
 int fork(void);
     ^~~~
init/main.c:24:29: error: static declaration of ‘pause’ follows non-static declaration
 static inline _syscall0(int,pause)
                             ^
include/unistd.h:134:6: note: in definition of macro ‘_syscall0’
 type name(void) \
      ^~~~
include/unistd.h:224:5: note: previous declaration of ‘pause’ was here
 int pause(void);
     ^~~~~
init/main.c:26:29: error: static declaration of ‘sync’ follows non-static declaration
 static inline _syscall0(int,sync)
                             ^
include/unistd.h:134:6: note: in definition of macro ‘_syscall0’
 type name(void) \
      ^~~~
include/unistd.h:235:5: note: previous declaration of ‘sync’ was here
 int sync(void);
     ^~~~
init/main.c:104:6: warning: return type of ‘main’ is not ‘int’ [-Wmain]         void main(void)  /* This really IS void, no error here. */
      ^~~~
Makefile:35: recipe for target 'init/main.o' failed
make: *** [init/main.o] Error 1
```

修改 `init/main.c`

```c
static inline _syscall0(int,fork)
static inline _syscall0(int,pause)
static inline _syscall1(int,setup,void *,BIOS)
static inline _syscall0(int,sync)
```

改成

```c
__attribute__((always_inline)) inline _syscall0(int,fork)
__attribute__((always_inline)) inline _syscall0(int,pause)
__attribute__((always_inline)) inline _syscall1(int,setup,void *,BIOS)
__attribute__((always_inline)) inline _syscall0(int,sync)
```

#### 0x06
```
../include/string.h:29:1: error: ‘asm’ operand has impossible constraints       __asm__("cld\n"
 ^~~~~~~
```

把内联汇编中出现的所有“可能被修改的寄存器”(即 Clobber)删除


#### 0x07
```
fork.c: In function ‘copy_mem’:
../include/linux/sched.h:189:1: error: ‘asm’ operand has impossible constraints __asm__("movw %%dx,%0\n\t" \
 ^
```

同上，例子如下：

```c
#define _set_base(addr,base) \
__asm__("movw %%dx,%0\n\t" \
	"rorl $16,%%edx\n\t" \
	"movb %%dl,%1\n\t" \
	"movb %%dh,%2" \
	::"m" (*((addr)+2)), \
	  "m" (*((addr)+4)), \
	  "m" (*((addr)+7)), \
	  "d" (base) \
	:"dx")
```

改成

```c
#define _set_base(addr,base) \
__asm__("movw %%dx,%0\n\t" \
	"rorl $16,%%edx\n\t" \
	"movb %%dl,%1\n\t" \
	"movb %%dh,%2" \
	::"m" (*((addr)+2)), \
	  "m" (*((addr)+4)), \
	  "m" (*((addr)+7)), \
	  "d" (base) \
	)
```

#### 0x08
```
traps.o: In function `get_fs_byte':
traps.c:(.text+0x391): multiple definition of `get_fs_byte'
sched.o:sched.c:(.text+0x10f): first defined here
traps.o: In function `get_fs_word':
traps.c:(.text+0x399): multiple definition of `get_fs_word'
sched.o:sched.c:(.text+0x117): first defined here
traps.o: In function `get_fs_long':
traps.c:(.text+0x3a2): multiple definition of `get_fs_long'
sched.o:sched.c:(.text+0x120): first defined here
traps.o: In function `put_fs_byte':
traps.c:(.text+0x3aa): multiple definition of `put_fs_byte'
sched.o:sched.c:(.text+0x128): first defined here
traps.o: In function `put_fs_word':
traps.c:(.text+0x3b6): multiple definition of `put_fs_word'
sched.o:sched.c:(.text+0x134): first defined here
traps.o: In function `put_fs_long':
traps.c:(.text+0x3c3): multiple definition of `put_fs_long'
sched.o:sched.c:(.text+0x141): first defined here
traps.o: In function `get_fs':
traps.c:(.text+0x3cf): multiple definition of `get_fs'
sched.o:sched.c:(.text+0x14d): first defined here
traps.o: In function `get_ds':
traps.c:(.text+0x3d6): multiple definition of `get_ds'
sched.o:sched.c:(.text+0x154): first defined here
traps.o: In function `set_fs':
traps.c:(.text+0x3dd): multiple definition of `set_fs'
sched.o:sched.c:(.text+0x15b): first defined here
fork.o: In function `get_fs_byte':
fork.c:(.text+0x0): multiple definition of `get_fs_byte'
sched.o:sched.c:(.text+0x10f): first defined here
fork.o: In function `get_fs_word':
fork.c:(.text+0x8): multiple definition of `get_fs_word'
sched.o:sched.c:(.text+0x117): first defined here
fork.o: In function `get_fs_long':
fork.c:(.text+0x11): multiple definition of `get_fs_long'
sched.o:sched.c:(.text+0x120): first defined here
fork.o: In function `put_fs_byte':
fork.c:(.text+0x19): multiple definition of `put_fs_byte'
sched.o:sched.c:(.text+0x128): first defined here
fork.o: In function `put_fs_word':
fork.c:(.text+0x25): multiple definition of `put_fs_word'
sched.o:sched.c:(.text+0x134): first defined here
fork.o: In function `put_fs_long':
fork.c:(.text+0x32): multiple definition of `put_fs_long'
sched.o:sched.c:(.text+0x141): first defined here
fork.o: In function `get_fs':
fork.c:(.text+0x3e): multiple definition of `get_fs'
sched.o:sched.c:(.text+0x14d): first defined here
fork.o: In function `get_ds':
fork.c:(.text+0x45): multiple definition of `get_ds'
sched.o:sched.c:(.text+0x154): first defined here
fork.o: In function `set_fs':
fork.c:(.text+0x4c): multiple definition of `set_fs'
sched.o:sched.c:(.text+0x15b): first defined here
vsprintf.o: In function `strcpy':
vsprintf.c:(.text+0x242): multiple definition of `strcpy'
traps.o:traps.c:(.text+0x182): first defined here
vsprintf.o: In function `strcat':
vsprintf.c:(.text+0x258): multiple definition of `strcat'
traps.o:traps.c:(.text+0x198): first defined here
vsprintf.o: In function `strcmp':
vsprintf.c:(.text+0x27b): multiple definition of `strcmp'
traps.o:traps.c:(.text+0x1bb): first defined here
vsprintf.o: In function `strspn':
vsprintf.c:(.text+0x29e): multiple definition of `strspn'
traps.o:traps.c:(.text+0x1de): first defined here
vsprintf.o: In function `strcspn':
vsprintf.c:(.text+0x2d1): multiple definition of `strcspn'
traps.o:traps.c:(.text+0x211): first defined here
vsprintf.o: In function `strpbrk':
vsprintf.c:(.text+0x304): multiple definition of `strpbrk'
traps.o:traps.c:(.text+0x244): first defined here
vsprintf.o: In function `strstr':
vsprintf.c:(.text+0x337): multiple definition of `strstr'
traps.o:traps.c:(.text+0x277): first defined here
vsprintf.o: In function `strlen':
vsprintf.c:(.text+0x36a): multiple definition of `strlen'
traps.o:traps.c:(.text+0x2aa): first defined here
vsprintf.o: In function `strtok':
vsprintf.c:(.text+0x383): multiple definition of `strtok'
traps.o:traps.c:(.text+0x2c3): first defined here
vsprintf.o: In function `memmove':
vsprintf.c:(.text+0x404): multiple definition of `memmove'
traps.o:traps.c:(.text+0x344): first defined here
vsprintf.o: In function `memchr':
vsprintf.c:(.text+0x42a): multiple definition of `memchr'
traps.o:traps.c:(.text+0x36a): first defined here
sys.o: In function `get_fs_byte':
sys.c:(.text+0x0): multiple definition of `get_fs_byte'
sched.o:sched.c:(.text+0x10f): first defined here
sys.o: In function `get_fs_word':
sys.c:(.text+0x8): multiple definition of `get_fs_word'
sched.o:sched.c:(.text+0x117): first defined here
sys.o: In function `get_fs_long':
sys.c:(.text+0x11): multiple definition of `get_fs_long'
sched.o:sched.c:(.text+0x120): first defined here
sys.o: In function `put_fs_byte':
sys.c:(.text+0x19): multiple definition of `put_fs_byte'
sched.o:sched.c:(.text+0x128): first defined here
sys.o: In function `put_fs_word':
sys.c:(.text+0x25): multiple definition of `put_fs_word'
sched.o:sched.c:(.text+0x134): first defined here
sys.o: In function `put_fs_long':
sys.c:(.text+0x32): multiple definition of `put_fs_long'
sched.o:sched.c:(.text+0x141): first defined here
sys.o: In function `get_fs':
sys.c:(.text+0x3e): multiple definition of `get_fs'
sched.o:sched.c:(.text+0x14d): first defined here
sys.o: In function `get_ds':
sys.c:(.text+0x45): multiple definition of `get_ds'
sched.o:sched.c:(.text+0x154): first defined here
sys.o: In function `set_fs':
sys.c:(.text+0x4c): multiple definition of `set_fs'
sched.o:sched.c:(.text+0x15b): first defined here
exit.o: In function `get_fs_byte':
exit.c:(.text+0x0): multiple definition of `get_fs_byte'
sched.o:sched.c:(.text+0x10f): first defined here
exit.o: In function `get_fs_word':
exit.c:(.text+0x8): multiple definition of `get_fs_word'
sched.o:sched.c:(.text+0x117): first defined here
exit.o: In function `get_fs_long':
exit.c:(.text+0x11): multiple definition of `get_fs_long'
sched.o:sched.c:(.text+0x120): first defined here
exit.o: In function `put_fs_byte':
exit.c:(.text+0x19): multiple definition of `put_fs_byte'
sched.o:sched.c:(.text+0x128): first defined here
exit.o: In function `put_fs_word':
exit.c:(.text+0x25): multiple definition of `put_fs_word'
sched.o:sched.c:(.text+0x134): first defined here
exit.o: In function `put_fs_long':
exit.c:(.text+0x32): multiple definition of `put_fs_long'
sched.o:sched.c:(.text+0x141): first defined here
exit.o: In function `get_fs':
exit.c:(.text+0x3e): multiple definition of `get_fs'
sched.o:sched.c:(.text+0x14d): first defined here
exit.o: In function `get_ds':
exit.c:(.text+0x45): multiple definition of `get_ds'
sched.o:sched.c:(.text+0x154): first defined here
exit.o: In function `set_fs':
exit.c:(.text+0x4c): multiple definition of `set_fs'
sched.o:sched.c:(.text+0x15b): first defined here
signal.o: In function `get_fs_byte':
signal.c:(.text+0x0): multiple definition of `get_fs_byte'
sched.o:sched.c:(.text+0x10f): first defined here
signal.o: In function `get_fs_word':
signal.c:(.text+0x8): multiple definition of `get_fs_word'
sched.o:sched.c:(.text+0x117): first defined here
signal.o: In function `get_fs_long':
signal.c:(.text+0x11): multiple definition of `get_fs_long'
sched.o:sched.c:(.text+0x120): first defined here
signal.o: In function `put_fs_byte':
signal.c:(.text+0x19): multiple definition of `put_fs_byte'
sched.o:sched.c:(.text+0x128): first defined here
signal.o: In function `put_fs_word':
signal.c:(.text+0x25): multiple definition of `put_fs_word'
sched.o:sched.c:(.text+0x134): first defined here
signal.o: In function `put_fs_long':
signal.c:(.text+0x32): multiple definition of `put_fs_long'
sched.o:sched.c:(.text+0x141): first defined here
signal.o: In function `get_fs':
signal.c:(.text+0x3e): multiple definition of `get_fs'
sched.o:sched.c:(.text+0x14d): first defined here
signal.o: In function `get_ds':
signal.c:(.text+0x45): multiple definition of `get_ds'
sched.o:sched.c:(.text+0x154): first defined here
signal.o: In function `set_fs':
signal.c:(.text+0x4c): multiple definition of `set_fs'
sched.o:sched.c:(.text+0x15b): first defined here
ld: Relocatable linking with relocations from format elf32-i386 (sched.o) to format elf64-x86-64 (kernel.o) is not supported
Makefile:32: recipe for target 'kernel.o' failed
```

找到对应的函数，将他们的 `extern` 改成 `static`


#### 0x09
```
ld -r -o kernel.o sched.o system_call.o traps.o asm.o fork.o panic.o printk.o vsprintf.o sys.o exit.o signal.o mktime.o
ld: Relocatable linking with relocations from format elf32-i386 (sched.o) to format elf64-x86-64 (kernel.o) is not supported
```

`kernel/Makefile`

```makefile
kernel.o: $(OBJS)
	$(LD) -r -o kernel.o $(OBJS)
	sync
```

改成

```makefile
kernel.o: $(OBJS)
	$(LD) -m elf_i386 -r -o kernel.o $(OBJS)
	sync
```


#### 0x0A
```
exec.c: In function ‘copy_strings’:
exec.c:139:44: error: lvalue required as left operand of assignment
         !(pag = (char *) page[p/PAGE_SIZE] =
                                            ^
```

修改

```c
if (!(pag = (char *) page[p/PAGE_SIZE]) &&
    !(pag = (char *) page[p/PAGE_SIZE] =
       (unsigned long *) get_free_page())) 
     return 0;
```

改成

```c
if (!(pag = (char *) page[p/PAGE_SIZE])) {
      page[p/PAGE_SIZE] = (unsigned long *) get_free_page();
      if (!(pag = (char *)page[p/PAGE_SIZE]))
              return 0;
}   
```

#### 0x0B
```
malloc.c: In function ‘malloc’:
malloc.c:156:46: error: lvalue required as left operand of assignment
   bdesc->page = bdesc->freeptr = (void *) cp = get_free_page();
                                              ^
```

修改

```c
bdesc->page = bdesc->freeptr = (void *) cp = get_free_page();
```

改为

```c
cp = (char*)get_free_page();
bdesc->page = bdesc->freeptr = (void *) cp;
```


#### 0x0C
```
keyboard.S: Assembler messages:
keyboard.S:47: Error: `%al' not allowed with `xorl'
```

修改 `keyboard.S:47`

```asm
xorl %al,%al		/* %eax is scan code */
```

改成

```asm
xorb %al,%al		/* %eax is scan code */
```

#### 0x0D
```
ld: i386 architecture of input file `boot/head.o' is incompatible with i386:x86-64 output
ld: i386 architecture of input file `init/main.o' is incompatible with i386:x86-64 output
ld: i386 architecture of input file `kernel/kernel.o' is incompatible with i386:x86-64 output
ld: i386 architecture of input file `mm/mm.o' is incompatible with i386:x86-64 output
ld: i386 architecture of input file `fs/fs.o' is incompatible with i386:x86-64 output
ld: warning: cannot find entry symbol _start; defaulting to 0000000008048098
```

同上，LDFLAGS 里加入 `-m elf_i386`，
给 `head.s` 的 `.text` 段添加一句 `.globl startup_32`，
然后给根目录下的 `Makefile` 中的 `ld` 加上选项 `-e startup_32 -Ttext 0` 以指定入口点。


#### 0x0E
```
boot/head.o: In function `startup_32':
(.text+0x10): undefined reference to `_stack_start'
(.text+0x2e): undefined reference to `_stack_start'
boot/head.o: In function `after_page_tables':
(.text+0x540c): undefined reference to `_main'
boot/head.o: In function `ignore_int':
(.text+0x5440): undefined reference to `_printk'
```

曾经的 C 语言也存在 name mangle，即所有 C 的符号在编译后都会多一个下划线的前缀，
而现在的 C 语言编译器不会这么做，所以把上面出现的那些符号手动 demangle 了~


#### 0x0F
```
init/main.o: In function `_printf':
main.c:(.text+0xa): undefined reference to `__GLOBAL_OFFSET_TABLE_'
init/main.o: In function `_fork':
main.c:(.text+0x44): undefined reference to `__GLOBAL_OFFSET_TABLE_'
init/main.o: In function `_pause':
main.c:(.text+0x71): undefined reference to `__GLOBAL_OFFSET_TABLE_'
init/main.o: In function `_sync':
main.c:(.text+0x9e): undefined reference to `__GLOBAL_OFFSET_TABLE_'
init/main.o: In function `_init':
main.c:(.text+0xd2): undefined reference to `__GLOBAL_OFFSET_TABLE_'
```

所有的 CFLAGS 加上 `-fno-pie`


#### 0x10
```
kernel/chr_drv/chr_drv.a(console.o): In function `_con_write':
console.c:(.text+0x329): undefined reference to `_attr'
```

修改 `console.c:77`

```c
static unsigned char	attr=0x07;
```

改成

```c
unsigned char	attr=0x07;
```


#### 0x11
```
/usr/include/stdio.h:27:10: fatal error: bits/libc-header-start.h: No such file or directory
 #include <bits/libc-header-start.h>
          ^~~~~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
```

```shell
$ apt install gcc-multilib
```


#### 0x12
```
build.c:(.text+0x10a): undefined reference to `MAJOR'
build.c:(.text+0x124): undefined reference to `MINOR'
```

从 `include/linux/fs.h` 里复制这两个宏到 `tools/build.c`

#### 0x13
```
Non-GCC header of 'system'
Makefile:41: recipe for target 'Image' failed
```

修改 `tools/build.c:161`

```c
if (((long *) buf)[5] != 0)
    die("Non-GCC header of 'system'");
```

改成

```c
if (((long *) buf)[6] != 0)
    die("Non-GCC header of 'system'");
```


## 为了进一步的调试

首先需要把所有的 Makefile 里的 CFLAGS 中的 `-O` 改成 `-g`，
并且把 LDFLAGS 中的 `-x` 删掉。

调试中遇到了很多问题，可以参考[这篇文章](http://blog.chinaunix.net/uid-23917107-id-3173253.html)


## 不想折腾？
直接下载吧！
[linux-0.11-fix.tar.gz](http://mirror.imkiva.com/linux-0.11-fix.tar.gz)

## ~~实用例子~~

#### 必备环境
```shell
$ sudo apt install build-essential make gcc-multilib bin86 qemu
```

#### 下载 & 编译
```shell
$ cd /some/fucking/directory
$ wget http://mirror.imkiva.com/linux-0.11-fix.tar.gz
$ tar zxvf linux-0.11-fix.tar.gz
$ cd linux-0.11
$ make
```

#### 运行 & 调试
```shell
$ make run
```

或者使用 `gdb` 调试

```shell
$ make debug
```

在另一个终端中输入

```shell
$ gdb tools/system
  ...
(gdb) target remote :1234
(gdb) b main
(gdb) r
```
