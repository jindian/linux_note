# kernel_init

* The first function called by `kernel_init` is wait_for_completion, it checks completion `kthreadd_done`. `kthreadd_done` is updated after kernel daemon thread created, it's used to ignore race conditon between root thread and init thread. If `kthead_done` isn't updated, the event `kthreadd_done.done is zero`, `kernel_init` won't continue its execution until timeout or `kthreadd_done.done` is non-zero.

```kernel_init
Breakpoint 2, kernel_init (unused=0x0) at init/main.c:859
859		wait_for_completion(&kthreadd_done);
(gdb) s
wait_for_completion (x=x@entry=0xc1730480 <kthreadd_done>)
    at kernel/sched.c:6091
6091		wait_for_common(x, MAX_SCHEDULE_TIMEOUT, TASK_UNINTERRUPTIBLE);
(gdb) s
wait_for_common (x=x@entry=0xc1730480 <kthreadd_done>, 
    timeout=timeout@entry=2147483647, state=state@entry=2)
    at kernel/sched.c:6070
6070	{
(gdb) n
6071		might_sleep();                                                    # might be preemted
(gdb) 
6073		spin_lock_irq(&x->wait.lock);
(gdb) 
6074		timeout = do_wait_for_common(x, timeout, state);
(gdb) s
do_wait_for_common (state=2, timeout=2147483647, x=0xc1730480 <kthreadd_done>)
    at kernel/sched.c:6045
6045		if (!x->done) {
(gdb) p *x
$1 = {done = 1, wait = {lock = {raw_lock = {slock = 513}, magic = 3735899821, 
      owner_cpu = 0, owner = 0xc7070000, dep_map = {
        key = 0xc1730494 <kthreadd_done+20>, 
        class_cache = 0xc1b5f8d0 <lock_classes+151696>, 
        name = 0xc15c53c4 "(kthreadd_done).wait.lock", cpu = 0, 
        ip = 3242755061}}, task_list = {next = 0xc17304a8 <kthreadd_done+40>, 
      prev = 0xc17304a8 <kthreadd_done+40>}}}
(gdb) n
6064		x->done--;
(gdb) 
6065		return timeout ?: 1;
(gdb) 
wait_for_common (x=x@entry=0xc1730480 <kthreadd_done>, 
    timeout=<optimized out>, timeout@entry=2147483647, state=state@entry=2)
    at kernel/sched.c:6075
6075		spin_unlock_irq(&x->wait.lock);
(gdb) 
6077	}
(gdb) 
wait_for_completion (x=x@entry=0xc1730480 <kthreadd_done>)
    at kernel/sched.c:6092
6092	}
```

* Call `lock_kernel` to get BKL

```lock_kernel
kernel_init (unused=<optimized out>) at init/main.c:860
860		lock_kernel();
(gdb) s
lock_kernel () at lib/kernel_lock.c:117
117	{
(gdb) n
118		int depth = current->lock_depth+1;
(gdb) 
119		if (likely(!depth))
(gdb) p depth
$2 = 0
(gdb) n
120			__lock_kernel();
(gdb) s
__lock_kernel () at lib/kernel_lock.c:96
96		_raw_spin_lock(&kernel_flag);
(gdb) p kernel_flag 
$3 = {raw_lock = {slock = 257}, magic = 3735899821, owner_cpu = 4294967295, 
  owner = 0xffffffff, dep_map = {key = 0x0, class_cache = 0x0, 
    name = 0xc1613adc "kernel_flag", cpu = 0, ip = 0}}
(gdb) s
_raw_spin_lock (lock=lock@entry=0xc16922a0 <kernel_flag>)
    at lib/spinlock_debug.c:130
130		debug_spin_lock_before(lock);
(gdb) n
131		if (unlikely(!__raw_spin_trylock(&lock->raw_lock)))
(gdb) s
__raw_spin_trylock (lock=0xc16922a0 <kernel_flag>) at lib/spinlock_debug.c:131
131		if (unlikely(!__raw_spin_trylock(&lock->raw_lock)))
(gdb) s
__ticket_spin_trylock (lock=0xc16922a0 <kernel_flag>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/spinlock.h:84
84		asm volatile("movzwl %2, %0\n\t"
(gdb) n
_raw_spin_lock (lock=lock@entry=0xc16922a0 <kernel_flag>)
    at lib/spinlock_debug.c:131
131		if (unlikely(!__raw_spin_trylock(&lock->raw_lock)))
(gdb) 
133		debug_spin_lock_after(lock);
(gdb) 
134	}
(gdb) 
lock_kernel () at lib/kernel_lock.c:121
121		current->lock_depth = depth;
(gdb) p depth
$4 = 0
(gdb) n
122	}
(gdb) 
```

* Sets zone of memory where allow init to allocate

```
kernel_init (unused=<optimized out>) at init/main.c:865
865		set_mems_allowed(node_states[N_HIGH_MEMORY]);
(gdb) s
set_mems_allowed (nodemask=...) at include/linux/cpuset.h:91
91		current->mems_allowed = nodemask;
(gdb) p /d N_HIGH_MEMORY
$8 = 3
(gdb) n
kernel_init (unused=<optimized out>) at init/main.c:869
869		set_cpus_allowed_ptr(current, cpu_all_mask);
```

* Change a given task's CPU affinity. Migrate the thread to a proper CPU and schedule it away if the CPU it's executing on is removed from the allowed bitmask.

```
kernel_init (unused=<optimized out>) at init/main.c:869
869		set_cpus_allowed_ptr(current, cpu_all_mask);
(gdb) s
set_cpus_allowed_ptr (p=p@entry=0xc7070000, new_mask=0xc1497fa4 <cpu_all_bits>)
    at kernel/sched.c:7372
7372		while (task_is_waking(p))                                           # check state of the task, if it's waking, wait here
(gdb) p p->state
$9 = 0
(gdb) n
7374		rq = task_rq_lock(p, &flags);
(gdb) 
7375		if (task_is_waking(p)) {
(gdb) 
7380		if (!cpumask_intersects(new_mask, cpu_active_mask)) {
(gdb) 
7385		if (unlikely((p->flags & PF_THREAD_BOUND) && p != current &&
(gdb) p p->flags
$10 = 2105408
(gdb) n
7391		if (p->sched_class->set_cpus_allowed)
(gdb) p p->sched_class->set_cpus_allowed
$11 = (void (*)(struct task_struct *, const struct cpumask *)) 0x0
(gdb) n
7394			cpumask_copy(&p->cpus_allowed, new_mask);
(gdb) 
7395			p->rt.nr_cpus_allowed = cpumask_weight(new_mask);
(gdb) 
7399		if (cpumask_test_cpu(task_cpu(p), new_mask))                        # test of task is allowed to executing with new mask
                                                                                # if yes, complete the function execution
(gdb) p p->rt.nr_cpus_allowed
$12 = 8
(gdb) p *new_mask
$13 = {bits = {255}}
(gdb) n
7365		int ret = 0;
(gdb) 
7415		task_rq_unlock(rq, &flags);
(gdb) 
7417		return ret;
(gdb) 
7418	}
```

# Links
* [Optimizing preemption](https://lwn.net/Articles/563185/)
* [completions - wait for completion handling](https://www.kernel.org/doc/Documentation/scheduler/completion.txt)
