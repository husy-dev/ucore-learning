---
title: 【ucore】lab1-2
date: 2018-06-13 11:21:03
tags:
- ucore
- OS
---
## 实验内容

为了熟悉使用qemu和gdb进行的调试工作，我们进行如下的小练习：

1. 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。

2. 在初始化位置0x7c00设置实地址断点,测试断点正常。

3. 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。

4. 自己找一个bootloader或内核中的代码位置，设置断点并进行测试。

## 前置知识

 首先我们注意到makefile的这些规则:

```makefile
# files for grade script
.PHONY: qemu qemu-nox debug debug-nox
qemu-mon: $(UCOREIMG)
	$(V)$(QEMU)  -no-reboot -monitor stdio -hda $< -serial null
qemu: $(UCOREIMG)
	$(V)$(QEMU) -no-reboot -parallel stdio -hda $< -serial null
log: $(UCOREIMG)
	$(V)$(QEMU) -no-reboot -d int,cpu_reset  -D q.log -parallel stdio -hda $< -serial null
qemu-nox: $(UCOREIMG)
	$(V)$(QEMU)   -no-reboot -serial mon:stdio -hda $< -nographic
TERMINAL        :=gnome-terminal
debug: $(UCOREIMG)
	$(V)$(QEMU) -S -s -parallel stdio -hda $< -serial null &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -tui -x tools/gdbinit"
	
debug-nox: $(UCOREIMG)
	$(V)$(QEMU) -S -s -serial mon:stdio -hda $< -nographic &
	$(V)sleep 2
	$(V)$(TERMINAL) -e "gdb -q -x tools/gdbinit"

```

可以看到有各种各样的伪指令,也就是没有目标文件生成的指令.我们使用```make debug```这条指令模拟开机的同时进入debug模式.输出如下:

```bash
0x000f0090 in ?? ()
Breakpoint 1 at 0x100006: file kern/init/init.c, line 19.

Breakpoint 1, kern_init () at kern/init/init.c:19
(gdb) 
```

发现这里已经有一个断点了,为什么?仔细看debug这条伪指令, 下面是来自[GDB](https://sourceware.org/gdb/current/onlinedocs/gdb/Mode-Options.html#index-_002d_002dquiet)的参考

> 1. `-q` 
> “Quiet”. Do not print the introductory and copyright messages. These messages are also suppressed in batch mode.
> 2. `-tui`
> Activate the *Text User Interface* when starting. The Text User Interface manages several text windows on the terminal, showing source, assembly, registers and GDB command outputs 
> 3. `-x file`
>    Execute commands from file file. 

所以gdb的开启命令表示,

 	1. 不打印任何介绍或者版权信息,
		2. 开启Text User Interface,也就是在文本窗口里展示多个输出,比如源代码,汇编代码,寄存器等的信息.这样帮助我们更好地调试.
		3. 从当前目录下的tools/gdbinit文件读取命令,一行一行依次执行.

然后我们看tools/gdbinit文件里的内容

```
file bin/kernel
target remote :1234
break kern_init
continue
```

参考:

>`file filename`
>
>Use filename as the program to be debugged.

因此我们大概知道make gdb后gdb把bin/kernel当作debug的程序,并且在kern_init处设置了断点.但是我们是要从BIOS处开始debug,所以我们可以修改一下gdbinit来达到我们的目的.当然也可以使用make qemu

[qemu](https://qemu.weilnetz.de/doc/qemu-doc.html#pcsys_005fmonitor),也就可以参考[ucore的说明](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab0/lab0_2_4_2_1_qemu_runtime_arguments.html)

> 1. -no-reboot: Exit instead of rebooting.
>
> 2. -hda file
>
>    -hdb file
>
>    -hdc file
>
>    -hdd file
>
>    Use file as hard disk 0, 1, 2 or 3 image
>
> 3.  -serial dev
>
>    Redirect the virtual serial port to host character device dev. The default device is `vc` in graphical mode and `stdio` in non graphical mode.
>
>    -serial null: void device



OK, 那么我们要单步追踪BIOS的话,因为这个BIOS并不在我们写的文件里.但是我们知道BIOS的执行位置其实是约定俗成的,具体参考[BIOS启动过程](https://chyyuu.gitbooks.io/ucore_os_docs/content/lab1/lab1_3_1_bios_booting.html). PC通电后会自动把CS=0xf000, IP=0xfff0.因此我们可以在内存0xffff0处设置一个断点.

## 从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行
由于现在我们假设的架构是i8086,也就是16位模式实模式.我们知道PC加点后的CS:IP=0xF000:0XFFF0. 那么我们可以给这个地址加一个断点看看.将tools/gdbinit设置成如下:
```
set architecture i8086
target remote localhost:1234
break *0xfff0
continue
x /2i $pc
```

- [ ] 然而输出很奇怪


## 在初始化位置0x7c00设置实地址断点,测试断点正常

比如将tools/gdbinit改成下面:

```
set architecture i8086
target remote localhost:1234
break *0x7c00
continue
x /2i $pc
```

于是输出便是:

```The target architecture is assumed to be i8086
0x0000fff0 in ?? ()
Breakpoint 1 at 0x7c00

Breakpoint 1, 0x00007c00 in ?? ()
=> 0x7c00:      cli
   0x7c01:      cld
```

接下来的不再赘述.