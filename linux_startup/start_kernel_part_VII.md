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




# Links
  * [82093AA I/O ADVANCED PROGRAMMABLE INTERRUPT CONTROLLER (IOAPIC)](http://download.intel.com/design/chipsets/datashts/29056601.pdf)