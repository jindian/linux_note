# start_kernel part V

## trim RAM not covered by MTRRs

  Some buggy BIOSes don't setup the MTRRs properly for systems with certain memory configurations.  This routine checks that the highest MTRR matches the end of memory, to make sure the MTRRs having a write back type cover all of the memory the kernel is intending to use.  If not, it'll trim any memory off the end by adjusting end_pfn, removing it from the kernel's allocation pools, warning the user with an obnoxious message.
  
```mtrr_trim_uncached_memory

mtrr_trim_uncached_memory (end_pfn=32766)
    at arch/x86/kernel/cpu/mtrr/cleanup.c:993
993	{
(gdb) n
1004		if (!is_cpu(INTEL) || disable_mtrr_trim)
(gdb) p disable_mtrr_trim
$18 = 0
(gdb) n
1035			return 0;
```

## determine low and high memory ranges

  All of RAM fits into lowmem - but if user wants highmem artificially via the highmem=x boot parameter then create it

```find_low_pfn_range

find_low_pfn_range () at arch/x86/mm/init_32.c:699
699		if (max_pfn <= MAXMEM_PFN)
(gdb) p /x max_pfn
$19 = 0x7ffe
(gdb) n
696	{
(gdb) 
699		if (max_pfn <= MAXMEM_PFN)
(gdb) 
700			lowmem_pfn_init();
(gdb) s
lowmem_pfn_init () at arch/x86/mm/init_32.c:625
625		max_low_pfn = max_pfn;
(gdb) p max_pfn
$20 = 32766
(gdb) n
627		if (highmem_pages == -1)
(gdb) p highmem_pages
$21 = 4294967295
(gdb) p /x highmem_pages
$22 = 0xffffffff
(gdb) n
625		max_low_pfn = max_pfn;
(gdb) 
627		if (highmem_pages == -1)
(gdb) 
628			highmem_pages = 0;
(gdb) 
630		if (highmem_pages >= max_pfn) {
(gdb) 
635		if (highmem_pages) {
(gdb) 
647	}
(gdb) 
find_low_pfn_range () at arch/x86/mm/init_32.c:703
703	}
```

## setup bios corruption check

  Find memory aread need to be scanned and update the found address and its size to a dedicated array.

