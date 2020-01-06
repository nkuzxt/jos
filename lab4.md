# Lab 4

## Part A

### Exercise 1

实现mmio_map_region，该函数在lapic_init中被调用，将从 lapicaddr 开始的 4kB 物理地址映射到虚拟地址，并返回其起始地址。调用 boot_map_region 来建立所需要的映射，需要注意的是，每次需要更改base的值，使得每次都是映射到一个新的页面。

### Exercise 2

修改page_init，标记 MPENTRY_PADDR 开始的一个物理页为已使用，只需在循环中做一个判断即可。

### 问题

逐行比较 kern/mpentry.S 和 boot/boot.S。牢记 kern/mpentry.S 和其他内核代码一样也是被编译和链接在 KERNBASE 之上运行的。那么，MPBOOTPHYS 这个宏定义的目的是什么呢？为什么它在 kern/mpentry.S 中是必要的，但在 boot/boot.S 却不用？换句话说，如果我们忽略掉 kern/mpentry.S 哪里会出现问题呢？ 提示：回忆一下我们在 Lab 1 讨论的链接地址和装载地址的不同之处。

MPBOOTPHYS 的定义表示从 mpentry_start 到 MPENTRY_PADDR 建立映射，将 mpentry_start + offset 地址转为 MPENTRY_PADDR + offset 地址。

mpentry.S的代码都在kernbase上，所以实模式是没办法直接寻址的，
这时候需要MPBOOTPHYS起一个地址转换的作用，而boot.S就被加载在实模式可寻址的低地址，所以不需要地址转换。

### Exercise 3/4

- 修改mem_init_mp，将每个CPU堆栈映射在 KSTACKTOP 开始的区域，就像 inc/memlayout.h 中描述的那样。每个堆栈的大小都是 KSTKSIZE 字节，加上 KSTKGAP 字节没有被映射的守护页。使用boot_map_region映射即可。

- 修改trap_init_percpu，在 inc/memlayout.h 中可以找到 GD_TSS0 的定义，但是并没有说其他CPU的任务选择器在哪，查阅知道偏移是cpu_id<<3即cpu_id*8。



### Exercise 5

加锁很简单，为什么在这几个地方加，应该是保证独立性，类似数据库里的加锁。

### 问题

看起来使用全局内核锁能够保证同一时段内只有一个 CPU 能够运行内核代码。既然这样，我们为什么还需要为每个 CPU 分配不同的内核堆栈呢？请描述一个即使我们使用了全局内核锁，共享内核堆栈仍会导致错误的情形。

在某进程即将陷入内核态的时候但尚未获得锁，在 trap() 函数之前已经在 trapentry.S 中对内核栈进行了操作，压入了寄存器信息。如果共用一个内核栈，那显然会导致信息错误。

### Exercise 6

具体实现是如果存在上一个running environment，就从它开始，否则从头开始搜索envs数组，找到一个runnable的environment并进入。如果没有其他environment而之前有一个正在运行的则继续执行它。其他情况执行cpuhalt，注意不能直接由curenv->env_id得到序号，应使用宏定义ENVX(idle->env_id)。

### 问题

- 在你实现的 env_run() 中你应当调用了 lcr3()。在调用 lcr3() 之前和之后，你的代码应当都在引用 变量 e，就是 env_run() 所需要的参数。 在装载 %cr3 寄存器之后， MMU 使用的地址上下文立刻发生改变，但是处在之前地址上下文的虚拟地址（比如说 e ）却还能够正常工作，为什么 e 在地址切换前后都可以被正确地解引用呢？

    在 lab3 中的函数 env_setup_vm，它直接以内核的页目录作为模版稍做修改。因此两个页目录的 e 地址映射到同一物理地址。

- 无论何时，内核在从一个进程切换到另一个进程时，它应当确保旧的寄存器被保存，以使得以后能够恢复。为什么？在哪里实现的呢？

    - 保存发生在 kern/trapentry.S 的自己实现的_alltraps中。
    
    - 恢复发生在 kern/env.c 的 env_pop_tf函数中。

        ```C
        void env_pop_tf(struct Trapframe *tf)
        {
            ...
            asm volatile(
                "\tmovl %0,%%esp\n"    // 恢复栈顶指针
                "\tpopal\n"    // 恢复其他寄存器
                "\tpopl %%es\n"    // 恢复段寄存器
                "\tpopl %%ds\n"
                "\taddl $0x8,%%esp\n" /* skip tf_trapno and tf_errcode */
                "\tiret\n"
                : : "g" (tf) : "memory");
            ...
        }
        ```

### Exercise 7

根据提示信息补全函数并实现系统调用。

- sys_exofork

- sys_env_set_status

- sys_page_alloc

- sys_page_map

- sys_page_unmap

## Part B

### Exercise 8

通过修改相应的struct Env的'env\_pgfault\_upcall'字段，为'envid'设置页面错误upcall。 当'envid'导致页面错误时，内核会将错误记录推送到异常堆栈，然后转移到'func'。成功时返回0，错误时即如果环境envid当前不存在，或者调用者没有更改envid的权限返回<0，即-E\_BAD\_ENV。 

### Exercise 9/10/11

- page_fault_handler

    首先要知道用户级别的页错误处理的步骤是：进程A(正常栈) -> 内核 -> 进程A(异常栈) -> 进程A(正常栈)

    那么内核的工作就是修改进程 A 的某些寄存器，并初始化异常栈，确保能顺利切换到异常栈运行。需要注意的是，由于修改了eip， env_run 是不会返回的，因此不会继续运行后面销毁进程的代码。

    没有空间的话，下面一页是空页，内核和用户访问都会报错。

- _pgfault_upcall



- set_pgfault_handler

    是用户用来指定缺页异常处理方式的函数。代码比较简单，但是需要区分清楚 handler，_pgfault_handler，_pgfault_upcall 三个变量。

    - handler 是传入的用户自定义页错误处理函数指针。
    
    - _pgfault_upcall 是一个全局变量，在 lib/pfentry.S 中完成的初始化。它是页错误处理的总入口，页错误除了运行 page fault handler，还需要切换回正常栈。

    - _pgfault_handler 被赋值为handler，会在 _pgfault_upcall 中被调用，是页错误处理的一部分。

    若是第一次调用，需要首先分配一个页面作为异常栈，并且将该进程的 upcall 设置为 Exercise 10 中的程序。此后如果需要改变handler，不需要再重复这个工作。

### Exercise 12

- fork 大体结构可以仿造 user/dumbfork.c 写，但是有关键几处不同：

    - 设置 page fault handler，即 page fault upcall 调用的函数

    - duppage 的范围不同，fork() 不需要复制内核区域的映射

    - 为子进程设置 page fault upcall，之所以这么做，是因为 sys_exofork() 并不会复制父进程的 e->env_pgfault_upcall 给子进程。

- duppage 作用是复制父、子进程的页面映射。尤其注意一个权限问题。由于 sys_page_map() 页面的权限有硬性要求，因此必须要修正一下权限。之前没有修正导致一直报错，后来发现页面权限为 0x865，不符合 sys_page_map() 要求。

- pgfault 作用是分配一个物理页面使得两者独立。首先，它分配一个页面，映射到了交换区 PFTEMP 这个虚拟地址，然后通过 memmove() 函数将 addr 所在页面拷贝至 PFTEMP，此时有两个物理页保存了同样的内容。再将 addr 也映射到 PFTEMP 对应的物理页，最后解除了 PFTEMP 的映射，此时就只有 addr 指向新分配的物理页了，如此就完成了错误处理。

## Part C

### Exercise 13/14

和Lab3的练习差不多。

### Exercise 15

注意通信流程是

1. 调用 ipc_recv，设置好 Env 结构体中的相关 field

1. 调用 ipc_send，它会通过 envid 找到接收进程，并读取 Env 中刚才设置好的 field，进行通信。

1. 最后返回实际上是在 ipc_send 中设置好 reg_eax，在调用结束，退出内核态时返回。

lib部分，主要陷阱是如果不需要共享页面，则把作为参数的虚拟地址设为 UTOP，这个地址在下面的系统调用实现中，会被忽略掉。

sys_ipc_recv 如果作为参数的虚拟地址在 UTOP 之上，只需要忽略，而不是报错退出。因为这种情况是说明接收者只需要接收值，而不需要共享页面（联想在 lib/ipc.c 中的处理）。

sys_ipc_try_send 与sys_page_map非常相似，尝试通过调用 sys_page_map() 解决，可以避免编写大量重复代码。但是发现其中最大的区别在于，ipc 通信并不限于父子进程之间，而 sys_page_map 最初设计的作用就是用于 fork，因此需要做改动才能用于这里，也就是说改变 envid2env 的参数。