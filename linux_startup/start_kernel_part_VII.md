# start kernel part VII

## I/O APIC mapping initialize

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

## probe number of gsi

  ACPI uses a cookie system to “name” in-terrupts known as Global System Interrupts. Each interrupt controller input pin is assigned a GSI using a fairly simple scheme. For the 8259A case, the GSIs map directly to ISA IRQs. Thus, IRQ 0 is GSI 0, etc. The APIC case is slightly more complicated, but still simple. Each I/O APIC is assigned a base GSI by the BIOS. Each input pin on the I/O APIC is mapped to a GSI number by adding the pin number (zero-based) to the base GSI. Thus, if an I/O APIC has a base GSI of N, pin 0 on that I/O APIC has a GSI of N, pin 1 has a GSI of N + 1, etc. The I/O APIC with a base GSI of 0 maps the ISA IRQs onto its first 16 input pins. Thus, the ISA IRQs are ef-fectively always mapped 1:1 onto GSIs. More details about GSIs can be found in Section 5.2.11 of the ACPI 2.0c spec.
  
  `probe_nr_irqs_gsi` involve `acpi_probe_gsi` to get number of gsi and update `nr_irqs_gsi` if the result is bigger than it.

## mark e820 reserved area as busy

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

## mark pages don't correcspond to e820 RAM areas as nosave

  `e820_mark_nosave_regions` requires the e820 map to be sorted and without any overlapping entries and assumes the first area to be RAM.
  
  `e820_mark_nosave_regions` involves `register_nosave_region` to add memory region to nosave list.

## reserve resources

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
  

## search biggest gap in e820 memory space and pass the result to PCI to assign MIMO resources

  `e820_setup_gap` involves `e820_search_gap` to search grap from address `0x10000000` size `0x400000`.

  The result of searching gap as follow:

```

(gdb) p /x *gapstart
$21 = 0x8000000
(gdb) p /x *gapsize
$22 = 0xf7fc0000
```
  
  Finally saves the start address of gap to `pci_mem_start`

## save the init thermal LVT value

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

## store untouched and touched command line

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

## _set nr_cu_ids_

# Links
  * [82093AA I/O ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER (IOAPIC)](http://download.intel.com/design/chipsets/datashts/29056601.pdf)