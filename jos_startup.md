# Boot过程
计算机上电后由BIOS程序接管，并将boot镜像load至0x7c00并跳转至此执行boot程序。
在boot中，做了一下事情：

* 装载GDT
	* GDT第一项(64bit)：0x 0000 0000 0000 0000
	* GDT第二项(64bit)-code seg： 0x0000 0000 - 0xffff ffff +权限设置
	* GDT第三项(64bit)-data seg：  0x0000 0000 - 0xffff ffff +权限设置
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
	* 调用boot_alloc，从end地址开始（4k对齐）分配一个PGSIZE（PGSIZE=4k）。返回的指针作为kern_pgdir的地址（0xf011 5000）供给内核建立页目录.
	* 然后建立kern_pgdir虚拟地址到物理地址的映射（0xf011 5000 -> 0x0011 5000）， 并将kern_pgdir[957]设置PTE_U | PTE_P;
	* 为pages开辟内存，开辟内存的大小为物理内存页npages * sizeof(struct PageInfo)
	* 
