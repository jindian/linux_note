# start kernel part VIII

## _parse early command line parameter_

```parse_early_param

/* Arch code calls this early on, or if not, just before other parsing. */
void __init parse_early_param(void)
{
	static __initdata int done = 0;
	static __initdata char tmp_cmdline[COMMAND_LINE_SIZE];

	if (done)
		return;

	/* All fall through to do_early_param. */
	strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
	parse_early_options(tmp_cmdline);
	done = 1;
}
```

  This function already involved in `setup_arch`, when checking value `done` we can find it was set as 1.
  
```parse_early_param_done

565		parse_early_param();
(gdb) s
parse_early_param () at init/main.c:473
473		if (done)
(gdb) p done
$13 = 1
```

## _parse kernel boot arguments_

```parse_args

566		parse_args("Booting kernel", static_command_line, __start___param,
(gdb) s
parse_args (name=name@entry=0xc15c5350 "Booting kernel", 
    args=0xc2126ca0 "BOOT_IMAGE=/boot/vmlinuz-2.6.32.69 root=/dev/sda", 
    params=0xc1687b18 <__param_initcall_debug>, num=168, 
    unknown=unknown@entry=0xc16fa21a <unknown_bootoption>)
    at kernel/params.c:136
136	{
```

  Parse kernel boot parameter, same with it did in `parse_early_param` when it involved in `setup_arch`

## _allocate memory for pid hash_

```pidhash_init

/*
 * The pid hash table is scaled according to the amount of memory in the
 * machine.  From a minimum of 16 slots up to 4096 slots at one gigabyte or
 * more.
 */
void __init pidhash_init(void)
{
	int i, pidhash_size;

	pid_hash = alloc_large_system_hash("PID", sizeof(*pid_hash), 0, 18,
					   HASH_EARLY | HASH_SMALL,
					   &pidhash_shift, NULL, 4096);
	pidhash_size = 1 << pidhash_shift;

	for (i = 0; i < pidhash_size; i++)
		INIT_HLIST_HEAD(&pid_hash[i]);
}
```

  `pidhash_init` is simple, just allocates memory from boot memory and initialize pid hash table. Let's step into `alloc_large_system_hash` to check how memory allocated in kernel initialization.

```alloc_large_system_hash

506		pid_hash = alloc_large_system_hash("PID", sizeof(*pid_hash), 0, 18,
(gdb) s
alloc_large_system_hash (tablename=tablename@entry=0xc15d6031 "PID", 
    bucketsize=bucketsize@entry=4, numentries=numentries@entry=0, 
    scale=scale@entry=18, flags=flags@entry=3, 
    _hash_shift=_hash_shift@entry=0xc169dc0c <pidhash_shift>, 
    _hash_mask=_hash_mask@entry=0x0, limit=limit@entry=4096)
    at mm/page_alloc.c:4853
4853		unsigned long long max = limit;
(gdb) 
4858		if (!numentries) {
(gdb) p nr_kernel_pages 
$19 = 32185
(gdb) n
4861			numentries += (1UL << (20 - PAGE_SHIFT)) - 1;
(gdb) 
4863			numentries <<= 20 - PAGE_SHIFT;
(gdb) 
4866			if (scale > PAGE_SHIFT)
(gdb)
4867				numentries >>= (scale - PAGE_SHIFT);
(gdb) 
4872			if (unlikely(flags & HASH_SMALL)) {
(gdb) p numentries 
$20 = 504
(gdb) n
4874				WARN_ON(!(flags & HASH_EARLY));
(gdb) 
4875				if (!(numentries >> *_hash_shift)) {
(gdb) p *_hash_shift 
$21 = 4
(gdb) n
4882		numentries = roundup_pow_of_two(numentries);
(gdb) 
4885		if (max == 0) {
(gdb) p numentries 
$22 = 512
(gdb) n
4890		if (numentries > max)
(gdb) 
4893		log2qty = ilog2(numentries);
(gdb) 
4897			if (flags & HASH_EARLY)
(gdb) 
4896			size = bucketsize << log2qty;
(gdb) 
4897			if (flags & HASH_EARLY)
(gdb) 
4898				table = alloc_bootmem_nopanic(size);
(gdb) p size
$24 = 2048
(gdb) s
__phys_addr (x=x@entry=3238002688) at arch/x86/mm/physaddr.c:51
51		VIRTUAL_BUG_ON(x < PAGE_OFFSET);
(gdb) n
52		VIRTUAL_BUG_ON(__vmalloc_start_set && is_vmalloc_addr((void *) x));
(gdb) 
53		return x - PAGE_OFFSET;
(gdb) 
54	}
(gdb) s
__alloc_bootmem_nopanic (size=size@entry=2048, align=align@entry=32, 
    goal=16777216) at mm/bootmem.c:610
610		return ___alloc_bootmem_nopanic(size, align, goal, 0);
(gdb) s
___alloc_bootmem_nopanic (size=size@entry=2048, align=align@entry=32, 
    goal=16777216, limit=limit@entry=0) at mm/bootmem.c:578
578			if (limit && bdata->node_min_pfn >= PFN_DOWN(limit))
(gdb) n
571		region = alloc_arch_preferred_bootmem(NULL, size, align, goal, limit);
(gdb) s
alloc_arch_preferred_bootmem (size=size@entry=2048, limit=0, goal=16777216, 
    align=32, bdata=0x0) at mm/bootmem.c:541
541	static void * __init alloc_arch_preferred_bootmem(bootmem_data_t *bdata,
(gdb) n
545		if (WARN_ON_ONCE(slab_is_available()))
(gdb) 
559		return NULL;
(gdb) 
560	}
(gdb) 
___alloc_bootmem_nopanic (size=size@entry=2048, align=align@entry=32, 
    goal=16777216, limit=limit@entry=0) at mm/bootmem.c:572
572		if (region)
(gdb) n
575		list_for_each_entry(bdata, &bdata_list, list) {
(gdb) 
(gdb) n
576			if (goal && bdata->node_low_pfn <= PFN_DOWN(goal))
(gdb) 
575		list_for_each_entry(bdata, &bdata_list, list) {
(gdb) 
576			if (goal && bdata->node_low_pfn <= PFN_DOWN(goal))
(gdb) 
578			if (limit && bdata->node_min_pfn >= PFN_DOWN(limit))
(gdb) 
581			region = alloc_bootmem_core(bdata, size, align, goal, limit);
(gdb) p bdata
$25 = (bootmem_data_t *) 0xc173e788 <bootmem_node_data>
(gdb) p *bdata
$26 = {node_min_pfn = 0, node_low_pfn = 32766, node_bootmem_map = 0xc0035000, 
  last_end_off = 37028264, hint_idx = 8487, list = {
    next = 0xc173e780 <bdata_list>, prev = 0xc173e780 <bdata_list>}}
(gdb) s
alloc_bootmem_core (bdata=bdata@entry=0xc173e788 <bootmem_node_data>, 
    size=size@entry=2048, align=align@entry=32, goal=goal@entry=16777216, 
    limit=limit@entry=0) at mm/bootmem.c:437
437	{

    .......
    # find memory area not allocated from node_bootmem_map #

(gdb) 
539	}
(gdb) 
___alloc_bootmem_nopanic (size=size@entry=2048, align=align@entry=32, 
    goal=16777216, limit=limit@entry=0) at mm/bootmem.c:582
582			if (region)
(gdb) p region
$27 = (void *) 0xc2129000
(gdb) n
592	}
(gdb) 
__alloc_bootmem_nopanic (size=size@entry=2048, align=align@entry=32, 
    goal=<optimized out>) at mm/bootmem.c:611
611	}
(gdb) n
alloc_large_system_hash (tablename=tablename@entry=0xc15d6031 "PID", 
    bucketsize=bucketsize@entry=4, numentries=<optimized out>, 
    numentries@entry=0, scale=scale@entry=18, flags=flags@entry=3, 
    _hash_shift=_hash_shift@entry=0xc169dc0c <pidhash_shift>, 
    _hash_mask=_hash_mask@entry=0x0, limit=limit@entry=4096)
    at mm/page_alloc.c:4912
4912		} while (!table && size > PAGE_SIZE && --log2qty);
(gdb) n
4914		if (!table)
(gdb) 
4917		printk(KERN_INFO "%s hash table entries: %d (order: %d, %lu bytes)\n",
(gdb) 
4923		if (_hash_shift)
(gdb) 
4924			*_hash_shift = log2qty;
(gdb) 
4925		if (_hash_mask)
(gdb) 
4929	}
```

  Because memory management is not ready, `alloc_arch_preferred_bootmem` returned NULL, later allocated memory from boot memory.

## _early initialization of virtual file system_

  The Virtual File System (also known as the Virtual Filesystem Switch) is the software layer in the kernel that provides the filesystem interface to userspace programs. It also provides an abstraction within the kernel which allows different filesystem implementations to coexist.
  
  Directory Entry Cache (dcache) is meant to be a view into your entire filespace. As most computers cannot fit all dentries in the RAM at the same time, some bits of the cache are missing. In order to resolve your pathname into a dentry, the VFS may have to resort to creating dentries along the way, and then loading the inode. This is done by looking up the inode.

  An individual dentry usually has a pointer to an inode. Inodes are filesystem objects such as regular files, directories, FIFOs and other beasts.  They live either on the disc (for block device filesystems) or in the memory (for pseudo filesystems). Inodes that live on the disc are copied into the memory when required and changes to the inode are written back to disc. A single inode can be pointed to by multiple dentries (hard links, for example, do this).

  Opening a file requires another operation: allocation of a file structure (this is the kernel-side implementation of file descriptors). The freshly allocated file structure is initialized with a pointer to the dentry and a set of file operation member functions. These are taken from the inode data.

  `vfs_caches_init_early` involves `dcache_init_early` and `inode_init_early` to allocate memory for hash tables `dentry_hashtable` and `inode_hashtable`

## _sort the kernel's built-in exception table_

  The section of exception table defined as follow, which could be found in include/asm-generic/vmlinux.ls.h

```defination_of_exception_table

/*
 * Exception table
 */
#define EXCEPTION_TABLE(align)						\
	. = ALIGN(align);						\
	__ex_table : AT(ADDR(__ex_table) - LOAD_OFFSET) {		\
		VMLINUX_SYMBOL(__start___ex_table) = .;			\
		*(__ex_table)						\
		VMLINUX_SYMBOL(__stop___ex_table) = .;			\
	}
```

  The concept of exception table, take reference in [Kernel level exception handling in Linux](https://www.kernel.org/doc/Documentation/x86/exception-tables.txt)

```sort_main_extable

575		sort_main_extable();
(gdb) s
sort_main_extable () at kernel/extable.c:41
41		sort_extable(__start___ex_table, __stop___ex_table);
(gdb) s
sort_extable (start=0xc148fb90, finish=0xc1490ad0) at lib/extable.c:38
38	{
(gdb) n
39		sort(start, finish - start, sizeof(struct exception_table_entry),
(gdb) s
sort (base=0xc148fb90, num=488, size=size@entry=8, 
    cmp_func=cmp_func@entry=0xc1237870 <cmp_ex>, swap_func=swap_func@entry=0x0)
    at lib/sort.c:50
50	{
```

  `cmp_ex` as the input parameter of `sort` routine, in which compares index of the input `exception_table_entry`

```cmp_ex

/*
 * The exception table needs to be sorted so that the binary
 * search that we use to find entries in it works properly.
 * This is used both for the kernel exception table and for
 * the exception tables of modules that get loaded.
 */
static int cmp_ex(const void *a, const void *b)
{
	const struct exception_table_entry *x = a, *y = b;

	/* avoid overflow */
	if (x->insn > y->insn)
		return 1;
	if (x->insn < y->insn)
		return -1;
	return 0;
}
```

## _initialize interrupt_

  In `trap_init`, if EISA configured, unmap memory region of EISA, here `0x0FFFD9` is the start address of EISA area.

  The Interrupt Descriptor Table (IDT) is a data structure used by the x86 architecture to implement an interrupt vector table. The IDT is used by the processor to determine the correct response to interrupts and exceptions.
  
  The Interrupt Descriptor Table (IDT) is specific to the IA-32 architecture. It is the Protected mode counterpart to the Real Mode Interrupt Vector Table (IVT) telling where the Interrupt Service Routines (ISR) are located (one per interrupt vector). It is similar to the Global Descriptor Table in structure.

  The IDT entries are called gates. It can contain Interrupt Gates, Task Gates and Trap Gates.

  `trap_init` sets 20 reserved interrupt with `set_intr_gate`, here is the function declaration of `set_intr_gate`, with interrupt number and address of interrupt handler function as its input parameters. `set_intr_gate` involves `_set_gate` which initializes a gate_desc struction and write to specified interrupt in `idt_table`.

```set_intr_gate

/*
 * This needs to use 'idt_table' rather than 'idt', and
 * thus use the _nonmapped_ version of the IDT, as the
 * Pentium F0 0F bugfix can have resulted in the mapped
 * IDT being write-protected.
 */
static inline void set_intr_gate(unsigned int n, void *addr)
{
	BUG_ON((unsigned)n > 0xFF);
	_set_gate(n, GATE_INTERRUPT, addr, 0, 0, __KERNEL_CS);
}

arch/x86/include/asm/desc.h:320

static inline void _set_gate(int gate, unsigned type, void *addr,
			     unsigned dpl, unsigned ist, unsigned seg)
{
	gate_desc s;
	pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
	/*
	 * does not need to be atomic because it is only done once at
	 * setup time
	 */
	write_idt_entry(idt_table, gate, &s);
}
```

  `trap_init` reserves all the builtin and syscall vector, checks if [control register CR4](https://en.wikipedia.org/wiki/Control_register#CR4) has fxsr and xmm, set specified bit of the control register, sets system call interrput.
  
  `trap_init` initialize cpu with `cpu_init`, in `cpu_init` initialize resources for current cpu.

```cpu_init

1011		cpu_init();
(gdb) s
cpu_init () at arch/x86/kernel/cpu/common.c:1201
1201	{
(gdb) n
1202		int cpu = smp_processor_id();
(gdb) 
1203		struct task_struct *curr = current;
(gdb) 
1204		struct tss_struct *t = &per_cpu(init_tss, cpu);
(gdb) 
1207		if (cpumask_test_and_set_cpu(cpu, cpu_initialized_mask)) {
(gdb) p cpu
$6 = 0
(gdb) p cpu_initialized_mask 
$7 = {{bits = {0}}}
(gdb) n
1208			printk(KERN_WARNING "CPU#%d already initialized!\n", cpu);
(gdb) 
1213		printk(KERN_INFO "Initializing CPU#%d\n", cpu);
(gdb) 
1215		if (cpu_has_vme || cpu_has_tsc || cpu_has_de)
(gdb) 
1216			clear_in_cr4(X86_CR4_VME|X86_CR4_PVI|X86_CR4_TSD|X86_CR4_DE);
(gdb) 
1218		load_idt(&idt_descr);
(gdb) s
native_load_idt (dtr=<optimized out>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/desc.h:224
224		asm volatile("lidt %0"::"m" (*dtr));
(gdb) n
cpu_init () at arch/x86/kernel/cpu/common.c:1219
1219		switch_to_new_gdt(cpu);c
(gdb) s
switch_to_new_gdt (cpu=cpu@entry=0) at arch/x86/kernel/cpu/common.c:348
348		gdt_descr.address = (long)get_cpu_gdt_table(cpu);
(gdb) n
349		gdt_descr.size = GDT_SIZE - 1;
(gdb) 
350		load_gdt(&gdt_descr);
(gdb) n
353		load_percpu_segment(cpu);
(gdb) s
load_percpu_segment (cpu=0) at arch/x86/kernel/cpu/common.c:332
332		loadsegment(fs, __KERNEL_PERCPU);
(gdb) n
337		load_stack_canary_segment();
(gdb) s
load_stack_canary_segment ()
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/stackprotector.h:101
101		asm("mov %0, %%gs" : : "r" (__KERNEL_STACK_CANARY) : "memory");
(gdb) n
switch_to_new_gdt (cpu=cpu@entry=0) at arch/x86/kernel/cpu/common.c:354
354	}
(gdb) 
cpu_init () at arch/x86/kernel/cpu/common.c:1224
1224		atomic_inc(&init_mm.mm_count);
(gdb) p init_mm.mm_count 
$8 = {counter = 1}
(gdb) p init_mm.mm_count 
$9 = {counter = 2}
(gdb) n
1225		curr->active_mm = &init_mm;
(gdb) 
1226		BUG_ON(curr->mm);
(gdb) 
1227		enter_lazy_tlb(&init_mm, curr);
(gdb) 
1229		load_sp0(t, thread);
(gdb) s
load_sp0 (thread=0xc1692794 <init_task+852>, tss=<optimized out>)
    at arch/x86/kernel/cpu/common.c:1229
1229		load_sp0(t, thread);
(gdb) s
native_load_sp0 (tss=<optimized out>, tss=<optimized out>, 
    thread=<optimized out>, thread=<optimized out>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/processor.h:557
557		tss->x86_tss.sp0 = thread->sp0;
(gdb) n
560		if (unlikely(tss->x86_tss.ss1 != thread->sysenter_cs)) {
(gdb) 
cpu_init () at arch/x86/kernel/cpu/common.c:1230
1230		set_tss_desc(cpu, t);
cpu_init () at arch/x86/kernel/cpu/common.c:1231
1231		load_TR_desc();
(gdb) s
native_load_tr_desc ()
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/desc.h:214
214		asm volatile("ltr %w0"::"q" (GDT_ENTRY_TSS*8));
(gdb) n
1231		load_TR_desc();
(gdb) 
1232		load_LDT(&init_mm.context);
(gdb) s
load_LDT_nolock (pc=<optimized out>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/desc.h:287
287		set_ldt(pc->ldt, pc->size);
(gdb) n
cpu_init () at arch/x86/kernel/cpu/common.c:1234
1234		t->x86_tss.io_bitmap_base = offsetof(struct tss_struct, io_bitmap);
(gdb) 
1238		__set_tss_desc(cpu, GDT_ENTRY_DOUBLEFAULT_TSS, &doublefault_tss);
(gdb) 
1241		clear_all_debug_regs();
1246		if (cpu_has_xsave)
(gdb) 
1249			current_thread_info()->status = 0;
(gdb) 
1250		clear_used_math();
(gdb) 
1251		mxcsr_feature_mask_init();
(gdb) s
mxcsr_feature_mask_init () at arch/x86/kernel/i387.c:49
49		clts();
(gdb) n
47		unsigned long mask = 0;
(gdb) 
50		if (cpu_has_fxsr) {
(gdb) 
51			memset(&fx_scratch, 0, sizeof(struct i387_fxsave_struct));
(gdb) 
52			asm volatile("fxsave %0" : : "m" (fx_scratch));
(gdb) 
53			mask = fx_scratch.mxcsr_mask;
(gdb) 
55				mask = 0x0000ffbf;
(gdb) 
57		mxcsr_feature_mask &= mask;
(gdb) 
58		stts();
(gdb) 
59	}
(gdb) n
cpu_init () at arch/x86/kernel/cpu/common.c:1256
1256		if (smp_processor_id() == boot_cpu_id)
(gdb) 
1257			init_thread_xstate();
(gdb) s
init_thread_xstate () at arch/x86/kernel/i387.c:68
68		if (cpu_has_xsave) {
(gdb) n
73		if (cpu_has_fxsr)
(gdb) 
74			xstate_size = sizeof(struct i387_fxsave_struct);
(gdb) 
79	}
(gdb) p xstate_size 
$10 = 512
(gdb) n
cpu_init () at arch/x86/kernel/cpu/common.c:1259
1259		xsave_init();
(gdb) s
xsave_init () at arch/x86/kernel/xsave.c:291
291		if (!cpu_has_xsave)
(gdb) n
301	}
cpu_init () at arch/x86/kernel/cpu/common.c:1260
1260	}
```

## _initialize memory management_c

  Declaration of routine `mm_init` could be found in file init/main.c:507

```mm_init

/*
 * Set up kernel memory allocators
 */
static void __init mm_init(void)
{
	/*
	 * page_cgroup requires countinous pages as memmap
	 * and it's bigger than MAX_ORDER unless SPARSEMEM.
	 */
	page_cgroup_init_flatmem();
	mem_init();
	kmem_cache_init();
	pgtable_cache_init();
	vmalloc_init();
}
```

  Because `CONFIG_CGROUP_MEM_RES_CTLR` doesn't enabled, declaration of routine `page_cgroup_init_flatmem` is empty.
  
  Next involved routine in `mm_init` is `mem_init`:
  
  In computing, an input–output memory management unit (IOMMU) is a memory management unit (MMU) that connects a direct-memory-access–capable (DMA-capable) I/O bus to the main memory. Like a traditional MMU, which translates CPU-visible virtual addresses to physical addresses, the IOMMU maps device-visible virtual addresses (also called device addresses or I/O addresses in this context) to physical addresses. Some units also provide memory protection from faulty or malicious devices.
  
  * `mem_init` involves `pci_iommu_alloc` to allocates IOMMU resource, actually because all configurations using IOMMU are disabled, soon returned from `pci_iommu_alloc` after stepped into it.
  * Put all low memory onto freelist with `free_all_bootmem`
  * save level 4 page data and zap initialize low memory mapping

```totalram_pages

(gdb) p totalram_pages 
$1 = 27200
```
  * Count reserved RAM pages

```reservedpages

881		set_highmem_pages_init();
(gdb) p reservedpages 
$4 = 5464
```
  * Loop all memory zones add active high memory regions to total RAM page with `set_highmem_pages_init`

```totalhigh_pages
130		totalram_pages += totalhigh_pages;
(gdb) p totalhigh_pages 
$8 = 0
(gdb) n
131	}
(gdb) p totalram_pages 
$10 = 27200
(gdb) 
```

  `kmem_cache_init` is used to basic setup of slabs.
  
  [Slab allocation](https://en.wikipedia.org/wiki/Slab_allocation) is a memory management mechanism intended for the efficient memory allocation of kernel objects. It eliminates fragmentation caused by allocations and deallocations. The technique is used to retain allocated memory that contains a data object of a certain type for reuse upon subsequent allocations of objects of the same type. It is analogous to an object pool, but only applies to memory, not other resources.
  
  * `init_alloc_cpu` initializes per cpu array kmem cache.
  * `create_kmalloc_cache` creates several type of caches for kernel memory allocation, let's deep into `create_kmalloc_cache`

```create_kmalloc_cache

mm/slub.c:2693

static struct kmem_cache *create_kmalloc_cache(struct kmem_cache *s,
		const char *name, int size, gfp_t gfp_flags)
{
	unsigned int flags = 0;

	if (gfp_flags & SLUB_DMA)
		flags = SLAB_CACHE_DMA;

	/*
	 * This function is called with IRQs disabled during early-boot on
	 * single CPU so there's no need to take slub_lock here.
	 */
	if (!kmem_cache_open(s, gfp_flags, name, size, ARCH_KMALLOC_MINALIGN,
								flags, NULL))
		goto panic;

	list_add(&s->list, &slab_caches);

	if (sysfs_slab_add(s))
		goto panic;
	return s;

panic:
	panic("Creation of kmalloc slab %s size=%d failed.\n", name, size);
}

mm/slub.c:2464

static int kmem_cache_open(struct kmem_cache *s, gfp_t gfpflags,
		const char *name, size_t size,
		size_t align, unsigned long flags,
		void (*ctor)(void *))
{
	memset(s, 0, kmem_size);
	s->name = name;
	s->ctor = ctor;
	s->objsize = size;
	s->align = align;
	s->flags = kmem_cache_flags(size, flags, name, ctor);

	if (!calculate_sizes(s, -1))
		goto error;
	if (disable_higher_order_debug) {
		/*
		 * Disable debugging flags that store metadata if the min slab
		 * order increased.
		 */
		if (get_order(s->size) > get_order(s->objsize)) {
			s->flags &= ~DEBUG_METADATA_FLAGS;
			s->offset = 0;
			if (!calculate_sizes(s, -1))
				goto error;
		}
	}

	/*
	 * The larger the object size is, the more pages we want on the partial
	 * list to avoid pounding the page allocator excessively.
	 */
	set_min_partial(s, ilog2(s->size));
	s->refcount = 1;
#ifdef CONFIG_NUMA
	s->remote_node_defrag_ratio = 1000;
#endif
	if (!init_kmem_cache_nodes(s, gfpflags & ~SLUB_DMA))
		goto error;

	if (alloc_kmem_cache_cpus(s, gfpflags & ~SLUB_DMA))
		return 1;
	free_kmem_cache_nodes(s);
error:
	if (flags & SLAB_PANIC)
		panic("Cannot create slab %s size=%lu realsize=%u "
			"order=%u offset=%u flags=%lx\n",
			s->name, (unsigned long)size, s->size, oo_order(s->oo),
			s->offset, flags);
	return 0;
}

```
  
  `create_kmalloc_cache` checks GFP bitmap if memory allocated in zone DMA, after that it involves `kmem_cache_open` to allocate needed kmem_cache. `kmem_cache_open` firstly initializes kmem_cache structure, then determines the order and the distribution of data within a slab object with `calculate_sizes`, sets min partial with `set_min_partial` because the larger the object size is, the more pages we want on the partial list to avoid pounding the page allocator excessively, initializes nodes of the kmem_cache with `init_kmem_cache_nodes` which initialzes partial list of node in kmem_cache, initializes kmem_cache_cpu data of per cpu data with `alloc_kmem_cache_cpus`. After `kmem_cache_open` executing completed, add new kmem_cache to `slab_cache` list. Finally `create_kmalloc_cache` add sysfs slab if `slab_state` is `SYSFS`.
  
  * After needed kmem_cache created, updates `slab_state` to `UP`
  * Updates kmalloc name
  * Registers notifier to allocate new kmem_cache for new up cpu
  * Updates kmem_size

 Go back to `mm_init`, it involves `vmalloc_init` to initialize vmalloc, understanding the difference between kmalloc and vmalloc [here](http://naveengopala-embeddedlinux.blogspot.fi/2012/01/linux-kernel-programmingmemory.html), `vmalloc_init` gets `vmap_block_queue` from per cpu data area and initializes it's spin lock and free list, loops `vmlist` and allocates vmap_area for all entres, inserts allocated vmap_area.
 
 
## _initialize scheduler_

  In computing, scheduling is the method by which work specified by some means is assigned to resources that complete the work. The work may be virtual computation elements such as threads, processes or data flows, which are in turn scheduled onto hardware resources such as processors, network links or expansion cards.

  A scheduler is what carries out the scheduling activity. Schedulers are often implemented so they keep all computer resources busy (as in load balancing), allow multiple users to share system resources effectively, or to achieve a target quality of service. Scheduling is fundamental to computation itself, and an intrinsic part of the execution model of a computer system; the concept of scheduling makes it possible to have computer multitasking with a single central processing unit (CPU).

  A scheduler may aim at one of many goals, for example, maximizing throughput (the total amount of work completed per time unit), minimizing response time (time from work becoming enabled until the first point it begins execution on resources), or minimizing latency (the time between work becoming enabled and its subsequent completion),[1] maximizing fairness (equal CPU time to each process, or more generally appropriate times according to the priority and workload of each process). In practice, these goals often conflict (e.g. throughput versus latency), thus a scheduler will implement a suitable compromise. Preference is given to any one of the concerns mentioned above, depending upon the user's needs and objectives.

  `sched_init` allocates memory from boot memory block if `CONFIG_FAIR_GROUP_SCHED`, `CONFIG_RT_GROUP_SCHED`, `CONFIG_CPUMASK_OFFSTACK` configured and initializes specified area of global variable `init_task_group` and per cpu variable separately. In our environment, above configurations are not enabled, `alloc_size` is zero.

```alloc_size

Breakpoint 2, sched_init () at kernel/sched.c:9658
9658	{
(gdb) n
9703		init_defrootdomain();
(gdb) p alloc_size 
$1 = 0
```

  `sched_init` initializes default root domain including specific cpu mask and cpu priority for each cpu set. Defination of root domain as follow.

```

/*
 * We add the notion of a root-domain which will be used to define per-domain
 * variables. Each exclusive cpuset essentially defines an island domain by
 * fully partitioning the member cpus from any other cpuset. Whenever a new
 * exclusive cpuset is created, we also create and attach a new root-domain
 * object.
 *
 */
struct root_domain {
	atomic_t refcount;
	cpumask_var_t span;
	cpumask_var_t online;

	/*
	 * The "RT overload" flag: it gets set if a CPU has more than
	 * one runnable RT task.
	 */
	cpumask_var_t rto_mask;
	atomic_t rto_count;
#ifdef CONFIG_SMP
	struct cpupri cpupri;
#endif
};

/*
 * By default the system creates a single root-domain with all cpus as
 * members (mimicking the global state we have today).
 */
static struct root_domain def_root_domain;
```
  
  `sched_init` initializes default real time bandwidth, about `rt_bandwidth`, it can be found [here](https://lwn.net/Articles/296419/).
  rt_bandwith consists of a pair of values: a CPU time accounting period, and the amount of CPU that the group is allowed to use - at realtime priority - during that period. Once a SCHED_FIFO task causes a group to exceed its rt_bandwidth, it will be pushed out of the processor whether it wants to go or not.

  Context of `init_rt_bandwidth` when initializing default real time bandwidth.

```init_rt_bandwidth

9706		init_rt_bandwidth(&def_rt_bandwidth,
(gdb) s
init_rt_bandwidth (rt_b=0xc18ec340 <def_rt_bandwidth>, runtime=950000000, 
    period=1000000000) at kernel/sched.c:181
181		spin_lock_init(&rt_b->rt_runtime_lock);
```

  `sched_init` intializes per cpu run queue structure.
  
  In modern computers many processes run at once. Active processes are placed in an array called a run queue, or runqueue. The run queue may contain priority values for each process, which will be used by the scheduler to determine which process to run next. To ensure each program has a fair share of resources, each one is run for some time period (quantum) before it is paused and placed back into the run queue. When a program is stopped to let another run, the program with the highest priority in the run queue is then allowed to execute.

  Processes are also removed from the run queue when they ask to sleep, are waiting on a resource to become available, or have been terminated.

  In the Linux operating system (prior to kernel 2.6.23), each CPU in the system is given a run queue, which maintains both an active and expired array of processes. Each array contains 140 (one for each priority level) pointers to doubly linked lists, which in turn reference all processes with the given priority. The scheduler selects the next process from the active array with highest priority. When a process' quantum expires, it is placed into the expired array with some priority. When the active array contains no more processes, the scheduler swaps the active and expired arrays, hence the name O(1) scheduler.
  
  `sched_init` sets load weight for `init_task`, the defination of `init_task` could be found in arch/x86/kernel/init_task.c:31

```init_task

/*
 * Initial task structure.
 *
 * All other task structs will be allocated on slabs in fork.c
 */
struct task_struct init_task = INIT_TASK(init_task);
```

  `set_load_weight` checks if init task has real time schedule policy, if yes, initialzes load weight base on real time policy and returns.

```policy_init_task

(gdb) where
#0  set_load_weight (p=0xc1692440 <init_task>) at kernel/sched.c:1921
#1  0xc170dada in sched_init () at kernel/sched.c:9797
#2  0xc16fa624 in start_kernel () at init/main.c:583
#3  0xc16fa0a1 in i386_start_kernel () at arch/x86/kernel/head32.c:62
#4  0x00000000 in ?? ()
(gdb) s
task_has_rt_policy (p=0xc1692440 <init_task>) at kernel/sched.c:1920
1920	{
(gdb) n
131		return rt_policy(p->policy);
(gdb) s
rt_policy (policy=0) at kernel/sched.c:124
124		if (unlikely(policy == SCHED_FIFO || policy == SCHED_RR))
(gdb) 
```
  
  `set_load_weight` then checks if init task has idle schedule policy, if yes, initialize load weight base on idle policy and returns.
  
  finally initialzes load weight base on priority of init task. The static priority of init task and the result of load weight:

```static_priority_of_init_task

(gdb) p p->static_prio 
$7 = 120
(gdb) p p->se.load 
$8 = {weight = 1024, inv_weight = 4194304}
```

  The mapping of priority to weight and weight multiplications could be found in kernel/sched.c:1345

```priority_to_weight

/*
 * Nice levels are multiplicative, with a gentle 10% change for every
 * nice level changed. I.e. when a CPU-bound task goes from nice 0 to
 * nice 1, it will get ~10% less CPU time than another CPU-bound task
 * that remained on nice 0.
 *
 * The "10% effect" is relative and cumulative: from _any_ nice level,
 * if you go up 1 level, it's -10% CPU usage, if you go down 1 level
 * it's +10% CPU usage. (to achieve that we use a multiplier of 1.25.
 * If a task goes up by ~10% and another task goes down by ~10% then
 * the relative distance between them is ~25%.)
 */
static const int prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};

/*
 * Inverse (2^32/x) values of the prio_to_weight[] array, precalculated.
 *
 * In cases where the weight does not change often, we can use the
 * precalculated inverse to speed up arithmetics by turning divisions
 * into multiplications:
 */
static const u32 prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

  After set load weight for init task, `sched_init` initializes preempt notifiers list of init task.
  
  `sched_init` registers for `SCHED_SOFTIRQ` with  function `run_rebalance_domains` which is triggered when needed from the scheduler tick. In CONFIG_NO_HZ case, the idle load balance owner will do the rebalancing for all the cpus for whom scheduler ticks are stopped.
  
  `sched_init` initializes priority and node list of PI waiters blocked on a rt_mutex held by init task.
  
  The kernel represents a process's address space with a data structure called the memory descriptor. This structure contains all the information related to the process address space. The memory descriptor is represented by struct mm_struct and defined in include/linux/mm_type.h
  
  The mm_users field of mm_struct is the number of processes using this address space. For example, if two threads share this address space, mm_users is equal to two. The mm_count field is the primary reference count for the mm_struct. All mm_users equate to one increment of mm_count. Only when mm_users reaches zero (when both threads exit) is mm_count decremented. When mm_count finally reaches zero, there are no remaining references to this mm_struct and it is freed. Having two counters enables the kernel to differentiate between the main usage counter (mm_count) and the number of processes using the address space (mm_users).
  
  All of the mm_struct structures are strung together in a doubly linked list via the mmlist field. The initial element in the list is the init_mm memory descriptor, which describes the address space of kernel, all kernel threads use kernel space and have the same access permission as kernel.
  
  `sched_init` increases mm_count of `init_mm` and updates per cpu variable `cpu_tlbstate.state` as `TLBSTATE_LAZY` if current state is `TLBSTATE_OK`.
  
  `sched_init` makes idle thread for cpu 0, the context of `init_idle` is `Breakpoint 3, init_idle (idle=idle@entry=0xc1692440 <init_task>, cpu=0)`.

# Links

  * [A tour of the Linux VFS](http://www.tldp.org/LDP/khg/HyperNews/get/fs/vfstour.html)
  * [Overview of the Linux Virtual File System](https://www.kernel.org/doc/Documentation/filesystems/vfs.txt)
  * [Kernel level exception handling in Linux](https://www.kernel.org/doc/Documentation/x86/exception-tables.txt)
  * [Interrupt descriptor table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
  * [Interrupt Descriptor Table](http://wiki.osdev.org/Interrupt_Descriptor_Table)
  * [EISA bus support](https://www.kernel.org/doc/Documentation/eisa.txt)
  * [CR4](https://en.wikipedia.org/wiki/Control_register#CR4)
  * [Input–output memory management unit](https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit)
  * [Slab allocation](https://en.wikipedia.org/wiki/Slab_allocation)
  * [vmalloc VS kmalloc](http://naveengopala-embeddedlinux.blogspot.fi/2012/01/linux-kernel-programmingmemory.html)
  * [Multiprocessing with the Completely Fair Scheduler](http://homepage.smc.edu/morgan_david/cs40/multimedia/cfs_scheduler_ibm1.pdf)
  * [Linux Scheduler](https://www.cs.columbia.edu/~smb/classes/s06-4118/l13.pdf)
  * [Completely Fair Scheduler](https://en.wikipedia.org/wiki/Completely_Fair_Scheduler)
  * [Scheduling](https://en.wikipedia.org/wiki/Scheduling_(computing))
  * [SCHED_FIFO and realtime throttling](https://lwn.net/Articles/296419/)
  * [Run queue](https://en.wikipedia.org/wiki/Run_queue)
  
  