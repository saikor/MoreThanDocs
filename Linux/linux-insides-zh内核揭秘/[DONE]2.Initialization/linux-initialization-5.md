内核初始化 第五部分
===========================================================

与系统架构有关的初始化后续分析
===========================================================

在之前的[章节](http://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-4.html)中，
我们讲到了与系统架构有关的 [setup_arch](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/setup.c#L856) 函数部分，本文会继续从这里开始。
我们为 [initrd](http://en.wikipedia.org/wiki/Initrd) 预留了内存之后，下一步是执行 `olpc_ofw_detect` 函数检测系统是否支持 [One Laptop Per Child support](http://wiki.laptop.org/go/OFW_FAQ)。
我们不会考虑与平台有关的东西，且会忽略与平台有关的函数。所以我们继续往下看。
下一步是执行 `early_trap_init` 函数。这个函数会初始化调试功能 （`#DB` -当 `TF` 标志位和rflags被设置时会被使用）和 `int3` （`#BP`）中断门。

> 在5.10.13中没有 `early_trap_init` 函数,改为`idt_setup_early_traps`。

如果你不了解中断，你可以从 [初期中断和异常处理](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-2.html) 中学习有关中断的内容。
在 `x86` 架构中，`INT`，`INT0` 和 `INT3` 是支持任务显式调用中断处理函数的特殊指令。`INT3` 指令调用断点（`#BP`）处理函数。
你如果记得，我们在这[部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-2.html) 看到过中断和异常概念：

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                                |
----------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                                 |
----------------------------------------------------------------------------------------------
```

调试中断 `#DB` 是激活调试器的重要方法。`early_trap_init` 函数的定义在 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) 中。这个函数用来设置 `#DB` 和 `#BP` 处理函数，并且实现重新加载 [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)。

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```
给出5.10.13中的函数：
```c
/**
 * idt_setup_early_traps - Initialize the idt table with early traps
 *
 * On X8664 these traps do not use interrupt stacks as they can't work
 * before cpu_init() is invoked and sets up TSS. The IST variants are
 * installed after that.
 */
void __init idt_setup_early_traps(void)/* 中断描述符表 */
{
	idt_setup_from_table(idt_table, early_idts, ARRAY_SIZE(early_idts),
			     true);
	load_idt(&idt_descr);/* 中断描述符 */
}
```
我们之前中断相关章节中看到过 `set_intr_gate` 的实现。

```c
static __init void set_intr_gate(unsigned int n, const void *addr)
{
	struct idt_data data;

    /*  */
	init_idt_data(&data, n/* 终端号 */, addr/* 处理地址 */);

    //将中断门插入至 `IDT` 表中
	idt_setup_from_table(idt_table, &data, 1, false);
}
```

这里的 `set_intr_gate_ist` 和 `set_system_intr_gate_ist` 也是类似的实现。
这两个函数都需要三个参数：

* 中断号
* 中断/异常处理函数的基地址
* 第三个参数是 `Interrupt Stack Table`。 `IST` 是 [TSS](http://en.wikipedia.org/wiki/Task_state_segment) 的部分内容，是 `x86_64` 引入的新机制。

在内核态处于活跃状态的线程拥有 `16kb` 的内核栈空间。但是在用户空间的线程的内核栈是空的。
除了线程栈，还有一些与**每个 `CPU`** 有关的特殊栈。你可以查阅 linux 内核文档 - [Kernel stacks](https://rtoax.blog.csdn.net/article/details/114995648) 部分了解这些栈信息。
`x86_64` 提供了像在非屏蔽中断等类似事件中切换新的特殊栈的特性支持。这个特性的名字是 `Interrupt Stack Table`。
每个CPU最多可以有 7 个 `IST` 条目，每个条目有自己特定的栈。在我们的案例中使用的是 `DEBUG_STACK`。

`set_intr_gate_ist` 和 `set_system_intr_gate_ist` 与 `set_intr_gate` 的工作原理几乎一样，只有一个区别。
这些函数检查中断号并在内部调用 `_set_gate` ：

```C
BUG_ON((unsigned)n > 0xFF);
_set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
```

其中， `set_intr_gate` 把 [dpl](http://en.wikipedia.org/wiki/Privilege_level) 和 `ist` 置为 0 来调用 `_set_gate`。
但是 `set_intr_gate_ist` 和 `set_system_intr_gate_ist` 把 `ist` 设置为 `DEBUG_STACK`，并且 `set_system_intr_gate_ist` 把 `dpl` 设置为优先级最低的 `0x3`。

当中断发生时，硬件加载这个描述符，然后硬件根据 `IST` 的值自动设置新的栈指针。
之后激活对应的中断处理函数。所有的特殊内核栈会在 `cpu_init` 函数中设置好（我们会在后文中提到）。

当 `#DB` 和 `#BP` 门向 `idt_descr` 有写操作，我们会调用 `load_idt` 函数来执行 `ldtr` 指令来重新加载 `IDT` 表。
现在我们来了解下中断处理函数并尝试理解它的工作原理。当然，我们不可能在这本书中讲解所有的中断处理函数。
深入学习 linux 的内核源码是很有意思的事情，我们会在这里讲解 `debug` 处理函数的实现。请自行学习其他的中断处理函数实现。

`#DB` 处理函数
-----------------------------------------------------------------------

像上文中提到的，我们在 `set_intr_gate_ist` 中通过 `&debug` 的地址传送 `#DB` 处理函数。[lxr.free-electorns.com](http://lxr.free-electrons.com/ident) 是很好的用来搜索 linux 源代码中标识符的资源。
遗憾的是，你在其中找不到 `debug` 处理函数。你只能在 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/traps.h) 中找到 `debug` 的定义：

```C
asmlinkage void debug(void);
```

从 `asmlinkage` 属性我们可以知道 `debug` 是由 [assembly](http://en.wikipedia.org/wiki/Assembly_language) 语言实现的函数。是的，又是汇编语言 :)。
和其他处理函数一样，`#DB` 处理函数的实现可以在 [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S) 文件中找到。
都是由 `idtentry` 汇编宏定义的：

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

`idtentry` 是一个定义中断/异常指令入口点的宏。它需要五个参数：

* 中断条目点的名字
* 中断处理函数的名字
* 是否有中断错误码
* paranoid - 如果这个参数置为 1，则切换到特殊栈
* shift_ist - 支持中断期间切换栈

现在我们来看下 `idtentry` 宏的实现。这个宏的定义也在相同的汇编文件中，并且定义了有 `ENTRY` 宏属性的 `debug` 函数。
首先，`idtentry` 宏检查所有的参数是否正确，是否需要切换到特殊栈。接下来检查中断返回的错误码。例如本案例中的 `#DB` 不会返回错误码。
如果有错误码返回，它会调用 `INTR_FRAME` 或者 `XCPT_FRAM` 宏。其实 `XCPT_FRAME` 和 `INTR_FRAME` 宏什么也不会做，只是对中断初始状态编译的时候有用。
它们使用 `CFI` 指令用来调试。你可以查阅更多有关 `CFI` 指令的信息 [CFI](https://sourceware.org/binutils/docs/as/CFI-directives.html)。
就像 [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64) 中解释：`CFI` 宏是用来产生更好的回溯的 `dwarf2` 的解开信息。
它们不会改变任何代码。因此我们可以忽略它们。

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	/* Sanity check */
	.if \shift_ist != -1 && \paranoid == 0
	.error "using shift_ist requires paranoid=1"
	.endif

	.if \has_error_code
	XCPT_FRAME
	.else
	INTR_FRAME
	.endif
	...
	...
	...
```
在5.10.13中是这样的：  
```c
/**
 * idtentry - Macro to generate entry stubs for simple IDT entries
 * @vector:		Vector number
 * @asmsym:		ASM symbol for the entry point
 * @cfunc:		C function to be called
 * @has_error_code:	Hardware pushed error code on stack
 *
 * The macro emits code to set up the kernel context for straight forward
 * and simple IDT entries. No IST stack, no paranoid entry checks.
 */
.macro idtentry vector asmsym cfunc has_error_code:req
SYM_CODE_START(\asmsym)
	UNWIND_HINT_IRET_REGS offset=\has_error_code*8
	ASM_CLAC

	.if \has_error_code == 0
		pushq	$-1			/* ORIG_RAX: no syscall to restart */
	.endif

	.if \vector == X86_TRAP_BP
		/*
		 * If coming from kernel space, create a 6-word gap to allow the
		 * int3 handler to emulate a call instruction.
		 */
		testb	$3, CS-ORIG_RAX(%rsp)
		jnz	.Lfrom_usermode_no_gap_\@
		.rept	6
		pushq	5*8(%rsp)
		.endr
		UNWIND_HINT_IRET_REGS offset=8
.Lfrom_usermode_no_gap_\@:
	.endif

	idtentry_body \cfunc \has_error_code

_ASM_NOKPROBE(\asmsym)
SYM_CODE_END(\asmsym)
.endm
```
当中断发生后经过初期的中断/异常处理，我们可以知道栈内的格式是这样的：

```
    +-----------------------+
    |                       |
+40 |         SS            |
+32 |         RSP           |
+24 |        RFLAGS         |
+16 |         CS            |
+8  |         RIP           |
 0  |       Error Code      | <---- rsp
    |                       |
    +-----------------------+
```

`idtentry` 实现中的另外两个宏分别是

```assembly
	ASM_CLAC
	PARAVIRT_ADJUST_EXCEPTION_FRAME
```

第一个 `ASM_CLAC` 宏依赖于 `CONFIG_X86_SMAP` 这个配置项和考虑安全因素，你可以从[这里](https://lwn.net/Articles/517475)了解更多内容。
第二个 `PARAVIRT_EXCEPTION_FRAME` 宏是用来处理 `Xen` 类型异常（这章只讲解内核初始化，不会考虑虚拟化的内容）。
下一段代码会检查中断是否有错误码。如果没有则会把 `$-1`(在 `x86_64` 架构下值为 `0xffffffffffffffff`)压入栈：

```assembly
	.ifeq \has_error_code
	pushq_cfi $-1
	.endif
```

为了保证对于所有中断的栈的一致性，我们会把它处理为 `dummy` 错误码。下一步我们从栈指针中减去 `$ORIG_RAX-R15`：

```assembly
	subq $ORIG_RAX-R15, %rsp
```

其中，`ORIG_RAX`，`R15` 和其他宏都定义在 [arch/x86/include/asm/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 中。`ORIG_RAX-R15` 是 120 字节。
我们在中断处理过程中需要把所有的寄存器信息存储在栈中，所有通用寄存器会占用这个 120 字节。
为通用寄存器设置完栈之后，下一步是检查从用户空间产生的中断：

```assembly
testl $3, CS(%rsp)
jnz 1f
```

我们查看段寄存器 `CS` 的前两个比特位。你应该记得 `CS` 寄存器包含段选择器，它的前两个比特是 `RPL`。所有的权限等级是0-3范围内的整数。
数字越小代表权限越高。因此当中断来自内核空间，我们会调用 `save_paranoid`，如果不来自内核空间，我们会跳转到标签 `1` 处处理。
在 `save_paranoid` 函数中，我们会把所有的通用寄存器存储到栈中，如果需要的话会用户态 `gs` 切换到内核态 `gs`：

```assembly
movl $1,%ebx
	movl $MSR_GS_BASE,%ecx
	rdmsr
	testl %edx,%edx
	js 1f
	SWAPGS
	xorl %ebx,%ebx
1:	ret
```

下一步我们把 `pt_regs` 指针存在 `rdi` 中，如果存在错误码就把它存储到 `rsi` 中，然后调用中断处理函数，例如就像 [arch/x86/kernel/trap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)中的 `do_debug`。
`do_debug` 像其他处理函数一样需要两个参数：

* pt_regs - 是一个存储在进程内存区域的一组CPU寄存器
* error code - 中断错误码

中断处理函数完成工作后会调用 `paranoid_exit` 还原栈区。如果中断来自用户空间则切换回用户态并调用 `iret`。我们会在不同的章节继续深入分析中断。

> 中断部分的代码在5.10.13中已经做出了较大程度的改进，asm_exc_debug我目前没有找到，只找到了不可屏蔽中断NMI部分的汇编代码，后续这里需要继续学习研究。

这是用在 `#DB` 中断中的 `idtentry` 宏的基本介绍。所有的中断都和这个实现类似，都定义在 `idtentry`中。`early_trap_init` 执行完后，下一个函数是 `early_cpu_init`。
这个函数定义在 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c) 中，负责收集 `CPU` 和其供应商的信息。

```c
void __init early_cpu_init(void)/* CPU 初始化 -> 收集 `CPU` 和其供应商的信息 */
{
	const struct cpu_dev *const *cdev;/*  */
	int count = 0;

#ifdef CONFIG_PROCESSOR_SELECT
	pr_info("KERNEL supported cpus:\n");
#endif
    /* arch/x86/kernel/vmlinux.lds.S:248 */
	for (cdev = __x86_cpu_dev_start/* CPU设备 */; cdev < __x86_cpu_dev_end; cdev++) {
		const struct cpu_dev *cpudev = *cdev;

		if (count >= X86_VENDOR_NUM/* x86架构的所有厂商，有中国厂商，也有中国控股的厂商 */)
			break;
		cpu_devs[count] = cpudev;/* CPU 设备 */
		count++;

#ifdef CONFIG_PROCESSOR_SELECT
		{
			unsigned int j;

			for (j = 0; j < 2; j++) {
				if (!cpudev->c_ident[j])
					continue;
				pr_info("  %s %s\n", cpudev->c_vendor,
					cpudev->c_ident[j]);
			}
		}
#endif
	}
	early_identify_cpu(&boot_cpu_data);/*  */
}
```
这里列出厂商的宏定义序号：
```c
#define X86_VENDOR_INTEL	0/* 英特尔 */
#define X86_VENDOR_CYRIX	1/* 赛瑞克斯 */
#define X86_VENDOR_AMD		2/* AMD */
#define X86_VENDOR_UMC		3/* 联华电子 */
#define X86_VENDOR_CENTAUR	5/*  */
#define X86_VENDOR_TRANSMETA	7/*  */
#define X86_VENDOR_NSC		8/*  */
#define X86_VENDOR_HYGON	9/* 海光国产 */
#define X86_VENDOR_ZHAOXIN	10/* 兆芯，国资控股 */
#define X86_VENDOR_NUM		11/* 别的厂商还不认了？ */
```

早期ioremap初始化
------------------------------------------------------------------------------

下一步是初始化早期的 `ioremap`。通常有两种实现与设备通信的方式：

* I/O端口
* 设备内存

我们在 linux [内核启动过程](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-3.html)中见过第一种方法（通过 `outb/inb` 指令实现）。
第二种方法是把 `I/O` 的物理地址映射到虚拟地址。当 `CPU` 读取一段物理地址时，它可以读取到映射了 `I/O` 设备的物理 `RAM` 区域。
`ioremap` 就是用来把设备内存映射到内核地址空间的。

像我上面提到的下一个函数时 `early_ioremap_init`，它可以在正常的像 `ioremap` 这样的映射函数可用之前，把 `I/O` 内存映射到内核地址空间以方便读取。



我们需要在初期的初始化代码中初始化临时的 `ioremap` 来映射 `I/O` 设备到内存区域。初期的 `ioremap` 实现在 [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/ioremap.c) 中可以找到。


```c
/**
 *  `ioremap` 就是用来把设备内存映射到内核地址空间的
 * 
 *  通常有两种实现与设备通信的方式：
 *
 *      (1)I/O端口:(通过 `outb/inb` 指令实现)
 *      (2)设备内存:(把 `I/O` 的物理地址映射到虚拟地址)
 *
 */
void __init early_ioremap_init(void)/* 初始化早期的 `ioremap` */
{
	pmd_t *pmd;

#ifdef CONFIG_X86_64
	BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
#else
	WARN_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
#endif

	early_ioremap_setup();

	pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
	memset(bm_pte, 0, sizeof(bm_pte));
	pmd_populate_kernel(&init_mm, pmd, bm_pte);

	/*
	 * The boot-ioremap range spans multiple pmds, for which
	 * we are not prepared:
	 */
#define __FIXADDR_TOP (-PAGE_SIZE)
	BUILD_BUG_ON((__fix_to_virt(FIX_BTMAP_BEGIN) >> PMD_SHIFT)
		     != (__fix_to_virt(FIX_BTMAP_END) >> PMD_SHIFT));
#undef __FIXADDR_TOP
	if (pmd != early_ioremap_pmd(fix_to_virt(FIX_BTMAP_END))) {
		WARN_ON(1);
		printk(KERN_WARNING "pmd %p != %p\n",
		       pmd, early_ioremap_pmd(fix_to_virt(FIX_BTMAP_END)));
		printk(KERN_WARNING "fix_to_virt(FIX_BTMAP_BEGIN): %08lx\n",
			fix_to_virt(FIX_BTMAP_BEGIN));
		printk(KERN_WARNING "fix_to_virt(FIX_BTMAP_END):   %08lx\n",
			fix_to_virt(FIX_BTMAP_END));

		printk(KERN_WARNING "FIX_BTMAP_END:       %d\n", FIX_BTMAP_END);
		printk(KERN_WARNING "FIX_BTMAP_BEGIN:     %d\n",
		       FIX_BTMAP_BEGIN);
	}
}
```

在 `early_ioremap_init` 的一开始我们可以看到 `pmd_t` 类型的 `pmd` 指针定义（代表页中间目录条目 `typedef struct {pmdval_t pmd; } pmd_t;` 其中 `pmdval_t` 是无符号长整型）。
然后检查 `fixmap` 是正确对齐的：

```C
pmd_t *pmd;
BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
```

`fixmap` - 是一段从 `FIXADDR_START` 到 `FIXADDR_TOP` 的固定虚拟地址映射区域。它在子系统需要知道虚拟地址的编译过程中会被使用。
之后 `early_ioremap_init` 函数会调用 [mm/early_ioremap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/early_ioremap.c) 中的 `early_ioremap_setup` 函数。

```c
void __init early_ioremap_setup(void)
{
	int i;

	for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
		if (WARN_ON(prev_map[i]))
			break;

	for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
		slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
}
```

`early_ioremap_setup` 会填充512个临时的启动时固定映射表来完成无符号长整型矩阵 `slot_virt` 的初始化：

```C
for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
    slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
```

之后我们就获得了 `FIX_BTMAP_BEGIN` 的页中间目录条目，并把它赋值给了 `pmd` 变量，把启动时间页表 `bm_pte` 写满 0。然后调用 `pmd_populate_kernel` 函数设置给定的页中间目录的页表条目：

```C
pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
memset(bm_pte, 0, sizeof(bm_pte));
pmd_populate_kernel(&init_mm, pmd, bm_pte);
```

这就是所有过程。如果你仍然觉得困惑，不要担心。在 [内核内存管理，第二部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/MM/linux-mm-2.html) 章节会有单独一部分讲解 `ioremap` 和 `fixmaps`。

获取根设备的主次设备号
----------------------------------------------------------------------------

`ioremap` 初始化完成后，紧接着是执行下面的代码：

```C
ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
```

这段代码用来获取根设备的主次设备号。后面 `initrd` 会通过 `do_mount_root` 函数挂载到这个根设备上。其中主设备号用来识别和这个设备有关的驱动。
次设备号用来表示使用该驱动的各设备。注意 `old_decode_dev` 函数是从 `boot_params_structure` 中获取了一个参数。我们可以从 x86 linux 内核启动协议中查到：

```
Field name:	root_dev
Type:		modify (optional)
Offset/size:	0x1fc/2
Protocol:	ALL

  The default root device device number.  The use of this field is
  deprecated, use the "root=" option on the command line instead
```

现在我们来看看 `old_decode_dev` 如何实现的。实际上它只是根据主次设备号调用了 `MKDEV` 来生成一个 `dev_t` 类型的设备。它的实现很简单：

```C
static inline dev_t old_decode_dev(u16 val)
{
         return MKDEV((val >> 8) & 255, val & 255);
}
```

其中 `dev_t` 是用来表示主/次设备号对的一个内核数据类型。但是这个奇怪的 `old` 前缀代表了什么呢？出于历史原因，有两种管理主次设备号的方法。
第一种方法主次设备号占用 2 字节。你可以在以前的代码中发现：主设备号占用 8 bit，次设备号占用 8 bit。但是这会引入一个问题：最多只能支持 256 个主设备号和 256 个次设备号。
因此后来引入了 32 bit 来表示主次设备号，其中 12 位用来表示主设备号，20 位用来表示次设备号。你可以在 `new_decode_dev` 的实现中找到：

```C
static inline dev_t new_decode_dev(u32 dev)
{
         unsigned major = (dev & 0xfff00) >> 8;
         unsigned minor = (dev & 0xff) | ((dev >> 12) & 0xfff00);
         return MKDEV(major, minor);
}
```

如果 `dev` 的值是 `0xffffffff`，经过计算我们可以得到用来表示主设备号的 12 位值 `0xfff`，表示次设备号的20位值 `0xfffff`。因此经过 `old_decode_dev` 我们最终可以得到在 `ROOT_DEV` 中根设备的主次设备号。

Memory Map设置
-----------------------------------------------------------------------

下一步是调用 `setup_memory_map` 函数设置内存映射。但是在这之前我们需要设置与显示屏有关的参数（目前有行、列，视频页等，你可以在 [显示模式初始化和进入保护模式](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-3.html) 中了解），
与拓展显示识别数据，视频模式，引导启动器类型等参数：

```C
screen_info = boot_params.screen_info;
edid_info = boot_params.edid_info;
saved_video_mode = boot_params.hdr.vid_mode;
bootloader_type = boot_params.hdr.type_of_loader;
if ((bootloader_type >> 4) == 0xe) {
	bootloader_type &= 0xf;
	bootloader_type |= (boot_params.hdr.ext_loader_type+0x10) << 4;
}
bootloader_version  = bootloader_type & 0xf;
bootloader_version |= boot_params.hdr.ext_loader_ver << 4;
```

我们可以从启动时候存储在 `boot_params` 结构中获取这些参数信息。之后我们需要设置 `I/O` 内存。众所周知，内核主要做的工作就是资源管理。其中一个资源就是内存。
我们也知道目前有通过 `I/O` 口和设备内存两种方法实现设备通信。所有有关注册资源的信息可以通过 `/proc/ioports` 和 `/proc/iomem` 获得：

* /proc/ioports - 提供用于设备输入输出通信的一租注册端口区域
* /proc/iomem - 提供每个物理设备的系统内存映射地址

我们先来看下 `/proc/iomem`：

```
cat /proc/iomem
00000000-00000fff : reserved
00001000-0009d7ff : System RAM
0009d800-0009ffff : reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000cffff : Video ROM
000d0000-000d3fff : PCI Bus 0000:00
000d4000-000d7fff : PCI Bus 0000:00
000d8000-000dbfff : PCI Bus 0000:00
000dc000-000dffff : PCI Bus 0000:00
000e0000-000fffff : reserved
000e0000-000e3fff : PCI Bus 0000:00
000e4000-000e7fff : PCI Bus 0000:00
000f0000-000fffff : System ROM
```
看下 `/proc/ioports`：
```
cat /proc/ioports 
0000-0cf7 : PCI Bus 0000:00
0000-001f : dma1
0020-0021 : pic1
0040-0043 : timer0
0050-0053 : timer1
0060-0060 : keyboard
0064-0064 : keyboard
0070-0071 : rtc0
0080-008f : dma page reg
00a0-00a1 : pic2
00c0-00df : dma2
```

可以看到iomem，根据不同属性划分为以十六进制符号表示的一段地址范围。linux 内核提供了用来管理所有资源的一种通用 API。全局资源（比如 PICs 或者 I/O 端口）可以划分为与硬件总线插槽有关的子集。
`resource` 的主要结构是：

```C
struct resource {
	resource_size_t start;
	resource_size_t end;
	const char *name;
	unsigned long flags;
	unsigned long desc;
	struct resource *parent, *sibling, *child;
};
```

例如下图中的树形系统资源子集示例。这个结构提供了资源占用的从 `start` 到 `end` 的地址范围（`resource_size_t` 是 `phys_addr_t` 类型，在 `x86_64` 架构上是 `u64`）。
资源名（你可以在 `/proc/iomem` 输出中看到），资源标记（所有的资源标记定义在 [include/linux/ioport.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/ioport.h) 文件中）。最后三个是资源结构体指针，如下图所示：

```
+-------------+      +-------------+
|             |      |             |
|    parent   |------|    sibling  |
|             |      |             |
+-------------+      +-------------+
       |
       |
+-------------+
|             |
|    child    | 
|             |
+-------------+
```

每个资源子集有自己的根范围资源。`iomem` 的资源 `iomem_resource` 的定义是：

```C
struct resource iomem_resource = {
        .name   = "PCI mem",
        .start  = 0,
        .end    = -1,
        .flags  = IORESOURCE_MEM,
};
EXPORT_SYMBOL(iomem_resource);

/*
 * IO resources have these defined flags.
 *
 * PCI devices expose these flags to userspace in the "resource" sysfs file,
 * so don't move them.
 */
#define IORESOURCE_BITS		0x000000ff	/* Bus-specific bits */

#define IORESOURCE_TYPE_BITS	0x00001f00	/* Resource type */
#define IORESOURCE_IO		0x00000100	/* PCI/ISA I/O ports */
#define IORESOURCE_MEM		0x00000200
#define IORESOURCE_REG		0x00000300	/* Register offsets */
#define IORESOURCE_IRQ		0x00000400
#define IORESOURCE_DMA		0x00000800
#define IORESOURCE_BUS		0x00001000

#define IORESOURCE_PREFETCH	0x00002000	/* No side effects */
#define IORESOURCE_READONLY	0x00004000
#define IORESOURCE_CACHEABLE	0x00008000
#define IORESOURCE_RANGELENGTH	0x00010000
#define IORESOURCE_SHADOWABLE	0x00020000

#define IORESOURCE_SIZEALIGN	0x00040000	/* size indicates alignment */
#define IORESOURCE_STARTALIGN	0x00080000	/* start field is alignment */

#define IORESOURCE_MEM_64	0x00100000
#define IORESOURCE_WINDOW	0x00200000	/* forwarded by bridge */
#define IORESOURCE_MUXED	0x00400000	/* Resource is software muxed */

#define IORESOURCE_EXT_TYPE_BITS 0x01000000	/* Resource extended types */
#define IORESOURCE_SYSRAM	0x01000000	/* System RAM (modifier) */

/* IORESOURCE_SYSRAM specific bits. */
#define IORESOURCE_SYSRAM_DRIVER_MANAGED	0x02000000 /* Always detected via a driver. */
#define IORESOURCE_SYSRAM_MERGEABLE		0x04000000 /* Resource can be merged. */

#define IORESOURCE_EXCLUSIVE	0x08000000	/* Userland may not map this resource */

#define IORESOURCE_DISABLED	0x10000000
#define IORESOURCE_UNSET	0x20000000	/* No address assigned yet */
#define IORESOURCE_AUTO		0x40000000
#define IORESOURCE_BUSY		0x80000000	/* Driver has marked this resource busy */

```

`iomem_resource` 利用 `PCI mem` 名字和 `IORESOURCE_MEM (0x00000200)` 标记定义了 `io` 内存的根地址范围。就像上文提到的，我们目前的目的是设置 `iomem` 的结束地址，我们需要这样做：

```C
iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1;
```

我们对1左移 `boot_cpu_data.x86_phys_bits`。`boot_cpu_data` 是我们在执行 `early_cpu_init` 的时候初始化的 `cpuinfo_x86` 结构。从字面理解，`x86_phys_bits` 代表系统可达到的最大内存地址时需要的比特数。
另外，`iomem_resource` 是通过 `EXPORT_SYMBOL` 宏传递的。这个宏可以把指定的符号（例如 `iomem_resource`）做动态链接。换句话说，它可以支持动态加载模块的时候访问对应符号。

设置完根 `iomem` 的资源地址范围的结束地址后，下一步就是设置内存映射。它通过调用 `setup_memory_map` 函数实现(5.10.13中叫做e820__memory_setup)：

```c
void __init setup_memory_map(void)
{
    char *who;

    who = x86_init.resources.memory_setup();
    memcpy(&e820_saved, &e820, sizeof(struct e820map));
    printk(KERN_INFO "e820: BIOS-provided physical RAM map:\n");
    e820_print_map(who);
}
```

首先，我们来看下 `x86_init.resources.memory_setup`。`x86_init` 是一种 `x86_init_ops` 类型的结构体，用来表示项资源初始化，`pci` 初始化平台特定的一些设置函数。
`x86_init` 的初始化实现在 [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c) 文件中。我不会全部解释这个初始化过程，因为我们只关心一个地方：

```C
struct x86_init_ops x86_init __initdata = {
	.resources = {
            .probe_roms             = probe_roms,
            .reserve_resources      = reserve_standard_io_resources,
            .memory_setup           = default_machine_specific_memory_setup,
    },
    ...
    ...
    ...
}
```

我们可以看到，这里的 `memory_setup` 赋值为 `default_machine_specific_memory_setup`（5.10.13中为e820__memory_setup_default），它是我们在对 [内核启动](https://xinqiu.gitbooks.io/linux-insides-cn/content/Booting/linux-bootstrap-2.html) 过程中的所有 [e820](http://en.wikipedia.org/wiki/E820) 条目经过整理和把内存分区填入 `e820map` 结构体中获得的。

```c
/*
 * Pass the firmware (bootloader) E820 map to the kernel and process it:
 */
char *__init e820__memory_setup_default(void)
{
	char *who = "BIOS-e820";

	/*
	 * Try to copy the BIOS-supplied E820-map.
	 *
	 * Otherwise fake a memory map; one section from 0k->640k,
	 * the next section from 1mb->appropriate_mem_k
	 */
	if (append_e820_table(boot_params.e820_table, boot_params.e820_entries) < 0) {
		u64 mem_size;

		/* Compare results from other methods and take the one that gives more RAM: */
		if (boot_params.alt_mem_k < boot_params.screen_info.ext_mem_k) {
			mem_size = boot_params.screen_info.ext_mem_k;
			who = "BIOS-88";
		} else {
			mem_size = boot_params.alt_mem_k;
			who = "BIOS-e801";
		}

		e820_table->nr_entries = 0;
		e820__range_add(0, LOWMEMSIZE(), E820_TYPE_RAM);
		e820__range_add(HIGH_MEMORY, mem_size << 10, E820_TYPE_RAM);
	}

	/* We just appended a lot of ranges, sanitize the table: */
	e820__update_table(e820_table);

	return who;
}
```
> e820是和BIOS的一个中断相关的，具体说是 int 0x15。之所以叫 e820 是因为在用这个中断时 ax 必须是 0xe820。这个中断的作用是得到系统的内存布局。因为系统内存会有很多段，每段的类型属性也不一样，所以这个查询是“迭代式”的，每次求得一个段。

所有收集的内存分区会用 `printk` 打印出来。你可以通过运行 `dmesg` 命令找到类似于下面的信息：

```
$ dmesg | grep e820
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000bff7ffff] usable
[    0.000000] BIOS-e820: [mem 0x00000000bff80000-0x00000000bfffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000feffc000-0x00000000feffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000100000000-0x000000023fffffff] usable
[    0.000000] e820: update [mem 0x00000000-0x00000fff] usable ==> reserved
[    0.000000] e820: remove [mem 0x000a0000-0x000fffff] usable
[    0.000000] e820: last_pfn = 0x240000 max_arch_pfn = 0x400000000
[    0.000000] e820: last_pfn = 0xbff80 max_arch_pfn = 0x400000000
[    0.000000] e820: [mem 0xc0000000-0xfeffbfff] available for PCI devices
[    1.146134] e820: reserve RAM buffer [mem 0x0009fc00-0x0009ffff]
[    1.146136] e820: reserve RAM buffer [mem 0xbff80000-0xbfffffff]
```

复制 `BIOS` 增强磁盘设备信息
--------------------------------------------------------------------------------------------

下面两部是通过 `parse_setup_data` 函数解析 `setup_data`，并且把 `BIOS` 的 `EDD` 信息复制到安全的地方。
`setup_data` 是内核启动头中包含的字段，我们可以在 `x86` 的启动协议中了解：

```
Field name:	setup_data
Type:		write (special)
Offset/size:	0x250/8
Protocol:	2.09+

  The 64-bit physical pointer to NULL terminated single linked list of
  struct setup_data. This is used to define a more extensible boot  
  parameters passing mechanism.
```

它用来存储不同类型的设置信息，例如设备树 `blob`，`EFI` 设置数据等等。第二步是从 `boot_params` 结构中复制我们在 [arch/x86/boot/edd.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/edd.c) 中 `BIOS` 的 `EDD` 信息到 `edd` 结构中。

```C
static inline void __init copy_edd(void)
{
     memcpy(edd.mbr_signature, boot_params.edd_mbr_sig_buffer,
            sizeof(edd.mbr_signature));
     memcpy(edd.edd_info, boot_params.eddbuf, sizeof(edd.edd_info));
     edd.mbr_signature_nr = boot_params.edd_mbr_sig_buf_entries;
     edd.edd_info_nr = boot_params.eddbuf_entries;
}
```

内存描述符初始化
--------------------------------------------------------------------------------------------------

下一步是在初始化阶段完成内存描述符的初始化。我们知道每个进程都有自己的运行内存地址空间。通过调用 `memory descriptor` 可以看到这些特殊数据结构。
在 linux 内核源码中内存描述符是用 `mm_struct` 结构体表示的。`mm_struct` 包含许多不同的与进程地址空间有关的字段，像内核代码/数据段的起始和结束地址，`brk` 的起始和结束，内存区域的数量，内存区域列表等。这些结构定义在 [include/linux/mm_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/mm_types.h) 中。`task_struct` 结构的 `mm` 和 `active_mm` 字段包含了每个进程自己的内存描述符。
我们的第一个 `init` 进程也有自己的内存描述符。在之前的[章节](https://xinqiu.gitbooks.io/linux-insides-cn/content/Initialization/linux-initialization-4.html)我们看到过通过 `INIT_TASK` 宏实现 `task_struct` 的部分初始化信息：

```C
#define INIT_TASK(tsk)  \
{
    ...
	...
	...
	.mm = NULL,         \
    .active_mm  = &init_mm, \
	...
}
```

`mm` 指向进程地址空间，`active_mm` 指向像内核线程这样子不存在地址空间的有效地址空间（你可以在这个[文档](https://www.kernel.org/doc/Documentation/vm/active_mm.txt) 中了解更多内容）。
接下来我们在初始化阶段完成内存描述符中内核代码段，数据段和 `brk` 段的初始化：

```C
    init_mm.start_code = (unsigned long) _text;
	init_mm.end_code = (unsigned long) _etext;
	init_mm.end_data = (unsigned long) _edata;
	init_mm.brk = _brk_end;
```

`init_mm` 是初始化阶段的内存描述符定义：

```C

/*
 * For dynamically allocated mm_structs, there is a dynamically sized cpumask
 * at the end of the structure, the size of which depends on the maximum CPU
 * number the system can see. That way we allocate only as much memory for
 * mm_cpumask() as needed for the hundreds, or thousands of processes that
 * a system typically runs.
 *
 * Since there is only one init_mm in the entire system, keep it simple
 * and size this cpu_bitmask to NR_CPUS.
 */
struct mm_struct init_mm = {/* 初始的 mm_struct 结构 */
	.mm_rb		= RB_ROOT,/* 红黑树 */
	.pgd		= swapper_pg_dir,/* 一级页表 */
	.mm_users	= ATOMIC_INIT(2),/* 用户空间使用的用户数 */
	.mm_count	= ATOMIC_INIT(1),/* 内存引用次数 */
	.write_protect_seq = SEQCNT_ZERO(init_mm.write_protect_seq),/* 写被保护的 pages 强制后期 COW */
	MMAP_LOCK_INITIALIZER(init_mm)/* 读写锁 */
	.page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),/* 保护页表和一些计数器 */
	.arg_lock	=  __SPIN_LOCK_UNLOCKED(init_mm.arg_lock),/* 保护 mm_struct 中的一些参数 */
	.mmlist		= LIST_HEAD_INIT(init_mm.mmlist),/* 保存可能被 swap 的链表 */
	.user_ns	= &init_user_ns,/* namespace 资源的隔离， cgroup=资源的限制*/
	.cpu_bitmap	= CPU_BITS_NONE,/*  */
	INIT_MM_CONTEXT(init_mm)/* TODO */
};
```

其中 `mm_rb` 是虚拟内存区域的红黑树结构，`pgd` 是全局页目录的指针，`mm_user` 是使用该内存空间的进程数目，`mm_count` 是主引用计数，`mmap_sem` 是内存区域信号量(在5.10.13中已经改为了读写信号量mmap_lock)。

在初始化阶段完成内存描述符的设置后，下一步是通过 `mpx_mm_init` 完成 `Intel` 内存保护扩展的初始化。下一步是代码/数据/`bss` 资源的初始化：

```C
code_resource.start = __pa_symbol(_text);   /* 代码段 */
code_resource.end = __pa_symbol(_etext)-1;
rodata_resource.start = __pa_symbol(__start_rodata);    /* 只读区域 */
rodata_resource.end = __pa_symbol(__end_rodata)-1;
data_resource.start = __pa_symbol(_sdata);      /* 数据段 */
data_resource.end = __pa_symbol(_edata)-1;
bss_resource.start = __pa_symbol(__bss_start);  /* bss 存放静态全局变量的区域 */
bss_resource.end = __pa_symbol(__bss_stop)-1;
```
他们的静态初始化如下：
```c
static struct resource rodata_resource = {
	.name	= "Kernel rodata",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_SYSTEM_RAM
};

static struct resource data_resource = {
	.name	= "Kernel data",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_SYSTEM_RAM
};

static struct resource code_resource = {/*  */
	.name	= "Kernel code",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_SYSTEM_RAM
};

static struct resource bss_resource = {/*  */
	.name	= "Kernel bss",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_SYSTEM_RAM
};
```
通过上面我们已经知道了一小部分关于 `resource` 结构体的样子。在这里，我们把物理地址段赋值给代码/数据/`bss` 段。你可以在 `/proc/iomem` 中看到：

```
$ sudo cat /proc/iomem | grep "Kernel " -B 1
100000000-23fffffff : System RAM
  164800000-164f9325e : Kernel code
  164f9325f-16556817f : Kernel data
  165762000-165a6efff : Kernel bss
```
在 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) 中有所有这些结构体的定义：

```c
static struct resource code_resource = {
	.name	= "Kernel code",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_MEM
};
```

本章节涉及的最后一部分就是 `NX` 配置。`NX-bit` 或者 `no-execute` 位是页目录条目的第 63 比特位。它的作用是控制被映射的物理页面是否具有执行代码的能力。

这个比特位只会在通过把 `EFER.NXE` 置为1使能 `no-execute` 页保护机制的时候被使用/设置。在 `x86_configure_nx` 函数中会检查 `CPU` 是否支持 `NX-bit`，以及是否被禁用。
经过检查后，我们会根据结果给 `_supported_pte_mask` 赋值：

```C
void x86_configure_nx(void)/*No Exec  */
{
	if (boot_cpu_has(X86_FEATURE_NX/* 不可执行 */) && !disable_nx)
		__supported_pte_mask |= _PAGE_NX;/* No exec */
	else
		__supported_pte_mask &= ~_PAGE_NX;
}
```
再给出每位的宏定义：
```c
#define _PAGE_BIT_PRESENT	0	/* is present */
#define _PAGE_BIT_RW		1	/* writeable */
#define _PAGE_BIT_USER		2	/* userspace addressable */
#define _PAGE_BIT_PWT		3	/* page write through */
#define _PAGE_BIT_PCD		4	/* page cache disabled */
#define _PAGE_BIT_ACCESSED	5	/* was accessed (raised by CPU) */
#define _PAGE_BIT_DIRTY		6	/* was written to (raised by CPU) */
#define _PAGE_BIT_PSE		7	/* 4 MB (or 2MB) page */
#define _PAGE_BIT_PAT		7	/* on 4KB pages */
#define _PAGE_BIT_GLOBAL	8	/* Global TLB entry PPro+ */
#define _PAGE_BIT_SOFTW1	9	/* available for programmer */
#define _PAGE_BIT_SOFTW2	10	/* " */
#define _PAGE_BIT_SOFTW3	11	/* " */
#define _PAGE_BIT_PAT_LARGE	12	/* On 2MB or 1GB pages */
#define _PAGE_BIT_SOFTW4	58	/* available for programmer */
#define _PAGE_BIT_PKEY_BIT0	59	/* Protection Keys, bit 1/4 */
#define _PAGE_BIT_PKEY_BIT1	60	/* Protection Keys, bit 2/4 */
#define _PAGE_BIT_PKEY_BIT2	61	/* Protection Keys, bit 3/4 */
#define _PAGE_BIT_PKEY_BIT3	62	/* Protection Keys, bit 4/4 */
#define _PAGE_BIT_NX		63	/* No execute: only valid after cpuid check */
```

结论
---------------------------------------------------------------------------------------------------

以上是 linux 内核初始化过程的第五部分。在这一章我们讲解了有关架构初始化的 `setup_arch` 函数。内容很多，但是我们还没有学习完。其中，`setup_arch`
 是一个很复杂的函数，甚至我不确定我们能在以后的章节中讲完它的所有内容。在这一章节中有一些很有趣的概念像 `Fix-mapped` 地址，`ioremap` 等等。
如果没听明白也不用担心，在 [内核内存管理，第二部分](https://xinqiu.gitbooks.io/linux-insides-cn/content/MM/linux-mm-2.html) 还会有更详细的解释。在下一章节我们会继续讲解有关结构初始化的东西，
以及初期内核参数的解析，`pci` 设备的早期转存，直接媒体接口扫描等等。

如果你有任何问题或者建议，你可以留言，也可以直接发送消息给我[twitter](https://twitter.com/0xAX)。

**很抱歉，英语并不是我的母语，非常抱歉给您阅读带来不便，如果你发现文中描述有任何问题，清提交一个 PR 到 [linux-insides](https://github.com/MintCN/linux-insides-zh)。**

链接
--------------------------------------------------------------------------------

* [mm vs active_mm](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)
* [e820](http://en.wikipedia.org/wiki/E820)
* [Supervisor mode access prevention](https://lwn.net/Articles/517475/)
* [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Memory mapped I/O](http://en.wikipedia.org/wiki/Memory-mapped_I/O)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [PDF. dwarf4 specification](http://dwarfstd.org/doc/DWARF4.pdf)
* [Call stack](http://en.wikipedia.org/wiki/Call_stack)
* [内核初始化. Part 4.](http://github.com/MintCN/linux-insides-zh/blob/master/Initialization/linux-initialization-4.md)
