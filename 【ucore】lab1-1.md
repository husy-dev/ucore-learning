---
title: 【ucore】lab1-1
date: 2018-06-12 11:19:31
tags:
- ucore
- OS
---

## 前置知识
1. [跟我一起写Makefile](https://seisman.github.io/how-to-write-makefile/overview.html)
1. [GUN make](https://www.gnu.org/software/make/manual/html_node/index.html#SEC_Contents)
1. [ucore中Makefile 内核文件组织全解析，学习软件的组织方式](http://blog.csdn.net/u013484370/article/details/50638353)

### 注意
1. 命令部分的语法主要是来自于UNIX下的标准shell语法
1. 变量的取值是简单的文本替换 

## 操作系统镜像文件ucore.img是如何一步一步生成的？
* 程序静态分析（Program Static Analysis）是指在不运行代码的方式下，通过词法分析、语法分析、控制流、数据流分析等技术对程序代码进行扫描，验证代码是否满足规范性、安全性、可靠性、可维护性等指标的一种代码分析技术。
* 首先我们要知道这个make过程,大部分是通过变量定义来实现编译的.而不是最普通的目标:前置文件;生成命令这种形式. 因为初始化参数实在分析规则前.
  * makefile的步骤是首先定义了一些参数, 然后include tools/function.mk.
  * 这个文件里虽然没有目标文件,但其实通过define cc_template这个变量,定义了很多生成,o文件的命令.
  * 在makefile里通过传入bootblock和kernel的源文件,使用function.mk生成.o文件.
* 所以其实我们的第一条正式的target,是\$(kernel): tools/kernel.ld.但是它是一个目标文件多规则,所以最终会执行kernel的第二个规则:\$(kernel): $(KOBJS)......
  * 这个规则主要是给编译后的kernel中间文件做链接.
  * 然后生成一些asm文件,也就是汇编代码和c语言对应的文件
* 然后是生成bootblock,和kernel类似,只不过多了一步启动区的启动扇区签名的步骤，即有效启动扇区，最后字节为0x55aa。而这一步是由sign来做的.
  *  但是这一步为什么会执行呢?变量会自动生成,但是这是一个生成规则,我们的第一个target也就是kernel并没有把bootblock作为前置文件,那么make就不会执行这一条规则才对?
* 最后ucore.img的操作是模拟一个硬盘,通过dd把kernel和bootblock都写进去.
## ucore.img
```makefile
# create ucore.img
UCOREIMG:= $(call totarget,ucore.img)
$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
$(call create_target,ucore.img)
```
这个是生成ucore.img的代码,这里可以看到ucore.img是以kernel和bootblock为前置条件,然后通过dd命令写入10000byte的0到目标文件,然后把bootblock和kernel都写进去.至于totarget是在function.mk里定义的,其中BINDIR在makefile里定义的,等于bin
```makefile
totarget = $(addprefix $(BINDIR)$(SLASH),$(1))
```
所以,这句话的作用就是给$(1)加一个前缀bin/,所以UCOREIMG:= $(call totarget,ucore.img)整句话的意思就是给ucore.img加个前缀.让UCOREIMG=bin/ucore.img,那么我生成的ucore.img就会存放在bin目录下.

### 输出内容检查

> dd if=/dev/zero of=bin/ucore.img count=10000
> dd if=bin/bootblock of=bin/ucore.img conv=notrunc
> dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc

## bootblock
```makefile
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
bootblock = $(call totarget,bootblock)
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
$(call create_target,bootblock)
```
一句句看:

#### 找到编译bootblock的源文件

```makefile
bootfiles = $(call listf_cc,boot)
#与此有关的是:
listf_cc = $(call listf,$(1),$(CTYPE))
listf = $(filter $(if $(2),$(addprefix %.,$(2)),%),\$(wildcard $(addsuffix $(SLASH)*,$(1))))
CTYPE	:= c S
```
listf的作用是做一个fliter, 把\$(1)目录下的所有文件(因为这里是一个*的通配符,)留下类似于%.\$(2)形式的.也就是挑选$(1)下后缀是在\$(2)中的文件. CTYPE中给出了c和S,也就是把所有boot下.S和.c文件都返回给bootfiles

#### 编译成.o文件

```makefile
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
#与此有关的代码有:
define do_cc_compile
$$(foreach f,$(1),cc_template,$$(f),$(2),$(3),$(4))))
endef

cc_compile = $(eval $(call do_cc_compile,$(1),$(2),$(3),$(4)))

define cc_template
$$(call todep,$(1),$(4)): $(1) | $$$$(dir $$$$@)
	@$(2) -I$$(dir $(1)) $(3) -MM $$< -MT "$$(patsubst %.d,%.o,$$@) $$@"> $$@
$$(call toobj,$(1),$(4)): $(1) | $$$$(dir $$$$@)
	@echo + cc $$<
	$(V)$(2) -I$$(dir $(1)) $(3) -c $$< -o $$@
ALLOBJS += $$(call toobj,$(1),$(4))
endef
```
- \$(eval text)的作用是把text作为makefile的一部分通过make解析.因为一般来讲,makefile里的函数都是通过参数动态生成命令传递给shell执行,也就是gcc之类的. 但是eval是一个可以通过参数动态生成makefile自身代码的函数.所以要注意的是, 如果在text中有参数,要考虑到eval函数的两次展开.因为text中的一个\$var通过eval后呈现给make的makefile指令是var,所以如果我们是要给makefile呈现出一个\$var则var应该写成\$\$var.

- 于是我们知道其实最终的代码主要是写在do\_cc\_template里,而网上的包含关系是:do\_cc\_template->cc\_template->do_cc_compile->cc_compile->\$(bootblock)下的命令 ,而没脱去一次父调用,eval函数就要做一次对变量的取值操作,相当于去掉一个\$.但是对于临时变量\$(1)等不要变.最后一层层解析上去后的代码是:

  - cc\_compile最终会呈现的语法是:```$(foreach f,$(1),$(eval $(call cc_template,$(f),$(2),$(3),$(4))))```

  - 于是第一行对cc_compile有调用, 代码变成对于每一个在\$(1)的f执行

    ```makefile
    $(call todep,$(1),$(4)): $(1) | $$(dir $$@)
    	@$(2) -I$(dir $(1)) $(3) -MM $< -MT "$(patsubst %.d,%.o,$@) $@"> $@
    $(call toobj,$(1),$(4)): $(1) | $$(dir $$@)
    	@echo + cc $<
    	@$(2) -I$(dir $(1)) $(3) -c $< -o $@  #注意前面的$(V)已经被解析成@
    ALLOBJS += $(call toobj,$(1),$(4))
    ```

于是我们看参数```CC:= $(GCCPREFIX)gcc```, 细节太多,可以慢慢去代入,一步步推敲,总的来说这一句话就把boot/目录下所有的.c和.S都编译成了.o文件.

- 编辑目标文件的位置

```bootblock = $(call totarget,bootblock)```易知这一步就是把生成的bootblock放到bin/目录下

#### 开始第二条bootblock规则

```@echo + ld $@```打印"+ ld <所有当前目录下的文件名>,然后第二条就是ld命令,也就是对生成的.o文件进行链接.

最后两个OBJDUMP和OBJCOPY的命令主要是gcc里的命令,用来输出汇编文件与C语言对应输出的.asm文件.

#### 查找对应的输出语句

+ 对于```$(V)$(2) -I$$(dir $(1)) $(3) -c $$< -o $$@```这一句,由于我们使用的make V:=因此这里的输出没有被屏蔽, 可以从输出里看到

+ > gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
  >
  > gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o

+ 注意到call toobj处下面有一行命令```+ cc $$<```这里可以对应make输出文件里一系列+ cc 前置文件名
  > \+ cc boot/bootasm.S
  > \+ cc boot/bootmain.c
  > \+ cc tools/sign.c

+ 而@echo + ld $@也可以找到其输出为:\+ ld bin/bootblock 

还有一些都是可以一一对应的.


## kernel

然后我们可以看看前置条件之一的kernel的规则:
```makefile
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
$(call create_target,kernel)
```
这里的\$(kernel)同理可得是bin/kernel,然后kernel规则有两个一个的前置条件是tools/kernel.ld,一个是$(KOBJS). \$(KOBJS)的定义如下:
```makefile
KOBJS	= $(call read_packet,kernel libs)
#有关的代码
read_packet = $(foreach p,$(call packetname,$(1)),$($(p)))
# change $(name) to $(OBJPREFIX)$(name): (#names)
packetname = $(if $(1),$(addprefix $(OBJPREFIX),$(1)),$(OBJPREFIX))
OBJPREFIX	:= __objs_
```

所以这里\$(1)就是kernel libs,依次对\_\_objs\_/kernel和\_\_objs\_/libs做\$($(p))操作.不太懂这里要做什么

然后就是输出 + ld bin/kernel,然后就是开始链接操作,需要注意的是, 这里-o后面有两个参数,一个$@可以理解,生成当前的目标文件bin/kernel, 然而还要生成KOBJ, 那么可以猜到KOBJ是我们文件里多出来的obj/目录下的kern文件夹和libs文件夹.


##一个被系统认为是符合规范的硬盘主引导扇区的特征是什么
###makefile部分

我们知道给引导扇区签名的逻辑是写在sign.c里的.那么我们先来看看相关代码:
首先是makefile里的:

```makefile
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)#1
$(call create_target_host,sign,sign)#2
# 相关代码

# for hostcc
add_files_host = $(call add_files,$(1),$(HOSTCC),$(HOSTCFLAGS),$(2),$(3))
create_target_host = $(call create_target,$(1),$(2),$(3),$(HOSTCC),$(HOSTCFLAGS))

# add files to packet: (#files, cc[, flags, packet, dir])
add_files = $(eval $(call do_add_files_to_packet,$(1),$(2),$(3),$(4),$(5)))

# add files to packet: (#files, cc[, flags, packet, dir])
define do_add_files_to_packet
__temp_packet__ := $(call packetname,$(4)) #__objs_sign_
ifeq ($$(origin $$(__temp_packet__)),undefined)
$$(__temp_packet__) :=
endif
__temp_objs__ := $(call toobj,$(1),$(5))#obj/sign/tools/sign.o
$$(foreach f,$(1),$$(eval $$(call cc_template,$$(f),$(2),$(3),$(5))))
$$(__temp_packet__) += $$(__temp_objs__)
endef

# add packets and objs to target (target, #packes, #objs[, cc, flags])
define do_create_target
__temp_target__ = $(call totarget,$(1)) #bin/sign
__temp_objs__ = $$(foreach p,$(call packetname,$(2)),$$($$(p))) $(3)#__objs_sign_
TARGETS += $$(__temp_target__)
ifneq ($(4),)#clang不为空,进入逻辑 
$$(__temp_target__): $$(__temp_objs__) | $$$$(dir $$$$@)
	$(V)$(4) $(5) $$^ -o $$@#执行clang -g -Wall -O2 所有依赖文件 -o bin/sign
else
$$(__temp_target__): $$(__temp_objs__) | $$$$(dir $$$$@)
endif
endef

# change $(name) to $(OBJPREFIX)$(name): (#names)
packetname = $(if $(1),$(addprefix $(OBJPREFIX),$(1)),$(OBJPREFIX))

# get .o obj files: (#files[, packet])
toobj = $(addprefix $(OBJDIR)$(SLASH)$(if $(2),$(2)$(SLASH)),\
		$(addsuffix .o,$(basename $(1))))

OBJDIR	:= obj
OBJPREFIX	:= __objs_
HOSTCC		:= clang
HOSTCFLAGS	:= -g -Wall -O2
```
上面的注释写得很清楚,第一条指令是通过cc_compile生成对应的sign.d和sign.o文件.第二条指令是通过clang指令生成bin/sign执行文件. 然后在bootblock的生成规则的命令里最后一行有执行bin/sign这个执行文件.这样就达到了给bootblock签名的目的.为了下面一步的分析: 把执行命令写在这:

```makefile
@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
一个一个转化,也就是bin/sign obj/bootblock.out bin/bootblcok.
```

###sign.c部分

于是, 对于sign.c里的main函数来讲,其argc=3, argv[1]=obj/bootblock.argv[2]=out bin/bootblcok.一句句往下执行.

首先是一个[stat](http://pubs.opengroup.org/onlinepubs/7908799/xsh/sysstat.h.html)的结构st,然后是函数[stat](http://pubs.opengroup.org/onlinepubs/7908799/xsh/stat.html),```int stat(const char *path, struct stat *buf);```于是```stat(argv[1], &st)```的作用是把obj/bootblock.out的stat信息存到st里去,返回0表示读取成功. stat信息包括很多,比如

```
dev_t     st_dev     ID of device containing file
ino_t     st_ino     file serial number
mode_t    st_mode    mode of file (see below)
nlink_t   st_nlink   number of links to the file
uid_t     st_uid     user ID of file
gid_t     st_gid     group ID of file
dev_t     st_rdev    device ID (if file is character or block special)
off_t     st_size    file size in bytes (if file is a regular file)
time_t    st_atime   time of last access
time_t    st_mtime   time of last data modification
time_t    st_ctime   time of last status change
blksize_t st_blksize a filesystem-specific preferred I/O block size for
                     this object.  In some filesystem types, this may
                     vary from file to file
blkcnt_t  st_blocks  number of blocks allocated for this object
```

读取完后,就会有一个输出,从make的输出文件可以知道输出的是:'obj/bootblock.out' size: 488 bytes.

然后判断bootblock是不是小于510bytes,因此一个扇区是512,启动扇区只能占用一个扇区也就是512bytes.由于我们还要签名2bytes,所以代码部分最多是510bytes.

然后申请512bytes的内存,把obj/bootblock写进去

然后在第510byte里写0x55,第511byte里写0xAA.

最后把这段内存写进argv[2],也就是文件bin/bootblcok.

打印信息: build 512 bytes boot sector: 'bin/bootblock' success!

至此,一个启动扇区就做好了.

### 总结

一个合格的启动扇区是最后两byte的内容为0xAA55,然后不能实际内容不能超过510bytes.

## 遗留问题

1. 第一个target是生成kernel,那么bootblock和ucore.img不应该会执行,因为kernel生成规则里没有说kernel的生成依赖于bootblock和ucore.img.那么bootblock和ucore.img是怎样生成的呢?

2. ```bootfiles = $(call listf_cc,boot)
     \$(foreach f,\$(bootfiles),\$(call cc_compile,\$(f),\$(CC),\$(CFLAGS) -Os -nostdinc))```这里的foreach会执行么?这里既不是在规则的命令内,也不是变量.
     ```
  ```

3. KOBJS是什么,也就是对一个文件夹目录做\$(\$(p))有什么用?

4. \_\_objs\_是什么

5. tools/kernel.ld 这个是最后的生成的可执行文件的header信息么?

4. sign.c里使用了memst这个内存分配函数,这个内存分配在这个阶段是有谁分配的?我们自己电脑的操作系统?