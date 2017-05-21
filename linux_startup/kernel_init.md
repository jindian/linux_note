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

# Links
* [Optimizing preemption](https://lwn.net/Articles/563185/)
* [completions - wait for completion handling](https://www.kernel.org/doc/Documentation/scheduler/completion.txt)
