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



# Links

* [User-space lockdep](https://lwn.net/Articles/536363/)
* [The kernel lock validator](https://lwn.net/Articles/185666/)
