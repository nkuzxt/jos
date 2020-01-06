![pgdir](C:/Users/zxt/Desktop/1.png)



# Lab 2

## Part 1

### Exercise 1

手打断点测试。

1. boot_alloc(uint32_t n)

    在此函数中，我们为内核页表目录分配了一块大小至少是n bytes的连续的内存。如果内存超限需要panic。且下一个空内存的大小也至少是n bytes。

1. mem_init
    
    为页申请空间并初始化，可以用到之前实现的boot_alloc函数，参数为页数*结构体大小，这样保证可以装下，不会溢出。

1. page_init

    对所有物理页初始化。按注释要求初始化，pp_ref==1代表该页被占用。

1. page_alloc

    分配一个新页，如果没有新页可供分配，返回NULL。否则进行分配，注意这里分配的新页可能被赋过值，用page2kva(作用是通过物理页获取其内核线性地址)初始化这个新页。

1. page_free

    把这页加入空页列表。

## Part 2

### Question 1

假设以下 JOS 内核代码是正确的，变量 x 应该是什么类型？uintptr_t 还是 physaddr_t？

```C
mystery_t x;
char * value = return_a_pointer（）;
* value = 10;
x =（mystery_t）value;
```

因为这里变量value使用了*操作符解析地址，即先转为指针再解析引用的方式，是虚拟地址。所以变量x也应该是虚拟地址即uintptr_t类型。

### Exercise 4

写一组程序来管理页表：插入和删除线性到物理映射，并在需要时创建新的页以存储页表。

```C
// The PDX, PTX, PGOFF, and PGNUM macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).

// page number field of address
#define PGNUM(la)   (((uintptr_t) (la)) >> PTXSHIFT)

// page directory index
// 取31到22 bit
#define PDX(la)     ((((uintptr_t) (la)) >> PDXSHIFT) & 0x3FF)

// page table index
// 取21到12 bit
#define PTX(la)     ((((uintptr_t) (la)) >> PTXSHIFT) & 0x3FF)

// offset in page
// 取11到0 bit
#define PGOFF(la)   (((uintptr_t) (la)) & 0xFFF)

// construct linear address from indexes and offset
#define PGADDR(d, t, o) ((void*) ((d) << PDXSHIFT | (t) << PTXSHIFT | (o)))

// Page directory and page table constants.
#define NPDENTRIES  1024        // page directory entries per page directory
#define NPTENTRIES  1024        // page table entries per page table

#define PGSIZE      4096        // bytes mapped by a page, 4kB
#define PGSHIFT     12      // log2(PGSIZE)

#define PTSIZE      (PGSIZE*NPTENTRIES) // bytes mapped by a page directory entry, 4MB
#define PTSHIFT     22      // log2(PTSIZE)

#define PTXSHIFT    12      // offset of PTX in a linear address
#define PDXSHIFT    22      // offset of PDX in a linear address

// The PTE_AVAIL bits aren't used by the kernel or interpreted by the
// hardware, so user processes are allowed to set them arbitrarily.
#define PTE_AVAIL   0xE00   // Available for software use

// Flags in PTE_SYSCALL may be used in system calls.  (Others may not.)
#define PTE_SYSCALL (PTE_AVAIL | PTE_P | PTE_W | PTE_U)

// Address in page table or page directory entry
// 将页目录项的后12位（flag 位）全部置 0 获得对应的页表项物理地址
#define PTE_ADDR(pte)   ((physaddr_t) (pte) & ~0xFFF)

// Page table/directory entry flags.
#define PTE_P       0x001   // 该项是否存在
#define PTE_W       0x002   // 可写入
#define PTE_U       0x004   // 用户有权限读取
#define PTE_PWT     0x008   // Write-Through
#define PTE_PCD     0x010   // Cache-Disable
#define PTE_A       0x020   // Accessed
#define PTE_D       0x040   // Dirty
#define PTE_PS      0x080   // Page Size
#define PTE_G       0x100   // Global

//接受一个内核虚拟地址，并返回相应的物理地址。
#define PADDR(kva) _paddr(__FILE__, __LINE__, kva)

//KADDR接受一个物理地址并返回相应的内核虚拟地址。
#define KADDR(pa) _kaddr(__FILE__, __LINE__, pa)
```

1. pgdir_walk

    用于查找线性地址对应的页表地址，传入参数分别为页目录项指针、线性地址和一个用于判断页目录项不存在时是否进行创建参数。返回值为页表项指针，在页表不存在且create==false或创建页表失败时，返回NULL。

    查找页表项地址之前首先要获得页目录地址，对页表是否存在进行判断，如果页目录存在，直接进行映射；如果不存在，则新建一个页表后再进行映射。创建页表时使用page_alloc分配新的页表页，并为新建的物理页设置页目录时添加上权限位。

1. boot_map_region

    将线性地址[va, va+size)映射到物理地址[pa, pa+size)。

1. page_lookup

    查找线性地址va对应的物理页面，找到就返回这个物理页，否则返回NULL。

1. page_remove

    对线性地址va取消其对物理页面的映射，如果物理页面不存在就什么都不做。

1. page_insert

    建立一个线性地址与物理页的映射，注意如果所要线性地址有映射应先移除。

## Part 3

### Exercise 5

通过 inc/memlayout.h 文件中展示的内存地址空间的布局，在meminit中完成线性地址到物理地址映射，以建立这个布局。

主要用到之前实现的 boot_map_region 函数，完成线性地址到物理地址的映射。注意权限问题。

### Question 2

倒数三行的第三列依次是。

- Page table for third lowest 4MB of phys memory

- Page table for second lowest 4MB of phys memory

- Page table for lowest 4MB of phys memory

### Question 3

我们已经将内核和用户环境放在同一地址空间内。为什么用户的程序不能读取内核的内存？有什么具体的机制保护内核空间吗？

因为有设置权限。

### Question 4

JOS 操作系统可以支持的最大物理内存是多少？为什么？

PGSIZE=4096*1024=4MB，一个PageInfo类型的大小是8bytes，所以可以有4M/8=512K页，每页大小4KB，最大物理内存不超过2G。

### Question 5

如果我们的硬件配置了可以支持的最大的物理内存，那么管理内存空间的开销是多少？这一开销是怎样划分的？

6M+2K。4M用于页面信息，2M用于页表，2K用于页目录表。

### Question 6

再次分析 kern/entry.S 和 kern/entrypgdir.c 的页表设置的过程，在打开分页之后，EIP 依然是一个数字（稍微超过 1MB）。在什么时刻我们才开始在 KERNBASE 上运行 EIP 的？当我们启动分页并在 KERNBASE 上开始运行 EIP 之时，我们能否以低地址的 EIP 继续执行？这个过渡为什么是必须要的？

在set cr3之后打断点可知eip是低地址，跳到新地址，会重置eip。

通过此过渡，我们可以使用虚拟地址进行页面转换，从而可以解决内存浪费问题，并使用户空间和内核空间彼此隔离。