# start_kernel part II

`lock_kernel` get [big kernel lock](https://kernelnewbies.org/BigKernelLock).

In operating systems, a giant lock, also known as a big-lock or kernel-lock, is a lock that may be used in the kernel to provide concurrency control required by symmetric multiprocessing (SMP) systems.

A giant lock is a solitary global lock that is held whenever a thread enters kernel space and released when the thread returns to user space; a system call is the archetypal example. In this model, threads in user space can run concurrently on any available processors or processor cores, but no more than one thread can run in kernel space; any other threads that try to enter kernel space are forced to wait. In other words, the giant lock eliminates all concurrency in kernel space.

By isolating the kernel from concurrency, many parts of the kernel no longer need to be modified to support SMP. However, as in giant-lock SMP systems only one processor can run the kernel code at a time, performance for applications spending significant amounts of time in the kernel is not much improved. Accordingly, the giant-lock approach is commonly seen as a preliminary means of bringing SMP support to an operating system, yielding benefits only in user space. Most modern operating systems use a fine-grained locking approach.

Linux kernel 2.6.39 removed the final part of the BKL, the whole BKL locking mechanism. BKL is now finally totally gone!

`tick_init` initialize the tick control, register tick notifier for clock event devices, the tick notifier defined in code kernel/time/tick-common.c line 363.

`boot_cpu_init` activate the first cpu, all routines involved in `boot_cpu_init` finally call `cpumask_set_cpu` to set specified bit of following cpu mask as ture, every bit of a dedicated cpu mask represent a cpu.

```cpu_mask
cpu_possible_mask
cpu_online_mask
cpu_present_mask
cpu_active_mask
```

`page_address_init` intialize `page_address_pool`, it's freelist of `page_address_map`, add list of all elements in `page_address_maps` to freelist, array length of `page_address_maps` in this kernel image is 
```
(gdb) p sizeof(page_address_maps)/sizeof(page_address_maps[0])
$3 = 512
```
initialize page address hash table `page_address_htable` and lock of `page_address_pool`.

`setup_arch` the most important routine involved in `start_kernel`, every architecture has its own implementation for it.

  1. Initialize common cpu data of all cpus, source information of cpu initialized before start_kernel in arch/x86/kernel/head_32.S.
  2. We don't config CONFIG_X86_VISWS, the defination of function `visws_early_detect` is NULL.
  3. CONFIG_VMI is not configured, the defination of function `vmi_init` is same as `visws_early_detect`.
  4. `early_cpu_init` intialize cpu information, it firstly get information of kernel support cpu, listed as follow, after that identify the cpu it used in this linux initialization with `early_identify_cpu`.
  
kernel supports cpu listed as follow:

```kernel_support_cpu

(gdb) p count
$1 = 7
(gdb) p cpu_devs[0]
$2 = (const struct cpu_dev *) 0xc1687140 <intel_cpu_dev>
(gdb) p cpu_devs[0].c_ident
$3 = {0xc15d066a "GenuineIntel", 0x0}
(gdb) p cpu_devs[1]
$4 = (const struct cpu_dev *) 0xc1687280 <amd_cpu_dev>
(gdb) p cpu_devs[1].c_ident
$5 = {0xc15d085e "AuthenticAMD", 0x0}
(gdb) p cpu_devs[2]
$6 = (const struct cpu_dev *) 0xc16873c0 <nsc_cpu_dev>
(gdb) p cpu_devs[2].c_ident
$7 = {0xc15d08a5 "Geode by NSC", 0x0}
(gdb) p cpu_devs[3]
$8 = (const struct cpu_dev *) 0xc1687500 <cyrix_cpu_dev>
(gdb) p cpu_devs[3].c_ident
$9 = {0xc15d087f "CyrixInstead", 0x0}
(gdb) p cpu_devs[4]
$10 = (const struct cpu_dev *) 0xc1687720 <centaur_cpu_dev>
(gdb) p cpu_devs[4].c_ident
$11 = {0xc15d0919 "CentaurHauls", 0x0}
(gdb) p cpu_devs[5]
$12 = (const struct cpu_dev *) 0xc1687860 <transmeta_cpu_dev>
(gdb) p cpu_devs[5].c_ident
$13 = {0xc15d093c "GenuineTMx86", 0xc15d0949 "TransmetaCPU"}
(gdb) p cpu_devs[6]
$14 = (const struct cpu_dev *) 0xc16879a0 <umc_cpu_dev>
(gdb) p cpu_devs[6].c_ident
$15 = {0xc15d0956 "UMC UMC UMC", 0x0}
```

  In `early_identify_cpu`:
  
  a. Check cpu has cpuid instruction enabled or not with `have_cpuid_p` which further involves `flag_is_changeable_p`, the entire proceduce of `flag_is_changeable_p` please take following debug information as reference:
  
  ```flag_is_changeable_p
  
  flag_is_changeable_p (flag=2097152) at arch/x86/kernel/cpu/common.c:186
186		asm volatile ("pushfl		\n\t"
(gdb) x/10i $eip
=> 0xc146f11f <have_cpuid_p+3>:	pushf  
   0xc146f120 <have_cpuid_p+4>:	pushf  
   0xc146f121 <have_cpuid_p+5>:	pop    %eax
   0xc146f122 <have_cpuid_p+6>:	mov    %eax,%edx
   0xc146f124 <have_cpuid_p+8>:	xor    $0x200000,%eax
   0xc146f129 <have_cpuid_p+13>:	push   %eax
   0xc146f12a <have_cpuid_p+14>:	popf   
   0xc146f12b <have_cpuid_p+15>:	pushf  
   0xc146f12c <have_cpuid_p+16>:	pop    %eax
   0xc146f12d <have_cpuid_p+17>:	popf   
(gdb) info registers eflags
eflags         0x46	[ PF ZF ]
(gdb) si
0xc146f120	186		asm volatile ("pushfl		\n\t"
(gdb) 
0xc146f121	186		asm volatile ("pushfl		\n\t"
(gdb) 
0xc146f122	186		asm volatile ("pushfl		\n\t"
(gdb) info registers eax
eax            0x46	70
(gdb) si
0xc146f124	186		asm volatile ("pushfl		\n\t"
(gdb) info registers edx
edx            0x46	70
(gdb) si
0xc146f129	186		asm volatile ("pushfl		\n\t"
(gdb) info registers eax
eax            0x200046 	2097222
(gdb) si
0xc146f12a	186		asm volatile ("pushfl		\n\t"
(gdb) 
0xc146f12b	186		asm volatile ("pushfl		\n\t"
(gdb) info registers eflags
eflags         0x200046	[ PF ZF ID ]
(gdb) si
0xc146f12c	186		asm volatile ("pushfl		\n\t"
(gdb) 
0xc146f12d	186		asm volatile ("pushfl		\n\t"
(gdb) info registers eax
eax            0x200046	       2097222
(gdb) si
200		return ((f1^f2) & flag) != 0;
(gdb) info registers eflags
eflags         0x46	[ PF ZF ]
(gdb) info registers eax edx
eax            0x200046	2097222
edx            0x46	70
(gdb) p f1
$16 = 2097222
(gdb) p /x f1
$17 = 0x200046
(gdb) p /x f2
$18 = 0x46
(gdb) p /x flag
$19 = 0x200000
(gdb) p  (f1^f2)
$20 = 2097152
(gdb) p  /x (f1^f2)
$21 = 0x200000
(gdb) p /x ((f1^f2) & flag)
$22 = 0x200000
  ```

  b. Get cpu information with `cpu_detect` in which it involves [cpuid](https://en.wikipedia.org/wiki/CPUID) instruction finally with different operation id, get verder id with opcode 0 and get processor info and feature bits.
  




# Links
  * [Giant lock](https://en.wikipedia.org/wiki/Giant_lock)
  * [Big Kernel Lock](https://kernelnewbies.org/BigKernelLock)
  * [Clock Event Devices](http://www.halolinux.us/kernel-architecture/clock-event-devices.html)
  * [Clock sources, Clock events, sched_clock() and delay timers](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
  * [Paging](https://en.wikipedia.org/wiki/Paging)
  * [CONFIG_X86_VISWS: SGI 320/540 (Visual Workstation)](http://cateee.net/lkddb/web-lkddb/X86_VISWS.html)
  * [SGI Visual Workstation](http://www.nekochan.net/wiki/SGI_Visual_Workstation)
  * [CPUID](https://en.wikipedia.org/wiki/CPUID)