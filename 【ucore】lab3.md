---
title: 【ucore】lab3
date: 2018-07-22 11:00:08
tags: 
- ucore
- OS
---
## 实验目标
 练习0：填写已有实验
 练习1：给未被映射的地址映射上物理页（需要编程）
 练习2：补充完成基于FIFO的页面替换算法（需要编程）
 扩展练习 Challenge：实现识别dirty bit的 extended clock页替换算法（需要编程）
数据结构分析
上一节已经设置好了物理内存以及分配方法.这一节主要是把虚拟内存和物理内存联系起来(也就是虚拟页和物理帧的对应,而物理帧我们有Pages数组来描述.所以其实是虚拟内存页的数据结构和ages的对应),并完成页面替换算法. 逻辑上来讲,我们要设立一个数据结构来描述虚拟内存,肯定要有下面的内容:

1. 对虚拟内存本身的描述: 起始位置(也许还有结束位置),某一写flags(可读可写?)
   2. 对对应物理页的描述: 对应的页帧数, 对应的物理页的地址
   3. 供页面替换算法使用的数据: 上一次访问时间,或者访问次数等等
看一下代码里是怎么写的:

```
/* *
 * struct Page - Page descriptor structures. Each Page describes one
 * physical page. In kern/mm/pmm.h, you can find lots of useful functions
 * that convert Page to other data types, such as phyical address.
 * */
struct Page {
    int ref;                        // page frame's reference counter
    uint32_t flags;                 // array of flags that describe the status of the page frame
    unsigned int property;          // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
    list_entry_t pra_page_link;     // used for pra (page replace algorithm)
    uintptr_t pra_vaddr;            // used for pra (page replace algorithm)
};

// the virtual continuous memory area(vma), [vm_start, vm_end), 
// addr belong to a vma means  vma.vm_start<= addr <vma.vm_end 
struct vma_struct {
    struct mm_struct *vm_mm; // the set of vma using the same PDT 
    uintptr_t vm_start;      // start addr of vma      
    uintptr_t vm_end;        // end addr of vma, not include the vm_end itself
    uint32_t vm_flags;       // flags of vma
    list_entry_t list_link;  // linear list link which sorted by start addr of vma
};

// 
struct mm_struct {
    list_entry_t mmap_list;        // linear list link which sorted by start addr of vma
    struct vma_struct *mmap_cache; // current accessed vma, used for speed purpose
    pde_t *pgdir;                  // the PDT of these vma
    int map_count;                 // the count of these vma
    void *sm_priv;                 // the private data for swap manager
};
```
### 代码片段解析

#### 练习一
这个实验是让我们处理do_pgfault函数的,而整个逻辑是:

产生页访问异常后 –> CPU把引起页访问异常的线性地址装到寄存器CR2中，并给出了出错码errorCode，说明了页访问异常的类型, OS把这个值保存在struct trapframe 中tf_err成员变量中。
中断服务例程会调用页访问异常处理函数do_pgfault进行具体处理
它根据从CPU的控制寄存器CR2中获取的页访问异常的物理地址以及根据errorCode的错误类型来查找此地址是否在某个VMA的地址范围内以及是否满足正确的读写权限
如果在此范围内并且权限也正确，这认为这是一次合法访问，但没有建立虚实对应关系。所以需要分配一个空闲的内存页，并修改页表完成虚地址到物理地址的映射，刷新TLB，然后调用iret中断，返回到产生页访问异常的指令处重新执行此指令。
如果该虚地址不在某VMA范围内，则认为是一次非法访问
首先页访问异常是CPU在找不到pa是的异常,这个异常是由硬件发出的. 那么我们代码可以不管.但是异常产生后,就是OS的责任来处理了.

根据lab1的中断分析,我们知道最终是由下面的函数来处理所有中断

```C
static void
trap_dispatch(struct trapframe *tf) {
    char c;
    int ret;
    switch (tf->tf_trapno) {
    case T_PGFLT:  //page fault
        if ((ret = pgfault_handler(tf)) != 0) {
            print_trapframe(tf);
            panic("handle pgfault failed. %e\n", ret);
        }
        break;
    case IRQ_OFFSET + IRQ_TIMER:
        if(!(++ticks%TICK_NUM)){
			print_ticks();
		}
        break;
   ......
    default:
        // in kernel, it must be a mistake
        if ((tf->tf_cs & 3) == 0) {
            print_trapframe(tf);
            panic("unexpected trap in kernel.\n");
        }
    }
}
```
于是当中断号是T_PGFLT时,我们就调用pgfault_handler来处理,处理不成功就打印栈调用信息. 而pgfault_handler代码如下:

```C
static int pgfault_handler(struct trapframe *tf) {
    extern struct mm_struct *check_mm_struct;
    print_pgfault(tf);
    if (check_mm_struct != NULL) {
        return do_pgfault(check_mm_struct, tf->tf_err, rcr2());
    }
    panic("unhandled page fault.\n");
}
```
这里就正式开始调用我们要填写的do_pgfault函数. 传入的参数是是一个mm_struct,一个中断里的错误,是由x86直接设置的,然后就是rcr2()函数, 这里有一个疑问,看遗留问题第二条. 有一个提示就是:

[提示]页访问异常错误码有32位。位0为１表示对应物理页不存在；位１为１表示写异常（比如写了只读页；位２为１表示访问权限异常（比如用户态程序访问内核空间的数据）

位2的访问权限异常在这个阶段还没有开始设置,因此这里暂且不说.

```C
int do_pgfault(struct mm_struct *mm, uint32_t error_code, uintptr_t addr) {
    int ret = -E_INVAL;
    struct vma_struct *vma = find_vma(mm, addr); //找到包含有addr的vma
    pgfault_num++;
    //先判断一下当前的虚拟地址是不是在我们的mm控制的范围里,说明当前虚拟地址是不合法的
    if (vma == NULL || vma->vm_start > addr) {//如果没找到,或者找到了但是地址不符合(一般是不会出现找到了但是地址不在vma所规定区间的)
        cprintf("not valid addr %x, and  can not find it in vma\n", addr);
        goto failed; //返回-E_INVAL也就是-3,代表无效参数
    }   
    //下面是找到了vma但是还是产生了访问页异常错误,我们根据error_code来判断是哪一种
    switch (error_code & 3) {
    default: //error_code=011,也就是物理页面不存在并且有写异常,那么就留到switch下面去处理
    case 2: //说明error_code是010,是写异常,物理页面存在
        if (!(vma->vm_flags & VM_WRITE)) { //判断要访问的虚拟地址的vm_flags是不是真的不可写
            cprintf("do_pgfault failed: error code flag = write AND not present, but the addr's vma cannot write\n");
            goto failed;
        }
        break;
    case 1: //error_code=001,也就是对应物理页面不存在.
        cprintf("do_pgfault failed: error code flag = read AND present\n");
        goto failed;
    case 0: //error_code=000,也就是物理页面存在,也没有写异常,但是还是出现了访问异常,就要看看读和执行是不是有问题
        if (!(vma->vm_flags & (VM_READ | VM_EXEC))) {//判断要访问的虚拟地址的vm_flags是不是真的不可读或者可执行.
            cprintf("do_pgfault failed: error code flag = read AND not present, but the addr's vma cannot read or exec\n");
            goto failed;
        }
    }
    //上面处理了一波后,执行到下面的要么是物理页面不存在,要么是访问权限异常,而写异常已经处理完了
    uint32_t perm = PTE_U;
    if (vma->vm_flags & VM_WRITE) {
        perm |= PTE_W;
    }
    addr = ROUNDDOWN(addr, PGSIZE);
    ret = -E_NO_MEM; //此时返回的错误代号是E_NO_MEM,意思是没有内存

    pte_t *ptep=NULL;
    ptep =get_pte(mm->pgdir,addr,1); //通过addr这个线性地址(按照代码注释来),返回对应的pte,如果没有get_pte会创一个pt,但是ptep是还是空的.
    if (*ptep == 0) { //如果ptep是空的,说明此时对应的页表项还没有物理页面和它对应,就要分配一个页面
  		pgdir_alloc_page(mm->pgdir,addr,perm)  //分配一个物理页面,权限是用户态的可写  
    }else{ //也就是说页表里有对应的物理地址,但是在内存里没找到,说明被换入了swap空间中.
     	 if(swap_init_ok) {
                struct Page *page=NULL;
                if(swap_in(mm,addr,&page)!=0){
                	cprintf("swap_init_ok but swap in function is failed\n");
                	goto failed;
                }
                page_insert(mm->pgdir,page,addr,perm);
                swap_map_swappable(mm,addr,page,1); //最后的swap_in=1应该是表示该页面可以换入swap里
             	page->pra_vaddr = addr;//这里很重要,我忘了写
         	  }else{
         		  cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);
         		  goto failed;
         	  }
        }
   ret = 0;
failed:
    return ret;
}
```

#### 练习二
这个实验主要是构造一个先入先出的页替换算法,而真正的替换其实在练习一已经完成了.这里主要是两个函数

_fifo_map_swappable: 维护FIFO算法所需要的数据结构,也就是mm->sm_priv所代表的一个list,把换进的页或者新读入物理内存的页放入这个list里.
_fifo_swap_out_victim:从上面的数据结构里得到应该换出的内存页,同时维护好这个list,也就是把换出的页从list里删掉
FIFO的话,其实就是每一次加入list的末尾, 链首的page就是应该换出的页.

注意,这里的作用仅仅是维护FIFO的数据结构,至于换出的页所对应的pte需要把present标志位改成0,还是如何换出去都不是这个函数来实现的.

## 知识点总结

### 操作系统理论
1. 产生页访问异常的原因主要有：
    - 目标页帧不存在（页表项全为0，即该线性地址与物理地址尚未建立映射或者已经撤销)；
    - 相应的物理页帧不在内存中（页表项非空，但Present标志位=0，比如在swap分区或磁盘文件上)，这在本次实验中会出现，我们将在下面介绍换页机制实现时进一步讲解如何处理；
    - 不满足访问权限(此时页表项P标志=1，但低权限的程序试图访问高权限的地址空间，或者有程序试图写只读页面).
1. 请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。
    我们在检测一个内存页是否是在物理内存里的方法就是通过PDE查找对应的PTE, 如果存在PTE(不全为0),但是present标志位为0,则说明我们需要进行页替换算法.

1.  如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
    要把页访问异常的错误代码压入到kernel stack中,然后把cs:ip指向中断服务例程.

### 代码层面
1. vma_struct里的list_link和mm_struct里的mmap_list是一个东西,逻辑上并无多大用处,只是为了代码上mm_struct能通过mmap_list访问所有的vma.以为如果vma_struct里只有数据部分,链接都由mm_struct来的话,其实是无法动态改变vma_struct这条链的,因为mmap_list只是提供一个入口,后面的部分要有vma_struct自己来.
1. 为了在页表项中区别 0 和 swap 分区的映射，将 swap 分区的一个 page 空出来不用. 这句话的意思是说 如果swap的24位映射是0的话,虽然是映射到swap分区的第0个扇区,但是由于此是全为0的pte与页目录不存在的pte看起来是一样的,所以干脆认为这个swap entry也是无效的. 是不可能存在某个内存页对应的物理内存是在swap空间的第0个扇区的.

## 遗留问题

1. cr2()的作用是返回一个从cr2寄存器里获取的页访问异常的物理地址.注意这里是的物理地址是指引起访问页异常的程序代码所在的物理地址,不是指当前要访问的但是却出现了异常没有获取到的物理地址.论坛是这么写的,但是代码里的注释写的是cr2里存放的是线性地址.见论坛

2. 类似这种swap_manager的代码处理模式的原理是什么?为什么可以这样做?

```C
struct swap_manager swap_manager_fifo =
{
     .name            = "fifo swap manager",
     .init            = &_fifo_init,
     .init_mm         = &_fifo_init_mm,
     .tick_event      = &_fifo_tick_event,
     .map_swappable   = &_fifo_map_swappable,
     .set_unswappable = &_fifo_set_unswappable,
     .swap_out_victim = &_fifo_swap_out_victim,
     .check_swap      = &_fifo_check_swap,
};
```

1. static int _fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in)
这里面的swap_in是代表什么?代表可以换出的意思还是?实验三的代码里还没有体现.
 page->pra_vaddr = addr;//这里很重要,我忘了写