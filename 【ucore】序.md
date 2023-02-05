---
title: 【ucore】序
date: 2018-03-24 11:31:06
tags:
- ucore
---

## 关于
这是一个关于清华大学ucore的实验博客。鉴于ucore提供了完整的答案，因此这里并不会提供具体的答案。更多的是理论与逻辑上的讨论思考，以及做实验时的一些感悟。

## 动机
第一次自学ucore这个课程，陷入了太多的细节，导致花了很多不必要的时间。事实上，这些细节相对于真正的操作系统来讲是很粗陋的，我们的目的是通过实验获取对理论的更深认识，而不是真的实现一个OS。因此，我希望能借这一系列的博客，第二次再来上这个课，有一些提纲挈领的感悟。

## 实验环境准备
1. OS： ubuntu18.04 LST
2. [qemu安装](https://www.qemu.org/download/)
3. [实验代码](https://github.com/chyyuu/ucore_os_lab)

### 命令
1. qemu
+ 首先是进入代码目录，make生成./bin/ucore.img
+ 这里makefile里提供了一个命令make qemu等。但下面这种更灵活。
+ 然后打开qemu-system-i386 ucore.img就能打开模拟控制台界面。
+ 这个里有很多参数可以调整，具体参考[qemu documents](https://qemu.weilnetz.de/doc/qemu-doc.html)
2. gdb的远程调试。
+ 首先```qemu-system-i386 -S -s -hda <ucore.img的路经> -monitor stdio```主要是参数-S -s要加上。
+ 然后打开一个新的终端，打开gdb，开始监听target remote 127.0.0.1:1234
+ 在gdb命令行下输入file <kernel的路径>，这样就能获取符号表，从而达到源码级别的调试。
+ 接下来设置断点，运行输出这些就是正常操作了。
+ 更为方便的方法是创建一个gdbinit文件，输入
>   target remote 127.0.0.1:1234
    file .bin/kernel
然后就可以直接输入命令 ```gdb -x <gdbinit的路径>```了。这里源代码在tools里已经帮我们创好了。
3. 说了这么多只是为了说明下面几个源代码的makefile里已经给我们写好的命令。
+ 直接编译加执行qemu  ```$make qemu```
+ 检查实验得分 ```$make grade```
+ 调试（使用gdb) ```$make debug```
+ 提交代码 ```$make handin```