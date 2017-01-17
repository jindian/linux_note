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

# Links
  * [Giant lock](https://en.wikipedia.org/wiki/Giant_lock)
  * [Big Kernel Lock](https://kernelnewbies.org/BigKernelLock)
  * [Clock Event Devices](http://www.halolinux.us/kernel-architecture/clock-event-devices.html)
  * [Clock sources, Clock events, sched_clock() and delay timers](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
  * [Paging](https://en.wikipedia.org/wiki/Paging)