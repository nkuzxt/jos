# Lab 3

## Part A

### Exercise 1

和lab2的第一个练习类似，envs指向大小为NENV的ENV结构体数组。并把envs和用户线性地址相连，权限只读。

### Exercise 2

1. env_init

    初始化全部 envs 数组中的 Env 结构体，并将它们加入到 env_free_list 中。还要调用 env_init_percpu ，这个函数会通过配置段硬件，将其分隔为特权等级 0 (内核) 和特权等级 3（用户）两个不同的段。

    和lab2的page_init类似，确保env_free_list=&envs[0]。

1. env_setup_vm 

    为新的进程分配一个页目录，并初始化新进程的地址空间对应的内核部分。

    把内核的内存拷贝过来，可以用memcpy函数一步搞定。

1. region_alloc 为进程分配和映射物理内存。

1. load_icode

    处理 ELF 二进制映像，将映像内容读入新进程的用户地址空间。大概要做：

	- 根据 ELF header 得出 Programm header。
	
	- 遍历所有 Programm header，分配好内存，加载类型为 ELF_PROG_LOAD 的段。
	
	- 分配用户栈。

	切换页目录：lcr3([页目录物理地址]) 将地址加载到 cr3 寄存器。

	更改函数入口：将 env->env_tf.tf_eip 设置为 elf->e_entry，等待之后的 env_pop_tf 调用。

1. env_create 通过调用 env_alloc 分配一个新进程，并调用 load_icode 读入 ELF 二进制映像。

1. env_run 启动给定的在用户模式运行的进程。

    Set the current environment (if any) back to ENV_RUNNABLE if it is ENV_RUNNING (think about what other states it can be in)，这一步可以不做，因为下面还会把状态设为 ENV_RUNNING。

### 练习

```C
(gdb) b env_pop_tf
Breakpoint 1 at 0xf0102e61: file kern/env.c, line 485.
(gdb) c
Continuing.
The target architecture is assumed to be i386
=> 0xf0102e61 <env_pop_tf>:     push   %ebp

Breakpoint 1, env_pop_tf (tf=0xf01f2000) at kern/env.c:485
485     {
(gdb) si
=> 0xf0102e62 <env_pop_tf+1>:   mov    %esp,%ebp
0xf0102e62      485     {
...
(gdb) si
=> 0xf0102e70 <env_pop_tf+15>:  iret   
0xf0102e70      486             asm volatile(
(gdb) info register
eax            0x0                 0
ecx            0x0                 0
edx            0x0                 0
ebx            0x0                 0
esp            0xf01f2030          0xf01f2030
ebp            0x0                 0x0
esi            0x0                 0
edi            0x0                 0
eip            0xf0102e70          0xf0102e70 <env_pop_tf+15>
eflags         0x96                [ PF AF SF ]
cs             0x8                 8
ss             0x10                16
ds             0x23                35
es             0x23                35
fs             0x23                35
gs             0x23                35
(gdb) si
=> 0x800020:    cmp    $0xeebfe000,%esp			//first instruction we execute in entry.S
0x00800020 in ?? ()													//See if we were started with arguments on the stack
	cmpl $USTACKTOP, %esp

(gdb) info register
eax            0x0                 0
ecx            0x0                 0
edx            0x0                 0
ebx            0x0                 0
esp            0xeebfe000          0xeebfe000
ebp            0x0                 0x0
esi            0x0                 0
edi            0x0                 0
eip            0x800020            0x800020
eflags         0x2                 [ ]
cs             0x1b                27
ss             0x23                35
ds             0x23                35
es             0x23                35
fs             0x23                35
gs             0x23                35

(gdb) si
=> 0x800a0a:    int    $0x30
0x00800a0a in ?? ()
```

### Exercise 4

在trap_init中初始化中断处理函数然后插入到中断描述符表中。

trapentry.S中有两个宏，定义了用于处理trap的全局可见函数。注意，如果trap没有错误代码，设为零，使与Trapframe的形式相同。

_alltraps 根据Trapframe的定义，我们需要依次push %ds、%es、所有通用寄存器，然后把%ds和%es的值都设为$GD_HD，再push %esp，最后call trap。

### 挑战

类比之前的代码可知，AT&T的大概汇编格式

```C
#define                     # 函数定义
.data                       # 数据段声明
.text                       # 代码段声明
.global _start              # 指定入口函数

_start:
```

```C
#define ec(name, num)\
.data 
	.long name 
.text 
	.globl name 
	.type name, @function 
	.align 2 
name:\
	pushl $(num) 
	jmp _alltraps



#define noec(name, num)\
.data 
	.long name 
.text 
	.globl name 
	.type name, @function 
	.align 2 
name:\
	pushl $0 
	pushl $(num) 
	jmp _alltraps



#define bubble()\
.data 
	.long 0
	.p2align 2
	.globl funs
funs:

.text

	noec(th0, 0)
	noec(th1, 1)
	bubble()
	noec(th3, 3)
	noec(th4, 4)
	noec(th5, 5)
	noec(th6, 6)
	noec(th7, 7)
	ec(th8, 8)
	noec(th9, 9)
	ec(th10, 10)
	ec(th11, 11)
	ec(th12, 12)
	ec(th13, 13)
	ec(th14, 14)
	bubble()
	noec(th16, 16)
.data
	.space 124
.text
	noec(th48, T_SYSCALL)
```

### 问题

- 对每一个中断/异常都分别给出中断处理函数的目的是什么？换句话说，如果所有的中断都交给同一个中断处理函数处理，现在我们实现的哪些功能就没办法实现了？

	每个异常和中断处理方式不同，例如除0异常不会返回程序继续执行，而I/O操作中断会返回程序继续执行。用一个handler难以实现。

- 你有没有额外做什么事情让 user/softint 这个程序按预期运行？打分脚本希望它产生一个一般保护错(陷阱 13)，可是 softint 的代码却发送的是 int $14。为什么 这个产生了中断向量 13 ？如果内核允许 softint 的 int $14 指令去调用内核中断向量 14 所对应的的缺页处理函数，会发生什么？

    在用户模式下，用户的CPL=3，而在SETGATE中对中断向量14设置的DPL=0，触发了13异常。

    如果允许这个操作会造成安全问题，而且用户不当的操作会使JOS崩溃。

## Part B

### Exercise 5/6

根据 Trapframe的tf_trapno 进行处理分配。

### 问题

- 断点那个测试样例可能会生成一个断点异常，或者生成一个一般保护错，这取决你是怎样在 IDT 中初始化它的入口的（换句话说，你是怎样在 trap_init 中调用 SETGATE 方法的）。为什么？你应该做什么才能让断点异常像上面所说的那样工作？怎样的错误配置会导致一般保护错？

	权限问题，SETGATE(idt[T_BRKPT], 1, GD_KT, th3, 3);

- 你认为这样的机制意义是什么？尤其要想想测试程序 user/softint 的所作所为 / 尤其要考虑一下 user/softint 测试程序的行为。

	优先级低的代码无法访问优先级高的代码，优先级高低由 gd_dpl 判断。数字越小越高。保证安全。

### Exercise 7

kern 中有syscall.h syscall.c，inc和lib中也有syscall.h syscall.c。需要理清之间的关系：

1. 用户进程使用 inc/ 目录下暴露的接口

1. lib/syscall.c 中的函数将系统调用号及必要参数传给寄存器，并引起一次 int $0x30 中断

1. kern/trap.c 捕捉到这个中断，并将 TrapFrame 记录的寄存器状态作为参数，调用处理中断的函数

1. kern/syscall.c 处理中断

trap_dispatch中用到的应该是kern/syscall.c中的syscall函数。

### Exercise 8

把全局指针 thisenv 指向该程序在 envs[] 数组中的位置。

### Exercise 9

- page_fault-handler 检查 tf_cs 的低位

- user_mem_check 内存检查

	使用pgdir查找线性地址对应的页表地址。
	
	注意需要存储第一个访问出错的地址，va 所在的页面需要单独处理一下，不能直接对齐。

- sys_cputs 和 debuginfo_eip 都是内存检查，使用 user_mem_check