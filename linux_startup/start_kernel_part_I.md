# start\_kernel part I

start\_kernel routine is the most important routine of linux startup, it includes many subroutine of the initialization, it's devided into several parts to study.

```start_kernel
init/main.c:520

asmlinkage void __init start_kernel(void)
{
    ......
}
```

Defination of `smp_setup_processor_id` is null for x86 architecture.

**Symmetric multiprocessing \(SMP\)** involves a symmetric multiprocessor system hardware and software architecture where two or more identical processors are connected to a single, shared main memory, have full access to all I/O devices, and are controlled by a single operating system instance that treats all processors equally, reserving none for special purposes. Most multiprocessor systems today use an SMP architecture. In the case of multi-core processors, the SMP architecture applies to the cores, treating them as separate processors.

SMP systems are tightly coupled multiprocessor systems with a pool of homogeneous processors running independent of each other. Each processor, executing different programs and working on different sets of data, has the capability of sharing common resources \(memory, I/O device, interrupt system and so on\) that are connected using a system bus or a crossbar.

`lockdep_init` initializes two hash tables. "Lockdep" is the kernel lock validator, which, when enabled, creates a detailed model of how locks are used in the kernel. This model can be used to find potential deadlocks and other problems.

To enable lockdep module, enable the configuration of lockdep as follow:

```enable_lockdep
1.  edit .config through menuconfig
      make menuconfig
2.  enable lockdep related hacking options
      [*] Detect Hard and Soft Lockups
      [*] Detect Hung Tasks
      [*] RT Mutex debugging, deadlock detection
      -*- Spinlock and rw-lock debugging: basic checks
      -*- Mutex debugging: basic checks
      -*- Lock debugging: detect incorrect freeing of live locks
      [*] Lock debugging: prove locking correctness
      [*] Lock usage statistics
3.  recompile linux kernel and make install
4.  create new bootable disk image
```

boot the new kernel image, under /proc you should see the following new folders:

```lockdep_folders_under_proc
/proc/lockdep
/proc/lockdep_chains
/proc/lockdep_stat
/proc/locks
/proc/lock_stats
```

If the app deadlocks(hangs), do a: `ps -aux | grep <app_name>`, you should see a +D (uninterruptible sleep) state for your app, do a: `dmesg`, the log it prints will include the function/file causing the deadlock.

`debug_objects_early_init` initialize the hash buckets and link the static object pool objects into the poll list. After this call the object tracker is fully operational.

`boot_init_stack_canary` intialize the stack canary value. Stack canaries are used to detect a stack buffer overflow before execution of malicious code can occur. This method works by placing a small integer, the value of which is randomly chosen at program start, in memory just before the stack return pointer. Most buffer overflows overwrite memory from lower to higher memory addresses, so in order to overwrite the return pointer (and thus take control of the process) the canary value must also be overwritten. This value is checked to make sure it has not changed before a routine uses the return pointer on the stack. This technique can greatly increase the difficulty of exploiting a stack buffer overflow because it forces the attacker to gain control of the instruction pointer by some non-traditional means such as corrupting other important variables on the stack.

`cgroup_init_early` 

# Links

* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [weak : GCC function attributes](https://gcc.gnu.org/onlinedocs/gcc-3.2/gcc/Function-Attributes.html)
* [Interrupts, threads, and lockdep](https://lwn.net/Articles/321663/)
* [The kernel lock validator](https://lwn.net/Articles/185666/)
* [How to use lockdep feature in linux kernel for deadlock detection](http://stackoverflow.com/questions/20892822/how-to-use-lockdep-feature-in-linux-kernel-for-deadlock-detection)
* [object debugging infrastructure](https://lwn.net/Articles/271582/)
* [Buffer overflow protection](https://en.wikipedia.org/wiki/Buffer_overflow_protection#Canaries)
* [Stack buffer overflow](https://en.wikipedia.org/wiki/Stack_buffer_overflow)


