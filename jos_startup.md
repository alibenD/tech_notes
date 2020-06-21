# Boot过程
计算机上电后由BIOS程序接管，并将boot镜像load至0x7c00并跳转至此执行boot程序。（此处省略如何将镜像文件load到0x7c00处的过程）
在boot中，做了一下事情：

1.  将bootloader读至memory（0x7c00处）, 开始位置为`start`

>  a. 初始化段寄存器（CS、DS、ES：Code、Data、Extra）
>  b. 打开A20地址线，装载GDT并设置CR0（&=0x01），长跳转进入保护模式。
> > * 装载GDT
	* GDT第一项(64bit)：0x 0000 0000 0000 0000
	* GDT第二项(64bit)-code seg： 0x0000 0000 - 0xffff ffff + 权限设置
	* GDT第三项(64bit)-data seg：  0x0000 0000 - 0xffff ffff + 权限设置
> 6222 0321 1000 3722 983
>  c. 初始化段寄存器并设置segment selector（0x08）, PROT_MODE_CODE_SEG:0x08, PROT_MODE_DATA_SEG:0x10 
>  d. 设置`%esp=$start`（bootloader的内容可以覆盖掉了）
> > [CR0-4寄存器介绍](https://blog.csdn.net/qq_37414405/article/details/84487591)

```
控制寄存器（CR0～CR3）用于控制和确定处理器的操作模式以及当前执行任务的特性，如图4-3所示。
CR0中含有控制处理器操作模式和状态的系统控制标志；
CR1保留不用；
CR2含有导致页错误的线性地址；
CR3中含有页目录表物理内存基地址，因此该寄存器也被称为页目录基地址寄存器PDBR（Page-Directory Base addressRegister）。

x86_32的CR0为32bit。X86_64下为64bit，其中低32bit与x86_32的CR0保持一致，高32bit没有定义，作保留使用，除了bit 4其他所有位都是可读可写的。
Protected-Mode Enable (PE) Bit. Bit0. PE=0,表示CPU处于实模式; PE=1表CPU处于保护模式，并使用分段机制。
Paging Enable (PG) Bit. Bit 31. 该位控制分页机制，PG=1，启动分页机制；PG=0,不使用分页机制。
```

2 .  将内核kernel镜像load至内存（镜像文件内存起始地址0x10000, elf格式）
> a. 将kernel的elf文件格式的镜像读入所有segment/section，按照va、pa映射关系将文件对应的section载入到对应的物理地址中去。
>> ```c++
// ELF文件头格式定义
struct Elf {
	uint32_t e_magic;	// must equal ELF_MAGIC
	uint8_t e_elf[12];
	uint16_t e_type;
	uint16_t e_machine;
	uint32_t e_version;
	uint32_t e_entry;
	uint32_t e_phoff;
	uint32_t e_shoff;
	uint32_t e_flags;
	uint16_t e_ehsize;
	uint16_t e_phentsize;
	uint16_t e_phnum;
	uint16_t e_shentsize;
	uint16_t e_shnum;
	uint16_t e_shstrndx;
};
// Program程序头
struct Proghdr {
	uint32_t p_type;
	uint32_t p_offset;
	uint32_t p_va;
	uint32_t p_pa;
	uint32_t p_filesz;
	uint32_t p_memsz;
	uint32_t p_flags;
	uint32_t p_align;
};
// 节头
struct Secthdr {
	uint32_t sh_name;
	uint32_t sh_type;
	uint32_t sh_flags;
	uint32_t sh_addr;
	uint32_t sh_offset;
	uint32_t sh_size;
	uint32_t sh_link;
	uint32_t sh_info;
	uint32_t sh_addralign;
	uint32_t sh_entsize;
};
```
>
> b. 载入kernel完毕后，将跳转至kernel entry(0x1000c)
> > 准备开启分页：
> > 设置页目录地址（entry_pgdir）： 0x0011 8000是页目录数组的首地址；
> > 	映射情况：1024个页目录项（PageDir， 4MB为单位进行管理），每个页目录项管理有1024个页表（PageTable），每个页表管理4kB内存。
> > 386_init时，只映射VA[0,4MB]->PA[0,4MB]以及VA[3840MB, 3844MB]->PA[0, 4MB], i.e. pgdir[0]和pgdir[960]
> 
> 打开分页模式(cr0, cr3)
> > CR0设置：开启分页PG=1以及写保护WP=1
>
> 设置kernel stack
> 长跳转，通过虚拟地址映射访问Kernel（调用i386_init）
>

```cpp
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |      Index     |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \---------- PGNUM(la) ----------/
#define NPDENTRIES	1024		// page directory entries per page directory
#define NPTENTRIES	1024		// page table entries per page table
#define PGSIZE		4096		// bytes mapped by a page
#define PGSHIFT		12		// log2(PGSIZE)
#define PTSIZE		(PGSIZE*NPTENTRIES) // bytes mapped by a page directory entry
#define PTSHIFT		22		// log2(PTSIZE)
#define PTXSHIFT	12		// offset of PTX in a linear address
#define PDXSHIFT	22		// offset of PDX in a linear address
typedef uint32_t pte_t;     // Page Table Entry
typedef uint32_t pde_t;    // Page Directory Entry
pte_t entry_pgtable[NPTENTRIES];

#define PTE_P		0x001	// Present
#define PTE_W		0x002	// Writeable
#define PTE_U		0x004	// User
#define PTE_PWT		0x008	// Write-Through
#define PTE_PCD		0x010	// Cache-Disable
#define PTE_A		0x020	// Accessed
#define PTE_D		0x040	// Dirty
#define PTE_PS		0x080	// Page Size
#define PTE_G		0x100	// Global

__attribute__((__aligned__(PGSIZE)))
pde_t entry_pgdir[NPDENTRIES] = 
{
	// Map VA's [0, 4MB) to PA's [0, 4MB)
	[0]  = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]. = ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};

__attribute__((__aligned__(PGSIZE)))
pte_t entry_pgtable[NPTENTRIES] = {
	0x000000 | PTE_P | PTE_W,
	0x001000 | PTE_P | PTE_W,
	0x002000 | PTE_P | PTE_W,
	0x003000 | PTE_P | PTE_W,
	0x004000 | PTE_P | PTE_W,
	......
	0x3fe000 | PTE_P | PTE_W,
	0x3ff000 | PTE_P | PTE_W,
};
```

3 .  Kernel内核开始自检-i386_init
> 初始化BSS区，通过memset(0, edata, (edata-eend))进行初始化。
> 初始化内存
> > 给Kernel 页目录分配内存（分配4kbytes内存，等价于分配了1k个页目录项），虚拟页表（但是不明白为何这么做）
> > > `kern_pgdir[PDX(UVPT)] = PADDR(kern_pgdir) | PTE_U | PTE_P;`
> > 
> > 创建Page数组，管理所有物理内存。数组的大小由实际内存页数决定。
> > > `  pages = (struct PageInfo*) boot_alloc( sizeof(struct PageInfo) * npages);`
> > 
> > 初始化Page数组，根据已经使用的物理内存对Page中对应的虚拟页进行标记
> > 检查空闲Page链表是否合法。
> > 检查Page Alloc是否正确分配内存页。
> > >  `PageInfo* page_alloc(int alloc_flags)` 根据page_free_list分配有效内存页
> > 检查Page。



* 打开保护模式、关闭中断
* 保护模式中读取kernel文件到内存位置（0x10000）
* 跳转至kernel的entry(0x10000c)
* 设置页目录地址（entry_pgdir）： 0x0011 3000
	* 映射情况：1024个页目录（PageDir， 4MB为单位进行管理），每个页目录有1024个页表项（PageTable），每个页表项管理4kB内存。
	* i386_init时，只映射VA[0,4MB]->PA[0,4MB]以及VA[3840MB, 3844MB]->PA[0, 4MB], i.e. pgdir[0]和pgdir[960]
* 打开分页模式(cr0, cr3)
* 设置kernel stack
* 长跳转，通过虚拟地址映射访问Kernel
* 内存初始化
	*  内核页表初始化： 	
		* 调用boot\_alloc，从end地址开始（4k对齐）分配一个PGSIZE（PGSIZE=4k）。返回的指针作为kern\_pgdir的地址（0xf011 6000）供给内核建立页目录. kern\_pgdir范围 [0xf011 6000, 0xf011 7000)
		* 然后建立kern\_pgdir虚拟地址到物理地址的映射（0xf011 6000 -> 0x0011 6000）， 并将kern\_pgdir[957]设置PTE_U | PTE_P;
	* 内核页表项初始化：
		* 为pages开辟内存，开辟内存的大小为物理内存页(npages = 0x8000), npages * sizeof(struct PageInfo) = 32768 * 8 bytes = 32k bytes，[0xf011 7000, 0xf015 7000);
		* pages[0]保留作为BIOS和RealMode的IDT，大小为4k，一个页的大小。 0xf011 7008为page[1];
		*  pgdir[0]管理pages[0]-pages[1023]一共1024个页，总计内存1024 * 4KB = 4MB内存
			*  &pages[0] = 0xf011 7000, &pages[1024] = 0xf011 9000
			*  管理的物理内存范围 [0x0000 0000, 0x0040 0000]，虚拟内存范围[0xf000 0000, 0xf040 0000]
		* 将0xA0000以下的物理内存页（Base memory），初始化为未使用。
		* 将I/O映射的部分内存页保留（Base memory -> BIOS_end, i.e. [0xA0000, 0x100000]），并设置为已使用, 不能被操作系统分配使用。
		* [0x100000, UPPER]排除已经分配的虚拟页（已经映射的），全都设置为未使用。
	* 
