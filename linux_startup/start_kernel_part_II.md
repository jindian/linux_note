# start_kernel part II

`lock_kernel` get [big kernel lock](https://kernelnewbies.org/BigKernelLock).

In operating systems, a giant lock, also known as a big-lock or kernel-lock, is a lock that may be used in the kernel to provide concurrency control required by symmetric multiprocessing (SMP) systems.

A giant lock is a solitary global lock that is held whenever a thread enters kernel space and released when the thread returns to user space; a system call is the archetypal example. In this model, threads in user space can run concurrently on any available processors or processor cores, but no more than one thread can run in kernel space; any other threads that try to enter kernel space are forced to wait. In other words, the giant lock eliminates all concurrency in kernel space.

By isolating the kernel from concurrency, many parts of the kernel no longer need to be modified to support SMP. However, as in giant-lock SMP systems only one processor can run the kernel code at a time, performance for applications spending significant amounts of time in the kernel is not much improved. Accordingly, the giant-lock approach is commonly seen as a preliminary means of bringing SMP support to an operating system, yielding benefits only in user space. Most modern operating systems use a fine-grained locking approach.

Linux kernel 2.6.39 removed the final part of the BKL, the whole BKL locking mechanism. BKL is now finally totally gone!

`tick_init` 

# Links
  * [Giant lock](https://en.wikipedia.org/wiki/Giant_lock)
  * [Big Kernel Lock](https://kernelnewbies.org/BigKernelLock)