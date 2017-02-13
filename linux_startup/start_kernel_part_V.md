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

## early reserve memory for brk erea

  Reserve memory for brk area.
  
  About brk here are some information from the description of heap
  
  The heap area commonly begins at the end of the .bss and .data segments and grows to larger addresses from there. The heap area is managed by malloc, calloc, realloc, and free, which may use the [brk and sbrk](https://en.wikipedia.org/wiki/Sbrk) system calls to adjust its size (note that the use of brk/sbrk and a single "heap area" is not required to fulfill the contract of malloc/calloc/realloc/free; they may also be implemented using mmap/munmap to reserve/unreserve potentially non-contiguous regions of virtual memory into the process' virtual address space). The heap area is shared by all threads, shared libraries, and dynamically loaded modules in a process.

## initialize memory map

  Setup the direct mapping of the physical memory at PAGE_OFFSET.
  This runs before bootmem is initialized and gets pages directly from the physical memory. To access them they are temporarily mapped.
  
  set [NX-bit](https://de.wikipedia.org/wiki/NX-Bit) if nx enabled, active pse and pge in cpu mask if dedicated feature available, finally initialize page table.
  
```max_pfn_mapped

(gdb) p /x max_pfn_mapped
$45 = 0x7ffe
```

## reserve memory for initrd

  [initrd (initial ramdisk)](https://en.wikipedia.org/wiki/Initrd) is a scheme for loading a temporary root file system into memory, which may be used as part of the Linux startup process. initrd and initramfs refer to two different methods of achieving this. Both are commonly used to make preparations before the real root file system can be mounted.

```reserve_initrd

ramdisk_image = 0x77b5000
ramdisk_size = 0x290498
ramdisk_end = 0x7a45498
end_of_lowmem = 0x7ffe000
```

  initrd finnaly locates at low memory
  
```initrd_location

initrd_start = 0xc77b5000
initrd_end = 0xc7a45498
```

## override I/O delay port
  
```io_delay_init

void __init io_delay_init(void)
{
        if (!io_delay_override)
                dmi_check_system(io_delay_0xed_port_dmi_table);
}
```
  Inside io_delay_init it first check the value of `io_delay_override` which is set when early parameter `io_delay` exists in linux start command line. If the option of `io_delay` doesn't set in command line, do further actions with `dmi_check_system` with `io_delay_0xed_port_dmi_table` as input parameter. `io_delay_0xed_port_dmi_table` is a list of quirk table for systems that misbehave (lock up, etc.) if port 0x80 is used. Scan the list with current system parameters, if matches call the configured function of the list item.
  

  
  
  
  
  
  

# Links:
  * [brk/sbrk](https://en.wikipedia.org/wiki/Sbrk)
  * [data segment](https://en.wikipedia.org/wiki/Data_segment)
  * [Physical Address Extension](https://en.wikipedia.org/wiki/Physical_Address_Extension)
  * [NX-Bit](https://de.wikipedia.org/wiki/NX-Bit)
  * [initrd](https://en.wikipedia.org/wiki/Initrd)
  * [kernel parameters](https://chromium.googlesource.com/chromiumos/third_party/kernel/+/master/Documentation/kernel-parameters.txt)
  

