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

`lockdep_init` initializes two hash tables. To enable lockdep module, enable the configuration of lockdep as follow:

```enable_lockdep
1. edit .config through menuconfig
   make menuconfig
2. Enable lockdep related hacking options
      [*] Detect Hard and Soft Lockups
      [*] Detect Hung Tasks
      [*] RT Mutex debugging, deadlock detection
      -*- Spinlock and rw-lock debugging: basic checks
      -*- Mutex debugging: basic checks
      -*- Lock debugging: detect incorrect freeing of live locks
      [*] Lock debugging: prove locking correctness
      [*] Lock usage statistics
3. make and make install
4. create new bootable disk image
```

"Lockdep" is the kernel lock validator, which, when enabled, creates a detailed model of how locks are used in the kernel. This model can be used to find potential deadlocks and other problems.

# Links

* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [weak : GCC function attributes](https://gcc.gnu.org/onlinedocs/gcc-3.2/gcc/Function-Attributes.html)
* [Interrupts, threads, and lockdep](https://lwn.net/Articles/321663/)
* [The kernel lock validator](https://lwn.net/Articles/185666/)


