---
title: 【ucore】lab2
date: 2018-07-12 11:38:18
tags:
- ucore
- OS
---

## 实验目标
1. 首先了解如何发现系统中的物理内存；
2. 然后了解如何建立对物理内存的初步管理，即了解连续物理内存管理；
3. 最后了解页表相关的操作，即如何建立页表来实现虚拟内存到物理内存之间的映射，对段页式内存管理机制有一个比较全面的了解。

## 练习1：实现 first-fit 连续物理内存分配算法（需要编程）

首先我们看一遍内存初始化的流程:
首先是/kern/init.c/kern\_init()里用pmm\_init()初始化内存设置
pmm\_init()是在kern/mm/pmm.c里, 主要做的事:

```C
static void init_pmm_manager(void) {
    pmm_manager = &default_pmm_manager;
    cprintf("memory management: %s\n", pmm_manager->name);
    pmm_manager->init();
}
```
这里其实就是通过default\_pmm_manager那个结构体设置好要pmm(page memory manager)的属性,比如它的名字,它分配内存用的方法,释放内存用的方法等.都在这里提供了统一的接口,我们只需要修改pmm\_manager的值就好.

然后就用pmm\_manager里的init接口执行对应的初始化. page_init();

这个函数主要是首先探测物理内存的大小,保留已经被使用的(比如硬件等),然后用pmm->init_memmap初始化空闲页的list

```C
boot_alloc_page()  //开始创建PDT,以及PT
boot_map_segment //做好PA和Va的映射.
enable_paging//通过改变enable page的标记通知硬件页表已经开始可以开始使用
```
细枝末节等后面再慢慢看.

### 重点
我们根据default_pmm.c的注释来看看细节:

首先是list_init,这个在list.h里
```C
static inline void list_init(list_entry_t *elm) {
    elm->prev = elm->next = elm;
}
```
这是一个内联静态函数,和普通的初始化双链表没什么特别的.只是这里的链表单元是list_entry_t,其实也就是list_entry.
```C
struct list_entry {
    struct list_entry *prev, *next;
};
```
这里不要有困惑,虽然看起来像是递归定义,实际上就是定义了两个指针作为一个结构体. 此时并没有在链表里存数据,只是单纯的构建一种前后连贯的关系.

然后我们要了解这个对这个list的一些操作list_add,list_del等等.这都是普通的数据结构. 不细讲.

但是这里的list_add_before和list_add_after对于这个free_area那个链来说,感觉是一样的.[TO-DO]

然后我们要知道如何让这个链表转化成各种特殊结构的链表. 举个例子–le2page.
```C
// convert list entry to page
\#define le2page(le, member) to_struct((le), struct Page, member)

/* *
 * to_struct - get the struct from a ptr
 * @ptr:    a struct pointer of member
 * @type:   the type of the struct this is embedded in
 * @member: the name of the member within the struct
 * */
\#define to_struct(ptr, type, member) ((type *)((char *)(ptr) - offsetof(type, member)))

/* Return the offset of 'member' relative to the beginning of a struct type */
\#define offsetof(type, member) ((size_t)(&((type *)0)->member))
```
这里是一个宏定义,to_struct的定义可以参考这个博客–利用宏来求结构体成员偏移值. 这个函数的作用就是当我们拿到一个地址时,我们就可以通过这个函数拿到这个地址所对应的页.

default_init已经帮我们写好了,free_list就是我们上面说的list_entry这个双链表, 用来记录空闲的内存块,nr_free记录空闲内存块的个数. 所以数据结构是free_area_t是总的数据结构,用来记录整个空闲内存的情况,它通过成员list_entry来记录空闲页链表的头和nr_free记录总共有的空闲内存大小(可以看到单位是页的数量,而不是bytes). 而list_entry则是通过两个指针讲空闲内存的地址穿起来.
default_init_memmap也就是pmm->init_memmap接口的默认实现. 主要的作用是初始化第一个空闲内存块(由于这里还是操作系统的部署时间,此时什么应用程序都没有运行,所以此时的第一个空闲内存块就是整个应用程序可以使用的内存.) 那么就要用的页这个结构
```C
/* *
 * struct Page - Page descriptor structures. Each Page describes one
 * physical page. In kern/mm/pmm.h, you can find lots of useful functions
 * that convert Page to other data types, such as phyical address.
 * */
struct Page {
    int ref;                // page frame's reference counter
    uint32_t flags;         // array of flags that describe the status of the page frame
    unsigned int property;  // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
};
```
初始化的过程是,首先应该根据内存大小创建好页的链表,然后把把ref都设为0,flags的PG位都设为保护模式. 然后property设为1,因为此时还只有一个free block. 随后page_link就是这个页的链表的入口.

```C
static void default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));
        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;
    list_add(&free_list, &(base->page_link));
}
```
\# 在page_init的函数里调用代码是:
init_memmap(pa2page(begin), (end - begin) / PGSIZE);
default_alloc_page这里就是我们内存分配的函数了. 我们的各种分配算法就是通过它来实现的.这个是练习一的内容.详情请看ucore-lab2-2.
然后我们看看页目录表和页表是如何创建的,
```C
// create boot_pgdir, an initial page directory(Page Directory Table, PDT)
boot_pgdir = boot_alloc_page(); //分配一个页面
memset(boot_pgdir, 0, PGSIZE);// 清空这个页面
boot_cr3 = PADDR(boot_pgdir); //将该页面的物理地址返回给boot_cr3,其实这个物理地址就是页目录表的起始地址.这里的PADDR的代码其实就是虚拟地址-KERNBASE.因为前面entry.s里有设置pa=va-KERNBASE
check_pgdir();//检查页目录表
static_assert(KERNBASE % PTSIZE == 0 && KERNTOP % PTSIZE == 0);//PTSIZE就是页表的大小,这里要对齐.因为我们是一个页表一个页表的创建虚拟内存,[TO-DO]
// recursively insert boot_pgdir in itself
// to form a virtual page table at virtual address VPT 
boot_pgdir[PDX(VPT)] = PADDR(boot_pgdir) | PTE_P | PTE_W;//这里其实是让pdt的第PDX[VPT]项指向自己,来形成一个虚拟的页表我们叫VPT,事实上我们并没有再开一个页表.VPT=0xFAC00000,那么PDX[VPT]实际上就是1023,这是一个自映射的机制

// map all physical memory to linear memory with base linear addr KERNBASE
//linear_addr KERNBASE~KERNBASE+KMEMSIZE = phy_addr 0~KMEMSIZE
//But shouldn't use this map until enable_paging() & gdt_init() finished.
boot_map_segment(boot_pgdir, KERNBASE, KMEMSIZE, 0, PTE_W);// 这个函数里有一个get_pte,实际上就创建了从KERBBASE映射到0二级页表.

//temporary map: 
//virtual_addr 3G~3G+4M = linear_addr 0~4M = linear_addr 3G~3G+4M = phy_addr 0~4M 
boot_pgdir[0] = boot_pgdir[PDX(KERNBASE)]; //因为这时的页机制还没有打开,va=la+KERNBASE=pa+2*KERNBASE,所以需要暂时设置一下这个转换方式.方法是让va在页目录表里映射两次,从而找到对应的物理页表.

enable_paging();//打开页机制,那么此时就要用到上面的转化方式.
//reload gdt(third time,the last time) to map all physical memory
//virtual_addr 0~4G=liear_addr 0~4G
//then set kernel stack(ss:esp) in TSS, setup TSS in gdt, load TSS
gdt_init();//重新创建新的gdt

//disable the map of virtual_addr 0~4M
boot_pgdir[0] = 0;//设置好新的gdt后pa=va-KERNBASE且这个偏移来自于页机制

//now the basic virtual memory map(see memalyout.h) is established.
//check the correctness of the basic virtual memory map.
check_boot_pgdir();

print_pgdir();

//-------------------------------------------------------------------------------//
static void
boot_map_segment(pde_t *pgdir, uintptr_t la, size_t size, uintptr_t pa, uint32_t perm) {
    assert(PGOFF(la) == PGOFF(pa)); //la和pa的偏移值应该是一样的
    size_t n = ROUNDUP(size + PGOFF(la), PGSIZE) / PGSIZE;//
    la = ROUNDDOWN(la, PGSIZE);
    pa = ROUNDDOWN(pa, PGSIZE);
    for (; n > 0; n --, la += PGSIZE, pa += PGSIZE) {
        pte_t *ptep = get_pte(pgdir, la, 1);
        assert(ptep != NULL);
        *ptep = pa | PTE_P | perm;
    }
}
//这个函数设置页表的过程是,通过get_pte设置好页目录表的内容(用alloc_page返回的page得到物理地址设置.),然后返回二级页表内容的地址,然后通过*ptep = pa | PTE_P | perm;设置好页表内容.所以页目录表里的页表起始地址以及页目录里的地址都是物理地址.
```
## 总结
这一部分主要是两个部分,一个是物理内存页的分配和维护,一个是物理内存和虚拟地址的映射,也就是页表机制的维护.

对于物理内存页,我们的做法是:
探测实际可用的物理内存
根据页的大小设置好Pages这个数组, 包含了所有页.在page_init函数里仅仅只是给每个页初始化了reserved这个属性.
然后我们开始设置好页目录表.
然后通过boot_map_segment把kernel所占用的页对应的页目录和页表设置好.因为就是把KERNBASE~KERNBASE+KMEMSIZE部分的线性地址和物理地址映射好.
所以我们做一个转换:
我们所谓的设置页目录表和页表就是让la和pa做一个映射.
当我们分配了一个内存页时,是先用alloc_page(),得到一个页的虚拟地址.然后根据这个虚拟地址去填写pte.
我们要释放一个内存页的操作就是让page的ref减1,然后把页表里的项删除.

## 遗留问题
1. default_init_memmap的调用链kern_init –> pmm_init–>page_init–>init_memmap–> pmm_manager->init_memmap,为什么要创一个init_memmap函数,然后再在这个函数里调用pmm_manager的init_memmap接口.为什么不直接在page_init里调用pmm_manager的init_memmap接口?