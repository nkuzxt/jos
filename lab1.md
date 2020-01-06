# LAB 1

## 准备工作

安装ubuntu子系统，开代理直接通过windows商店安装，比在镜像网站上下载快很多。

安装qemu成功后，运行qemu应使用指令sudo make qemu-nox，不能光用make qemu-nox。

## Part 1

### Exercise 1

阅读 Brenna’s Guide to Inline Assembly 的 The Sytax 这一部分的内容。

是对DJGPP下的内联汇编的介绍。DJGPP是GCC编译器在DOS操作系统上的一个移植版本，可以用来在DOS操作系统下编译生成32位保护模式的程序。所以使用AT&T/UNIX语法。
比较AT&T和Intel语法的区别：

1. 寄存器名字是否有%。AT&T有%。

2. 源/目标顺序不同。AT&T左边是源，右边是目标。

3. 地址前加不加$。AT&T有$，Intel可以忽略0x。

4. 是否明确寄存器的值。AT&T必须标明。

5. 引用内存的格式不同。
    
    - AT&T:  immed32(basepointer,indexpointer,indexscale)

    - Intel: [basepointer + indexpointer*indexscale + immed32]

### Exercise 2

使用GDB的si指令来跟踪 ROM BIOS 中的几个指令，并且尝试这些指令是要做什么的。

## Part 2

### Exercise 3

1. 处理器什么时候开始执行 32 位代码？如何完成的从 16 位到 32 位模式的切换？

    阅读boot.asm中的代码与注释，".code16"段为实模式，".code32"段为保护模式，由此可知，处理器在命令 ljmp $PROT_MODE_CSEG, $protcseg 之后开始执行32位代码。这一切换过程是通过如下命令实现的。
    
    ```C
    # Switch from real to protected mode, using a bootstrap GDT
    # and segment translation that makes virtual addresses 
    # identical to their physical addresses, so that the 
    # effective memory map does not change during the switch.
    lgdt    gdtdesc
      7c1e:	0f 01 16             	lgdtl  (%esi)
      7c21:	64 7c 0f             	fs jl  7c33 <protcseg+0x1>
    movl    %cr0, %eax
      7c24:	20 c0                	and    %al,%al
    orl     $CR0_PE_ON, %eax
      7c26:	66 83 c8 01          	or     $0x1,%ax
    movl    %eax, %cr0
      7c2a:	0f 22 c0             	mov    %eax,%cr0
    ```

1. 引导加载程序 bootloader 执行的最后一个指令是什么，加载的内核的第一个指令是什么？

    boot loader执行的最后一条指令为call ∗0x10018，调用了ELF的头部，实现跳转到kernel。 
    ```C
    ((void (*)(void)) (ELFHDR->e_entry))();
    ```

    内核执行的第一条指令从kernel.asm中找

    ```C
    .globl entry
    entry:
   movw	$0x1234,0x472			# warm boot
    ```

1. 内核的第一个指令在哪里？

    调用objdump\ -f\ obj/kern/kernel 得到信息可知第一条指令地址为0x0010000c

1. 引导加载程序如何决定为了从磁盘获取整个内核必须读取多少扇区？在哪里可以找到这些信息？

    boot loader会从硬盘中读入ELF File Header，对应代码在boot/main.c的bootmain函数中：

    ```C
    // read 1st page off disk
    readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC)
    	goto bad;

    // load each program segment (ignores ph flags)
    ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph++)
	// p_pa is the load address of this segment (as well
	// as the physical address)
	readseg(ph->p_pa, ph->p_memsz, ph->p_offset);

    // call the entry point from the ELF header
    // note: does not return!
    ((void (*)(void)) (ELFHDR->e_entry))();
	```

	从硬盘中读取ELFHDR，通过设定好的魔法数字来检验ELF头的有效性。

	可以看到，由ELFHDR的地址+程序头表的文件偏移e\_phoff能得到开始其中保存的起始程序头的地址ph，

	eph = ph + ELF Header中总的程序头个数e\_phnum为结束地址。

	利用ph和eph可遍历每一个程序头，并依次从中读取出kernel的内容。

### Exercise 4

指针部分

- 指针常量：该指针是一个常量。

- 常量指针：指向常量的指针。

ELF文件

- .text：程序的可执行指令。
- .rodata：只读数据，如 C 编译器生成的 ASCII 字符串常量（虽说是只读，我们并不会专门设置硬件来禁止写操作）。
- .data：数据部分保存程序的初始化数据，例如用 int x = 5 初始化时声明的全局变量。

## Part 3

### Exercise 8

1. o代表八进制，参考上方代码即可补全。

1. 解释 printf.c 和 console.c 之间的接口。具体来说，console.c 导出了什么函数？printf.c 是如何使用这些函数的？

    导出了cputchar(int c)，用于打印字符。

1. 解释代码

    ```C
    if (crt_pos >= CRT_SIZE) {
    	int i;
    	memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE -    CRT_COLS) * sizeof(uint16_t));
    	for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
    		crt_buf[i] = 0x0700 | ' ';
    	crt_pos -= CRT_COLS;
    }
    ```

    这段代码的目的是缓冲区满时清除一部分留出空间，可以理解成一页写满时自动下拉一行。比如设一页有25行，每行可放80个字符，则CRT_SIZE=25*80=2000，CRT_CLOS=80。当检测到crt_pos>CRT_SIZE，即光标位置超出屏幕外时，将缓冲区(crt_buf)中后24行复制到前24行，最后一行以' '填充。最后将光标位置上移一行。

1. 对于以下问题，你可能需要参考一些课外资料。

    跟踪以下代码的并单步执行：

    ```C
    int x = 1, y = 3, z = 4;
    cprintf("x %d, y %x, z %d\n", x, y, z);
    ```

    1. 在调用 cprintf() 时，fmt 是什么意思？ ap 是什么意思？

        fmt是字符指针，可以看作字符串。ap的类型是va_list，代表多个参数。
    
    1. 按执行顺序列出每次调用 cons_putc，va_arg 和 vcprintf 这三段代码时的细节。 对于 cons_putc，列出其参数。 对于 va_arg，列出调用之前和之后的 ap 指针的指向。 对于 vcprintf，列出其两个参数的值。

        cons_putc c=864、c=980、c=-1、c=980

        va_arg的作用是解析参数，它的第一个参数是ap，第二个参数是要获取的参数的指定类型，然后返回这个指定类型的值，并且把 ap 的位置指向变参表的下一个变量位置，以下是宏定义。

        vcprintf ( fmt=0xf01017f7 "x %d, y %x, z %d\n", ap=0xf010ffd4 "\001")

1. 运行下面的代码

    ```C
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
    ```

    输出是什么？如何按照上一个练习的执行步骤，说明为什么会显示这个输出信息。

    输出是He110 World。

    cprintf会调用vprintf进入while(1)的循环。

    ```C
    while (1) {
		while ((ch = *(unsigned char *) fmt++) != '%') {
			if (ch == '\0')
				return;
			putch(ch, putdat);
		}
    ```

    当遇到%x，会变成0x，继而把57616看作16进制输出成e110。然后遇到%s，会输出0x00646c72。因为x86是小端，第一个是0x72，第二个是0x6c，第三个是0x64，对应的ascII码是rld。

1. 在下面的代码中，将在 y = 之后打印什么（注意：答案不是一个固定的值）？为什么会发生这种情况？

    因为没给够参数，所以会输出下一个寄存器存的值，这个值是任意的。

1. 假设 GCC 更改了它的调用约定，以声明的顺序将参数压入栈中，这样会使最后一个参数最后被压入。 你将如何更改 cprintf 或其接口，以便仍然可以传递一个可变数量的参数？

    参数顺序正好相反，可以用一个函数再颠倒回来。

### Exercise 11

实现mon_backtrace()函数，显示ebp，eip和arg信息。

eip（返回指令指针）：存储当前执行指令的下一条指令在内存中的偏移地址。

ebp（基址指针）：存储指向当前函数需要使用的参数的指针。

esp （栈指针）：存储指向栈顶的指针。

在程序中，如果需要调用一个函数，首先会将函数需要的参数进栈，然后将 eip 中的内容进栈，也就是下一条指令在内存中的位置，这样在函数调用结束后便可以通过堆栈中的 eip 值。