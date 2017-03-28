# start kernel part VII

## _I/O APIC mapping initialize_

  The IOAPIC provides multi-processor interrupt management and incorporates both static and dynamic symmetric interrupt distribution across all processors. In systems with multiple I/O subsystems, each subsystem can have its own set of interrupts.

  `ioapic_init_mappings` setup resource for every I/O APIC with routine `ioapic_setup_resources`

```ioapic_setup_resources

4169		ioapic_res = ioapic_setup_resources(nr_ioapics);
(gdb) s
ioapic_setup_resources (nr_ioapics=1) at arch/x86/kernel/apic/io_apic.c:4140
4140		if (nr_ioapics <= 0)
(gdb) n
4146		mem = alloc_bootmem(n);
(gdb) 
4144		n *= nr_ioapics;
(gdb) 
4146		mem = alloc_bootmem(n);
(gdb) 
4149		mem += sizeof(struct resource) * nr_ioapics;
(gdb) 
4151		for (i = 0; i < nr_ioapics; i++) {
(gdb) 
4152			res[i].name = mem;
(gdb) 
4153			res[i].flags = IORESOURCE_MEM | IORESOURCE_BUSY;
(gdb) 
4154			sprintf(mem,  "IOAPIC %u", i);
(gdb) 
4155			mem += IOAPIC_RESOURCE_NAME_SIZE;
(gdb) 
4151		for (i = 0; i < nr_ioapics; i++) {
(gdb) 
4158		ioapic_resources = res;
```

  After completed resource setup, mapping response routine for each resource.
  
```ioapic_init_mappings

ioapic_init_mappings () at arch/x86/kernel/apic/io_apic.c:4170
4170		for (i = 0; i < nr_ioapics; i++) {
(gdb)
4171			if (smp_found_config) {
(gdb) 
4172				ioapic_phys = mp_ioapics[i].apicaddr;
(gdb) 
4174				if (!ioapic_phys) {
(gdb) 
4192			set_fixmap_nocache(idx, ioapic_phys);
(gdb) 
4193			apic_printk(APIC_VERBOSE,
(gdb) 
4198			ioapic_res->start = ioapic_phys;
(gdb) 
4199			ioapic_res->end = ioapic_phys + (4 * 1024) - 1;
(gdb) 
4200			ioapic_res++;
(gdb) 
4202	}
```

## _probe number of gsi_

  ACPI uses a cookie system to “name” in-terrupts known as Global System Interrupts. Each interrupt controller input pin is assigned a GSI using a fairly simple scheme. For the 8259A case, the GSIs map directly to ISA IRQs. Thus, IRQ 0 is GSI 0, etc. The APIC case is slightly more complicated, but still simple. Each I/O APIC is assigned a base GSI by the BIOS. Each input pin on the I/O APIC is mapped to a GSI number by adding the pin number (zero-based) to the base GSI. Thus, if an I/O APIC has a base GSI of N, pin 0 on that I/O APIC has a GSI of N, pin 1 has a GSI of N + 1, etc. The I/O APIC with a base GSI of 0 maps the ISA IRQs onto its first 16 input pins. Thus, the ISA IRQs are ef-fectively always mapped 1:1 onto GSIs. More details about GSIs can be found in Section 5.2.11 of the ACPI 2.0c spec.
  
  `probe_nr_irqs_gsi` involve `acpi_probe_gsi` to get number of gsi and update `nr_irqs_gsi` if the result is bigger than it.

## _mark e820 reserved area as busy_

  `e820_reserve_resources` allocate memory for `e820_res` and initialize every memory resource, insert resource to resource tree if the memory are isn't reserved or memory below 1M.
  
  Add saved memory region to firmware memory map.

```e820_reserve_resources

Breakpoint 2, e820_reserve_resources () at arch/x86/kernel/e820.c:1335
1335	{
(gdb) n
1340		res = alloc_bootmem(sizeof(struct resource) * e820.nr_map);
(gdb) 
1335	{
(gdb) 
1340		res = alloc_bootmem(sizeof(struct resource) * e820.nr_map);
(gdb) 
1342		for (i = 0; i < e820.nr_map; i++) {
(gdb) 
1340		res = alloc_bootmem(sizeof(struct resource) * e820.nr_map);
(gdb) 
1341		e820_res = res;
(gdb) 
1342		for (i = 0; i < e820.nr_map; i++) {
(gdb) 
1343			end = e820.map[i].addr + e820.map[i].size - 1;
(gdb) 
1348			res->name = e820_type_to_string(e820.map[i].type);
(gdb) 
1349			res->start = e820.map[i].addr;
(gdb) 
1350			res->end = end;
(gdb) 
1352			res->flags = IORESOURCE_MEM;
(gdb) 
1359			if (e820.map[i].type != E820_RESERVED || res->start < (1ULL<<20)) {
(gdb) 
1360				res->flags |= IORESOURCE_BUSY;
(gdb) 
1361				insert_resource(&iomem_resource, res);
(gdb) 
1363			res++;
(gdb) 
1342		for (i = 0; i < e820.nr_map; i++) {
(gdb) break if i==7
Breakpoint 3 at 0xc17006bc: file arch/x86/kernel/e820.c, line 1342.
(gdb) c
Continuing.

Breakpoint 3, e820_reserve_resources () at arch/x86/kernel/e820.c:1342
1342		for (i = 0; i < e820.nr_map; i++) {
(gdb) n
1366		for (i = 0; i < e820_saved.nr_map; i++) {
(gdb) p e820_saved.nr_map 
$1 = 6
(gdb) p e820.nr_map 
$2 = 8
(gdb) n
1368			firmware_map_add_early(entry->addr,
(gdb) s
e820_type_to_string (e820_type=1) at arch/x86/kernel/e820.c:1320
1320		switch (e820_type) {
(gdb) n
1322		case E820_RAM:	return "System RAM";
(gdb) 
e820_reserve_resources () at arch/x86/kernel/e820.c:1369
1368			firmware_map_add_early(entry->addr,
(gdb) s
firmware_map_add_early (start=0, end=654335, type=0xc15cfb9b "System RAM")
    at drivers/firmware/memmap.c:164
164	{
(gdb) n
167		entry = alloc_bootmem(sizeof(struct firmware_map_entry));
(gdb) 
168		if (WARN_ON(!entry))
(gdb) 
171		return firmware_map_add_entry(start, end, type, entry);
(gdb) s
firmware_map_add_entry (entry=0xc2126aa0, type=0xc15cfb9b "System RAM", 
    end=654335, start=0) at drivers/firmware/memmap.c:112
112		BUG_ON(start > end);
(gdb) n
171		return firmware_map_add_entry(start, end, type, entry);
(gdb) s
172	}
(gdb) 
e820_reserve_resources () at arch/x86/kernel/e820.c:1366
1366		for (i = 0; i < e820_saved.nr_map; i++) {
```

## _mark pages don't correcspond to e820 RAM areas as nosave_

  `e820_mark_nosave_regions` requires the e820 map to be sorted and without any overlapping entries and assumes the first area to be RAM.
  
  `e820_mark_nosave_regions` involves `register_nosave_region` to add memory region to nosave list.

## _reserve resources_

  32bit specific setup functions are initialized in `i386_default_early_setup`.
  
  `i386_reserve_resources` is the reserve resource function in `x86_init`. In `i386_reserve_resources` it reserves video 
ram resouce with function `request_resource`.

```request_resource

/**
 * request_resource - request and reserve an I/O or memory resource
 * @root: root resource descriptor
 * @new: resource descriptor desired by caller
 *
 * Returns 0 for success, negative error code on error.
 */
int request_resource(struct resource *root, struct resource *new)
{
	struct resource *conflict;

	write_lock(&resource_lock);
	conflict = __request_resource(root, new);
	write_unlock(&resource_lock);
	return conflict ? -EBUSY : 0;
}
```
  
  `request_resource` involves `__request_resource`, the entire debug information as follow:

```__request_resource

203		conflict = __request_resource(root, new);
(gdb) s
__request_resource (root=root@entry=0xc16997c0 <iomem_resource>, 
    new=new@entry=0xc16944e0 <video_ram_resource>) at kernel/resource.c:147
147		resource_size_t end = new->end;
(gdb) p new
$1 = (struct resource *) 0xc16944e0 <video_ram_resource>
(gdb) p *new
$2 = {start = 655360, end = 786431, name = 0xc15cf9eb "Video RAM area", 
  flags = 2147484160, parent = 0x0, sibling = 0x0, child = 0x0}
(gdb) p /x *new
$3 = {start = 0xa0000, end = 0xbffff, name = 0xc15cf9eb, flags = 0x80000200, 
  parent = 0x0, sibling = 0x0, child = 0x0}
(gdb) p root
$4 = (struct resource *) 0xc16997c0 <iomem_resource>
(gdb) p *root
$5 = {start = 0, end = 18446744073709551615, name = 0xc15d46e7 "PCI mem", 
  flags = 512, parent = 0x0, sibling = 0x0, child = 0xc2126980}
(gdb) p /x *root
$6 = {start = 0x0, end = 0xffffffffffffffff, name = 0xc15d46e7, flags = 0x200, 
  parent = 0x0, sibling = 0x0, child = 0xc2126980}
(gdb) n
146		resource_size_t start = new->start;
(gdb) 
147		resource_size_t end = new->end;
(gdb) 
150		if (end < start)
(gdb) 
152		if (start < root->start)
(gdb) 
154		if (end > root->end)
(gdb) 
156		p = &root->child;
(gdb) 
158			tmp = *p;
(gdb) 
159			if (!tmp || tmp->start > end) {
(gdb) p tmp
$7 = (struct resource *) 0xc2126980
(gdb) n
165			p = &tmp->sibling;
(gdb) 
166			if (tmp->end < start)
(gdb) 
158			tmp = *p;
(gdb) 
159			if (!tmp || tmp->start > end) {
(gdb) p tmp
$8 = (struct resource *) 0xc21269a4
(gdb) n
165			p = &tmp->sibling;
(gdb) 
166			if (tmp->end < start)
(gdb) 
158			tmp = *p;
(gdb) 
159			if (!tmp || tmp->start > end) {
(gdb) 
165			p = &tmp->sibling;
(gdb) 
166			if (tmp->end < start)
(gdb) 
158			tmp = *p;
(gdb) 
159			if (!tmp || tmp->start > end) {
(gdb) 
165			p = &tmp->sibling;
(gdb) 
166			if (tmp->end < start)
(gdb) 
158			tmp = *p;
(gdb) 
159			if (!tmp || tmp->start > end) {
(gdb) 
160				new->sibling = tmp;
(gdb) 
161				*p = new;
(gdb) 
162				new->parent = root;
(gdb) 
163				return NULL;
(gdb) p tmp
$9 = (struct resource *) 0xc1694900 <video_rom_resource>
(gdb) p *tmp
$10 = {start = 786432, end = 822783, name = 0xc15cfa8e "Video ROM", 
  flags = 2147492352, parent = 0xc16997c0 <iomem_resource>, 
  sibling = 0xc1694940 <adapter_rom_resources>, child = 0x0}
```

  After reserving video ram resource completed, continue resource standard I/O resources with function `reserve_standard_io_resources`.
  

## _search biggest gap in e820 memory space and pass the result to PCI to assign MIMO resources_

  `e820_setup_gap` involves `e820_search_gap` to search grap from address `0x10000000` size `0x400000`.

  The result of searching gap as follow:

```

(gdb) p /x *gapstart
$21 = 0x8000000
(gdb) p /x *gapsize
$22 = 0xf7fc0000
```
  
  Finally saves the start address of gap to `pci_mem_start`

## _save the init thermal LVT value_

  `mcheck_intel_therm_init` checks if cpu has feature ACPI and ACC, saves LVT value if cpu has above features.

```mcheck_intel_therm_init

1058		mcheck_intel_therm_init();
(gdb) s
mcheck_intel_therm_init () at arch/x86/kernel/cpu/mcheck/therm_throt.c:260
260	{
(gdb) n
266		if (cpu_has(&boot_cpu_data, X86_FEATURE_ACPI) &&
(gdb) 
269	}
```

  ## `setup_arch` routine ended here, let's continue start_kernel.

## _store untouched and touched command line_

```setup_command_line

/*
 * We need to store the untouched command line for future reference.
 * We also need to store the touched command line since the parameter
 * parsing is performed in place, and we should allow a component to
 * store reference of name/value for future reference.
 */
static void __init setup_command_line(char *command_line)
{
	saved_command_line = alloc_bootmem(strlen (boot_command_line)+1);
	static_command_line = alloc_bootmem(strlen (command_line)+1);
	strcpy (saved_command_line, boot_command_line);
	strcpy (static_command_line, command_line);
}
```

  `setup_command_line` allocates memory for boot command line and static command line and stores command line to allocated 
memory.

## _set number of cpu id_

```setup_nr_cpu_ids

/* An arch may set nr_cpu_ids earlier if needed, so this would be redundant */
static void __init setup_nr_cpu_ids(void)
{
	nr_cpu_ids = find_last_bit(cpumask_bits(cpu_possible_mask),NR_CPUS) + 1;
}
```

  `setup_nr_cpu_ids` gets number of cpus from cpu mask.

## _setup per cpu variables_

  Per cpu variable is a feature in linux kernel, every cpu has its own variable which stored in data section belong to it.
  
  To understand per cpu varaibe let's check declaration/defination for per cpu variable macro:
  
  per cpu variable declared/defined in `include/linux/percpu_defs.h:78`

```per_cpu_variables

/*
 * Variant on the per-CPU variable declaration/definition theme used for
 * ordinary per-CPU variables.
 */
#define DECLARE_PER_CPU(type, name)					\
	DECLARE_PER_CPU_SECTION(type, name, "")

#define DEFINE_PER_CPU(type, name)					\
	DEFINE_PER_CPU_SECTION(type, name, "")
```
  
  `DEFINE_PER_CPU_SECTION(type, name, "")` declared/defined in `include/linux/percpu_defs.h:67`

```per_cpu_seciton

/*
 * Normal declaration and definition macros.
 */
#define DECLARE_PER_CPU_SECTION(type, name, sec)			\
	extern __PCPU_ATTRS(sec) __typeof__(type) per_cpu__##name

#define DEFINE_PER_CPU_SECTION(type, name, sec)				\
	__PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES			\
	__typeof__(type) per_cpu__##name
```

  `__PCPU_ATTRS(sec)` declared/defined in `include/linux/percpu_defs.h:10`

```per_cpu_attribute

/*
 * Base implementations of per-CPU variable declarations and definitions, where
 * the section in which the variable is to be placed is provided by the
 * 'sec' argument.  This may be used to affect the parameters governing the
 * variable's storage.
 *
 * NOTE!  The sections for the DECLARE and for the DEFINE must match, lest
 * linkage errors occur due the compiler generating the wrong code to access
 * that section.
 */
#define __PCPU_ATTRS(sec)						\
	__attribute__((section(PER_CPU_BASE_SECTION sec)))		\
	PER_CPU_ATTRIBUTES
```

  `PER_CPU_BASE_SECTION` declared/defined in `include/asm-generic/percpu.h:73`, here we suppose smp configured.
  
```PER_CPU_BASE_SECTION

#ifdef CONFIG_SMP
#define PER_CPU_BASE_SECTION ".data.percpu"
#else
#define PER_CPU_BASE_SECTION ".data"
#endif
```
  
  `PER_CPU_ATTRIBUTES` declared/defined in `include/asm-generic/percpu.h:99`

```PER_CPU_ATTRIBUTES

#ifndef PER_CPU_ATTRIBUTES
#define PER_CPU_ATTRIBUTES
#endif
```

  If we define a variable with type `int` and name `x`, after macro expanded, the actual result as follow:
  
  `extern __attribute__((section(".data.percpu" ""))) int per_cpu_x`

  `setup_per_cpu_areas` is used to setup per cpu variable areas, default function for allocating per cpu area is `pcpu_embed_first_chunk` if `percpu_alloc` doesn't exist in startup command line. `pcpu_embed_first_chunk` allocates memory for all cpus' allocation information and setup first chunk.
  
  After `pcpu_embed_first_chunk` completed, setup offset and segment for every cpu, initialize per cpu variable `x86_cpu_to_apicid`, `irq_stack_ptr` ...


  Here is the debug information after `pcpu_build_alloc_info` completd in function `pcpu_embed_first_chunk` when I configure cpu number 2 (qemu-system-i386 -s -S -smp 2 disk.img).

```pcpu_build_alloc_info

(gdb) p ai
$11 = (struct pcpu_alloc_info *) 0xc2126ce0
(gdb) p *ai
$12 = {static_size = 1353816, reserved_size = 0, dyn_size = 22440, 
  unit_size = 2097152, atom_size = 2097152, alloc_size = 2097152, 
  __ai_size = 4096, nr_groups = 1, groups = 0xc2126d00}
(gdb) p ai->groups 
$13 = 0xc2126d00
(gdb) p *ai->groups 
$14 = {nr_units = 2, base_offset = 0, cpu_map = 0xc2126d0c}
```

  After allocation information allocated successfully, further allocates memory for group and cpus, copy content in `.data.percpu` to per cpu segments.
    
```

(gdb) n
pcpu_embed_first_chunk (reserved_size=reserved_size@entry=0, 
    dyn_size=dyn_size@entry=20480, atom_size=2097152, 
    cpu_distance_fn=cpu_distance_fn@entry=0xc1705aa1 <pcpu_cpu_distance>, 
    alloc_fn=alloc_fn@entry=0xc1705aca <pcpu_fc_alloc>, 
    free_fn=free_fn@entry=0xc1705ab5 <pcpu_fc_free>) at mm/percpu.c:1876
1876		if (IS_ERR(ai))
(gdb) 
1879		size_sum = ai->static_size + ai->reserved_size + ai->dyn_size;
(gdb) 
1880		areas_size = PFN_ALIGN(ai->nr_groups * sizeof(void *));
(gdb) 
1882		areas = alloc_bootmem_nopanic(areas_size);
(gdb) 
1883		if (!areas) {
(gdb) 
1891			unsigned int cpu = NR_CPUS;
(gdb) 
1894			for (i = 0; i < gi->nr_units && cpu == NR_CPUS; i++)
(gdb) 
1895				cpu = gi->cpu_map[i];
(gdb) 
1899			ptr = alloc_fn(cpu, gi->nr_units * ai->unit_size, atom_size);
(gdb) 
1900			if (!ptr) {
(gdb) 
1904			areas[group] = ptr;
(gdb) 
1906			base = min(ptr, base);
(gdb) 
1908			for (i = 0; i < gi->nr_units; i++, ptr += ai->unit_size) {
(gdb) 
1909				if (gi->cpu_map[i] == NR_CPUS) {
(gdb) 
1915				memcpy(ptr, __per_cpu_load, ai->static_size);
(gdb) 
```

  There will be N `.data.percpu` after `pcpu_embed_first_chunk` completes, here N is the number of cpus in system.

## _prepare for boot cpu_

  Do some preparation for boot cpu.

```native_smp_prepare_boot_cpu

void __init native_smp_prepare_boot_cpu(void)
{
	int me = smp_processor_id();
	switch_to_new_gdt(me);
	/* already set me in cpu_online_mask in boot_cpu_init() */
	cpumask_set_cpu(me, cpu_callout_mask);
	per_cpu(cpu_state, me) = CPU_ONLINE;
}
```
  
  Inside `native_smp_prepare_boot_cpu`, get id of boot cpu with `smp_processor_id`, with `switch_to_new_gdt` loading gdt and data sections, set cpu mask to indicate boot cpu is online.

## _create node and zone for memory management_

  Linux has a structure describing memory which is used to keep account of memory banks, pages and the flags that affect VM behaviour.

  The first principal concept prevalent in the VM is Non-Uniform Memory Access (NUMA). With large scale machines, memory may be arranged into banks that incur a different cost to access depending on the “distance” from the processor. For example, there might be a bank of memory assigned to each CPU or a bank of memory very suitable for DMA near device cards.

  Each bank is called a node and the concept is represented under Linux by a struct pglist_data even if the architecture is UMA. This struct is always referenced to by it's typedef pg_data_t. Every node in the system is kept on a NULL terminated list called pgdat_list and each node is linked to the next with the field pg_data_t→node_next. For UMA architectures like PC desktops, only one static pg_data_t structure called contig_page_data is used.

  Each node is divided up into a number of blocks called zones which represent ranges within memory. Zones should not be confused with zone based allocators as they are unrelated. A zone is described by a struct zone_struct, typedeffed to zone_t and each one is of type ZONE_DMA, ZONE_NORMAL or ZONE_HIGHMEM. Each zone type suitable a different type of usage. ZONE_DMA is memory in the lower physical memory ranges which certain ISA devices require. Memory within ZONE_NORMAL is directly mapped by the kernel into the upper region of the linear address space. ZONE_HIGHMEM is the remaining available memory in the system and is not directly mapped by the kernel.

  `build_all_zonelists` checks system state, if in system booting, build all zonelist with `__build_all_zonelists` which involves `build_zonelists_node` to add page presented zone to zone list.
  
  `build_all_zonelists` updates varaible of total number of pages `vm_total_pages` after accounting free RAM allocatable within all zones.

## _register callback function for CPU up/down_

  `page_alloc_init` involves `hotcpu_notifier` to register callback function `page_alloc_cpu_notify` to know CPUs going up/down when HOTPLUG supported.
  
  `hotcpu_notifier` is a MACRO, its defination can be found in include/linux/cpu.h:113

```

#define hotcpu_notifier(fn, pri)	cpu_notifier(fn, pri)
```

  `cpu_notifier` can be found in include/linux/cpu.h:52

```

#define cpu_notifier(fn, pri) {					\
	static struct notifier_block fn##_nb __cpuinitdata =	\
		{ .notifier_call = fn, .priority = pri };	\
	register_cpu_notifier(&fn##_nb);			\
}
```

  In linux to define a new lock, it uses macro `DEFINE_MUTEX`, from comment of mutex struct, the principle of it is clear. Mutex is used to protect critical region.

```

include/linux/mutex.h:97

#define __MUTEX_INITIALIZER(lockname) \
		{ .count = ATOMIC_INIT(1) \
		, .wait_lock = __SPIN_LOCK_UNLOCKED(lockname.wait_lock) \
		, .wait_list = LIST_HEAD_INIT(lockname.wait_list) \
		__DEBUG_MUTEX_INITIALIZER(lockname) \
		__DEP_MAP_MUTEX_INITIALIZER(lockname) }

#define DEFINE_MUTEX(mutexname) \
	struct mutex mutexname = __MUTEX_INITIALIZER(mutexname)

include/linux/mutex.h:48

/*
 * Simple, straightforward mutexes with strict semantics:
 *
 * - only one task can hold the mutex at a time
 * - only the owner can unlock the mutex
 * - multiple unlocks are not permitted
 * - recursive locking is not permitted
 * - a mutex object must be initialized via the API
 * - a mutex object must not be initialized via memset or copying
 * - task may not exit with mutex held
 * - memory areas where held locks reside must not be freed
 * - held mutexes must not be reinitialized
 * - mutexes may not be used in hardware or software interrupt
 *   contexts such as tasklets and timers
 *
 * These semantics are fully enforced when DEBUG_MUTEXES is
 * enabled. Furthermore, besides enforcing the above rules, the mutex
 * debugging code also implements a number of additional features
 * that make lock debugging easier and faster:
 *
 * - uses symbolic names of mutexes, whenever they are printed in debug output
 * - point-of-acquire tracking, symbolic lookup of function names
 * - list of all locks held in the system, printout of them
 * - owner tracking
 * - detects self-recursing locks and prints out all relevant info
 * - detects multi-task circular deadlocks and prints out all affected
 *   locks and tasks (and only those tasks)
 */
struct mutex {
	/* 1: unlocked, 0: locked, negative: locked, possible waiters */
	atomic_t		count;
	spinlock_t		wait_lock;
	struct list_head	wait_list;
	
	...
};
```

  Inside `register_cpu_notifier`, it locks mutex `cpu_add_remove_lock` to protect data `cpu_chain`. `cpu_maps_update_begin` involves `mutex_lock` with reference of `cpu_add_remove_lock` as its input parameter.

```cpu_maps_update_begin

void cpu_maps_update_begin(void)
{
	mutex_lock(&cpu_add_remove_lock);
}
```

  Here is the declaration of function `mutex_lock`, it changes count from 1 to a 0 value with `__mutex_fastpath_lock` and set current task as the owner of the lock with `mutex_set_owner`.

```mutex_lock

/***
 * mutex_lock - acquire the mutex
 * @lock: the mutex to be acquired
 *
 * Lock the mutex exclusively for this task. If the mutex is not
 * available right now, it will sleep until it can get it.
 *
 * The mutex must later on be released by the same task that
 * acquired it. Recursive locking is not allowed. The task
 * may not exit without first unlocking the mutex. Also, kernel
 * memory where the mutex resides mutex must not be freed with
 * the mutex still locked. The mutex must first be initialized
 * (or statically defined) before it can be locked. memset()-ing
 * the mutex to 0 is not allowed.
 *
 * ( The CONFIG_DEBUG_MUTEXES .config option turns on debugging
 *   checks that will enforce the restrictions and will also do
 *   deadlock debugging. )
 *
 * This function is similar to (but not equivalent to) down().
 */
void __sched mutex_lock(struct mutex *lock)
{
	might_sleep();
	/*
	 * The locking fastpath is the 1->0 transition from
	 * 'unlocked' into 'locked' state.
	 */
	__mutex_fastpath_lock(&lock->count, __mutex_lock_slowpath);
	mutex_set_owner(lock);
}
```
  
  After register completed, unlock lock with `cpu_maps_update_done` which involves `mutex_unlock` to release the mutex.

```cpu_maps_update_done

void cpu_maps_update_done(void)
{
	mutex_unlock(&cpu_add_remove_lock);
}
```

  `mutex_unlock` changes count from 0 to 1 with `__mutex_fastpath_unlock`.

```mutex_unlock

/***
 * mutex_unlock - release the mutex
 * @lock: the mutex to be released
 *
 * Unlock a mutex that has been locked by this task previously.
 *
 * This function must not be used in interrupt context. Unlocking
 * of a not locked mutex is not allowed.
 *
 * This function is similar to (but not equivalent to) up().
 */
void __sched mutex_unlock(struct mutex *lock)
{
	/*
	 * The unlocking fastpath is the 0->1 transition from 'locked'
	 * into 'unlocked' state:
	 */
#ifndef CONFIG_DEBUG_MUTEXES
	/*
	 * When debugging is enabled we must not clear the owner before time,
	 * the slow path will always be taken, and that clears the owner field
	 * after verifying that it was indeed current.
	 */
	mutex_clear_owner(lock);
#endif
	__mutex_fastpath_unlock(&lock->count, __mutex_unlock_slowpath);
}
```


# Links
  * [82093AA I/O ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER (IOAPIC)](http://download.intel.com/design/chipsets/datashts/29056601.pdf)
  * [Symmetric multiprocessing](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
  * [Describing Physical Memory](https://www.kernel.org/doc/gorman/html/understand/understand005.html)
  
  
  
  
  
