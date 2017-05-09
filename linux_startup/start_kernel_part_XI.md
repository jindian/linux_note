# start\_kernel part XI

## _initialize cgoup_

[Control Groups](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt) provide a mechanism for aggregating/partitioning sets of tasks, and all their future children, into hierarchical groups with specialized behaviour.

`cgroup_init`

* initializes `cgroup_backing_devinfo`\_, \_the routine `bdi_init` already introduced in `mount_init`

```
start_kernel () at init/main.c:681
681        cgroup_init();
(gdb) s
cgroup_init () at kernel/cgroup.c:3267
3267        err = bdi_init(&cgroup_backing_dev_info);
(gdb) n
3268        if (err)
```

* initializes any cgroup subsystems that don't need early init, all configured cgroup subsystems are defined in linux/cgroup\_subsys.h.

```
(gdb) 
3272            struct cgroup_subsys *ss = subsys[i];
(gdb) p subsys
$1 = {0xc16a1500 <cpuset_subsys>, 0xc1697d80 <cpuacct_subsys>, 
  0xc16a0c60 <freezer_subsys>}
(gdb) n
3273            if (!ss->early_init)
(gdb) p subsys[i]->early_init 
$4 = 1
(gdb) p subsys[i]->use_id 
$5 = false
(gdb) n
3275            if (ss->use_id)
(gdb) 
3271        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3272            struct cgroup_subsys *ss = subsys[i];
(gdb) 
3273            if (!ss->early_init)
(gdb) p subsys[i]->early_init 
$6 = 0
(gdb) p subsys[i]->use_id 
$7 = false
(gdb) n
3274                cgroup_init_subsys(ss);
(gdb) s
cgroup_init_subsys (ss=ss@entry=0xc1697d80 <cpuacct_subsys>)
    at kernel/cgroup.c:3180
3180        printk(KERN_INFO "Initializing cgroup subsys %s\n", ss->name);
(gdb) n
3183        list_add(&ss->sibling, &rootnode.subsys_list);
(gdb) 
3184        ss->root = &rootnode;
(gdb) n
3185        css = ss->create(ss, dummytop);
(gdb) s
cpuacct_create (ss=0xc1697d80 <cpuacct_subsys>, cgrp=0xc1e30b78 <rootnode+24>)
    at kernel/sched.c:10842
10842        struct cpuacct *ca = kzalloc(sizeof(*ca), GFP_KERNEL);
(gdb) n
10845        if (!ca)
(gdb) 
10848        ca->cpuusage = alloc_percpu(u64);
(gdb) 
10849        if (!ca->cpuusage)
(gdb) 
10853            if (percpu_counter_init(&ca->cpustat[i], 0))
(gdb) 
10856        if (cgrp->parent)
(gdb) 
10859        return &ca->css;
(gdb) 
10869    }
(gdb) 
cgroup_init_subsys (ss=ss@entry=0xc1697d80 <cpuacct_subsys>)
    at kernel/cgroup.c:3187
3187        BUG_ON(IS_ERR(css));
(gdb) 
3188        init_cgroup_css(css, ss, dummytop);
(gdb) 
3194        init_css_set.subsys[ss->subsys_id] = dummytop->subsys[ss->subsys_id];
(gdb) 
3196        need_forkexit_callback |= ss->fork || ss->exit;
(gdb) 
3201        BUG_ON(!list_empty(&init_task.tasks));
(gdb) 
3203        mutex_init(&ss->hierarchy_mutex);
(gdb) 
3204        lockdep_set_class(&ss->hierarchy_mutex, &ss->subsys_key);
(gdb) 
3205        ss->active = 1;
(gdb) 
3206    }
(gdb) 
cgroup_init () at kernel/cgroup.c:3275
3275            if (ss->use_id)
(gdb) 
3271        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3272            struct cgroup_subsys *ss = subsys[i];
(gdb) 
3273            if (!ss->early_init)
(gdb) p subsys[i]->early_init 
$8 = 0
(gdb) p subsys[i]->use_id 
$9 = false
(gdb) n
3274                cgroup_init_subsys(ss);
(gdb) s
cgroup_init_subsys (ss=ss@entry=0xc16a0c60 <freezer_subsys>)
    at kernel/cgroup.c:3180
3180        printk(KERN_INFO "Initializing cgroup subsys %s\n", ss->name);
(gdb) n
3183        list_add(&ss->sibling, &rootnode.subsys_list);
(gdb) 
3184        ss->root = &rootnode;
(gdb) 
3185        css = ss->create(ss, dummytop);
(gdb) s
freezer_create (ss=0xc16a0c60 <freezer_subsys>, 
    cgroup=0xc1e30b78 <rootnode+24>) at kernel/cgroup_freezer.c:136
136        freezer = kzalloc(sizeof(struct freezer), GFP_KERNEL);
(gdb) n
137        if (!freezer)
(gdb) 
140        spin_lock_init(&freezer->lock);
(gdb) 
141        freezer->state = CGROUP_THAWED;
(gdb) 
142        return &freezer->css;
(gdb) 
143    }
(gdb) 
cgroup_init_subsys (ss=ss@entry=0xc16a0c60 <freezer_subsys>)
    at kernel/cgroup.c:3187
3187        BUG_ON(IS_ERR(css));
(gdb) 
3188        init_cgroup_css(css, ss, dummytop);
(gdb) 
3194        init_css_set.subsys[ss->subsys_id] = dummytop->subsys[ss->subsys_id];
(gdb) 
3196        need_forkexit_callback |= ss->fork || ss->exit;
(gdb) 
3201        BUG_ON(!list_empty(&init_task.tasks));
(gdb) 
3203        mutex_init(&ss->hierarchy_mutex);
(gdb) 
3204        lockdep_set_class(&ss->hierarchy_mutex, &ss->subsys_key);
(gdb) 
3205        ss->active = 1;
(gdb) 
3206    }
(gdb) 
cgroup_init () at kernel/cgroup.c:3275
3275            if (ss->use_id)
(gdb) 
3271        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
```

* Add `init_css_set` to the hash table

```
3280        hhead = css_set_hash(init_css_set.subsys);
(gdb) p init_css_set.subsys
$10 = {0xc16a14a0 <top_cpuset>, 0xc701b780, 0xc7001150}
(gdb) s
css_set_hash (css=<optimized out>) at kernel/cgroup.c:262
262            tmp += (unsigned long)css[i];
(gdb) n
263        tmp = (tmp >> 16) ^ tmp;
(gdb) 
265        index = hash_long(tmp, CSS_SET_HASH_BITS);
(gdb) 
cgroup_init () at kernel/cgroup.c:3281
3281        hlist_add_head(&init_css_set.hlist, hhead);
(gdb) 
3282        BUG_ON(!init_root_id(&rootnode));
(gdb) p init_css_set.subsys[0]
$12 = (struct cgroup_subsys_state *) 0xc16a14a0 <top_cpuset>
(gdb) p init_css_set.subsys[1]
$13 = (struct cgroup_subsys_state *) 0xc701b780
(gdb) p init_css_set.subsys[2]
$14 = (struct cgroup_subsys_state *) 0xc7001150
(gdb) p css_set_table 
$15 = {{first = 0x0} <repeats 46 times>, {
    first = 0xc1e30b24 <init_css_set+4>}, {first = 0x0} <repeats 81 times>}
```

* Registers cgroup filesystem

```
(gdb) 
3283        err = register_filesystem(&cgroup_fs_type);
(gdb) 
3284        if (err < 0)
```

* Creates /proc/cgroups

```
3287        proc_create("cgroups", 0, NULL, &proc_cgroupstats_operations);
(gdb) 
3290        if (err)
(gdb) 
3294    }
```

## _initialize cpusets_

Cpusets provide a mechanism for assigning a set of CPUs and Memory Nodes to a set of tasks.

Cpusets constrain the CPU and Memory placement of tasks to only the resources within a tasks current cpuset.  They form a nested hierarchy visible in a virtual file system.  These are the essential hooks, beyond what is already present, required to manage dynamic job placement on large systems.

Each task has a pointer to a cpuset.  Multiple tasks may reference the same cpuset.  Requests by a task, using the sched\_setaffinity\(2\) system call to include CPUs in its CPU affinity mask, and using the mbind\(2\) and set\_mempolicy\(2\) system calls to include Memory Nodes in its memory policy, are both filtered through that tasks cpuset, filtering out any CPUs or Memory Nodes not in that cpuset.  The scheduler will not schedule a task on a CPU that is not allowed in its cpus\_allowed vector, and the kernel page allocator will not allocate a page on a node that is not allowed in the requesting tasks mems\_allowed vector.

If a cpuset is cpu or mem exclusive, no other cpuset, other than a direct ancestor or descendent, may share any of the same CPUs or Memory Nodes.

User level code may create and destroy cpusets by name in the cpuset virtual file system, manage the attributes and permissions of these cpusets and which CPUs and Memory Nodes are assigned to each cpuset, specify and query to which cpuset a task is assigned, and list the task pids assigned to a cpuset.

* `cpuset_init` allocates cpu mask variable of `top_cpuset` and initializes `top_cpuset`
* Registers cpuset filesystem
* Allocates cpu mask variable `cpus_attach`

## _early init task statistics_

Taskstats is a netlink-based interface for sending per-task and per-process statistics from the kernel to userspace.  
Taskstats was designed for the following benefits:

* efficiently provide statistics during lifetime of a task and on its exit
* unified interface for multiple accounting subsystems
* extensibility for use by future accounting patches

`taskstats_init_early` allocates slab cache memory for struct taskstats and initializes list and semaphore for per cpu variable `listener_array`.

## _initialize per task delay accounting_

Tasks encounter delays in execution when they wait for some kernel resource to become available e.g. a runnable task may wait for a free CPU to run on.

The per-task delay accounting functionality measures the delays experienced by a task while

a\) waiting for a CPU \(while being runnable\)  
b\) completion of synchronous block I/O initiated by the task  
c\) swapping in pages  
d\) memory reclaim

and makes these statistics available to userspace through the taskstats interface.

Such delays provide feedback for setting a task's cpu priority, io priority and rss limit values appropriately. Long delays for important tasks could be a trigger for raising its corresponding priority.

The functionality, through its use of the taskstats interface, also provides delay statistics aggregated for all tasks \(or threads\) belonging to a thread group \(corresponding to a traditional Unix process\). This is a commonly needed aggregation that is more efficiently done by the kernel.

Userspace utilities, particularly resource management applications, can also aggregate delay statistics into arbitrary groups. To enable this, delay statistics of a task are available both during its lifetime as well as on its exit, ensuring continuous and complete monitoring can be done.

`delayacct_init` allocates slab cache memory for struct `task_delay_info` and initializes delay accounting for init task.

```delayacct\_init
Breakpoint 2, delayacct_init () at kernel/delayacct.c:34
34    {
(gdb) n
35        delayacct_cache = KMEM_CACHE(task_delay_info, SLAB_PANIC);
(gdb) 
36        delayacct_tsk_init(&init_task);
(gdb) s
delayacct_tsk_init (tsk=<optimized out>) at include/linux/delayacct.h:67
67        tsk->delays = NULL;
(gdb) n
68        if (delayacct_on)
(gdb) p delayacct_on 
$1 = 1
(gdb) n
69            __delayacct_tsk_init(tsk);
(gdb) s
__delayacct_tsk_init (tsk=0xc1692440 <init_task>) at kernel/delayacct.c:41
41        tsk->delays = kmem_cache_zalloc(delayacct_cache, GFP_KERNEL);
(gdb) n
42        if (tsk->delays)
(gdb) 
43            spin_lock_init(&tsk->delays->lock);
(gdb) 
44    }
(gdb) 
delayacct_init () at kernel/delayacct.c:37
37    }
(gdb)
```

## _check bugs of cpu_

`check_bugs` is used to intialize boot cpu and checks bugs of the cpu.

* `check_bugs` identifies boot cpu to initialize `boot_cpu_data`; checks powermanagerment idle function, if it's c1e\_idle, allocates c1e\_mask; setup syscall enter with vDSO mechanism; enables SYSENTER/SYSEXIT feature if cpu has this feature and update MSR register; initializes performance events.

```identify\_boot\_cpu
Breakpoint 2, check_bugs () at arch/x86/kernel/cpu/bugs.c:156
156	{
(gdb) n
157		identify_boot_cpu();
(gdb) s
identify_boot_cpu () at arch/x86/kernel/cpu/common.c:866
866		identify_cpu(&boot_cpu_data);
(gdb) n
867		init_c1e_mask();
(gdb) p boot_cpu_data 
$1 = {x86 = 6 '\006', x86_vendor = 0 '\000', x86_model = 6 '\006', 
  x86_mask = 3 '\003', wp_works_ok = 1 '\001', hlt_works_ok = 1 '\001', 
  hard_math = 1 '\001', rfu = 0 '\000', fdiv_bug = -1 '\377', 
  f00f_bug = 0 '\000', coma_bug = 0 '\000', pad0 = 0 '\000', 
  x86_virt_bits = 32 ' ', x86_phys_bits = 36 '$', x86_coreid_bits = 0 '\000', 
  extended_cpuid_level = 2147483652, cpuid_level = 4, x86_capability = {
    125873145, 0, 0, 262208, 2155872257, 0, 0, 0, 0}, 
  x86_vendor_id = "GenuineIntel\000\000\000", 
  x86_model_id = "QEMU Virtual CPU version 2.0.0", '\000' <repeats 33 times>, 
  x86_cache_size = 4096, x86_cache_alignment = 32, x86_power = 0, 
  loops_per_jiffy = 13568436, llc_shared_map = {{bits = {0}}}, 
  x86_max_cores = 1, apicid = 0, initial_apicid = 0, x86_clflush_size = 32, 
  booted_cores = 0, phys_proc_id = 0, cpu_core_id = 0, cpu_index = 0, 
  x86_hyper_vendor = 0}
(gdb) s
init_c1e_mask () at arch/x86/kernel/process.c:529
529		if (pm_idle == c1e_idle)
(gdb) p pm_idle
$2 = (void (*)(void)) 0xc1009d60 <default_idle>
(gdb) p c1e_idle 
$3 = {void (void)} 0xc1009de0 <c1e_idle>
(gdb) n
527	{
(gdb) 
529		if (pm_idle == c1e_idle)
(gdb) 
531	}
(gdb) 
identify_boot_cpu () at arch/x86/kernel/cpu/common.c:869
869		sysenter_setup();
(gdb) s
sysenter_setup () at arch/x86/vdso/vdso32-setup.c:285
285		void *syscall_page = (void *)get_zeroed_page(GFP_ATOMIC);
(gdb) n
289		vdso32_pages[0] = virt_to_page(syscall_page);
(gdb) 
292		gate_vma_init();
(gdb) s
gate_vma_init () at arch/x86/vdso/vdso32-setup.c:248
248		gate_vma.vm_mm = NULL;
(gdb) 
249		gate_vma.vm_start = FIXADDR_USER_START;
(gdb) 
250		gate_vma.vm_end = FIXADDR_USER_END;
(gdb) 
252		gate_vma.vm_page_prot = __P101;
(gdb) 
259		gate_vma.vm_flags |= VM_ALWAYSDUMP;
(gdb) 
sysenter_setup () at arch/x86/vdso/vdso32-setup.c:298
298		} else if (vdso32_sysenter()){
(gdb) 
299			vsyscall = &vdso32_sysenter_start;
(gdb) 
300			vsyscall_len = &vdso32_sysenter_end - &vdso32_sysenter_start;
(gdb) 
306		memcpy(syscall_page, vsyscall, vsyscall_len);
(gdb) p vsyscall
$4 = (const void *) 0xc173c7f8
(gdb) p vsyscall_len 
$5 = 1652
(gdb) n
307		relocate_vdso(syscall_page);
(gdb) 
310	}
(gdb) 
identify_boot_cpu () at arch/x86/kernel/cpu/common.c:870
870		enable_sep_cpu();
(gdb) s
enable_sep_cpu () at arch/x86/vdso/vdso32-setup.c:228
228		int cpu = get_cpu();
(gdb) n
229		struct tss_struct *tss = &per_cpu(init_tss, cpu);
(gdb) 
231		if (!boot_cpu_has(X86_FEATURE_SEP)) {
(gdb) 
236		tss->x86_tss.ss1 = __KERNEL_CS;
(gdb) 
237		tss->x86_tss.sp1 = sizeof(struct tss_struct) + (unsigned long) tss;
(gdb) 
238		wrmsr(MSR_IA32_SYSENTER_CS, __KERNEL_CS, 0);
(gdb) 
242	}
(gdb) 
identify_boot_cpu () at arch/x86/kernel/cpu/common.c:874
874		init_hw_perf_events();
(gdb) s
init_hw_perf_events () at arch/x86/kernel/cpu/perf_event.c:2152
2152		pr_info("Performance Events: ");
(gdb) n
2154		switch (boot_cpu_data.x86_vendor) {
(gdb) 
2156			err = intel_pmu_init();
(gdb) s
intel_pmu_init () at arch/x86/kernel/cpu/perf_event.c:2059
2059		if (!cpu_has(&boot_cpu_data, X86_FEATURE_ARCH_PERFMON)) {
(gdb) n
2061		   if (boot_cpu_data.x86 == 6) {
(gdb) 
2062			return p6_pmu_init();
(gdb) s
p6_pmu_init () at arch/x86/kernel/cpu/perf_event.c:2021
2021		switch (boot_cpu_data.x86_model) {
(gdb) p boot_cpu_data.x86_model
$8 = 6 '\006'
(gdb) n
2040		x86_pmu = p6_pmu;
(gdb) 
2042		if (!cpu_has_apic) {
(gdb) 
init_hw_perf_events () at arch/x86/kernel/cpu/perf_event.c:2169
2169		pr_cont("%s PMU driver.\n", x86_pmu.name);
(gdb) 
2171		if (x86_pmu.num_events > X86_PMC_MAX_GENERIC) {
(gdb) 
2176		perf_event_mask = (1 << x86_pmu.num_events) - 1;
(gdb) 
2177		perf_max_events = x86_pmu.num_events;
(gdb) 
2179		if (x86_pmu.num_events_fixed > X86_PMC_MAX_FIXED) {
(gdb) 
2185		perf_event_mask |=
(gdb) 
2186			((1LL << x86_pmu.num_events_fixed)-1) << X86_PMC_IDX_FIXED;
(gdb) 
2187		x86_pmu.intel_ctrl = perf_event_mask;
(gdb) 
2189		perf_events_lapic_init();
(gdb) s
perf_events_lapic_init () at arch/x86/kernel/cpu/perf_event.c:1897
1897		if (!x86_pmu.apic || !x86_pmu_initialized())
(gdb) n
1903		apic_write(APIC_LVTPC, APIC_DM_NMI);
(gdb) 
1905	}
(gdb) 
init_hw_perf_events () at arch/x86/kernel/cpu/perf_event.c:2190
2190		register_die_notifier(&perf_event_nmi_notifier);
(gdb) 
2192		pr_info("... version:                %d\n",     x86_pmu.version);
(gdb) 
2193		pr_info("... bit width:              %d\n",     x86_pmu.event_bits);
(gdb) 
2194		pr_info("... generic registers:      %d\n",     x86_pmu.num_events);
(gdb) 
2195		pr_info("... value mask:             %016Lx\n", x86_pmu.event_mask);
(gdb) 
2196		pr_info("... max period:             %016Lx\n", x86_pmu.max_period);
(gdb) 
2197		pr_info("... fixed-purpose events:   %d\n",     x86_pmu.num_events_fixed);
(gdb) 
2198		pr_info("... event mask:             %016Lx\n", perf_event_mask);
(gdb) 
2199	}
```

* Checks cpu bugs
* Replaces instructions(`__alt_instructions`, `__alt_instructions_end`) with architecture specified instructions

## _early initialize acpi_

`acpi_early_init` reallocates the root table if the host provided a static buffer for the table array in the call to acpi_initialize_tables; initializes acpi global variables; loads the ACPI tables from the RSDT/XSDT; completes the subsystem initialization including hardware, puts system into ACPI mode if it isn't already.

## _late initialize sfi_

If sfi is enabled, unmapps early mapped memory with early_ioremap, uses ioremap instead after it is ready; parses sfi table and update sfi table.

## _initialze ftrace_

Ftrace is an internal tracer designed to help out developers and designers of systems to find what is going on inside the kernel.
It can be used for debugging or analyzing latencies and performance issues that take place outside of user-space.

Although ftrace is typically considered the function tracer, it is really a frame work of several assorted tracing utilities.
There's latency tracing to examine what occurs between interrupts disabled and enabled, as well as for preemption and from a time a task is woken to the task is actually scheduled in.

One of the most common uses of ftrace is the event tracing. Through out the kernel is hundreds of static event points that can be enabled via the debugfs file system to see what is going on in certain parts of the kernel.

* The default nop is P6_NOP5, tests it to make sure that the nop actually work on this CPU. If it faults, then go to a lesser efficient 5 byte nop. If that fails then just use a jmp as nop. This isn't the most efficient nop, but we can not use a multi part nop since we would then risk being preempted in the middle of that nop, and if we enabled tracing then, it might cause a system crash.

```ftrace_dyn_arch_init
Breakpoint 2, ftrace_init () at kernel/trace/ftrace.c:2735
2735	{
(gdb) n
2740		addr = (unsigned long)ftrace_stub;
(gdb) p ftrace_stub 
$1 = {<text variable, no debug info>} 0xc1003b0b <ftrace_stub>
(gdb) n
2742		local_irq_save(flags);
(gdb) 
2743		ftrace_dyn_arch_init(&addr);
(gdb) s
ftrace_dyn_arch_init (data=data@entry=0xc168bfac <init_thread_union+8108>)
    at arch/x86/kernel/ftrace.c:315
315		asm volatile (
(gdb) n
337		switch (faulted) {
(gdb) 
339			pr_info("ftrace: converting mcount calls to 0f 1f 44 00 00\n");
(gdb) p faulted 
$2 = 0
(gdb) n
340			memcpy(ftrace_nop, ftrace_test_p6nop, MCOUNT_INSN_SIZE);
(gdb) 
353		*(unsigned long *)data = 0;
(gdb) 
356	}
(gdb) n
ftrace_init () at kernel/trace/ftrace.c:2744
2744		local_irq_restore(flags);
(gdb) 
2747		if (addr)
(gdb) 
```

* Allocates pages for ftrace dynamic table

```ftrace_dyn_table_alloc
2750		count = __stop_mcount_loc - __start_mcount_loc;
(gdb) 
2752		ret = ftrace_dyn_table_alloc(count);
(gdb) s
ftrace_dyn_table_alloc (num_to_init=<optimized out>)
    at kernel/trace/ftrace.c:1292
1292		ftrace_pages_start = (void *)get_zeroed_page(GFP_KERNEL);
(gdb) n
1293		if (!ftrace_pages_start)
(gdb) 
1310		pg = ftrace_pages = ftrace_pages_start;
(gdb) 
1312		cnt = num_to_init / ENTRIES_PER_PAGE;
(gdb) 
1313		pr_info("ftrace: allocating %ld entries in %d pages\n",
(gdb) 
1316		for (i = 0; i < cnt; i++) {
(gdb) p cnt
$5 = 41
(gdb) n
1317			pg->next = (void *)get_zeroed_page(GFP_KERNEL);
(gdb) break if i==40
Breakpoint 3 at 0xc171162a: file kernel/trace/ftrace.c, line 1317.
(gdb) c
Continuing.

Breakpoint 3, ftrace_dyn_table_alloc (num_to_init=<optimized out>)
    at kernel/trace/ftrace.c:1317
1317			pg->next = (void *)get_zeroed_page(GFP_KERNEL);
(gdb) n
1320			if (!pg->next)
(gdb) 
1323			pg = pg->next;
(gdb) 
1316		for (i = 0; i < cnt; i++) {
(gdb) 
```

* Replaces code with nop code got in previous subroutine

```
2756		last_ftrace_enabled = ftrace_enabled = 1;
(gdb) n
2758		ret = ftrace_convert_nops(NULL,
(gdb) s
ftrace_convert_nops (mod=mod@entry=0x0, start=0xc1768530, end=0xc177cfbc)
    at kernel/trace/ftrace.c:2645
2645		mutex_lock(&ftrace_lock);
(gdb) n
2647		while (p < end) {
(gdb) n
2648			addr = ftrace_call_adjust(*p++);
(gdb) 
2655			if (!addr)
(gdb) 
2657			ftrace_record_ip(addr);
(gdb) s
ftrace_record_ip (ip=3238007027) at kernel/trace/ftrace.c:973
973		if (ftrace_disabled)
(gdb) n
976		rec = ftrace_alloc_dyn_node(ip);
(gdb) s
ftrace_alloc_dyn_node (ip=<optimized out>) at kernel/trace/ftrace.c:940
940		if (ftrace_free_records) {
(gdb) p ftrace_free_records 
$6 = (struct dyn_ftrace *) 0x0
(gdb) n
954		if (ftrace_pages->index == ENTRIES_PER_PAGE) {
(gdb) p ftrace_pages->index 
$7 = 0
(gdb) n
965		return &ftrace_pages->records[ftrace_pages->index++];
(gdb) 
ftrace_record_ip (ip=3238007027) at kernel/trace/ftrace.c:977
977		if (!rec)
(gdb) 
981		rec->newlist = ftrace_new_addrs;
(gdb) p ftrace_new_addrs 
$8 = (struct dyn_ftrace *) 0x0
(gdb) n
ftrace_convert_nops (mod=mod@entry=0x0, start=<optimized out>, end=0xc177cfbc)
    at kernel/trace/ftrace.c:2647
2647		while (p < end) {
(gdb) break if p>=end
Breakpoint 4 at 0xc10b4836: file kernel/trace/ftrace.c, line 2647.
(gdb) c
Continuing.

Breakpoint 4, ftrace_convert_nops (mod=mod@entry=0x0, start=<optimized out>, 
    end=0xc177cfbc) at kernel/trace/ftrace.c:2647
2647		while (p < end) {
(gdb) n
2661		local_irq_save(flags);
(gdb) 
2662		ftrace_update_code(mod);
(gdb) s
ftrace_update_code (mod=0x0) at kernel/trace/ftrace.c:1257
1257		start = ftrace_now(raw_smp_processor_id());
(gdb) s
ftrace_now (cpu=0) at kernel/trace/trace.c:185
185		if (!global_trace.buffer)
(gdb) n
186			return trace_clock_local();
(gdb) s
trace_clock_local () at kernel/trace/trace_clock.c:39
39		raw_local_irq_save(flags);
(gdb) n
40		clock = sched_clock();
(gdb) s
native_sched_clock () at arch/x86/kernel/tsc.c:55
55		if (unlikely(tsc_disabled)) {
(gdb) n
44	{
(gdb) 
61		rdtscll(this_offset);
(gdb) 
64		return __cycles_2_ns(this_offset);
(gdb) 
65	}
(gdb) 
trace_clock_local () at kernel/trace/trace_clock.c:41
41		raw_local_irq_restore(flags);
(gdb) 
44	}
(gdb) 
ftrace_now (cpu=0) at kernel/trace/trace.c:192
192	}
(gdb) 
ftrace_update_code (mod=0x0) at kernel/trace/ftrace.c:1258
1258		ftrace_update_cnt = 0;
(gdb) 
1257		start = ftrace_now(raw_smp_processor_id());
(gdb) 
1260		while (ftrace_new_addrs) {
(gdb) n
1263			if (unlikely(ftrace_disabled))
(gdb) p ftrace_new_addrs 
$9 = (struct dyn_ftrace *) 0xc7063660
(gdb) n
1267			ftrace_new_addrs = p->newlist;
(gdb) 
1268			p->flags = 0L;
(gdb) n
1271			if (ftrace_code_disable(mod, p)) {
(gdb) s
ftrace_code_disable (rec=<optimized out>, mod=<optimized out>)
    at kernel/trace/ftrace.c:1100
1100		ip = rec->ip;
(gdb) n
1102		ret = ftrace_make_nop(mod, rec, MCOUNT_ADDR);
(gdb) s
ftrace_make_nop (mod=mod@entry=0x0, rec=rec@entry=0xc7063660, addr=3238017760)
    at arch/x86/kernel/ftrace.c:260
260	{
(gdb) n
262		unsigned long ip = rec->ip;
(gdb) 
264		old = ftrace_call_replace(ip, addr);
(gdb) s
ftrace_call_replace (addr=3238017760, ip=3242601459)
    at arch/x86/kernel/ftrace.c:60
60		calc.e8		= 0xe8;
(gdb) n
61		calc.offset	= ftrace_calc_offset(ip + MCOUNT_INSN_SIZE, addr);
(gdb) 
ftrace_make_nop (mod=mod@entry=0x0, rec=rec@entry=0xc7063660, 
    addr=<optimized out>) at arch/x86/kernel/ftrace.c:264
264		old = ftrace_call_replace(ip, addr);
(gdb) n
267		return ftrace_modify_code(rec->ip, old, new);
(gdb) s
ftrace_modify_code (ip=3242601459, 
    old_code=old_code@entry=0xc18e62a0 <calc> "\350\350\016\272\377", 
    new_code=new_code@entry=0xc18e62ac <ftrace_nop> "\017\037D")
    at arch/x86/kernel/ftrace.c:242
242		if (probe_kernel_read(replaced, (void *)ip, MCOUNT_INSN_SIZE))
(gdb) n
228	{
(gdb) 
242		if (probe_kernel_read(replaced, (void *)ip, MCOUNT_INSN_SIZE))
(gdb) 
246		if (memcmp(replaced, old_code, MCOUNT_INSN_SIZE) != 0)
(gdb) 
250		if (do_ftrace_mod_code(ip, new_code))
(gdb) 
253		sync_core();
(gdb) 
256	}
(gdb) 
ftrace_make_nop (mod=mod@entry=0x0, rec=rec@entry=0xc7063660, 
    addr=<optimized out>) at arch/x86/kernel/ftrace.c:268
268	}
(gdb) n
ftrace_code_disable (rec=0xc7063660, mod=0x0) at kernel/trace/ftrace.c:1103
1103		if (ret) {
(gdb) 
ftrace_update_code (mod=0x0) at kernel/trace/ftrace.c:1272
1272				p->flags |= FTRACE_FL_CONVERTED;
(gdb) 
1273				ftrace_update_cnt++;
1260		while (ftrace_new_addrs) {
(gdb) 
(gdb) break if ftrace_new_addrs == 0
Breakpoint 5 at 0xc10b48b1: file kernel/trace/ftrace.c, line 1273.
(gdb) c
Continuing.

Breakpoint 5, ftrace_update_code (mod=0x0) at kernel/trace/ftrace.c:1273
1273				ftrace_update_cnt++;
(gdb) n
1260		while (ftrace_new_addrs) {
(gdb) 
1278		stop = ftrace_now(raw_smp_processor_id());
(gdb) 
1279		ftrace_update_time = stop - start;
(gdb) 
1280		ftrace_update_tot_cnt += ftrace_update_cnt;
(gdb) 
ftrace_convert_nops (mod=mod@entry=0x0, start=<optimized out>, 
    end=<optimized out>) at kernel/trace/ftrace.c:2663
2663		local_irq_restore(flags);
(gdb) 
2664		mutex_unlock(&ftrace_lock);
(gdb) 
2667	}
(gdb) 
```

* Registers notifier to module notify list and sets ftrace early filter

```
(gdb) 
ftrace_init () at kernel/trace/ftrace.c:2762
2762		ret = register_module_notifier(&ftrace_module_nb);
(gdb) 
2763		if (ret)
(gdb) 
2766		set_ftrace_early_filters();
(gdb) s
set_ftrace_early_filters () at kernel/trace/ftrace.c:2335
2335		if (ftrace_filter_buf[0])
(gdb) 
2337		if (ftrace_notrace_buf[0])
(gdb) 
ftrace_init () at kernel/trace/ftrace.c:2771
2771	}
(gdb) 
```

## _rest of kernel initailization_

End of `start_kernel`, rest of kernel initialization will be introduced in new chapter.

# Links

* [CGROUPS](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [CPUSETS](https://lwn.net/Articles/127936/)
* [Per-task statistics interface](https://www.kernel.org/doc/Documentation/accounting/taskstats.txt)
* [Delay accounting](https://www.kernel.org/doc/Documentation/accounting/delay-accounting.txt)
* [Simple Firmware Interface](https://en.wikipedia.org/wiki/Simple_Firmware_Interface)
* [ftrace - Function Tracer](https://www.kernel.org/doc/Documentation/trace/ftrace.txt)
* [vDSO](https://en.wikipedia.org/wiki/VDSO)
* [SYSENTER](http://wiki.osdev.org/SYSENTER)
* [model-specific register (MSR)](https://en.wikipedia.org/wiki/Model-specific_register)



