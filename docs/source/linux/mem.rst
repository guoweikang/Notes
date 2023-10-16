
===========
内存管理
===========

前言
=====
前提至少学习过 X86/ARM/RISC-V 一门体系架构

什么是内存
----------
我的个人看法: 在计算机体系架构中，ALU计算单元必须需要存储参与工作，用来作为 数据输入、输出 保存(简单点就是数据需要存放)
正常来说，能够直接被CPU *寻址* 访问到的存储单元，都可以叫做内存，而其物理、电子形态、大小则是会随着基础科学发展而不断演进

下面是 DDRAM 工作方式的一个小科普，DDRAM 目前应该是主流 内存形态的一种
https://www.youtube.com/watch?v=fpnE6UAfbtU&list=PL8dPuuaLjXtNlUrzyH5r6jN9ulIgZBpdo&index=7


什么是内存管理
---------------
下图是一个非常简单的场景, CPU指令执行，需要加载指令代码，在执行函数，需要用到栈, 因此需要根据某种规定，对内存进行功能划分，比如哪段内存是用来存放代码的，
哪段内存又是用来作为程序的栈等

.. image:: ./images/mem/1.png
 :width: 400px

上面我们只是描述了一个非常简单的场景，只是想表达，内存管理功能之一是保证内存被合理的分配使用，当然还有许多其他工作需要完成，接下来会一一展开


基础知识背景
=============
本章主要是对内存管理的基础概念和技术做一个介绍，为后面更深入的讨论做好基础

什么是虚拟内存
----------------
我们先假设一下，如果没有虚拟内存，应用是如何工作的

.. image:: ./images/mem/2.png
 :width: 400px

上图是假设有两个应用同时运行的情况，我标注了一些可能发生的情况 

1. 如果进程都是直接访问物理内存，意味着进程可以访问任意的物理内存，则可能发生内存踩踏， 当然这种情况可以通过一些约束限制
2. 物理内存可能是不连续的，如果一个进程分配比较多的连续内存，内存不足，则该进程无法被加载，当然也可以通过交换进程(把没有在调度的进程放到磁盘)
3. 假设我们在给每个进程分配物理内存，是按照连续内存分配(难道不应该吗)，则交换进程的代价过于庞大
4. 编译器在编译用户态应用，应该怎么给变量、函数分配地址。

总之，出于用户态进程对于*空间隔离*的需求，虚拟内存技术孕育而生

空间隔离
^^^^^^^^^

 - 屏蔽掉物理内存的存在, 让不同进程 以为是自己在独享一段内存
 - 屏蔽掉物理内存的大小、分布；让不通进程以为自己有很大的内存，并且内存是连续的

虚拟内存地址翻译
------------------
这里简单介绍一下目前主流的虚拟内存翻译技术:MMU 

.. image:: ./images/mem/3.png
  :width: 400px

知识点: 

 - CPU在执行指令，指令访问的地址是虚拟地址，但是实际上，该虚拟地址会经过MMU硬件+映射表查找，最终访问物理内存地址
 - MMU的配置以及页表的维护 是内存管理模块完成的


什么是页表
^^^^^^^^^^^
页表就是一个用于存放物理内存信息的 数组结构

.. image:: ./images/mem/7.png
 :width: 400px

 - 页表存放在物理内存中，MMU使用指定的寄存器存储 页表基址
 - 利用要访问的虚拟内存的信息，获得页表中索引 
 - 利用索引得到的 页表条目(entry) 获得实际的物理内存
 - 访问实际的物理内存 

什么是页表大小
^^^^^^^^^^^^^^^
页表大小一般指每一个 页表条目所能代表的 物理内存范围 

.. image:: ./images/mem/15.png
 :width: 400px

上图很好的解释了为什么需要页表大小 

.. image:: ./images/mem/16.png
 :width: 400px


什么是多级页表
^^^^^^^^^^^^^^^

- 如果页表条目太大，内存会浪费
- 但是如果页表条目太小，会占用过多内存 

.. image:: ./images/mem/17.png
 :width: 400px

多级页表，第一级页表表示范围更大

ARM64架构下的MMU
^^^^^^^^^^^^^^^^^^^
MMU对内存的处理，一般都会和架构相关，我们选择ARM64架构下MMU作为一个介绍

下图是一个MMU寻址的流水线图


.. image:: ./images/mem/8.png
 :width: 400px


下图是一个MMU实际寻址的过程 

.. image:: ./images/mem/10.png
 :width: 400px
 

上面我们大致介绍了寻址流程，命中的情况暂且不考虑，我们主要看一下，虚拟内存是如何访问物理内存的

ARM64支持
^^^^^^^^^^^^^
虚拟地址到物理地址的映射通过查表的机制来实现，ARMv8中，Kernel Space的页表基地址存放在TTBR1_EL1寄存器中，
User Space页表基地址存放在TTBR0_EL0寄存器中，其中内核地址空间的高位为全1，(0xFFFF0000_00000000 ~ 0xFFFFFFFF_FFFFFFFF)，
用户地址空间的高位为全0，(0x00000000_00000000 ~ 0x0000FFFF_FFFFFFFF)

.. image:: ./images/mem/11.png
 :width: 400px
 

64位虚拟地址中，并不是所有位都用上，除了高16位用于区分内核空间和用户空间外，有效位的配置可以是：36, 39, 42, 47。

.. image:: ./images/mem/12.png
 :width: 400px
 
页面大小支持3种页面大小：4KB, 16KB, 64KB

页表支持至少两级页表，至多四级页表，Level 0 ~ Level 3

ARM会根据实际使用的VA大小，以及选择的页面大小，来决定应该使用几级页表 
页表映射层级: 支持2-4级，

 - 16K页表，36VA 2级页表
 - 64K页表，42VA 2级页表
 - 64K页表，48/52VA 3级页表
 - 4K页表，39VA 3级页表
 - 不是64K页表，也不是48VA，默认都是4级页表


寻址介绍
^^^^^^^^^^^^^

第一种情况: 虚拟内存范围39BIT，也就是512GB的内存范围，并且使用4KB页表，默认使用3级页表 

.. image:: ./images/mem/18.png
 :width: 400px

下图我们是我们自己的一个分解 

.. image:: ./images/mem/19.png
 :width: 400px

下图是页表内容的描述

.. image:: ./images/mem/20.png
 :width: 400px


Linux对页表的定义
^^^^^^^^^^^^^^^^^^
由于不同页表的描述 在ARM下有规定，因此，linux把各级页表分别命名: PGD PUD(3级页表没有)  PMD PTE

.. image:: ./images/mem/21.png
 :width: 400px


Linux源码分析
^^^^^^^^^^^^^^
代码路径：

 - arch/arm64/include/asm/pgtable-types.h：定义pgd_t, pud_t, pmd_t, pte_t等类型；
 - arch/arm64/include/asm/pgtable-prot.h：针对页表中entry中的权限内容设置；
 - arch/arm64/include/asm/pgtable-hwdef.h：主要包括虚拟地址中PGD/PMD/PUD等的划分，这个与虚拟地址的有效位及分页大小有关，
   此外还包括硬件页表的定义， TCR寄存器中的设置等；
 - arch/arm64/include/asm/pgtable.h：页表设置相关；

在ARM中，与页表相关的寄存器有：TCR_EL1, TTBRx_EL1

关键代码：


.. code-block:: c
    :linenos:

	/* 
	 * 计算不同 页表映射的entry表示的大小位移，比如:
     * 假设当前是4级页表 ( 0<=n <= 3)
     * PTE SHIFT：4K = (12 - 3) * 1 + 3 = 12    	 
	 * PMD SHIFT: 2M = (12 - 3) * 2 + 3 = 21   
	 * PUD SHIFT: 1G = (12 - 3) * 3 + 3 = 30 
	 * PGD SHIFT：512G = (12 - 3) * 4 + 3 = 39 
	 * 假设当前是3级页表 ( 0<=n <= 3)
     * PTE SHIFT：4K = (12 - 3) * 1 + 3 = 12    	 
	 * PMD SHIFT: 2M = (12 - 3) * 2 + 3 = 21   
	 * PGD SHIFT：1G = (12 - 3) * 3 + 3 = 30 
	*/
	#define ARM64_HW_PGTABLE_LEVEL_SHIFT(n) ((PAGE_SHIFT - 3) * (4 - (n)) + 3)

	#define PMD_SHIFT               ARM64_HW_PGTABLE_LEVEL_SHIFT(2)  //如上21
	#define PMD_SIZE                (_AC(1, UL) << PMD_SHIFT) // 2M
	
	// 注意: 三级页表 没有PUD 
	#define PUD_SHIFT               ARM64_HW_PGTABLE_LEVEL_SHIFT(1) //如上30
	#define PUD_SIZE                (_AC(1, UL) << PUD_SHIFT) // 1G
	
	// 注意: 三四级页表下 PGD 含义不同，三级页表，PGD = PUD 
	#define PGDIR_SHIFT             ARM64_HW_PGTABLE_LEVEL_SHIFT(4 - CONFIG_PGTABLE_LEVELS)
	#define PGDIR_SIZE              (_AC(1, UL) << PGDIR_SHIFT)
	
	
	
虚拟内存空间整体分布
---------------------

空间布局整体介绍
^^^^^^^^^^^^^^^^

虽然不同进程的虚拟内存是隔离的，但是实际上隔离的主要是用户态，内核态依然是共享的
下图分别是32位 64位下用户态和内核态的内存分布

.. image:: ./images/mem/5.png
 :width: 400px
 
arm64的内存布局在linux文档中有比较详细的描述  
https://www.kernel.org/doc/Documentation/arm64/memory.rst

文档中基本上可以知道 arm64架构下: 
 - 内核态地址范围从 0xffff000000000000 到0xffffffffffffffff (前16bit都是1)
 - 用户态地址范围从 0x0000000000000000 到0x0000ffffffffffff (前16bit都是0)
 - 有一段空洞范围地址不使用(前16bit) 也就是一共只使用了2个48bit的地址范围(256TB)，一共是512TB
 - 内核态256TB的内存又根据功能划分了不同的地址范围


虚拟内存的访问
--------------
不通用户态访问进程访问各自的用户态虚拟内存

.. image:: ./images/mem/4.png
 :width: 400px
 
知识点:
   - 不通进程的虚拟内存，就算虚拟内存是相同的，但是实际上访问的不通的物理内存
   - 内存管理模块会管理这种关系，硬件会通过某种机制自动完成地址翻译(后面讲)

下图是不同用户态访问内核地址空间

.. image:: ./images/mem/6.png
 :width: 400px

知识点: 
 - 只有在内核态才能访问内核的地址
 - 用户态程序进入内核态之后，访问的是同一套内核地址


内核内存管理
=============

前置基础
--------------------

用到的掌握的汇编指令
^^^^^^^^^^^^^^^^^^^^^^

adrp 指令: ADRP  Xd, label; 利用当前PC 和label的相对地址，计算label 内存地址的4KB基址 

.. code-block:: c
    :linenos:
	
	//如果PC 当前指令地址为 0x1000 0000 ; data 相对 0x1000 0000 的偏移是 0x1234，
	//可以得到data的地址为0x1000 1234，他的内存基址就是 0x1000 1000
	// X0的值就为  0x1000 1000
	ADRP  X0, data；


内核内存分布
-------------

整体布局描述
^^^^^^^^^^^^^

内核的地址分布描述定义在:  arch/arm64/include/asm/memory.h 
假设当前配置: 4K页(CONFIG_PAGE_SHIFT=12) VA地址是48BIT(256TB)

.. code-block:: c
    :linenos:
	
	/* 
	 * STRUCT_PAGE_MAX_SHIFT 定义了一个 管理页表结构(struct page)的大小
	 * PAGE_SHIT 是页表大小位移(比如 4K是12 16K是14 64K是16)
	 * VMEMMAP_SHIFT 是用于计算线性地址大小的除数
	 * 举例: 为了管理4GB大小的线性地址，需要使用 4GB/4KB = 1024 个页表, 每个页表大小如果占1B, 需要 1024 * 1B 的内存 
	 * 因此页表所占内存计算公式为 : 需要映射的内存大小/页大小*页表内存 =   需要映射的内存大小/ (页大小 - 页表内存) 
	*/
	
	#define VMEMMAP_SHIFT   (PAGE_SHIFT - STRUCT_PAGE_MAX_SHIFT) // 目前是: 12（4KB） - 6(64B) = 6
	// 计算管理128TB(0xffff800000000000 - 0xffff000000000000) 的线性内存 页表条目需要使用的内存大小	
	#define VMEMMAP_SIZE    ((_PAGE_END(VA_BITS_MIN) - PAGE_OFFSET) >> VMEMMAP_SHIFT) // 目前是 128TB/4KB*64B=2TB  
	
	#define VA_BITS                 (CONFIG_ARM64_VA_BITS)                           
	#define _PAGE_OFFSET(va)        (-(UL(1) << (va)))      //内核地址起始地址  0xffff000000000000                        
	#define PAGE_OFFSET             (_PAGE_OFFSET(VA_BITS)) //内核地址起始地址  0xffff 0000 0000 0000                            
	#define KIMAGE_VADDR            (MODULES_END)   //kernel image的VA地址 位于modules 结束 0xffff800007ffffff                                     
	#define MODULES_END             (MODULES_VADDR + MODULES_VSIZE)   //modules结束地址 0xffff800007ffffff                   
	#define MODULES_VADDR           (_PAGE_END(VA_BITS_MIN))  //modules起始地址 0xffff800000000000                                 
	#define MODULES_VSIZE           (SZ_128M)  //modules大小 128M                                       
	#define VMEMMAP_START           (-(UL(1) << (VA_BITS - VMEMMAP_SHIFT))) // fffffc0000000000        
	#define VMEMMAP_END             (VMEMMAP_START + VMEMMAP_SIZE) // 2TB大小: fffffdffffffffff                 
	#define PCI_IO_END              (VMEMMAP_START - SZ_8M)                          
	#define PCI_IO_START            (PCI_IO_END - PCI_IO_SIZE)                       
	#define FIXADDR_TOP             (VMEMMAP_START - SZ_32M) 
	
	#define _PAGE_END(va)           (-(UL(1) << ((va) - 1)))

下图以VA 39BIT 和 4K页做演示

.. image:: ./images/mem/22.png
 :width: 800px
 
 
 
下图以VA48 BIT 和 4K页做演示

.. image:: ./images/mem/23.png
 :width: 800px
 
内核内存管理的一部分工作，就是负责管理不同区域的内存的分配、释放

接下来我们将先按照 内核启动顺序 剖析内核内存初始化过程  


内核启动的内存初始化
--------------------


内核镜像
^^^^^^^^^^

内核镜像我们简单也可以理解为是一个二进制的文件，主要定义了代码段的布局情况,用于指导内核镜像分段的文件位于: 
arch/arm64/kernel/vmlinux.lds.S, SECTIONS 描述了段的定义

也可以直接通过 *readelf -d  vmlinux* 获取内核链接后的文件 查看布局情况

.. image:: ./images/mem/24.png
 :width: 800px
 
 一阶段:镜像1:l映射
^^^^^^^^^^^^^^^^^^^^^^

当uboot 加载完成内核，并且跳转到内核起始位置的时候，此时MMU处于未打开的状态，因此此时CPU在执行内核代码是直接访问的物理内存;
这段代码执行期间，严格意义上来说不能够访问类似于全局变量、函数等会涉及到 虚拟内存地址的代码

内存初始化会分几个阶段，第一阶段，使能mmu，为了MMU使能后能够正常工作，需要先把 内核的镜像代码建立 VA 到PA的映射

.. image:: ./images/mem/25.png
 :width: 800px

为什么是线性映射？因为PC在刚开启MMU的时候 PC的地址依然是物理内存地址 因此需要先建立1:1的映射


线性映射的页表准备
^^^^^^^^^^^^^^^^^^^^

映射表位置及大小: arch/arm64/kernel/vmlinux.lds.S 


.. code-block:: c
    :linenos:
	
	init_idmap_pg_dir = .;
    . += INIT_IDMAP_DIR_SIZE;
    init_idmap_pg_end = .;
	
	// 下面代码用于计算 虚拟内存需要多少的内存
	
	#define EARLY_ENTRIES(vstart, vend, shift, add) \
        ((((vend) - 1) >> (shift)) - ((vstart) >> (shift)) + 1 + add)

	#define EARLY_PGDS(vstart, vend, add) (EARLY_ENTRIES(vstart, vend, PGDIR_SHIFT, add))

	#define EARLY_PAGES(vstart, vend, add) ( 1                      /* PGDIR page */                                \
                        + EARLY_PGDS((vstart), (vend), add)     /* each PGDIR needs a next level page table */  \
                        + EARLY_PUDS((vstart), (vend), add)     /* each PUD needs a next level page table */    \
                        + EARLY_PMDS((vstart), (vend), add))    /* each PMD needs a next level page table */
						
	#define INIT_DIR_SIZE (PAGE_SIZE * EARLY_PAGES(KIMAGE_VADDR, _end, EARLY_KASLR))
     
	/* the initial ID map may need two extra pages if it needs to be extended */
	#if VA_BITS < 48
	#define INIT_IDMAP_DIR_SIZE     ((INIT_IDMAP_DIR_PAGES + 2) * PAGE_SIZE)
	#else   
	#define INIT_IDMAP_DIR_SIZE     (INIT_IDMAP_DIR_PAGES * PAGE_SIZE)
	#endif  
	#define INIT_IDMAP_DIR_PAGES    EARLY_PAGES(KIMAGE_VADDR, _end + MAX_FDT_SIZE + SWAPPER_BLOCK_SIZE, 1)
	
下图演示了 EARLY_PAGES 的计算公式 

.. image:: ./images/mem/26.png
 :width: 800px
 
如果映射1M，需要考虑0的情况，因此代码中都做了+1处理 


线性映射建立
^^^^^^^^^^^^^^

内核关键代码: arch/arm64/kernel/head.S是内核一开始启动的代码


此外，宏的实现也考虑了一些额外的级别和特定的位移量（extra_shift），以根据不同的条件填充页表。

.. code-block:: c
    :linenos:
          
	__HEAD                                                                   
	/*                                                                       
	* DO NOT MODIFY. Image header expected by Linux boot-loaders.           
	*/                                                                      
	efi_signature_nop                       // special NOP to identity as PE/COFF executable
	b       primary_entry                   // 跳转到内核入口
	.quad   0                               // Image load offset from start of RAM, little-endian
	le64sym _kernel_size_le                 // Effective size of kernel image, little-endian
	le64sym _kernel_flags_le                // Informative flags, little-endian
	.quad   0                               // reserved                      
	.quad   0                               // reserved                      
	.quad   0                               // reserved                      
	.ascii  ARM64_IMAGE_MAGIC               // Magic number                  
	.long   .Lpe_header_offset              // Offset to the PE header.      
	
	
	SYM_CODE_START(primary_entry)
     bl      preserve_boot_args
     bl      init_kernel_el                  // w0=cpu_boot_mode
     mov     x20, x0
     bl      create_idmap                    //建立内核代码内存映射 



.. code-block:: c
    :linenos:
          
    adrp    x0, init_idmap_pg_dir  // x0 = init_idmap_pg_dir 物理内存基址                                         
    adrp    x3, _text             // x3 =  内核镜像的起始地址的物理内存基址                                       
    adrp    x6, _end + MAX_FDT_SIZE + SWAPPER_BLOCK_SIZE  // x6 = 内核镜像结束的物理内存地址                
    mov     x7, SWAPPER_RX_MMUFLAGS  

    map_memory x0, x1, x3, x6, x7, x3, IDMAP_PGD_ORDER, x10, x11, x12, x13, x14, EXTRA_SHIFT


关键函数说明: .macro map_memory, tbl, rtbl, vstart, vend, flags, phys, order, istart, iend, tmp, count, sv, extra_shift
参数列表：
- tbl：页表的位置。
- rtbl：第一个级别的页表项应使用的虚拟地址。
- vstart：映射范围的起始虚拟地址。
- vend：映射范围的结束虚拟地址（实际映射的范围是vstart到vend - 1）。
- flags：用于映射最后级别页表项的标志。
- phys：与vstart对应的物理地址，假定物理内存是连续的。
- order：一个值，表示页表的级别，即#imm（立即数）的2的对数，它表示PGD表中的条目数。
- istart, iend, tmp, count, sv, extra_shift：这些是临时寄存器和标志，用于在宏内部进行计算和存储中间值。

map_memory 给定的参数映射虚拟地址到物理地址，计算页表级别，并填充页表的不同级别。根据宏的调用情况，它可能涉及多个级别的页表。

MMU开启
^^^^^^^

映射建立完成后就要准备开启MMU，代码依然位于 head.S 

.. code-block:: c
    :linenos:
	
	SYM_FUNC_START_LOCAL(__primary_switch)
        adrp    x1, reserved_pg_dir
        adrp    x2, init_idmap_pg_dir  //加载tTBR 基址为 init_idmap_pg_dir
        bl      __enable_mmu
	#ifdef CONFIG_RELOCATABLE
        adrp    x23, KERNEL_START
        and     x23, x23, MIN_KIMG_ALIGN - 1
	#ifdef CONFIG_RANDOMIZE_BASE
        mov     x0, x22
        adrp    x1, init_pg_end
        mov     sp, x1
        mov     x29, xzr
        bl      __pi_kaslr_early_init
        and     x24, x0, #SZ_2M - 1             // capture memstart offset seed
        bic     x0, x0, #SZ_2M - 1
        orr     x23, x23, x0                    // record kernel offset
	#endif
	#endif
        bl      clear_page_tables
        bl      create_kernel_mapping   

非线性二次映射 
^^^^^^^^^^^^^^^

下图演示了 二次映射的主要工作

.. image:: ./images/mem/27.png
 :width: 800px


.. code-block:: c
    :linenos:
	
	
	SYM_FUNC_START_LOCAL(create_kernel_mapping)
			adrp    x0, init_pg_dir
			mov_q   x5, KIMAGE_VADDR                // compile time __va(_text)
	#ifdef CONFIG_RELOCATABLE
			add     x5, x5, x23                     // add KASLR displacement
	#endif  
			adrp    x6, _end                        // runtime __pa(_end)
			adrp    x3, _text                       // runtime __pa(_text)
			sub     x6, x6, x3                      // _end - _text
			add     x6, x6, x5                      // runtime __va(_end)
			mov     x7, SWAPPER_RW_MMUFLAGS
			
			map_memory x0, x1, x5, x6, x7, x3, (VA_BITS - PGDIR_SHIFT), x10, x11, x12, x13, x14

上述动作 完成了第二阶段的映射  紧接着又通过绝对跳转 跳转到了 __primary_switched 

.. code-block:: c
    :linenos:
	
	ldr     x8, =__primary_switched 
    adrp    x0, KERNEL_START                // __pa(KERNEL_START)           
    br      x8             
	SYM_FUNC_END(__primary_switch) 

    bl      finalise_el2                    // Prefer VHE if possible
    ldp     x29, x30, [sp], #16
    bl      start_kernel                    // 正式进入内核
    ASM_BUG()
 
	





页表
-----

实验
-----

实验1
^^^^^^^^
在linux代码中找到关于页表的配置项 


在linux代码中找到关于VA的配置项 



用户态内存管理
===============


实验
======


NVME虚拟机环境
----------------






