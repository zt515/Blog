---
title: Linux LVM 更改物理分区大小
date: 2017-10-21 18:05:19
tags: linux
---

## 什么是 LVM
以下文字摘自`百度百科`
> LVM是逻辑盘卷管理（LogicalVolumeManager）的简称，它是Linux环境下对磁盘分区进行管理的一种机制，LVM是建立在硬盘和 分区之上的一个逻辑层，来提高磁盘分区管理的灵活性。通过LVM系统管理员可以轻松管理磁盘分区，如：将若干个磁盘分区连接为一个整块的卷组 （volumegroup），形成一个存储池。管理员可以在卷组上随意创建逻辑卷组（logicalvolumes），并进一步在逻辑卷组上创建文件系 统。管理员通过LVM可以方便的调整存储卷组的大小，并且可以对磁盘存储按照组的方式进行命名、管理和分配，例如按照使用用途进行定义：“development”和“sales”，而不是使用物理磁盘名“sda”和“sdb”。而且当系统添加了新的磁盘，通过LVM管理员就不必将磁盘的 文件移动到新的磁盘上以充分利用新的存储空间，而是直接扩展文件系统跨越磁盘即可。

<!--more-->

## 场景再现
我的电脑上本来只有一个 Ubuntu 系统，今天因为一些工作原因需要使用 Windows，而虚拟机那个效率太低了，明显不能满足我的需要（我需要在 Windows 上安装一系列开发工具并测试），于是决定装双系统。

但是，在安装 Ubuntu 的时候我为了省事就直接使用了 Ubuntu 的默认配置，即**使用整个磁盘和 LVM**，我按照原先的 `parted` 方案来更改分区时，parted 无法直接更改 `/dev/sda1` 的大小为我选的大小，后来了解到，要先缩减 lvm 分区大小才可以更改物理分区大小。

具体做法如下

## 第一部分：逻辑分区

0. 进入 live cd
1. 使用 `e2fsck` 检查分区可用空间，否则 `resize2fs` 会报错，`<path to lvm device>` 换成你的 lvm 挂载分区，一般为 `/dev/<VG>/<LV>`，比如我的就是 `/dev/ubuntu-vg/root`
```shell
sudo e2fsck -f <path to lvm device>
```

2. 使用 `resize2fs` 更改分区大小，`<size>` 为你需要的大小，可以用 `K`, `M`, `G` 等等做单位，比如 `100 MB` 就填 `100M`
```shell
sudo resize2fs <path to lvm device> <size>
```

3. 使用 `lvmreduce` 缩减 lvm 分区大小，注意：这里的 `<size>` 要跟上面的一样
```shell
sudo lvreduce -L <size> <path to lvm device>
```

现在，初步操作我们做完了，但是如果去 parted 里看，依旧不能更改物理分区的大小，因为前面我们修改的只是逻辑分区，现在我们开始对物理分区下手

## 第二部分：物理分区
我们可以使用 `lvm pvresize --setphysicalvolumesize` 来更改分区大小，但是我在这样做的时候，发现了一个新的问题：
```shell
ubuntu@ubuntu:~$ sudo lvm pvresize --setphysicalvolumesize 345G /dev/sda1
  /dev/sda1: cannot resize to 88319 extents as 89079 are allocated.
  0 physical volume(s) resized / 1 physical volume(s) not resized
```

原来，在更改物理分区之前，我们要重新分配未分配的空间到 LVM 的最后面，即：我们上面的情况中， `/dev/sda1` 开头部分是 lvm 分区，中间部分是空闲空间（上一步中减少出来的），结尾部分是 lvm 分区，我们得把 lvm 分区都放在一块去，把空间部分放到最后，这样就不会造成逻辑分区的数据丢失，具体做法如下：

1. 查看逻辑分区的具体分布情况：
```shell
ubuntu@ubuntu:~$ sudo pvs -v --segments /dev/sda1
```
 我们会看到类似这样的东西:
```shell
  PV         VG        Fmt  Attr PSize   PFree   Start  SSize LV     Start Type   PE Ranges              
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g      0 87040 root       0 linear /dev/sda1:0-87039      
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g  87040 30153            0 free                          
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g 117193  2039 swap_1     0 linear /dev/sda1:117193-119231
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g 119232     2            0 free   
```
2. 看到上面输出的 `root` 和 `swap_1` 之间的 `free` 了没？那个 `swap_1` 就是我们要移动的东西。记住 swap_1 后面的那一串 `/dev/sda1:117193-119231`，接下来要用到：
```shell
ubuntu@ubuntu:~$ sudo pvmove --alloc anywhere /dev/sda1:117193-119231
```
 静待完成，可能需要一段时间
```shell
  /dev/sda1: Moved: 0.00%
  /dev/sda1: Moved: 4.71%
  ...
  /dev/sda1: Moved: 100.00%
```

3. 如果不出意外的话，到这里就完成了，为了确认，我们再看看：
```shell
ubuntu@ubuntu:~$ sudo pvs -v --segments /dev/sda1
  PV         VG        Fmt  Attr PSize   PFree   Start SSize LV     Start Type   PE Ranges            
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g     0 87040 root       0 linear /dev/sda1:0-87039    
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g 87040  2039 swap_1     0 linear /dev/sda1:87040-89078
  /dev/sda1  ubuntu-vg lvm2 a--  465.76g 117.79g 89079 30155            0 free    
```
 不出所料，`free` 跑到了 `swap_1` 的后面

4. 接下来呢？当然是 `sudo parted` 咯～

## 后话
数据无价，请做好备份！！！

## 文中参考的资料
> [Ask Ubuntu - How to shrink Ubuntu LVM logical and physical volumes?](https://askubuntu.com/questions/252204/how-to-shrink-ubuntu-lvm-logical-and-physical-volumes)
