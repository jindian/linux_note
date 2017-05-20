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
7399		if (cpumask_test_cpu(task_cpu(p), new_mask))                        
                                                                                # test of task is allowed to executing with new mask
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

* Prepare for SMP bootup.  The MP table or ACPI has been read earlier.
Just do some sanity checking here and enable APIC mode.

```
882		smp_prepare_cpus(setup_max_cpus);
(gdb) s
smp_prepare_cpus (max_cpus=8)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/smp.h:91
91		smp_ops.smp_prepare_cpus(max_cpus);
(gdb) s
native_smp_prepare_cpus (max_cpus=8) at arch/x86/kernel/smpboot.c:1069
1069		smp_cpu_index_default();
(gdb) s
smp_cpu_index_default () at arch/x86/kernel/smpboot.c:1053
1053		for_each_possible_cpu(i) {
(gdb) n
1054			c = &cpu_data(i);
(gdb) 
1056			c->cpu_index = nr_cpu_ids;
(gdb) p nr_cpu_ids 
$25 = 1
(gdb) n
172		return native_smp_prepare_cpus (max_cpus=8) at arch/x86/kernel/smpboot.c:1070
1070		current_cpu_data = boot_cpu_data;
(gdb) 
1071		cpumask_copy(cpu_callin_mask, cpumask_of(0));
(gdb) p boot_cpu_data 
$26 = {x86 = 6 '\006', x86_vendor = 0 '\000', x86_model = 6 '\006', 
  x86_mask = 3 '\003', wp_works_ok = 1 '\001', hlt_works_ok = 1 '\001', 
  hard_math = 1 '\001', rfu = 0 '\000', fdiv_bug = 0 '\000', 
  f00f_bug = 0 '\000', coma_bug = 0 '\000', pad0 = 0 '\000', 
  x86_virt_bits = 32 ' ', x86_phys_bits = 36 '$', x86_coreid_bits = 0 '\000', 
  extended_cpuid_level = 2147483652, cpuid_level = 4, x86_capability = {
    125873145, 0, 0, 262720, 2155872257, 0, 0, 0, 0}, 
  x86_vendor_id = "GenuineIntel\000\000\000", 
  x86_model_id = "QEMU Virtual CPU version 2.0.0", '\000' <repeats 33 times>, 
  x86_cache_size = 4096, x86_cache_alignment = 32, x86_power = 0, 
  loops_per_jiffy = 13568660, llc_shared_map = {{bits = {0}}}, 
  x86_max_cores = 1, apicid = 0, initial_apicid = 0, x86_clflush_size = 32, 
  booted_cores = 0, phys_proc_id = 0, cpu_core_id = 0, cpu_index = 0, 
  x86_hyper_vendor = 0}
(gdb) p cpu_callin_mask 
$27 = {{bits = {0}}}
(gdb) n
1072		mb();
(gdb) p cpu_callin_mask 
$28 = {{bits = {1}}}
(gdb) 
1076		smp_store_cpu_info(0); /* Final full version of the data */
(gdb) s
smp_store_cpu_info (id=id@entry=0) at arch/x86/kernel/smpboot.c:388
388		struct cpuinfo_x86 *c = &cpu_data(id);
(gdb) n
390		copy_cpuinfo_x86(c, &boot_cpu_data);
(gdb) s
copy_cpuinfo_x86 (src=<optimized out>, dst=0xc234a200)
    at arch/x86/kernel/smpboot.c:377
377		*dst = *src;
(gdb) n
smp_store_cpu_info (id=id@entry=0) at arch/x86/kernel/smpboot.c:391
391		c->cpu_index = id;
(gdb) 
392		if (id != 0)
(gdb) 
394	}
(gdb) 
native_smp_prepare_cpus (max_cpus=8) at arch/x86/kernel/smpboot.c:1078
1078		boot_cpu_logical_apicid = logical_smp_processor_id();
(gdb) s
logical_smp_processor_id () at arch/x86/kernel/smpboot.c:1078
1078		boot_cpu_logical_apicid = logical_smp_processor_id();
(gdb) s
apic_read (reg=208)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/apic.h:378
378		return apic->read(reg);
(gdb) s
native_apic_mem_read (reg=208)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/apic.h:112
112		return *((volatile u32 *)(APIC_BASE + reg));
(gdb) n
113	}
(gdb) 
logical_smp_processor_id ()
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/smp.h:196
196		return GET_APIC_LOGICAL_ID(apic_read(APIC_LDR));
(gdb) n
native_smp_prepare_cpus (max_cpus=8) at arch/x86/kernel/smpboot.c:1080
1080		current_thread_info()->cpu = 0;  /* needed? */
(gdb) 
1081		for_each_possible_cpu(i) {                                          # allocate memory for per cpu variables
(gdb) 
1082			zalloc_cpumask_var(&per_cpu(cpu_sibling_map, i), GFP_KERNEL);
(gdb) 
1083			zalloc_cpumask_var(&per_cpu(cpu_core_map, i), GFP_KERNEL);
(gdb) 
1084			zalloc_cpumask_var(&cpu_data(i).llc_shared_map, GFP_KERNEL);
(gdb) 
1086		set_cpu_sibling_map(0);
(gdb) 
1088		enable_IR_x2apic();
(gdb) 
1089		default_setup_apic_routing();
(gdb) 
1091		if (smp_sanity_check(max_cpus) < 0) {
(gdb) p max_cpus 
$29 = 8
(gdb) n
1098		if (read_apic_id() != boot_cpu_physical_apicid) {
(gdb) p boot_cpu_physical_apicid
$30 = 0
(gdb) n
1105		connect_bsp_APIC();
(gdb) 
1110		setup_local_APIC();
(gdb) 
1115		if (!skip_ioapic_setup && nr_ioapics)
(gdb) 
1116			enable_IO_APIC();
(gdb) 
1118		end_local_APIC_setup();
(gdb) 
1120		map_cpu_to_logical_apicid();
(gdb) s
map_cpu_to_logical_apicid () at arch/x86/kernel/smpboot.c:172
172		int cpu = smp_processor_id();
(gdb) n
173		int apicid = logical_smp_processor_id();
(gdb) 
174		int node = apic->apicid_to_node(apicid);
(gdb) 
179		cpu_2_logical_apicid[cpu] = apicid;
(gdb) 
181	}
(gdb) 
native_smp_prepare_cpus (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1122
1122		if (apic->setup_portio_remap)
(gdb) 
1125		smpboot_setup_io_apic();
(gdb) s
smpboot_setup_io_apic ()
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/smpboot_hooks.h:47
47		if (!skip_ioapic_setup && nr_ioapics)
(gdb) n
48			setup_IO_APIC();
(gdb) 
native_smp_prepare_cpus (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1130
1130		printk(KERN_INFO "CPU%d: ", 0);
(gdb) 
1131		print_cpu_info(&cpu_data(0));
(gdb) 
1132		x86_init.timers.setup_percpu_clockev();
(gdb) s
setup_boot_APIC_clock () at arch/x86/kernel/apic/apic.c:749
749		if (disable_apic_timer) {
(gdb) n
759		apic_printk(APIC_VERBOSE, "Using local APIC timer interrupts.\n"
(gdb) 
762		if (calibrate_APIC_clock()) {
(gdb) 
774		if (nmi_watchdog != NMI_IO_APIC)
(gdb) 
775			lapic_clockevent.features &= ~CLOCK_EVT_FEAT_DUMMY;
(gdb) 
765				setup_APIC_timer();
(gdb) 
782	}
(gdb) 
native_smp_prepare_cpus (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1137
1137		set_mtrr_aps_delayed_init();
(gdb) s
set_mtrr_aps_delayed_init () at arch/x86/kernel/cpu/mtrr/main.c:768
768		if (!use_intel())
(gdb) n
772	}
(gdb) 
native_smp_prepare_cpus (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1140
1140	}
(gdb) 
```

# Links
* [Optimizing preemption](https://lwn.net/Articles/563185/)
* [completions - wait for completion handling](https://www.kernel.org/doc/Documentation/scheduler/completion.txt)
