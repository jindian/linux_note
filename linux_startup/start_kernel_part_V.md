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

