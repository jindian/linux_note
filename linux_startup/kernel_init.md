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

```smp_prepare_cpus
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

* Invoke SMP init routines

migration_init: create migration thread for boot cpu and wake up the created thread, registers notifier to response new added CPU.

spawn_ksoftirqd: create ksoftirqd thread and wake up the created thread, register notifier to response added CPU.

init_call_single_data: initialize per CPU variable `call_single_queue` and allocate CPU mask for up CPU, register notifier to response new added CPU.

spawn_softlockup_task: create watch dog thread for CPU up and down, register notifier to response new added CPU

relay_init: register CPU hotplug callback

tracer_alloc_buffers: prepare trace for all possible CPU and register response routine

init_trace_printk: register module notifier for treace printk


```do_pre_smp_initcalls
Breakpoint 3, kernel_init (unused=<optimized out>) at init/main.c:884
884		do_pre_smp_initcalls();
(gdb) s
do_pre_smp_initcalls () at init/main.c:797
797		for (call = __initcall_start; call < __early_initcall_end; call++)
(gdb) p __initcall_start
$1 = 0xc177d998
(gdb) p *(initcall_t*)__initcall_start
$3 = (initcall_t) 0xc170d5e3 <migration_init>
(gdb) p __early_initcall_end
$4 = 0xc177d9b4 <__initcall_init_mmap_min_addr0>
(gdb) p __initcall_start+1
$6 = (initcall_t *) 0xc177d99c <__initcall_spawn_ksoftirqdearly>
(gdb) p __initcall_start+2
$7 = (initcall_t *) 0xc177d9a0 <__initcall_init_call_single_dataearly>
(gdb) p __initcall_start+3
$8 = (initcall_t *) 0xc177d9a4 <__initcall_spawn_softlockup_taskearly>
(gdb) p __initcall_start+4
$9 = (initcall_t *) 0xc177d9a8 <__initcall_relay_initearly>
(gdb) p __initcall_start+5
$10 = (initcall_t *) 0xc177d9ac <__initcall_tracer_alloc_buffersearly>
(gdb) p __initcall_start+6
$11 = (initcall_t *) 0xc177d9b0 <__initcall_init_trace_printkearly>
(gdb) 
```

* rest of initialization of SMP in boot CPU

```smp_init
887		smp_init();
(gdb) s
smp_init () at init/main.c:373
373		for_each_present_cpu(cpu) {
(gdb) 
374			if (num_online_cpus() >= setup_max_cpus)
(gdb) 
376			if (!cpu_online(cpu))
(gdb) p setup_max_cpus 
$12 = 8
(gdb) n
381		printk(KERN_INFO "Brought up %ld CPUs\n", (long)num_online_cpus());
(gdb) 
382		smp_cpus_done(setup_max_cpus);
(gdb) s
smp_cpus_done (max_cpus=8)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/smp.h:96
96		smp_ops.smp_cpus_done(max_cpus);
(gdb) s
native_smp_cpus_done (max_cpus=8) at arch/x86/kernel/smpboot.c:1166
1166		pr_debug("Boot done.\n");
(gdb) n
1168		impress_friends();
(gdb) s
impress_friends () at arch/x86/kernel/smpboot.c:483
483		pr_debug("Before bogomips.\n");
(gdb) n
484		for_each_possible_cpu(cpu)
(gdb) 
485			if (cpumask_test_cpu(cpu, cpu_callout_mask))
(gdb) 
486				bogosum += cpu_data(cpu).loops_per_jiffy;
(gdb) 
484		for_each_possible_cpu(cpu)
(gdb) 
487		printk(KERN_INFO
(gdb) p bogosum
$14 = 13568592
(gdb) n
491			(bogosum/(5000/HZ))%100);
(gdb) 
493		pr_debug("Before bogocount - setting activated=1.\n");
(gdb) 
native_smp_cpus_done (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1170
1170		setup_ioapic_dest();
(gdb) s
setup_ioapic_dest () at arch/x86/kernel/apic/io_apic.c:4096
4096		if (skip_ioapic_setup == 1)
(gdb) p skip_ioapic_setup
$15 = 0
(gdb) n
4099		for (ioapic = 0; ioapic < nr_ioapics; ioapic++)
(gdb) p nr_ioapics
$16 = 1
(gdb) n
4100		for (pin = 0; pin < nr_ioapic_registers[ioapic]; pin++) {
(gdb) p nr_ioapic_registers[0]
$17 = 24
(gdb) n
4101			irq_entry = find_irq_entry(ioapic, pin, mp_INT);
(gdb) 
4102			if (irq_entry == -1)
(gdb) 
4104			irq = pin_2_irq(irq_entry, ioapic, pin);
(gdb) 
4106			if ((ioapic > 0) && (irq > 16))
(gdb) p irq
$19 = 1
(gdb) n
4109			desc = irq_to_desc(irq);
(gdb) 
4114			if (desc->status &
(gdb) p desc
$21 = (struct irq_desc *) 0xc1690c00 <irq_desc_legacy+160>
(gdb) p *desc
$22 = {irq = 1, kstat_irqs = 0xc7001004, 
  handle_irq = 0xc10ab4a0 <handle_edge_irq>, chip = 0xc16f7140 <ioapic_chip>, 
  msi_desc = 0x0, handler_data = 0x0, chip_data = 0xc1697194 <irq_cfgx+20>, 
  action = 0x0, status = 512, depth = 1, wake_depth = 0, irq_count = 0, 
  last_unhandled = 0, irqs_unhandled = 0, lock = {raw_lock = {slock = 1028}, 
    magic = 3735899821, owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
      key = 0xc1e3376c <irq_desc_lock_class>, class_cache = 0x0, 
      name = 0xc15dae6f "&irq_desc_lock_class", cpu = 0, ip = 3238703088}}, 
  affinity = {{bits = {255}}}, node = 0, pending_mask = {{bits = {0}}}, 
  threads_active = {counter = 0}, wait_for_threads = {lock = {raw_lock = {
        slock = 0}, magic = 0, owner_cpu = 0, owner = 0x0, dep_map = {
        key = 0x0, class_cache = 0x0, name = 0x0, cpu = 0, ip = 0}}, 
    task_list = {next = 0x0, prev = 0x0}}, dir = 0x0, name = 0xc15d11ec "edge"}
(gdb) n
4118				mask = apic->target_cpus();
(gdb) s
default_target_cpus ()
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/apic.h:475
475	}
(gdb) n
setup_ioapic_dest () at arch/x86/kernel/apic/io_apic.c:4123
4123				set_ioapic_affinity_irq_desc(desc, mask);
(gdb) p mask
$23 = (const struct cpumask *) 0xc16f7260 <cpu_online_bits>
(gdb) p *mask
$24 = {bits = {1}}
(gdb) s
set_ioapic_affinity_irq_desc (
    desc=desc@entry=0xc1690c00 <irq_desc_legacy+160>, 
    mask=0xc16f7260 <cpu_online_bits>) at arch/x86/kernel/apic/io_apic.c:2366
2366		cfg = desc->chip_data;
(gdb) n
2368		spin_lock_irqsave(&ioapic_lock, flags);
(gdb) 
2369		dest = set_desc_affinity(desc, mask);
(gdb) 
2370		if (dest != BAD_APICID) {
(gdb) 
2372			dest = SET_APIC_LOGICAL_ID(dest);
(gdb) 
2373			__target_IO_APIC_irq(irq, dest, cfg);
(gdb) 
2374			ret = 0;
(gdb) 
2376		spin_unlock_irqrestore(&ioapic_lock, flags);
(gdb) 
2379	}
(gdb) break check_nmi_watchdog
Breakpoint 9 at 0xc1707f40: file arch/x86/kernel/apic/nmi.c, line 134.
(gdb) c
Continuing.
Error in testing breakpoint condition:
value has been optimized out

Breakpoint 9, check_nmi_watchdog () at arch/x86/kernel/apic/nmi.c:134
134		if (!nmi_watchdog_active() || !atomic_read(&nmi_active))
(gdb) n
native_smp_cpus_done (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1173
1173		mtrr_aps_init();
(gdb) s
mtrr_aps_init () at arch/x86/kernel/cpu/mtrr/main.c:779
779		if (!use_intel())
(gdb) n
792	}
(gdb) 
native_smp_cpus_done (max_cpus=<optimized out>)
    at arch/x86/kernel/smpboot.c:1174
1174	}
(gdb) 
```

* Initialize schedule variables and doamins, register callback function to response CPU come and go

```sched_init_smp
Breakpoint 3, sched_init_smp () at kernel/sched.c:9509
9509		get_online_cpus();                                              # increase reference
(gdb) s
get_online_cpus () at kernel/cpu.c:48
48		might_sleep();
(gdb) n
49		if (cpu_hotplug.active_writer == current)
(gdb) p cpu_hotplug
$3 = {active_writer = 0x0, lock = {count = {counter = 1}, wait_lock = {
      raw_lock = {slock = 2056}, magic = 3735899821, owner_cpu = 4294967295, 
      owner = 0xffffffff, dep_map = {key = 0x0, class_cache = 0x0, 
        name = 0xc15d41ad "cpu_hotplug.lock.wait_lock", cpu = 0, ip = 0}}, 
    wait_list = {next = 0xc1698eec <cpu_hotplug+44>, 
      prev = 0xc1698eec <cpu_hotplug+44>}, owner = 0x0, name = 0x0, 
    magic = 0xc1698ec4 <cpu_hotplug+4>, dep_map = {
      key = 0xc1698f00 <cpu_hotplug+64>, 
      class_cache = 0xc1b605e0 <lock_classes+155040>, 
      name = 0xc15d41c8 "cpu_hotplug.lock", cpu = 0, ip = 3238298482}}, 
  refcount = 0}
(gdb) n
51		mutex_lock(&cpu_hotplug.lock);
(gdb) 
52		cpu_hotplug.refcount++;
(gdb) 
53		mutex_unlock(&cpu_hotplug.lock);
(gdb) 
55	}
(gdb) 
sched_init_smp () at kernel/sched.c:9510
9510		mutex_lock(&sched_domains_mutex);
(gdb) 
9511		arch_init_sched_domains(cpu_active_mask);
(gdb) p *cpu_active_mask 
$5 = {bits = {1}}
(gdb) s
arch_init_sched_domains (cpu_map=<optimized out>) at kernel/sched.c:9220
9220		arch_update_cpu_topology();
(gdb) s
arch_update_cpu_topology () at kernel/sched.c:9209
9209	}
(gdb) n
arch_init_sched_domains (cpu_map=<optimized out>) at kernel/sched.c:9221
9221		ndoms_cur = 1;
(gdb) 
9222		doms_cur = kmalloc(cpumask_size(), GFP_KERNEL);
(gdb) 
9225		cpumask_andnot(doms_cur, cpu_map, cpu_isolated_map);
(gdb) s
cpumask_andnot (src2p=0xc18eb14c <cpu_isolated_map>, src1p=<optimized out>, 
    dstp=<optimized out>) at kernel/sched.c:9225
9225		cpumask_andnot(doms_cur, cpu_map, cpu_isolated_map);
(gdb) s
bitmap_andnot (src2=0xc18eb14c <cpu_isolated_map>, nbits=8, 
    src1=<optimized out>, dst=<optimized out>) at include/linux/bitmap.h:204
204			return (*dst = *src1 & ~(*src2)) != 0;
(gdb) n
arch_init_sched_domains (cpu_map=<optimized out>) at kernel/sched.c:9226
9226		dattr_cur = NULL;
(gdb) 
9227		err = build_sched_domains(doms_cur);
(gdb) s
build_sched_domains (cpu_map=0xc7020188) at kernel/sched.c:9186
9186		return __build_sched_domains(cpu_map, NULL);
(gdb) n
arch_init_sched_domains (cpu_map=<optimized out>) at kernel/sched.c:9228
9228		register_sched_domain_sysctl();
(gdb) s
register_sched_domain_sysctl () at kernel/sched.c:7825
7825		struct ctl_table *entry = sd_alloc_ctl_entry(cpu_num + 1);
(gdb) n
7828		WARN_ON(sd_ctl_dir[0].child);
(gdb) 
7829		sd_ctl_dir[0].child = entry;
(gdb) 
7831		if (entry == NULL)
(gdb) 
7834		for_each_possible_cpu(i) {
(gdb) 
7835			snprintf(buf, 32, "cpu%d", i);
(gdb) 
7836			entry->procname = kstrdup(buf, GFP_KERNEL);
(gdb) 
7837			entry->mode = 0555;
(gdb) 
7838			entry->child = sd_alloc_ctl_cpu_table(i);
(gdb) 
7839			entry++;
(gdb) 
7834		for_each_possible_cpu(i) {
(gdb) 
7842		WARN_ON(sd_sysctl_header);
(gdb) 
7843		sd_sysctl_header = register_sysctl_table(sd_ctl_root);
(gdb) 
7844	}
(gdb) 
sched_init_smp () at kernel/sched.c:9512
9512		cpumask_andnot(non_isolated_cpus, cpu_possible_mask, cpu_isolated_map);
(gdb) 
9513		if (cpumask_empty(non_isolated_cpus))
(gdb) p non_isolated_cpus 
$14 = {{bits = {3245306464}}}
(gdb) n
9515		mutex_unlock(&sched_domains_mutex);
(gdb) 
9516		put_online_cpus();                                                  # decrease reference
(gdb) s
put_online_cpus () at kernel/cpu.c:60
60		if (cpu_hotplug.active_writer == current)
(gdb) n
62		mutex_lock(&cpu_hotplug.lock);
(gdb) 
63		if (!--cpu_hotplug.refcount && unlikely(cpu_hotplug.active_writer))
(gdb) 
65		mutex_unlock(&cpu_hotplug.lock);
(gdb) 
67	}
(gdb) 
sched_init_smp () at kernel/sched.c:9524
9524		hotcpu_notifier(update_runtime, 0);                                 # RT runtime code needs to handle some hotplug events
(gdb) s
register_cpu_notifier (nb=nb@entry=0xc16de1f0 <update_runtime_nb>) 
    at kernel/cpu.c:131
131		cpu_maps_update_begin();
(gdb) n
132		ret = raw_notifier_chain_register(&cpu_chain, nb);
(gdb) 
133		cpu_maps_update_done();
(gdb) 
135	}
(gdb) 
sched_init_smp () at kernel/sched.c:9526
9526		init_hrtick();                                                      # high resolutuib time to handle hotplug events
(gdb) s
init_hrtick () at kernel/sched.c:1102
1102		hotcpu_notifier(hotplug_hrtick, 0);
(gdb) n
sched_init_smp () at kernel/sched.c:9529
9529		if (set_cpus_allowed_ptr(current, non_isolated_cpus) < 0)
(gdb) 
sched_init_smp () at kernel/sched.c:9531
9531		sched_init_granularity();                                           # increase the granularity value when there are more CPUs
(gdb) 
9534		init_sched_rt_class();                                              # allocate cpu mask
(gdb) 
9535	}
(gdb) 
kernel_init (unused=<optimized out>) at init/main.c:890
890		do_basic_setup();
(gdb) 
```

* Initialize some basic initialization

init_workqueues: initialize work queue for all up CPUs

cpuset_init_smp: intialize cpu and memory allowed, register notifier to response CPU hotplug, create work queue and work queue thread for cpuset

usermodehelper_init: create work queue and work queue thread for khelper

```do_basic_setup
kernel_init (unused=<optimized out>) at init/main.c:890
890		do_basic_setup();
(gdb) s
do_basic_setup () at init/main.c:783
783		init_workqueues();
(gdb) s
init_workqueues () at kernel/workqueue.c:1049
1049		cpumask_copy(cpu_populated_map, cpu_online_mask);
(gdb) p *cpu_online_mask 
$5 = {bits = {1}}
(gdb) n
1050		singlethread_cpu = cpumask_first(cpu_possible_mask);
(gdb) p *cpu_possible_mask
$6 = {bits = {1}}
(gdb) n
1051		cpu_singlethread_map = cpumask_of(singlethread_cpu);
(gdb) 
1052		hotcpu_notifier(workqueue_cpu_callback, 0);
(gdb) 
1053		keventd_wq = create_workqueue("events");
(gdb) p *cpu_singlethread_map
$10 = {bits = {1}}
(gdb) s
__create_workqueue_key (name=name@entry=0xc1619d0a "events", 
    singlethread=singlethread@entry=0, freezeable=freezeable@entry=0, 
    rt=rt@entry=0, key=key@entry=0xc190e014 <__key.22871>, 
    lock_name=lock_name@entry=0xc1619d0a "events") at kernel/workqueue.c:839
839		wq = kzalloc(sizeof(*wq), GFP_KERNEL);
(gdb) n
840		if (!wq)
(gdb) 
843		wq->cpu_wq = alloc_percpu(struct cpu_workqueue_struct);
(gdb) 
844		if (!wq->cpu_wq) {
(gdb) 
849		wq->name = name;
(gdb) 
850		lockdep_init_map(&wq->lockdep_map, lock_name, key, 0);
(gdb) s
lockdep_init_map (lock=lock@entry=0xc7001e1c, 
    name=name@entry=0xc1619d0a "events", 
    key=key@entry=0xc190e014 <__key.22871>, subclass=subclass@entry=0)
    at kernel/lockdep.c:2683
2683		if (DEBUG_LOCKS_WARN_ON(!name)) {
(gdb) n
2678		lock->class_cache = NULL;
(gdb) 
2680		lock->cpu = raw_smp_processor_id();
(gdb) 
2683		if (DEBUG_LOCKS_WARN_ON(!name)) {
(gdb) 
2688		lock->name = name;
(gdb) 
2690		if (DEBUG_LOCKS_WARN_ON(!key))
(gdb) 
2695		if (!static_obj(key)) {
(gdb) 
2700		lock->key = key;
(gdb) 
2702		if (unlikely(!debug_locks))
(gdb) 
2705		if (subclass)
(gdb) 
2707	}
(gdb) 
__create_workqueue_key (name=name@entry=0xc1619d0a "events", 
    singlethread=singlethread@entry=0, freezeable=freezeable@entry=0, 
    rt=rt@entry=0, key=key@entry=0xc190e014 <__key.22871>, 
    lock_name=lock_name@entry=0xc1619d0a "events") at kernel/workqueue.c:851
851		wq->singlethread = singlethread;
(gdb) 
852		wq->freezeable = freezeable;
(gdb) 
853		wq->rt = rt;
(gdb) 
854		INIT_LIST_HEAD(&wq->list);
(gdb) 
856		if (singlethread) {
(gdb) 
861			cpu_maps_update_begin();
(gdb) s
cpu_maps_update_begin () at kernel/cpu.c:78
78		mutex_lock(&cpu_add_remove_lock);
(gdb) n
79	}
(gdb) 
__create_workqueue_key (name=name@entry=0xc1619d0a "events", 
    singlethread=singlethread@entry=0, freezeable=freezeable@entry=0, 
    rt=rt@entry=0, key=key@entry=0xc190e014 <__key.22871>, 
    lock_name=lock_name@entry=0xc1619d0a "events") at kernel/workqueue.c:868
868			spin_lock(&workqueue_lock);
(gdb) 
869			list_add(&wq->list, &workqueues);
(gdb) 
870			spin_unlock(&workqueue_lock);
(gdb) 
877			for_each_possible_cpu(cpu) {
(gdb) 
878				cwq = init_cpu_workqueue(wq, cpu);
(gdb) 
879				if (err || !cpu_online(cpu))
(gdb) 
881				err = create_workqueue_thread(cwq, cpu);
(gdb) s
create_workqueue_thread (cwq=cwq@entry=0xc234a8c0, cpu=cpu@entry=0)
    at kernel/workqueue.c:792
792		struct sched_param param = { .sched_priority = MAX_RT_PRIO-1 };
(gdb) 
793		struct workqueue_struct *wq = cwq->wq;
(gdb) n
794		const char *fmt = is_wq_single_threaded(wq) ? "%s" : "%s/%d";
(gdb) 
797		p = kthread_create(worker_thread, cwq, fmt, wq->name, cpu);
(gdb) 
806		if (IS_ERR(p))
(gdb) 
808		if (cwq->wq->rt)
(gdb) 
812		trace_workqueue_creation(cwq->thread, cpu);
(gdb) 
810		cwq->thread = p;
(gdb) 
812		trace_workqueue_creation(cwq->thread, cpu);
(gdb) 
814		return 0;
(gdb) 
815	}
(gdb) 
__create_workqueue_key (name=name@entry=0xc1619d0a "events", 
    singlethread=singlethread@entry=0, freezeable=freezeable@entry=0, 
    rt=rt@entry=0, key=key@entry=0xc190e014 <__key.22871>, 
    lock_name=lock_name@entry=0xc1619d0a "events") at kernel/workqueue.c:882
882				start_workqueue_thread(cwq, cpu);
(gdb) s
start_workqueue_thread (cpu=cpu@entry=0, cwq=0xc234a8c0)
    at kernel/workqueue.c:821
821		if (p != NULL) {
(gdb) n
822			if (cpu >= 0)
(gdb) 
823				kthread_bind(p, cpu);
(gdb) 
824			wake_up_process(p);
(gdb) 
826	}
(gdb) 
__create_workqueue_key (name=name@entry=0xc1619d0a "events", 
    singlethread=singlethread@entry=0, freezeable=freezeable@entry=0, 
    rt=rt@entry=0, key=key@entry=0xc190e014 <__key.22871>, 
    lock_name=lock_name@entry=0xc1619d0a "events") at kernel/workqueue.c:877
877			for_each_possible_cpu(cpu) {
(gdb) n
884			cpu_maps_update_done();
(gdb) 
887		if (err) {
(gdb) 
892	}
(gdb) 
init_workqueues () at kernel/workqueue.c:1054
1054		BUG_ON(!keventd_wq);
(gdb) 
1055	}
(gdb) 
do_basic_setup () at init/main.c:784
784		cpuset_init_smp();
(gdb)  s
cpuset_init_smp () at kernel/cpuset.c:2127
2127		cpumask_copy(top_cpuset.cpus_allowed, cpu_active_mask);
(gdb) n
2128		top_cpuset.mems_allowed = node_states[N_HIGH_MEMORY];
(gdb) 
2130		hotcpu_notifier(cpuset_track_online_cpus, 0);
(gdb) 
2133		cpuset_wq = create_singlethread_workqueue("cpuset");
(gdb) 
2134		BUG_ON(!cpuset_wq);
2135	}
(gdb) 
do_basic_setup () at init/main.c:785
785		usermodehelper_init();
(gdb) s
usermodehelper_init () at kernel/kmod.c:640
640		khelper_wq = create_singlethread_workqueue("khelper");
(gdb) n
641		BUG_ON(!khelper_wq);
(gdb) 
642	}
```

init_tmpfs: initialize tmp file system

super block -> inode　-> dentry

```init_tmpfs
786		init_tmpfs();
(gdb) s
init_tmpfs () at mm/shmem.c:2533
2533		error = bdi_init(&shmem_backing_dev_info);                              # initialize block device information
(gdb) n
2534		if (error)
(gdb) 
2537		error = init_inodecache();                                              # create shmem_inode_cache
(gdb) s
init_inodecache () at mm/shmem.c:2410
2410		shmem_inode_cachep = kmem_cache_create("shmem_inode_cache",
(gdb) n
init_tmpfs () at mm/shmem.c:2541
2541		error = register_filesystem(&tmpfs_fs_type);                            # register tmpfs_fs_type
(gdb) 
2542		if (error) {
(gdb) 
2547		shm_mnt = vfs_kern_mount(&tmpfs_fs_type, MS_NOUSER,
(gdb) s
vfs_kern_mount (type=type@entry=0xc16a4560 <tmpfs_fs_type>, 
    flags=flags@entry=-2147483648, name=0xc1620c4e "tmpfs", 
    data=data@entry=0x0) at fs/super.c:920
920		if (!type)
(gdb) n
924		mnt = alloc_vfsmnt(name);
(gdb) s
alloc_vfsmnt (name=name@entry=0xc1620c4e "tmpfs") at fs/namespace.c:130
130		struct vfsmount *mnt = kmem_cache_zalloc(mnt_cache, GFP_KERNEL);
(gdb) n
131		if (mnt) {
(gdb) 
134			err = mnt_alloc_id(mnt);
(gdb) s
mnt_alloc_id (mnt=<optimized out>) at fs/namespace.c:73
73		ida_pre_get(&mnt_id_ida, GFP_KERNEL);
(gdb) s
ida_pre_get (ida=ida@entry=0xc16a98c0 <mnt_id_ida>, 
    gfp_mask=gfp_mask@entry=208) at lib/idr.c:737
737		if (!idr_pre_get(&ida->idr, gfp_mask))
(gdb) s
idr_pre_get (idp=idp@entry=0xc16a98c0 <mnt_id_ida>, 
    gfp_mask=gfp_mask@entry=208) at lib/idr.c:122
122		while (idp->id_free_cnt < IDR_FREE_MAX) {
(gdb) 
124			new = kmem_cache_zalloc(idr_layer_cache, gfp_mask);
(gdb) 
125			if (new == NULL)
(gdb) 
127			move_to_free_list(idp, new);
(gdb) s
move_to_free_list (p=0xc7024300, idp=0xc16a98c0 <mnt_id_ida>) at lib/idr.c:83
83		spin_lock_irqsave(&idp->lock, flags);
(gdb) n
84		__move_to_free_list(idp, p);
(gdb) s
__move_to_free_list (idp=<optimized out>, idp=<optimized out>, p=0xc7024300)
    at lib/idr.c:71
71		p->ary[0] = idp->id_free;
(gdb) n
72		idp->id_free = p;
(gdb) 
73		idp->id_free_cnt++;
(gdb) 
move_to_free_list (p=0xc7024300, idp=0xc16a98c0 <mnt_id_ida>) at lib/idr.c:85
85		spin_unlock_irqrestore(&idp->lock, flags);
(gdb) 
idr_pre_get (idp=idp@entry=0xc16a98c0 <mnt_id_ida>, 
    gfp_mask=gfp_mask@entry=208) at lib/idr.c:122
122		while (idp->id_free_cnt < IDR_FREE_MAX) {
(gdb) p idp->id_free_cnt 
$1 = 14
(gdb) n
129		return 1;
(gdb) 
130	}
(gdb) 
ida_pre_get (ida=ida@entry=0xc16a98c0 <mnt_id_ida>, 
    gfp_mask=gfp_mask@entry=208) at lib/idr.c:741
741		if (!ida->free_bitmap) {
(gdb) 
751		return 1;
(gdb) 
752	}
(gdb) 
mnt_alloc_id (mnt=<optimized out>) at fs/namespace.c:74
74		spin_lock(&vfsmount_lock);
(gdb) 
75		res = ida_get_new_above(&mnt_id_ida, mnt_id_start, &mnt->mnt_id);
(gdb) s
ida_get_new_above (ida=ida@entry=0xc16a98c0 <mnt_id_ida>, starting_id=4, 
    p_id=p_id@entry=0xc701d360) at lib/idr.c:775
775		int idr_id = starting_id / IDA_BITMAP_BITS;
(gdb) n
776		int offset = starting_id % IDA_BITMAP_BITS;
(gdb) 
781		t = idr_get_empty_slot(&ida->idr, idr_id, pa);
(gdb) s
idr_get_empty_slot (idp=idp@entry=0xc16a98c0 <mnt_id_ida>, 
    starting_id=starting_id@entry=0, pa=pa@entry=0xc706bf38) at lib/idr.c:204
204	{
(gdb) n
211		p = idp->top;
(gdb) 
213		if (unlikely(!p)) {
(gdb) 
223		while ((layers < (MAX_LEVEL - 1)) && (id >= (1 << (layers*IDR_BITS)))) {
(gdb) p *idp
$3 = {top = 0xc701e9c0, id_free = 0xc7024300, layers = 1, id_free_cnt = 14, 
  lock = {raw_lock = {slock = 6939}, magic = 3735899821, 
    owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
      key = 0xc16a98e0 <mnt_id_ida+32>, 
      class_cache = 0xc1b5d070 <lock_classes+141360>, 
      name = 0xc15ef581 "mnt_id_ida.lock", cpu = 0, ip = 3240328551}}}
(gdb) n
255		rcu_assign_pointer(idp->top, p);
(gdb) 
256		idp->layers = layers;
(gdb) 
257		v = sub_alloc(idp, &id, pa);
(gdb) s
sub_alloc (pa=0xc706bf38, starting_id=<synthetic pointer>, 
    idp=0xc16a98c0 <mnt_id_ida>) at lib/idr.c:144
144		pa[l--] = NULL;
(gdb) n
149			n = (id >> (IDR_BITS*l)) & IDR_MASK;
(gdb) p p
$4 = (struct idr_layer *) 0xc701e9c0
(gdb) p l
$5 = 0
(gdb) n
149			n = (id >> (IDR_BITS*l)) & IDR_MASK;
(gdb) 
150			bm = ~p->bitmap;
(gdb) 
151			m = find_next_bit(&bm, IDR_SIZE, n);
(gdb) p /t p->bitmap
$6 = 0
(gdb) n
152			if (m == IDR_SIZE) {
(gdb) p m
$7 = 0
(gdb) n
175			if (m != n) {
(gdb) 
179			if ((id >= MAX_ID_BIT) || (id < 0))
(gdb) 
181			if (l == 0)
(gdb) 
idr_get_empty_slot (idp=idp@entry=0xc16a98c0 <mnt_id_ida>, 
    starting_id=starting_id@entry=0, pa=pa@entry=0xc706bf38) at lib/idr.c:257
257		v = sub_alloc(idp, &id, pa);
(gdb) 
261	}
(gdb) p pa[0]
$8 = (struct idr_layer *) 0xc701e9c0
(gdb) n
ida_get_new_above (ida=ida@entry=0xc16a98c0 <mnt_id_ida>, 
    starting_id=<optimized out>, p_id=p_id@entry=0xc701d360) at lib/idr.c:782
782		if (t < 0)
(gdb) 
785		if (t * IDA_BITMAP_BITS >= MAX_ID_BIT)
(gdb) 
789			offset = 0;
(gdb) 
793		bitmap = (void *)pa[0]->ary[idr_id & IDR_MASK];
(gdb) 
794		if (!bitmap) {
(gdb) 
810		t = find_next_zero_bit(bitmap->bitmap, IDA_BITMAP_BITS, offset);
(gdb) p *bitmap
$10 = {nr_busy = 4, bitmap = {15, 0 <repeats 30 times>}}
(gdb) n
811		if (t == IDA_BITMAP_BITS) {
(gdb) p t
$11 = 4
(gdb) n
819		if (id >= MAX_ID_BIT)
(gdb) 
822		__set_bit(t, bitmap->bitmap);
(gdb) p bitmap->bitmap 
$12 = {15, 0 <repeats 30 times>}
(gdb) n
823		if (++bitmap->nr_busy == IDA_BITMAP_BITS)
(gdb) p bitmap->bitmap 
$13 = {31, 0 <repeats 30 times>}
(gdb) n
826		*p_id = id;
(gdb) 
833		if (ida->idr.id_free_cnt || ida->free_bitmap) {
(gdb) 
834			struct idr_layer *p = get_from_free_list(&ida->idr);
(gdb) 
835			if (p)
(gdb) 
836				kmem_cache_free(idr_layer_cache, p);
(gdb) 
840	}
(gdb) p p_id
$14 = (int *) 0xc701d360
(gdb) p *p_id
$15 = 4
(gdb) n
839		return 0;
(gdb) 
840	}
(gdb) 
mnt_alloc_id (mnt=<optimized out>) at fs/namespace.c:76
76		if (!res)
(gdb) 
77			mnt_id_start = mnt->mnt_id + 1;
(gdb) 
78		spin_unlock(&vfsmount_lock);
(gdb) 
alloc_vfsmnt (name=name@entry=0xc1620c4e "tmpfs") at fs/namespace.c:138
138			if (name) {
(gdb) 
139				mnt->mnt_devname = kstrdup(name, GFP_KERNEL);
(gdb) 
140				if (!mnt->mnt_devname)
(gdb) 
144			atomic_set(&mnt->mnt_count, 1);
(gdb) 
145			INIT_LIST_HEAD(&mnt->mnt_hash);
(gdb) 
146			INIT_LIST_HEAD(&mnt->mnt_child);
(gdb) 
147			INIT_LIST_HEAD(&mnt->mnt_mounts);
(gdb) 
148			INIT_LIST_HEAD(&mnt->mnt_list);
(gdb) 
149			INIT_LIST_HEAD(&mnt->mnt_expire);
(gdb) 
150			INIT_LIST_HEAD(&mnt->mnt_share);
(gdb) 
151			INIT_LIST_HEAD(&mnt->mnt_slave_list);
(gdb) 
152			INIT_LIST_HEAD(&mnt->mnt_slave);
(gdb) 
154			mnt->mnt_writers = alloc_percpu(int);
(gdb) 
155			if (!mnt->mnt_writers)
(gdb) 
172	}
(gdb) 
vfs_kern_mount (type=type@entry=0xc16a4560 <tmpfs_fs_type>, 
    flags=flags@entry=-2147483648, name=0xc1620c4e "tmpfs", 
    data=data@entry=0x0) at fs/super.c:925
925		if (!mnt)
(gdb) 
928		if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {
(gdb) 
938		error = type->get_sb(type, flags, name, data, mnt);
(gdb) s
shmem_get_sb (fs_type=0xc16a4560 <tmpfs_fs_type>, flags=-2147483648, 
    dev_name=0xc1620c4e "tmpfs", data=0x0, mnt=0xc701d300) at mm/shmem.c:2519
2519		return get_sb_nodev(fs_type, flags, data, shmem_fill_super, mnt);
(gdb) s
get_sb_nodev (fs_type=0xc16a4560 <tmpfs_fs_type>, flags=-2147483648, 
    data=data@entry=0x0, fill_super=0xc10e49b0 <shmem_fill_super>, 
    mnt=mnt@entry=0xc701d300) at fs/super.c:859
859	{
(gdb) n
861		struct super_block *s = sget(fs_type, NULL, set_anon_super, NULL);
(gdb) 
863		if (IS_ERR(s))
(gdb) 
866		s->s_flags = flags;
(gdb) 
868		error = fill_super(s, data, flags & MS_SILENT ? 1 : 0);
(gdb) s
shmem_fill_super (sb=0xc7009920, data=0x0, silent=0) at mm/shmem.c:2319
2319		sbinfo = kzalloc(max((int)sizeof(struct shmem_sb_info),
(gdb) 
2321		if (!sbinfo)
(gdb) 
2324		sbinfo->mode = S_IRWXUGO | S_ISVTX;
(gdb) 
2325		sbinfo->uid = current_fsuid();
(gdb) 
2326		sbinfo->gid = current_fsgid();
(gdb) 
2327		sb->s_fs_info = sbinfo;
(gdb) 
2335		if (!(sb->s_flags & MS_NOUSER)) {
(gdb) 
2343		sb->s_export_op = &shmem_export_ops;
(gdb) 
2348		spin_lock_init(&sbinfo->stat_lock);
(gdb) 
2349		sbinfo->free_blocks = sbinfo->max_blocks;
(gdb) 
2350		sbinfo->free_inodes = sbinfo->max_inodes;
(gdb) 
2352		sb->s_maxbytes = SHMEM_MAX_BYTES;
(gdb) 
2353		sb->s_blocksize = PAGE_CACHE_SIZE;
(gdb) 
2354		sb->s_blocksize_bits = PAGE_CACHE_SHIFT;
(gdb) 
2355		sb->s_magic = TMPFS_MAGIC;
(gdb) 
2356		sb->s_op = &shmem_ops;
(gdb) 
2357		sb->s_time_gran = 1;
(gdb) 
2359		sb->s_xattr = shmem_xattr_handlers;
(gdb) 
2360		sb->s_flags |= MS_POSIXACL;
(gdb) 
2363		inode = shmem_get_inode(sb, S_IFDIR | sbinfo->mode, 0, VM_NORESERVE);
(gdb) s
shmem_get_inode (sb=sb@entry=0xc7009920, mode=17407, dev=dev@entry=0, 
    flags=flags@entry=2097152) at mm/shmem.c:1548
1548		if (shmem_reserve_inode(sb))
(gdb) s
shmem_reserve_inode (sb=sb@entry=0xc7009920) at mm/shmem.c:245
245		struct shmem_sb_info *sbinfo = SHMEM_SB(sb);
(gdb) n
246		if (sbinfo->max_inodes) {
(gdb) 
255		return 0;
(gdb) 
256	}
(gdb) 
shmem_get_inode (sb=sb@entry=0xc7009920, mode=17407, dev=dev@entry=0, 
    flags=flags@entry=2097152) at mm/shmem.c:1551
1551		inode = new_inode(sb);
(gdb) s
new_inode (sb=sb@entry=0xc7009920) at fs/inode.c:677
677		spin_lock_prefetch(&inode_lock);
(gdb) n
679		inode = alloc_inode(sb);
(gdb) s
alloc_inode (sb=sb@entry=0xc7009920) at fs/inode.c:213
213	{
(gdb) n
216		if (sb->s_op->alloc_inode)
(gdb) 
217			inode = sb->s_op->alloc_inode(sb);
(gdb) s
shmem_alloc_inode (sb=0xc7009920) at mm/shmem.c:2386
2386		p = (struct shmem_inode_info *)kmem_cache_alloc(shmem_inode_cachep, GFP_KERNEL);
(gdb) n
2387		if (!p)
(gdb) 
2389		return &p->vfs_inode;
(gdb) 
2390	}
(gdb) 
alloc_inode (sb=sb@entry=0xc7009920) at fs/inode.c:221
221		if (!inode)
(gdb) 
224		if (unlikely(inode_init_always(sb, inode))) {
(gdb) 
233	}
(gdb) 
new_inode (sb=sb@entry=0xc7009920) at fs/inode.c:680
680		if (inode) {
(gdb) 
679		inode = alloc_inode(sb);
(gdb) 
680		if (inode) {
(gdb) 
681			spin_lock(&inode_lock);
(gdb) 
682			__inode_add_to_lists(sb, NULL, inode);
(gdb) 
683			inode->i_ino = ++last_ino;
(gdb) 
684			inode->i_state = 0;
(gdb) 
685			spin_unlock(&inode_lock);
(gdb) 
688	}
(gdb) 
shmem_get_inode (sb=sb@entry=0xc7009920, mode=17407, dev=dev@entry=0, 
    flags=flags@entry=2097152) at mm/shmem.c:1552
1552		if (inode) {
(gdb) 
1553			inode->i_mode = mode;
(gdb) 
1554			inode->i_uid = current_fsuid();
(gdb) 
1555			inode->i_gid = current_fsgid();
(gdb) 
1556			inode->i_blocks = 0;
(gdb) 
1557			inode->i_mapping->backing_dev_info = &shmem_backing_dev_info;
(gdb) 
1558			inode->i_atime = inode->i_mtime = inode->i_ctime = CURRENT_TIME;
(gdb) 
1559			inode->i_generation = get_seconds();
(gdb) 
1560			info = SHMEM_I(inode);
(gdb) 
1561			memset(info, 0, (char *)inode - (char *)info);
(gdb) 
1562			spin_lock_init(&info->lock);
(gdb) 
1563			info->flags = flags & VM_NORESERVE;
(gdb) 
1564			INIT_LIST_HEAD(&info->swaplist);
(gdb) 
1565			cache_no_acl(inode);
(gdb) 
1567			switch (mode & S_IFMT) {
(gdb) 
1580				inc_nlink(inode);
(gdb) 
1582				inode->i_size = 2 * BOGO_DIRENT_SIZE;
(gdb) 
1583				inode->i_op = &shmem_dir_inode_operations;                      # operations of inode
(gdb) 
1584				inode->i_fop = &simple_dir_operations;
(gdb) 
1597	}
(gdb) 
shmem_fill_super (sb=0xc7009920, data=0x0, silent=<optimized out>)
    at mm/shmem.c:2364
2364		if (!inode)
(gdb) 
2366		inode->i_uid = sbinfo->uid;
(gdb) 
2367		inode->i_gid = sbinfo->gid;
(gdb) 
2368		root = d_alloc_root(inode);
(gdb) 
(gdb) s
d_alloc_root (root_inode=root_inode@entry=0xc709e080) at fs/dcache.c:1098
1098		if (root_inode) {
(gdb) n
1101			res = d_alloc(NULL, &name);
(gdb) s
d_alloc (parent=parent@entry=0x0, name=name@entry=0xc149fbb8 <name>)
    at fs/dcache.c:922
922		dentry = kmem_cache_alloc(dentry_cache, GFP_KERNEL);
(gdb) 
923		if (!dentry)
(gdb) 
926		if (name->len > DNAME_INLINE_LEN-1) {
(gdb) 
933			dname = dentry->d_iname;
(gdb) 
935		dentry->d_name.name = dname;
(gdb) 
937		dentry->d_name.len = name->len;
(gdb) 
938		dentry->d_name.hash = name->hash;
(gdb) 
939		memcpy(dname, name->name, name->len);
(gdb) 
940		dname[name->len] = 0;
(gdb) 
942		atomic_set(&dentry->d_count, 1);
(gdb) 
943		dentry->d_flags = DCACHE_UNHASHED;
(gdb) 
944		spin_lock_init(&dentry->d_lock);
(gdb) 
945		dentry->d_inode = NULL;
(gdb) 
946		dentry->d_parent = NULL;
(gdb) 
947		dentry->d_sb = NULL;
(gdb) 
948		dentry->d_op = NULL;
(gdb) 
949		dentry->d_fsdata = NULL;
(gdb) 
950		dentry->d_mounted = 0;
(gdb) 
951		INIT_HLIST_NODE(&dentry->d_hash);
(gdb) 
952		INIT_LIST_HEAD(&dentry->d_lru);
(gdb) 
953		INIT_LIST_HEAD(&dentry->d_subdirs);
(gdb) 
954		INIT_LIST_HEAD(&dentry->d_alias);
(gdb) 
956		if (parent) {
(gdb) 
960			INIT_LIST_HEAD(&dentry->d_u.d_child);
(gdb) 
963		spin_lock(&dcache_lock);
(gdb) 
966		dentry_stat.nr_dentry++;
(gdb) 
967		spin_unlock(&dcache_lock);
(gdb) 
969		return dentry;
(gdb) 
970	}
(gdb) 
d_alloc_root (root_inode=root_inode@entry=0xc709e080) at fs/dcache.c:1102
1102			if (res) {
(gdb) 
1103				res->d_sb = root_inode->i_sb;
(gdb) 
1104				res->d_parent = res;
(gdb) 
1105				d_instantiate(res, root_inode);
(gdb) s
d_instantiate (entry=0xc6c02340, inode=0xc709e080) at fs/dcache.c:1007
1007	{
(gdb) n
1008		BUG_ON(!list_empty(&entry->d_alias));
(gdb) 
1009		spin_lock(&dcache_lock);
(gdb) 
1010		__d_instantiate(entry, inode);
(gdb) s
__d_instantiate (dentry=dentry@entry=0xc6c02340, inode=inode@entry=0xc709e080)
    at fs/dcache.c:985
985		if (inode)
(gdb) n
986			list_add(&dentry->d_alias, &inode->i_dentry);
(gdb) 
987		dentry->d_inode = inode;
(gdb) 
988		fsnotify_d_instantiate(dentry, inode);
(gdb) s
fsnotify_d_instantiate (inode=0xc709e080, entry=0xc6c02340) at fs/dcache.c:988
988		fsnotify_d_instantiate(dentry, inode);
(gdb) s
__fsnotify_d_instantiate (inode=0xc709e080, dentry=0xc6c02340)
    at fs/dcache.c:988
988		fsnotify_d_instantiate(dentry, inode);
(gdb) n
989	}
(gdb) 
d_instantiate (entry=0xc6c02340, inode=0xc709e080) at fs/dcache.c:1011
1011		spin_unlock(&dcache_lock);
(gdb) 
1012		security_d_instantiate(entry, inode);
(gdb) 
1013	}
(gdb) 
d_alloc_root (root_inode=root_inode@entry=0xc709e080) at fs/dcache.c:1109
1109	}
(gdb) 
shmem_fill_super (sb=0xc7009920, data=<optimized out>, silent=<optimized out>)
    at mm/shmem.c:2369
2369		if (!root)
(gdb) 
2371		sb->s_root = root;
(gdb) p root
$2 = (struct dentry *) 0xc6c02340
(gdb) p *root
$3 = {d_count = {counter = 1}, d_flags = 16, d_lock = {raw_lock = {
      slock = 514}, magic = 3735899821, owner_cpu = 4294967295, 
    owner = 0xffffffff, dep_map = {key = 0xc1e6248c <__key.26347>, 
      class_cache = 0x0, name = 0xc15ef0b2 "&dentry->d_lock", cpu = 0, 
      ip = 3239351818}}, d_mounted = 0, d_inode = 0xc709e080, d_hash = {
    next = 0x0, pprev = 0x0}, d_parent = 0xc6c02340, d_name = {hash = 0, 
    len = 1, name = 0xc6c023bc "/"}, d_lru = {next = 0xc6c0238c, 
    prev = 0xc6c0238c}, d_u = {d_child = {next = 0xc6c02394, 
      prev = 0xc6c02394}, d_rcu = {next = 0xc6c02394, func = 0xc6c02394}}, 
  d_subdirs = {next = 0xc6c0239c, prev = 0xc6c0239c}, d_alias = {
    next = 0xc709e098, prev = 0xc709e098}, d_time = 1802201963, d_op = 0x0, 
  d_sb = 0xc7009920, d_fsdata = 0x0, 
  d_iname = "/\000", 'k' <repeats 37 times>, "\245"}
(gdb) n
2372		return 0;
(gdb) 
2379	}
(gdb) 
get_sb_nodev (fs_type=<optimized out>, flags=<optimized out>, 
    data=data@entry=0x0, fill_super=0xc10e49b0 <shmem_fill_super>, 
    mnt=mnt@entry=0xc701d300) at fs/super.c:869
869		if (error) {
(gdb) 
873		s->s_flags |= MS_ACTIVE;
(gdb) n
874		simple_set_mnt(mnt, s);
(gdb) s
simple_set_mnt (mnt=mnt@entry=0xc701d300, sb=sb@entry=0xc7009920)
    at fs/namespace.c:394
394		mnt->mnt_sb = sb;
(gdb) n
395		mnt->mnt_root = dget(sb->s_root);
(gdb) 
396	}
(gdb) 
get_sb_nodev (fs_type=<optimized out>, flags=<optimized out>, 
    data=data@entry=0x0, fill_super=0xc10e49b0 <shmem_fill_super>, 
    mnt=mnt@entry=0xc701d300) at fs/super.c:875
875		return 0;
(gdb) 
876	}
(gdb) 
shmem_get_sb (fs_type=<optimized out>, flags=<optimized out>, 
    dev_name=<optimized out>, data=0x0, mnt=0xc701d300) at mm/shmem.c:2520
2520	}
(gdb) 
vfs_kern_mount (type=type@entry=0xc16a4560 <tmpfs_fs_type>, 
    flags=flags@entry=-2147483648, name=<optimized out>, data=data@entry=0x0)
    at fs/super.c:939
939		if (error < 0)
(gdb) 
941		BUG_ON(!mnt->mnt_sb);
(gdb) 
943	 	error = security_sb_kern_mount(mnt->mnt_sb, flags, secdata);
(gdb) s
security_sb_kern_mount (sb=0xc7009920, flags=flags@entry=-2147483648, 
    data=data@entry=0x0) at security/security.c:273
273		return security_ops->sb_kern_mount(sb, flags, data);
(gdb) s
cap_sb_kern_mount (sb=0xc7009920, flags=-2147483648, data=0x0)
    at security/capability.c:65
65	}
(gdb) n
security_sb_kern_mount (sb=<optimized out>, flags=flags@entry=-2147483648, 
    data=data@entry=0x0) at security/security.c:274
274	}
(gdb) 
vfs_kern_mount (type=type@entry=0xc16a4560 <tmpfs_fs_type>, 
    flags=flags@entry=-2147483648, name=<optimized out>, data=data@entry=0x0)
    at fs/super.c:944
944	 	if (error)
(gdb) 
954		WARN((mnt->mnt_sb->s_maxbytes < 0), "%s set sb->s_maxbytes to "
(gdb) 
957		mnt->mnt_mountpoint = mnt->mnt_root;
(gdb) 
958		mnt->mnt_parent = mnt;
(gdb) 
959		up_write(&mnt->mnt_sb->s_umount);
(gdb) 
960		free_secdata(secdata);
(gdb) 
961		return mnt;
(gdb) 
971	}
(gdb) 
init_tmpfs () at mm/shmem.c:2549
2549		if (IS_ERR(shm_mnt)) {
(gdb) 
2554		return 0;
(gdb) 
2565	}
(gdb) 
do_basic_setup () at init/main.c:787
787		driver_init();
```

* initialize driver model

`devtmpfs_init`: initialize dev tmp file system

`devices_init`: initialize devices, it involke `kset_create_and_add` to create kset data. struct kset - a set of kobjects of a specific type, belonging to a specific subsystem. A kset defines a group of kobjects.  They can be individually different "types" but overall these kobjects all want to be grouped together and operated on in the same manner.  ksets are used to define the attribute callbacks and other common events that happen to a kobject.

```driver_init
do_basic_setup () at init/main.c:787
787		driver_init();
(gdb) s
driver_init () at drivers/base/init.c:23
23		devtmpfs_init();                                                                            # initialize dev tmp filesystem
(gdb) s
devtmpfs_init () at drivers/base/devtmpfs.c:369
369		char options[] = "mode=0755";
(gdb) n
371		err = register_filesystem(&dev_fs_type);
(gdb) 
372		if (err) {
(gdb) 
378		mnt = kern_mount_data(&dev_fs_type, options);
(gdb) s
kern_mount_data (type=type@entry=0xc16c7da0 <dev_fs_type>, 
    data=data@entry=0xc706bfa6) at fs/super.c:1016
1016		return vfs_kern_mount(type, MS_KERNMOUNT, type->name, data);                            # kern_mount_data invoke vfs_kern_mount, reference tmp file system
(gdb) n
1017	}
(gdb) 
devtmpfs_init () at drivers/base/devtmpfs.c:379
379		if (IS_ERR(mnt)) {
(gdb) 
387		printk(KERN_INFO "devtmpfs: initialized\n");
(gdb) p mnt
$1 = (struct vfsmount *) 0xc701d3c0
(gdb) p *mnt
$2 = {mnt_hash = {next = 0xc701d3c0, prev = 0xc701d3c0}, 
  mnt_parent = 0xc701d3c0, mnt_mountpoint = 0xc6c02410, mnt_root = 0xc6c02410, 
  mnt_sb = 0xc7009d50, mnt_mounts = {next = 0xc701d3d8, prev = 0xc701d3d8}, 
  mnt_child = {next = 0xc701d3e0, prev = 0xc701d3e0}, mnt_flags = 0, 
  mnt_devname = 0xc7000700 "devtmpfs", mnt_list = {next = 0xc701d3f0, 
    prev = 0xc701d3f0}, mnt_expire = {next = 0xc701d3f8, prev = 0xc701d3f8}, 
  mnt_share = {next = 0xc701d400, prev = 0xc701d400}, mnt_slave_list = {
    next = 0xc701d408, prev = 0xc701d408}, mnt_slave = {next = 0xc701d410, 
    prev = 0xc701d410}, mnt_master = 0x0, mnt_ns = 0x0, mnt_id = 5, 
  mnt_group_id = 0, mnt_count = {counter = 1}, mnt_expiry_mark = 0, 
  mnt_pinned = 0, mnt_ghosts = 0, mnt_writers = 0xc18d3a50}
(gdb) n
385		dev_mnt = mnt;
(gdb) 
387		printk(KERN_INFO "devtmpfs: initialized\n");
(gdb) 
388		return 0;
(gdb) 
389	}
(gdb) 
driver_init () at drivers/base/init.c:24
24		devices_init();
(gdb) s
devices_init () at drivers/base/core.c:1275
1275		devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);
(gdb) break kset_create_and_add 
Breakpoint 4 at 0xc1239840: file lib/kobject.c, line 837.
(gdb) c
Continuing.

Breakpoint 4, kset_create_and_add (name=name@entry=0xc15d7009 "devices", 
    uevent_ops=uevent_ops@entry=0xc16c78f8 <device_uevent_ops>, 
    parent_kobj=parent_kobj@entry=0x0) at lib/kobject.c:837
841		kset = kset_create(name, uevent_ops, parent_kobj);                              # allocate memory for kset and initialize it
(gdb) n
844		error = kset_register(kset);
(gdb) s
kset_register (k=k@entry=0xc70851b0) at lib/kobject.c:716
716		if (!k)
(gdb) n
719		kset_init(k);
(gdb) s
kset_init (k=k@entry=0xc70851b0) at lib/kobject.c:672
672	{
(gdb) n
673		kobject_init_internal(&k->kobj);
(gdb) s
kobject_init_internal (kobj=0xc70851dc) at lib/kobject.c:147
147		if (!kobj)
(gdb) n
149		kref_init(&kobj->kref);
(gdb) 
150		INIT_LIST_HEAD(&kobj->entry);
(gdb) 
151		kobj->state_in_sysfs = 0;
(gdb) 
153		kobj->state_remove_uevent_sent = 0;
(gdb) 
154		kobj->state_initialized = 1;
(gdb) 
kset_init (k=k@entry=0xc70851b0) at lib/kobject.c:674
674		INIT_LIST_HEAD(&k->list);
(gdb) 
675		spin_lock_init(&k->list_lock);
(gdb) 
676	}
(gdb) 
kset_register (k=k@entry=0xc70851b0) at lib/kobject.c:720
720		err = kobject_add_internal(&k->kobj);
(gdb) s
kobject_add_internal (kobj=0xc70851dc) at lib/kobject.c:163
163		if (!kobj)
(gdb) n
166		if (!kobj->name || !kobj->name[0]) {
(gdb) p *kobj
$1 = {name = 0xc7020230 "devices", entry = {next = 0xc70851e0, 
    prev = 0xc70851e0}, parent = 0x0, kset = 0x0, 
  ktype = 0xc16bb100 <kset_ktype>, sd = 0x0, kref = {refcount = {
      counter = 1}}, state_initialized = 1, state_in_sysfs = 0, 
  state_add_uevent_sent = 0, state_remove_uevent_sent = 0, uevent_suppress = 0}
(gdb) n
172		parent = kobject_get(kobj->parent);
(gdb) s
kobject_get (kobj=0x0) at lib/kobject.c:529
529		if (kobj)
(gdb) n
kobject_add_internal (kobj=0xc70851dc) at lib/kobject.c:175
175		if (kobj->kset) {
(gdb) 
182		pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
(gdb) 
187		error = create_dir(kobj);
(gdb) s
create_dir (kobj=0xc70851dc) at lib/kobject.c:50
50		if (kobject_name(kobj)) {
(gdb) s
51			error = sysfs_create_dir(kobj);
(gdb) s
sysfs_create_dir (kobj=kobj@entry=0xc70851dc) at fs/sysfs/dir.c:715
715		BUG_ON(!kobj);
(gdb) n
717		if (kobj->parent)
(gdb) 
720			parent_sd = &sysfs_root;
(gdb) 
722		error = create_dir(kobj, parent_sd, kobject_name(kobj), &sd);
(gdb) s
create_dir (kobj=kobj@entry=0xc70851dc, parent_sd=0xc16ac580 <sysfs_root>, 
    name=0xc7020230 "devices", p_sd=p_sd@entry=0xc706bf48)
    at fs/sysfs/dir.c:675
675	{
(gdb) n
682		sd = sysfs_new_dirent(name, mode, SYSFS_DIR);
(gdb) s
sysfs_new_dirent (name=0xc7020230 "devices", mode=mode@entry=16877, 
    type=type@entry=1) at fs/sysfs/dir.c:318
318		if (type & SYSFS_COPY_NAME) {
(gdb) 
319			name = dup_name = kstrdup(name, GFP_KERNEL);
(gdb) 
320			if (!name)
(gdb) 
324		sd = kmem_cache_zalloc(sysfs_dir_cachep, GFP_KERNEL);
(gdb) 
325		if (!sd)
(gdb) 
328		if (sysfs_alloc_ino(&sd->s_ino))
(gdb) s
sysfs_alloc_ino (pino=<optimized out>) at fs/sysfs/dir.c:264
264		*pino = ino;
(gdb) p ino
$2 = 3
(gdb) n
sysfs_new_dirent (name=0xc7020268 "devices", mode=mode@entry=16877, 
    type=type@entry=1) at fs/sysfs/dir.c:331
331		atomic_set(&sd->s_count, 1);
(gdb) 
332		atomic_set(&sd->s_active, 0);
(gdb) 
334		sd->s_name = name;
(gdb) 
335		sd->s_mode = mode;
(gdb) 
336		sd->s_flags = type;
(gdb) 
338		return sd;
(gdb) 
345	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70851dc, parent_sd=0xc16ac580 <sysfs_root>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf48) at fs/sysfs/dir.c:683
683		if (!sd)
(gdb) 
685		sd->s_dir.kobj = kobj;
(gdb) 
688		sysfs_addrm_start(&acxt, parent_sd);
(gdb) s
sysfs_addrm_start (acxt=acxt@entry=0xc706bf20, 
    parent_sd=parent_sd@entry=0xc16ac580 <sysfs_root>) at fs/sysfs/dir.c:374
374		memset(acxt, 0, sizeof(*acxt));
(gdb) n
375		acxt->parent_sd = parent_sd;
(gdb) 
381		mutex_lock(&sysfs_mutex);
(gdb) 
383		inode = ilookup5(sysfs_sb, parent_sd->s_ino, sysfs_ilookup_test,
(gdb) s
ilookup5 (sb=0xc7008430, hashval=1, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>) at fs/inode.c:1001
1001		struct hlist_head *head = inode_hashtable + hash(sb, hashval);
(gdb) n
1003		return ifind(sb, head, test, data, 1);
(gdb) s
ifind (sb=0xc7008430, head=0xc213e004, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>, wait=wait@entry=1)
    at fs/inode.c:904
904		spin_lock(&inode_lock);
(gdb) n
905		inode = find_inode(sb, head, test, data);
(gdb) s
find_inode (sb=sb@entry=0xc7008430, head=head@entry=0xc213e004, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>) at fs/inode.c:573
573		hlist_for_each_entry(inode, node, head, i_hash) {
(gdb) n
574			if (inode->i_sb != sb)
(gdb) p inode
$3 = (struct inode *) 0xc6c00000
(gdb) p *inode
$4 = {i_hash = {next = 0x0, pprev = 0xc213e004}, i_list = {
    next = 0xc16a946c <inode_in_use>, prev = 0xc6c00310}, i_sb_list = {
    next = 0xc7008514, prev = 0xc7008514}, i_dentry = {next = 0xc6c02064, 
    prev = 0xc6c02064}, i_ino = 1, i_count = {counter = 1}, i_nlink = 3, 
  i_uid = 0, i_gid = 0, i_rdev = 0, i_version = 0, i_size = 0, 
  i_size_seqcount = {sequence = 0}, i_atime = {tv_sec = 1495595682, 
    tv_nsec = 16001000}, i_mtime = {tv_sec = 1495595682, tv_nsec = 20001250}, 
  i_ctime = {tv_sec = 1495595682, tv_nsec = 20001250}, i_blocks = 0, 
  i_blkbits = 12, i_bytes = 0, i_mode = 16877, i_lock = {raw_lock = {
      slock = 0}, magic = 3735899821, owner_cpu = 4294967295, 
    owner = 0xffffffff, dep_map = {key = 0xc16ac550 <sysfs_fs_type+48>, 
      class_cache = 0x0, name = 0xc15ef1bb "&sb->s_type->i_lock_key", cpu = 0, 
      ip = 0}}, i_mutex = {count = {counter = 1}, wait_lock = {raw_lock = {
        slock = 514}, magic = 3735899821, owner_cpu = 4294967295, 
      owner = 0xffffffff, dep_map = {key = 0xc190e388 <__key.15367>, 
        class_cache = 0x0, name = 0xc15d6430 "&lock->wait_lock", cpu = 0, 
        ip = 0}}, wait_list = {next = 0xc6c000c0, prev = 0xc6c000c0}, 
    owner = 0x0, name = 0x0, magic = 0xc6c00098, dep_map = {
      key = 0xc16ac560 <sysfs_fs_type+64>, 
      class_cache = 0xc1b5deb0 <lock_classes+145008>, 
      name = 0xc15ef22d "&type->i_mutex_dir_key", cpu = 0, ip = 0}}, 
  i_alloc_sem = {count = 0, wait_lock = {raw_lock = {slock = 0}, 
      magic = 3735899821, owner_cpu = 4294967295, owner = 0xffffffff, 
---Type <return> to continue, or q <return> to quit---
      dep_map = {key = 0xc1e6ccd4 <__key.14570>, class_cache = 0x0, 
        name = 0xc1612d38 "&sem->wait_lock", cpu = 0, ip = 0}}, wait_list = {
      next = 0xc6c00110, prev = 0xc6c00110}, dep_map = {
      key = 0xc16ac568 <sysfs_fs_type+72>, class_cache = 0x0, 
      name = 0xc15ef210 "&sb->s_type->i_alloc_sem_key", cpu = 0, ip = 0}}, 
  i_op = 0xc14a36e0 <sysfs_dir_inode_operations>, 
  i_fop = 0xc14a3660 <sysfs_dir_operations>, i_sb = 0xc7008430, i_flock = 0x0, 
  i_mapping = 0xc6c00140, i_data = {host = 0xc6c00000, page_tree = {
      height = 0, gfp_mask = 32, rnode = 0x0}, tree_lock = {raw_lock = {
        slock = 0}, magic = 3735899821, owner_cpu = 4294967295, 
      owner = 0xffffffff, dep_map = {key = 0xc1e624ec <__key.26256>, 
        class_cache = 0x0, name = 0xc15ef13f "&mapping->tree_lock", cpu = 0, 
        ip = 0}}, i_mmap_writable = 0, i_mmap = {prio_tree_node = 0x0, 
      index_bits = 1, raw = 1}, i_mmap_nonlinear = {next = 0xc6c00180, 
      prev = 0xc6c00180}, i_mmap_lock = {raw_lock = {slock = 0}, 
      magic = 3735899821, owner_cpu = 4294967295, owner = 0xffffffff, 
      dep_map = {key = 0xc1e624e4 <__key.26257>, class_cache = 0x0, 
        name = 0xc15ef153 "&mapping->i_mmap_lock", cpu = 0, ip = 0}}, 
    truncate_count = 0, nrpages = 0, writeback_index = 0, 
    a_ops = 0xc14a3540 <sysfs_aops>, flags = 131290, 
    backing_dev_info = 0xc16ac140 <sysfs_backing_dev_info>, private_lock = {
      raw_lock = {slock = 0}, magic = 3735899821, owner_cpu = 4294967295, 
      owner = 0xffffffff, dep_map = {key = 0xc1e624dc <__key.26258>, 
---Type <return> to continue, or q <return> to quit---
        class_cache = 0x0, name = 0xc15ef169 "&mapping->private_lock", 
        cpu = 0, ip = 0}}, private_list = {next = 0xc6c001e8, 
      prev = 0xc6c001e8}, assoc_mapping = 0x0, unmap_mutex = {count = {
        counter = 1}, wait_lock = {raw_lock = {slock = 0}, magic = 3735899821, 
        owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
          key = 0xc190e388 <__key.15367>, class_cache = 0x0, 
          name = 0xc15d6430 "&lock->wait_lock", cpu = 0, ip = 0}}, 
      wait_list = {next = 0xc6c0021c, prev = 0xc6c0021c}, owner = 0x0, 
      name = 0x0, magic = 0xc6c001f4, dep_map = {
        key = 0xc1e624d4 <__key.26259>, class_cache = 0x0, 
        name = 0xc15ef180 "&mapping->unmap_mutex", cpu = 0, ip = 0}}}, 
  i_dquot = {0x0, 0x0}, i_devices = {next = 0xc6c0024c, prev = 0xc6c0024c}, {
    i_pipe = 0x0, i_bdev = 0x0, i_cdev = 0x0}, i_generation = 0, 
  i_fsnotify_mask = 0, i_fsnotify_mark_entries = {first = 0x0}, 
  inotify_watches = {next = 0xc6c00264, prev = 0xc6c00264}, inotify_mutex = {
    count = {counter = 1}, wait_lock = {raw_lock = {slock = 0}, 
      magic = 3735899821, owner_cpu = 4294967295, owner = 0xffffffff, 
      dep_map = {key = 0xc190e388 <__key.15367>, class_cache = 0x0, 
        name = 0xc15d6430 "&lock->wait_lock", cpu = 0, ip = 0}}, wait_list = {
      next = 0xc6c00294, prev = 0xc6c00294}, owner = 0x0, name = 0x0, 
    magic = 0xc6c0026c, dep_map = {key = 0xc1e624cc <__key.26270>, 
      class_cache = 0x0, name = 0xc15ef196 "&inode->inotify_mutex", cpu = 0, 
      ip = 0}}, i_state = 0, dirtied_when = 0, i_flags = 0, i_writecount = {
---Type <return> to continue, or q <return> to quit---
    counter = 0}, i_security = 0x0, i_acl = 0xffffffff, 
  i_default_acl = 0xffffffff, i_private = 0xc16ac580 <sysfs_root>}
(gdb) n
576			if (!test(inode, data))
(gdb) 
578			if (inode->i_state & (I_FREEING|I_CLEAR|I_WILL_FREE)) {
(gdb) 
579				__wait_on_freeing_inode(inode);
(gdb) 
585	}
(gdb) 
ifind (sb=0xc7008430, head=0xc213e004, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>, wait=wait@entry=1)
    at fs/inode.c:906
906		if (inode) {
(gdb) 
907			__iget(inode);
(gdb) 
908			spin_unlock(&inode_lock);
(gdb) 
909			if (likely(wait))
(gdb) 
910				wait_on_inode(inode);
(gdb) 
915	}
(gdb) 
ilookup5 (sb=<optimized out>, hashval=<optimized out>, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>) at fs/inode.c:1004
1004	}
(gdb) 
sysfs_addrm_start (acxt=acxt@entry=0xc706bf20, 
    parent_sd=parent_sd@entry=0xc16ac580 <sysfs_root>) at fs/sysfs/dir.c:385
385		if (inode) {
(gdb) 
386			WARN_ON(inode->i_state & I_NEW);
(gdb) 
389			acxt->parent_inode = inode;
(gdb) 
395			if (!mutex_trylock(&inode->i_mutex)) {
(gdb) 
401	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70851dc, parent_sd=0xc16ac580 <sysfs_root>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf48) at fs/sysfs/dir.c:689
689		rc = sysfs_add_one(&acxt, sd);
(gdb) s
sysfs_add_one (acxt=acxt@entry=0xc706bf20, sd=sd@entry=0xc7022058)
    at fs/sysfs/dir.c:482
482		ret = __sysfs_add_one(acxt, sd);
(gdb) s
__sysfs_add_one (acxt=acxt@entry=0xc706bf20, sd=sd@entry=0xc7022058)
    at fs/sysfs/dir.c:425
425		if (sysfs_find_dirent(acxt->parent_sd, sd->s_name))
(gdb) s
sysfs_find_dirent (parent_sd=0xc16ac580 <sysfs_root>, 
    name=0xc7020268 "devices") at fs/sysfs/dir.c:639
639		for (sd = parent_sd->s_dir.children; sd; sd = sd->s_sibling)
(gdb) n
640			if (!strcmp(sd->s_name, name))
(gdb) 
642		return NULL;
(gdb) 
643	}
(gdb) 
__sysfs_add_one (acxt=acxt@entry=0xc706bf20, sd=sd@entry=0xc7022058)
    at fs/sysfs/dir.c:428
428		sd->s_parent = sysfs_get(acxt->parent_sd);
(gdb) s
__sysfs_get (sd=0xc16ac580 <sysfs_root>) at fs/sysfs/sysfs.h:138
138		if (sd) {
(gdb) n
139			WARN_ON(!atomic_read(&sd->s_count));
(gdb) 
140			atomic_inc(&sd->s_count);
(gdb) 
__sysfs_add_one (acxt=acxt@entry=0xc706bf20, sd=sd@entry=0xc7022058)
    at fs/sysfs/dir.c:430
430		if (sysfs_type(sd) == SYSFS_DIR && acxt->parent_inode)
(gdb) 
428		sd->s_parent = sysfs_get(acxt->parent_sd);
(gdb) 
430		if (sysfs_type(sd) == SYSFS_DIR && acxt->parent_inode)
(gdb) 
431			inc_nlink(acxt->parent_inode);
(gdb) 
433		acxt->cnt++;
(gdb) 
435		sysfs_link_sibling(sd);
(gdb) s
sysfs_link_sibling (sd=0xc7022058) at fs/sysfs/dir.c:49
49		BUG_ON(sd->s_sibling);
(gdb) n
46		struct sysfs_dirent *parent_sd = sd->s_parent;
(gdb) 
49		BUG_ON(sd->s_sibling);
(gdb) 
55		for (pos = &parent_sd->s_dir.children; *pos; pos = &(*pos)->s_sibling) {
(gdb) 
56			if (sd->s_ino < (*pos)->s_ino)
(gdb) 
55		for (pos = &parent_sd->s_dir.children; *pos; pos = &(*pos)->s_sibling) {
(gdb) 
59		sd->s_sibling = *pos;
(gdb) 
60		*pos = sd;
(gdb) 
61	}
(gdb) 
__sysfs_add_one (acxt=acxt@entry=0xc706bf20, sd=sd@entry=0xc7022058)
    at fs/sysfs/dir.c:437
437		return 0;
(gdb) 
438	}
(gdb) 
sysfs_add_one (acxt=acxt@entry=0xc706bf20, sd=sd@entry=0xc7022058)
    at fs/sysfs/dir.c:483
483		if (ret == -EEXIST) {
(gdb) 
482		ret = __sysfs_add_one(acxt, sd);
(gdb) 
483		if (ret == -EEXIST) {
(gdb) 
496	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70851dc, parent_sd=<optimized out>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf48) at fs/sysfs/dir.c:690
690		sysfs_addrm_finish(&acxt);
(gdb) s
sysfs_addrm_finish (acxt=acxt@entry=0xc706bf20) at fs/sysfs/dir.c:593
593	{
(gdb) n
595		mutex_unlock(&sysfs_mutex);
(gdb) 
596		if (acxt->parent_inode) {
(gdb) 
600			if (acxt->cnt)
(gdb) 
601				inode->i_ctime = inode->i_mtime = CURRENT_TIME;
(gdb) 
603			mutex_unlock(&inode->i_mutex);
(gdb) 
604			iput(inode);
(gdb) 
608		while (acxt->removed) {
(gdb) 
619	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70851dc, parent_sd=<optimized out>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf48) at fs/sysfs/dir.c:692
692		if (rc == 0)
(gdb) 
693			*p_sd = sd;
(gdb) 
698	}
(gdb) 
sysfs_create_dir (kobj=kobj@entry=0xc70851dc) at fs/sysfs/dir.c:723
723		if (!error)
(gdb) 
724			kobj->sd = sd;
(gdb) 
726	}
(gdb) 
create_dir (kobj=0xc70851dc) at lib/kobject.c:52
52			if (!error) {
(gdb) 
51			error = sysfs_create_dir(kobj);
(gdb) 
52			if (!error) {
(gdb) 
kobject_add_internal (kobj=0xc70851dc) at lib/kobject.c:187
187		error = create_dir(kobj);
(gdb) 
204			kobj->state_in_sysfs = 1;
(gdb) 
169			return -EINVAL;
(gdb) 
207	}
(gdb) 
kset_register (k=k@entry=0xc70851b0) at lib/kobject.c:721
721		if (err)
(gdb) 
720		err = kobject_add_internal(&k->kobj);
(gdb) 
721		if (err)
(gdb) 
723		kobject_uevent(&k->kobj, KOBJ_ADD);
(gdb) s
kobject_uevent (kobj=kobj@entry=0xc70851dc, action=action@entry=KOBJ_ADD)
    at lib/kobject_uevent.c:282
282		return kobject_uevent_env(kobj, action, NULL);
(gdb) s
kobject_uevent_env (kobj=kobj@entry=0xc70851dc, action=action@entry=KOBJ_ADD, 
    envp_ext=envp_ext@entry=0x0) at lib/kobject_uevent.c:93
93		const char *action_string = kobject_actions[action];
(gdb) n
103		pr_debug("kobject: '%s' (%p): %s\n",
(gdb) 
91	{
(gdb) 
93		const char *action_string = kobject_actions[action];
(gdb) 
103		pr_debug("kobject: '%s' (%p): %s\n",
(gdb) 
91	{
(gdb) 
103		pr_debug("kobject: '%s' (%p): %s\n",
(gdb) 
107		top_kobj = kobj;
(gdb) 
108		while (!top_kobj->kset && top_kobj->parent)
(gdb) 
112			pr_debug("kobject: '%s' (%p): %s: attempted to send uevent "
(gdb) 
115			return -EINVAL;
(gdb) 
112			pr_debug("kobject: '%s' (%p): %s: attempted to send uevent "
(gdb) 
268	}
(gdb) 
kobject_uevent (kobj=kobj@entry=0xc70851dc, action=action@entry=KOBJ_ADD)
    at lib/kobject_uevent.c:283
283	}
(gdb) 
kset_register (k=k@entry=0xc70851b0) at lib/kobject.c:725
725	}
(gdb) 
kset_create_and_add (name=name@entry=0xc15d7009 "devices", 
    uevent_ops=uevent_ops@entry=0xc16c78f8 <device_uevent_ops>, 
    parent_kobj=parent_kobj@entry=0x0) at lib/kobject.c:845
845		if (error) {
(gdb) 
850	}
(gdb) 
devices_init () at drivers/base/core.c:1276
1276		if (!devices_kset)
(gdb) 
1275		devices_kset = kset_create_and_add("devices", &device_uevent_ops, NULL);
(gdb) 
1276		if (!devices_kset)
(gdb) 
```

After kset created, `device_init` invoke `kobject_create_and_add` to create kernel objects and register it with sysfs. 3 Types of kobject created, `dev` for normal device, `block` for block device and `char` for charactor device. Below debug information is the procedue of creating kobject `dev`.

```kobject_create_and_add
1278		dev_kobj = kobject_create_and_add("dev", NULL);
(gdb) s
kobject_create_and_add (name=name@entry=0xc1629db5 "dev", 
    parent=parent@entry=0x0) at lib/kobject.c:652
652		kobj = kobject_create();
(gdb) s
kobject_create () at lib/kobject.c:626
626		kobj = kzalloc(sizeof(*kobj), GFP_KERNEL);
(gdb) n
627		if (!kobj)
(gdb) 
630		kobject_init(kobj, &dynamic_kobj_ktype);
(gdb) s
kobject_init (kobj=kobj@entry=0xc70860e0, 
    ktype=ktype@entry=0xc16bb114 <dynamic_kobj_ktype>) at lib/kobject.c:274
274		if (!kobj) {
(gdb) n
278		if (!ktype) {
(gdb) 
282		if (kobj->state_initialized) {
(gdb) 
289		kobject_init_internal(kobj);
(gdb) s
kobject_init_internal (kobj=0xc70860e0) at lib/kobject.c:149
149		kref_init(&kobj->kref);
(gdb) s
kref_init (kref=kref@entry=0xc70860fc) at lib/kref.c:34
34		kref_set(kref, 1);
(gdb) n
35	}
(gdb) 
kobject_init_internal (kobj=0xc70860e0) at lib/kobject.c:150
150		INIT_LIST_HEAD(&kobj->entry);
(gdb) 
151		kobj->state_in_sysfs = 0;
(gdb) 
153		kobj->state_remove_uevent_sent = 0;
(gdb) n
154		kobj->state_initialized = 1;
(gdb) 
kobject_init (kobj=kobj@entry=0xc70860e0, 
    ktype=ktype@entry=0xc16bb114 <dynamic_kobj_ktype>) at lib/kobject.c:290
290		kobj->ktype = ktype;
(gdb) 
296	}
(gdb) 
kobject_create () at lib/kobject.c:631
631		return kobj;
(gdb) 
632	}
(gdb) 
kobject_create_and_add (name=name@entry=0xc1629db5 "dev", 
    parent=parent@entry=0x0) at lib/kobject.c:653
653		if (!kobj)
(gdb) 
656		retval = kobject_add(kobj, parent, "%s", name);
(gdb) s
kobject_add (kobj=kobj@entry=0xc70860e0, parent=parent@entry=0x0, 
    fmt=fmt@entry=0xc1627337 "%s") at lib/kobject.c:344
344		if (!kobj)
(gdb) n
347		if (!kobj->state_initialized) {
(gdb) 
355		retval = kobject_add_varg(kobj, parent, fmt, args);
(gdb) s
kobject_add_varg (
    vargs=0xc706bfa4 "\265\235b\301\360\021v\301`ro\301\\ro\301\274\277\006Q;\r\3
01L\\006\307G@r\301\340\277\006\307\340\250o\301\254S\\\301\001",
    fmt=0xc1627337 "%s", parent=0x0, kobj=0xc70860e0) at lib/kobject.c:304
304		retval = kobject_set_name_vargs(kobj, fmt, vargs);
(gdb) n
305		if (retval) {
(gdb) 
309		kobj->parent = parent;
(gdb) 
310		return kobject_add_internal(kobj);
(gdb) s
kobject_add_internal (kobj=0xc70860e0) at lib/kobject.c:163
163		if (!kobj)
(gdb) p kobj
$1 = (struct kobject *) 0xc70860e0
(gdb) p *kobj
$2 = {name = 0xc70202a0 "dev", entry = {next = 0xc70860e4, prev = 0xc70860e4}, 
  parent = 0x0, kset = 0x0, ktype = 0xc16bb114 <dynamic_kobj_ktype>, sd = 0x0, 
  kref = {refcount = {counter = 1}}, state_initialized = 1, 
  state_in_sysfs = 0, state_add_uevent_sent = 0, state_remove_uevent_sent = 0, 
  uevent_suppress = 0}
(gdb) n
166		if (!kobj->name || !kobj->name[0]) {
(gdb) 
172		parent = kobject_get(kobj->parent);
(gdb) 
175		if (kobj->kset) {
(gdb) 
182		pr_debug("kobject: '%s' (%p): %s: parent: '%s', set: '%s'\n",
(gdb) 
187		error = create_dir(kobj);
(gdb) s
create_dir (kobj=0xc70860e0) at lib/kobject.c:50
50		if (kobject_name(kobj)) {
(gdb) n
51			error = sysfs_create_dir(kobj);
(gdb) s
sysfs_create_dir (kobj=kobj@entry=0xc70860e0) at fs/sysfs/dir.c:715
715		BUG_ON(!kobj);
(gdb) n
717		if (kobj->parent)
(gdb) 
720			parent_sd = &sysfs_root;
(gdb) 
722		error = create_dir(kobj, parent_sd, kobject_name(kobj), &sd);
(gdb) s
create_dir (kobj=kobj@entry=0xc70860e0, parent_sd=0xc16ac580 <sysfs_root>, 
    name=0xc70202a0 "dev", p_sd=p_sd@entry=0xc706bf38) at fs/sysfs/dir.c:675
675	{
(gdb) n
682		sd = sysfs_new_dirent(name, mode, SYSFS_DIR);
(gdb) s
sysfs_new_dirent (name=0xc70202a0 "dev", mode=mode@entry=16877, 
    type=type@entry=1) at fs/sysfs/dir.c:318
318		if (type & SYSFS_COPY_NAME) {
(gdb) 
319			name = dup_name = kstrdup(name, GFP_KERNEL);
(gdb) 
320			if (!name)
(gdb) 
324		sd = kmem_cache_zalloc(sysfs_dir_cachep, GFP_KERNEL);
(gdb) 
325		if (!sd)
(gdb) 
328		if (sysfs_alloc_ino(&sd->s_ino))
(gdb) 
331		atomic_set(&sd->s_count, 1);
(gdb) 
332		atomic_set(&sd->s_active, 0);
(gdb) 
334		sd->s_name = name;
(gdb) 
335		sd->s_mode = mode;
(gdb) 
336		sd->s_flags = type;
(gdb) 
338		return sd;
(gdb) 
345	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70860e0, parent_sd=0xc16ac580 <sysfs_root>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf38) at fs/sysfs/dir.c:683
683		if (!sd)
(gdb) n
685		sd->s_dir.kobj = kobj;
(gdb) 
688		sysfs_addrm_start(&acxt, parent_sd);
(gdb) s
sysfs_addrm_start (acxt=acxt@entry=0xc706bf10, 
    parent_sd=parent_sd@entry=0xc16ac580 <sysfs_root>) at fs/sysfs/dir.c:374
374		memset(acxt, 0, sizeof(*acxt));
(gdb) n
375		acxt->parent_sd = parent_sd;
(gdb) 
381		mutex_lock(&sysfs_mutex);
(gdb) 
383		inode = ilookup5(sysfs_sb, parent_sd->s_ino, sysfs_ilookup_test,
(gdb) s
ilookup5 (sb=0xc7008430, hashval=1, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>) at fs/inode.c:1001
1001		struct hlist_head *head = inode_hashtable + hash(sb, hashval);
(gdb) n
1003		return ifind(sb, head, test, data, 1);
(gdb) s
ifind (sb=0xc7008430, head=0xc213e004, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>, wait=wait@entry=1)
    at fs/inode.c:904
904		spin_lock(&inode_lock);
(gdb) 
905		inode = find_inode(sb, head, test, data);
(gdb) s
find_inode (sb=sb@entry=0xc7008430, head=head@entry=0xc213e004, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>) at fs/inode.c:573
573		hlist_for_each_entry(inode, node, head, i_hash) {
(gdb) 
574			if (inode->i_sb != sb)
(gdb) p inode->i_sb
$3 = (struct super_block *) 0xc7008430
(gdb) n
576			if (!test(inode, data))
(gdb) 
578			if (inode->i_state & (I_FREEING|I_CLEAR|I_WILL_FREE)) {
(gdb) p inode->i_state 
$4 = 0
(gdb) n
579				__wait_on_freeing_inode(inode);
(gdb) 
585	}
(gdb) 
ifind (sb=0xc7008430, head=0xc213e004, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>, wait=wait@entry=1)
    at fs/inode.c:906
906		if (inode) {
(gdb) 
907			__iget(inode);
(gdb) 
908			spin_unlock(&inode_lock);
(gdb) 
909			if (likely(wait))
(gdb) 
910				wait_on_inode(inode);
(gdb) 
915	}
(gdb) 
ilookup5 (sb=<optimized out>, hashval=<optimized out>, 
    test=test@entry=0xc1170180 <sysfs_ilookup_test>, 
    data=data@entry=0xc16ac580 <sysfs_root>) at fs/inode.c:1004
1004	}
(gdb) 
sysfs_addrm_start (acxt=acxt@entry=0xc706bf10, 
    parent_sd=parent_sd@entry=0xc16ac580 <sysfs_root>) at fs/sysfs/dir.c:385
385		if (inode) {
(gdb) 
386			WARN_ON(inode->i_state & I_NEW);
(gdb) 
389			acxt->parent_inode = inode;
(gdb) 
395			if (!mutex_trylock(&inode->i_mutex)) {
(gdb) 
401	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70860e0, parent_sd=0xc16ac580 <sysfs_root>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf38) at fs/sysfs/dir.c:689
689		rc = sysfs_add_one(&acxt, sd);
(gdb) s
sysfs_add_one (acxt=acxt@entry=0xc706bf10, sd=sd@entry=0xc70220b0)
    at fs/sysfs/dir.c:482
482		ret = __sysfs_add_one(acxt, sd);
(gdb) s
__sysfs_add_one (acxt=acxt@entry=0xc706bf10, sd=sd@entry=0xc70220b0)
    at fs/sysfs/dir.c:425
425		if (sysfs_find_dirent(acxt->parent_sd, sd->s_name))
(gdb) s
sysfs_find_dirent (parent_sd=0xc16ac580 <sysfs_root>, name=0xc70202d8 "dev")
    at fs/sysfs/dir.c:639
639		for (sd = parent_sd->s_dir.children; sd; sd = sd->s_sibling)
(gdb) n
640			if (!strcmp(sd->s_name, name))
(gdb) p sd->s_name 
$5 = 0xc7020070 "fs"
(gdb) n
642		return NULL;
(gdb) 
643	}
(gdb) 
__sysfs_add_one (acxt=acxt@entry=0xc706bf10, sd=sd@entry=0xc70220b0)
    at fs/sysfs/dir.c:428
428		sd->s_parent = sysfs_get(acxt->parent_sd);
(gdb) 
430		if (sysfs_type(sd) == SYSFS_DIR && acxt->parent_inode)
(gdb) 
431			inc_nlink(acxt->parent_inode);
(gdb) 
433		acxt->cnt++;
(gdb) 
435		sysfs_link_sibling(sd);
(gdb) s
sysfs_link_sibling (sd=0xc70220b0) at fs/sysfs/dir.c:46
46		struct sysfs_dirent *parent_sd = sd->s_parent;
(gdb) 
49		BUG_ON(sd->s_sibling);
(gdb) 
55		for (pos = &parent_sd->s_dir.children; *pos; pos = &(*pos)->s_sibling) {
(gdb) 
56			if (sd->s_ino < (*pos)->s_ino)
(gdb) 
55		for (pos = &parent_sd->s_dir.children; *pos; pos = &(*pos)->s_sibling) {
(gdb) 
59		sd->s_sibling = *pos;
(gdb) 
60		*pos = sd;
(gdb) 
61	}
(gdb) 
__sysfs_add_one (acxt=acxt@entry=0xc706bf10, sd=sd@entry=0xc70220b0)
    at fs/sysfs/dir.c:437
437		return 0;
(gdb) 
438	}
(gdb) 
sysfs_add_one (acxt=acxt@entry=0xc706bf10, sd=sd@entry=0xc70220b0)
    at fs/sysfs/dir.c:483
483		if (ret == -EEXIST) {
(gdb) 
496	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70860e0, parent_sd=<optimized out>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf38) at fs/sysfs/dir.c:690
690		sysfs_addrm_finish(&acxt);
(gdb) s
sysfs_addrm_finish (acxt=acxt@entry=0xc706bf10) at fs/sysfs/dir.c:595
595		mutex_unlock(&sysfs_mutex);
(gdb) n
596		if (acxt->parent_inode) {
(gdb) 
600			if (acxt->cnt)
(gdb) 
601				inode->i_ctime = inode->i_mtime = CURRENT_TIME;
(gdb) 
603			mutex_unlock(&inode->i_mutex);
(gdb) 
604			iput(inode);
(gdb) 
608		while (acxt->removed) {
(gdb) 
619	}
(gdb) 
create_dir (kobj=kobj@entry=0xc70860e0, parent_sd=<optimized out>, 
    name=<optimized out>, p_sd=p_sd@entry=0xc706bf38) at fs/sysfs/dir.c:692
692		if (rc == 0)
(gdb) 
693			*p_sd = sd;
(gdb) 
698	}
(gdb) 
sysfs_create_dir (kobj=kobj@entry=0xc70860e0) at fs/sysfs/dir.c:723
723		if (!error)
(gdb) 
724			kobj->sd = sd;
(gdb) 
726	}
(gdb) 
create_dir (kobj=0xc70860e0) at lib/kobject.c:52
52			if (!error) {
(gdb) 
kobject_add_internal (kobj=0xc70860e0) at lib/kobject.c:187
187		error = create_dir(kobj);
(gdb) 
204			kobj->state_in_sysfs = 1;
(gdb) 
207	}
(gdb) 
kobject_add (kobj=kobj@entry=0xc70860e0, parent=parent@entry=0x0, 
    fmt=fmt@entry=0xc1627337 "%s") at lib/kobject.c:359
359	}
(gdb) 
kobject_create_and_add (name=name@entry=0xc1629db5 "dev", 
    parent=parent@entry=0x0) at lib/kobject.c:657
657		if (retval) {
(gdb) 
664	}
(gdb) 
devices_init () at drivers/base/core.c:1279
1279		if (!dev_kobj)
(gdb) 
```

# Links
* [Optimizing preemption](https://lwn.net/Articles/563185/)
* [completions - wait for completion handling](https://www.kernel.org/doc/Documentation/scheduler/completion.txt)
