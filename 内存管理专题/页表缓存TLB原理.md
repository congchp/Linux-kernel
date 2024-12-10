处理器的内存管理单元(Memory Management Uint，MMU)负责把虚拟地址转换成物理地址，为了加快虚拟地址到物理地址的转换速度，避免每次转换都需要查询内存中的页表，处理器厂商在MMU中增加了一个高速缓存TLB(Translation Lookaside Buffer)，TLB直译为转换后背缓冲区，也叫页表缓存；
页表缓存用来缓存最近使用过的页表项，有些处理器使用两级页表缓存：第一级TLB分为指令TLB和数据TLB，取指令和取数据可以并行执行；第二级TLB是统一TLB(Unified TLB)，即指指令和数据共用TLB；
## TLB表项格式
不同处理器架构的TLB表项的格式不同；ARM64处理器的每条TLB表项不仅包含虚拟地址和物理地址，也包含属性：内存类型、缓存策略、访问权限、地址空间标识符(Address Space Identifier，ASID)和虚拟机标识符(Virtual Machine Identifier，VMID)；ASID区分不同进程的页表项，VMID区分不同虚拟机的页表项；
## TLB管理
### flush_tlb相关函数
如果内核修改了可能缓存在TLB里面的页表项，那么内核必须负责使旧的TLB表项失效；内核定义了每种处理器架构必须实现的函数，主要是flush tlb，即冲刷tlb，使tlb失效；如下表所示：
arm64架构实现在`arch/arm64/include/asm/tlbflush.h`

| 函数 | 说明 |
| --- | --- |
| void flush_tlb_all(void); | 使所有tlb表项失效； |
| void flush_tlb_mm(struct mm_struct *mm)； | 使指定用户地址空间的所有tlb表项失效；参数mm是进程的内存描述符； |
| flush_tlb_range(struct vm_area_struct *vma,
unsigned long start, unsigned long end)； | 使指定用户地址空间的某个范围的TLB表项失效；参数vma是虚拟内存区域，start是起始地址，end是结束地址； |
| ......... |  |


arm64架构中关于tlb管理的说明如下：
```c
/*
 *	TLB Management
 *	==============
 *
 *	The TLB specific code is expected to perform whatever tests it needs
 *	to determine if it should invalidate the TLB for each call.  Start
 *	addresses are inclusive and end addresses are exclusive; it is safe to
 *	round these addresses down.
 *
 *	flush_tlb_all()
 *
 *		Invalidate the entire TLB.
 *
 *	flush_tlb_mm(mm)
 *
 *		Invalidate all TLB entries in a particular address space.
 *		- mm	- mm_struct describing address space
 *
 *	flush_tlb_range(mm,start,end)
 *
 *		Invalidate a range of TLB entries in the specified address
 *		space.
 *		- mm	- mm_struct describing address space
 *		- start - start address (may not be aligned)
 *		- end	- end address (exclusive, may not be aligned)
 *
 *	flush_tlb_page(vaddr,vma)
 *
 *		Invalidate the specified page in the specified address range.
 *		- vaddr - virtual address (may not be aligned)
 *		- vma	- vma_struct describing address range
 *
 *	flush_kern_tlb_page(kaddr)
 *
 *		Invalidate the TLB entry for the specified page.  The address
 *		will be in the kernels virtual memory space.  Current uses
 *		only require the D-TLB to be invalidated.
 *		- kaddr - Kernel virtual memory address
 */
```
### arm64 tlb失效指令
当tlb没有命中的时候，arm64处理器的MMU自动遍历内存中的页表，把页表项复制到tlb，不需要软件把页表写道tlb，所有arm64架构没有提供写tlb的指令；

arm64架构提供了一条tlb失效指令：

---

`TLBI <type><level>{IS} {, <Xt>}`

---

1）字段type的常见选项如下：

- ALL：所有表项；
- VMALL：当前虚拟机的阶段1的所有表项，即表项的VMID是当前虚拟机的VMID；虚拟机里面运行的客户操作系统的虚拟地址转换成物理地址分为两个阶段：第1阶段把虚拟地址转换成中间物理地址；第2阶段把中间物理地址转换成物理地址；
- VMALLS12：当前虚拟机的阶段1和阶段2的所有表项；
- ......

2）字段level指定异常级别；

- E1：异常级别1；
- E2：异常级别2；
- E3：异常级别3；

3）字段IS表示内部共享(Inner Shareable)，即多核共享；如果不使用IS字段，表示非共享，只被一个核使用；在SMP系统中，如果指令TLBI不携带字段IS，仅使当前核的TLB表项失效；如果指令TLBI携带字段IS，表示使所有核的TLB表项失效；
4）字段Xt是X0~X31中的任一寄存器；
### arm64 flush tlb函数分析
函数flush_tlb_all，用来使所有核的所有TLB表项失效；
```c
/*
 *	flush_tlb_all()
 *
 *		Invalidate the entire TLB.
*/

static inline void flush_tlb_all(void)
{
	dsb(ishst);
	__tlbi(vmalle1is);
	dsb(ish);
	isb();
}
```
是一些汇编代码，将宏展开后代码如下：
```c
static inline void flush_tlb_all(void)
{
    asm volatile("dsb ishst": : : "memory");
    asm ("tlbi vmalle1is" : :);
    asm volatile("dsb ish" : : : "memory");
    asm volatile("isb" : : : "memory");
}
```
最主要的是`tlbi vmalle1is`，根据前面介绍的arm64架构tlb失效指令：使所有核上匹配当前VMID、阶段1和异常级别1的所有TLB表项失效；

对比一下ARM64架构的函数local_flush_tlb_all，作用是使当前核的所有TLB表项失效，代码如下：
```c
static inline void local_flush_tlb_all(void)
{
	dsb(nshst);
	__tlbi(vmalle1);
	dsb(nsh);
	isb();
}
```
`vmalle1`，根据前面介绍的arm64架构tlb失效指令：使当前核上匹配当前VMID、阶段1和异常级别1的所有TLB表项失效；
## 地址空间标识符(Address Space Identifier，ASID)
为了减少进程切换时清空TLB的需要，ARM64处理器的TLB使用非全局(not global，nG)位区分内核和进程的页表项(nG位为0表示内核的页表项)，使用ASID区分不同进程的页表项；
ARM64处理器的ASID的长度时可以设置的，支持8bit或者16bit；
SMP系统中，ARM64架构要求ASID在处理器的所有核上是唯一的；
假设ASID长度是8bit，ASID只可以有256个值，其中0是保留值；有效的ASID范围是1~255；进程的数量可能超过255，那么两个进程的ASID值可能冲突，这个问题怎么解决呢？内核引入了ASID版本号，方法如下：

1. 每个进程有一个64位的软件ASID，低8位存放硬件ASID，高56位存放ASID版本号；
2. 64位全局变量`asid_generation`的高56位保存全局ASID版本号；
3. 当进程被调度时，比较进程的ASID版本号和全局ASID版本号；如果版本号相同，那么直接使用上次分配的硬件ASID，否则需要给进程重新分配硬件ASID；

1）如果存在空闲的硬件ASID，那么选择一个分配给进程；
2）如果没有空闲的硬件ASID，那么把全局的ASID版本号加1，重新从1开始分配硬件ASID，即硬件ASID从255回绕到1；因为刚分配得硬件ASID可能和某个进程的硬件ASID相同，只是ASID版本号不同；TLB中的ASID是硬件ASID，8bit；TLB中可能包含了这个进程的页表项，所以必须把所有处理器的TLB清空；
**引入ASID版本号的好处是**：避免每次进程切换都需要清空TLB，只需要在硬件ASID回绕的时候把处理器的TLB清空；
## 虚拟机标识符(Virtual Machine Identifier，VMID)
每个虚拟机有独立的ASID空间，TLB使用VMID区分不同虚拟机的转换表项，可以避免每次虚拟机切换都要清空TLB，只需要在VMID回绕时把处理器的TLB清空；
