# start kernel part VIII

## parse early command line parameter

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

## parse kernel boot arguments

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

## allocate memory for pid hash

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

  `pidhash_init` is simple, just allocates memory and initialize pid hash table. Let's step into `alloc_large_system_hash` to check how memory allocated in kernel initialization.

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

```

# Links

  * []()