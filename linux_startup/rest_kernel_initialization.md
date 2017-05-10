# rest kernel initialization

`start_kernel` calls `rest_init`, at this point things are almost all working. `rest_init` creates a kernel thread passing function `kernel_init` as the entry point. `rest_init` calls `schedule` to kickstart task scheduling and go to sleep by calling `cpu_idle`,  which is the idle thread for the Linux kernel. cpu_idle() runs forever and so does process zero, which hosts it. Whenever there is work to do ¨C a runnable process ¨C process zero gets booted out of the CPU, only to return when no runnable processes are available.

Detail steps of `rest_init`:

* Checks number of cpu and number of context switch; sets `rcu_scheduler_active` as 1

The rcu_scheduler_active variable transitions from zero to one just before the first task is spawned. So when this variable is zero, RCU can assume that there is but one task, allowing RCU to (for example) optimize synchronize_sched() to a simple barrier(). When this variable is one, RCU must actually do all the hard work required to detect real grace periods. This variable is also used to suppress boot-time false positives from lockdep-RCU error checking.


```
start_kernel () at init/main.c:694
694		rest_init();
(gdb) s
rest_init () at init/main.c:417
417		rcu_scheduler_starting();
(gdb) s
rcu_scheduler_starting () at kernel/rcupdate.c:184
184		WARN_ON(num_online_cpus() != 1);
(gdb) s
cpumask_weight (srcp=<optimized out>) at kernel/rcupdate.c:184
184		WARN_ON(num_online_cpus() != 1);
(gdb) s
bitmap_weight (nbits=8, src=<optimized out>) at include/linux/bitmap.h:263
263			return hweight_long(*src & BITMAP_LAST_WORD_MASK(nbits));
(gdb) s
hweight_long (w=<optimized out>) at include/linux/bitops.h:45
45		return sizeof(w) == 4 ? hweight32(w) : hweight64(w);
(gdb) s
hweight32 (w=1) at lib/hweight.c:14
14		unsigned int res = w - ((w >> 1) & 0x55555555);
(gdb) n
15		res = (res & 0x33333333) + ((res >> 2) & 0x33333333);
(gdb) 
16		res = (res + (res >> 4)) & 0x0F0F0F0F;
(gdb) 
17		res = res + (res >> 8);
(gdb) 
18		return (res + (res >> 16)) & 0x000000FF;
(gdb) 
19	}
(gdb) 
rcu_scheduler_starting () at kernel/rcupdate.c:184
185		WARN_ON(nr_context_switches() > 0);
(gdb) s
nr_context_switches () at kernel/sched.c:3088
3088		for_each_possible_cpu(i)
(gdb) n
3089			sum += cpu_rq(i)->nr_switches;
(gdb) 
3088		for_each_possible_cpu(i)
(gdb) 
3092	}
(gdb) 
rcu_scheduler_starting () at kernel/rcupdate.c:186
186		rcu_scheduler_active = 1;
(gdb) 
187	}
(gdb) 
```

* Creates a kernel thread.

    - `kernel_thread` initializes register parameters and invoke `do_fork` to create new process.

```kernel_thread
423		kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
(gdb) s
kernel_thread (fn=fn@entry=0xc16fa7e8 <kernel_init>, arg=arg@entry=0x0, 
    flags=flags@entry=2560) at arch/x86/kernel/process_32.c:205
205	{
(gdb) n
208		memset(&regs, 0, sizeof(regs));
(gdb) 
210		regs.bx = (unsigned long) fn;
(gdb) 
211		regs.dx = (unsigned long) arg;
(gdb) 
213		regs.ds = __USER_DS;
(gdb) 
214		regs.es = __USER_DS;
(gdb) 
215		regs.fs = __KERNEL_PERCPU;
(gdb) 
216		regs.gs = __KERNEL_STACK_CANARY;
(gdb) 
217		regs.orig_ax = -1;
(gdb) 
218		regs.ip = (unsigned long) kernel_thread_helper;
(gdb) 
219		regs.cs = __KERNEL_CS | get_kernel_rpl();
(gdb) 
220		regs.flags = X86_EFLAGS_IF | X86_EFLAGS_SF | X86_EFLAGS_PF | 0x2;
(gdb) 
223		return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
```

    - `do_fork` does some preliminary argument and permissions checking before actually start allocating stuff

```argument_and_permission_checking
223		return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
(gdb) s
do_fork (clone_flags=clone_flags@entry=8391424, 
    stack_start=stack_start@entry=0, 
    regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
    stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
    child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1374
1374	{
(gdb) n
1383		if (clone_flags & CLONE_NEWUSER) {
(gdb) 
1397		if (unlikely(clone_flags & CLONE_STOPPED)) {
(gdb) 
1414		if (likely(user_mode(regs)))
(gdb) p *regs
$2 = {bx = 3245320168, cx = 0, dx = 0, si = 0, di = 0, bp = 0, ax = 0, 
  ds = 123, es = 123, fs = 216, gs = 224, orig_ax = 4294967295, 
  ip = 3238017744, cs = 96, flags = 646, sp = 0, ss = 0}
(gdb) s
user_mode (regs=0xc168bf64 <init_thread_union+8036>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/ptrace.h:160
160		return (regs->cs & SEGMENT_RPL_MASK) == USER_RPL;
(gdb) n
do_fork (clone_flags=clone_flags@entry=8391424, 
    stack_start=stack_start@entry=0, 
    regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
    stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
    child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1414
1414		if (likely(user_mode(regs)))
(gdb) 
1417		p = copy_process(clone_flags, stack_start, regs, stack_size,
(gdb) 
```

    - `copy_process` is used to create a new process as a copy of old one, but doesn't actually start it yet. It copies the registers, and all the appropriate parts of the process environment(as per clone flags). Before copying stuff, do some argument checking, it's failed when checking `CLONE_PARENT`

```copy_process
1417		p = copy_process(clone_flags, stack_start, regs, stack_size,
(gdb) s
copy_process (trace=0, pid=0x0, child_tidptr=0x0, stack_size=0, 
    regs=0xc168bf64 <init_thread_union+8036>, stack_start=0, 
    clone_flags=8391424) at kernel/fork.c:993
993		if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))
(gdb) n
1000		if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
(gdb) 
1008		if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
(gdb) 
1017		if ((clone_flags & CLONE_PARENT) &&
(gdb) 
do_fork (clone_flags=clone_flags@entry=8391424, 
    stack_start=stack_start@entry=0, 
    regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
    stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
    child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1423
1423		if (!IS_ERR(p)) {
```

# Links

* [User-space lockdep](https://lwn.net/Articles/536363/)
* [The kernel lock validator](https://lwn.net/Articles/185666/)
