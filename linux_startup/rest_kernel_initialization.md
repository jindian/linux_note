# rest kernel initialization

`start_kernel` calls `rest_init`, at this point things are almost all working. `rest_init` creates a kernel thread passing function `kernel_init` as the entry point. `rest_init` calls `schedule` to kickstart task scheduling and go to sleep by calling `cpu_idle`,  which is the idle thread for the Linux kernel. cpu\_idle\(\) runs forever and so does process zero, which hosts it. Whenever there is work to do ¨C a runnable process ¨C process zero gets booted out of the CPU, only to return when no runnable processes are available.

Detail steps of `rest_init`:

* Checks number of cpu and number of context switch; sets `rcu_scheduler_active` as 1

The rcu\_scheduler\_active variable transitions from zero to one just before the first task is spawned. So when this variable is zero, RCU can assume that there is but one task, allowing RCU to \(for example\) optimize synchronize\_sched\(\) to a simple barrier\(\). When this variable is one, RCU must actually do all the hard work required to detect real grace periods. This variable is also used to suppress boot-time false positives from lockdep-RCU error checking.

```
start_kernel () at init/main.c:694
694        rest_init();
(gdb) s
rest_init () at init/main.c:417
417        rcu_scheduler_starting();
(gdb) s
rcu_scheduler_starting () at kernel/rcupdate.c:184
184        WARN_ON(num_online_cpus() != 1);
(gdb) s
cpumask_weight (srcp=<optimized out>) at kernel/rcupdate.c:184
184        WARN_ON(num_online_cpus() != 1);
(gdb) s
bitmap_weight (nbits=8, src=<optimized out>) at include/linux/bitmap.h:263
263            return hweight_long(*src & BITMAP_LAST_WORD_MASK(nbits));
(gdb) s
hweight_long (w=<optimized out>) at include/linux/bitops.h:45
45        return sizeof(w) == 4 ? hweight32(w) : hweight64(w);
(gdb) s
hweight32 (w=1) at lib/hweight.c:14
14        unsigned int res = w - ((w >> 1) & 0x55555555);
(gdb) n
15        res = (res & 0x33333333) + ((res >> 2) & 0x33333333);
(gdb) 
16        res = (res + (res >> 4)) & 0x0F0F0F0F;
(gdb) 
17        res = res + (res >> 8);
(gdb) 
18        return (res + (res >> 16)) & 0x000000FF;
(gdb) 
19    }
(gdb) 
rcu_scheduler_starting () at kernel/rcupdate.c:184
185        WARN_ON(nr_context_switches() > 0);
(gdb) s
nr_context_switches () at kernel/sched.c:3088
3088        for_each_possible_cpu(i)
(gdb) n
3089            sum += cpu_rq(i)->nr_switches;
(gdb) 
3088        for_each_possible_cpu(i)
(gdb) 
3092    }
(gdb) 
rcu_scheduler_starting () at kernel/rcupdate.c:186
186        rcu_scheduler_active = 1;
(gdb) 
187    }
(gdb)
```

* Creates a kernel thread.

  1. `kernel_thread` initializes register parameters and invoke `do_fork` to create new process
     ```
     423        kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);
     (gdb) s
     kernel_thread (fn=fn@entry=0xc16fa7e8 <kernel_init>, arg=arg@entry=0x0, 
         flags=flags@entry=2560) at arch/x86/kernel/process_32.c:205
     205    {
     (gdb) n
     208        memset(&regs, 0, sizeof(regs));
     (gdb) 
     210        regs.bx = (unsigned long) fn;
     (gdb) 
     211        regs.dx = (unsigned long) arg;
     (gdb) 
     213        regs.ds = __USER_DS;
     (gdb) 
     214        regs.es = __USER_DS;
     (gdb) 
     215        regs.fs = __KERNEL_PERCPU;
     (gdb) 
     216        regs.gs = __KERNEL_STACK_CANARY;
     (gdb) 
     217        regs.orig_ax = -1;
     (gdb) 
     218        regs.ip = (unsigned long) kernel_thread_helper;
     (gdb) 
     219        regs.cs = __KERNEL_CS | get_kernel_rpl();
     (gdb) 
     220        regs.flags = X86_EFLAGS_IF | X86_EFLAGS_SF | X86_EFLAGS_PF | 0x2;
     (gdb) 
     223        return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL)
     ```
  2. `do_fork` does some preliminary argument and permissions checking before actually start allocating stuff
     ```
     223        return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
     (gdb) s
     do_fork (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
         child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1374
     1374    {
     (gdb) n
     1383        if (clone_flags & CLONE_NEWUSER) {
     (gdb) 
     1397        if (unlikely(clone_flags & CLONE_STOPPED)) {
     (gdb) 
     1414        if (likely(user_mode(regs)))
     (gdb) p *regs
     $2 = {bx = 3245320168, cx = 0, dx = 0, si = 0, di = 0, bp = 0, ax = 0, 
       ds = 123, es = 123, fs = 216, gs = 224, orig_ax = 4294967295, 
       ip = 3238017744, cs = 96, flags = 646, sp = 0, ss = 0}
     (gdb) s
     user_mode (regs=0xc168bf64 <init_thread_union+8036>)
         at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/ptrace.h:160
     160        return (regs->cs & SEGMENT_RPL_MASK) == USER_RPL;
     (gdb) n
     do_fork (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
         child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1414
     1414        if (likely(user_mode(regs)))
     (gdb) 
     1417        p = copy_process(clone_flags, stack_start, regs, stack_size,
     (gdb)
     ```
  3. `copy_process` is used to create a new process as a copy of old one, but doesn't actually start it yet. It copies the registers, and all the appropriate parts of the process environment\(as per clone flags\). 

     Before copying stuff, do some argument checking is executed, after that calls `security_task_create` which invokes `cap_task_create` of `default_security_ops`; 
     ```
     1417        p = copy_process(clone_flags, stack_start, regs, stack_size,
     (gdb) break copy_process
     Breakpoint 3 at 0xc1044a60: copy_process. (3 locations)
     (gdb) s
     copy_process (trace=0, pid=0x0, child_tidptr=0x0, stack_size=0, 
         regs=0xc168bf64 <init_thread_union+8036>, stack_start=0, 
         clone_flags=8391424) at kernel/fork.c:993
     993        if ((clone_flags & (CLONE_NEWNS|CLONE_FS)) == (CLONE_NEWNS|CLONE_FS))
     (gdb) n
     1000        if ((clone_flags & CLONE_THREAD) && !(clone_flags & CLONE_SIGHAND))
     (gdb) 
     1008        if ((clone_flags & CLONE_SIGHAND) && !(clone_flags & CLONE_VM))
     (gdb) 
     1017        if ((clone_flags & CLONE_PARENT) &&
     (gdb) 
     1021        retval = security_task_create(clone_flags);
     (gdb) s
     security_task_create (clone_flags=clone_flags@entry=8391424)
         at security/security.c:683
     683        return security_ops->task_create(clone_flags);
     (gdb) s
     cap_task_create (clone_flags=8391424) at security/capability.c:374
     374    }
     (gdb) n
     security_task_create (clone_flags=clone_flags@entry=8391424)
         at security/security.c:684
     684    }
     (gdb) p security_ops 
     $1 = (struct security_operations *) 0xc16aeee0 <default_security_ops>
     (gdb) n
     copy_process (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
         pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1022
     1022        if (retval)
     (gdb) 
 
     ```

     Duplicates task\_struct of `init_task`; 

     ```
     1026        p = dup_task_struct(current);
     (gdb) s
     get_current ()
         at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/current.h:14
     14        return percpu_read_stable(current_task);
     (gdb) n
     copy_process (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
         pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1026
     1026        p = dup_task_struct(current);
     (gdb) s
     dup_task_struct (orig=0xc1692440 <init_task>) at kernel/fork.c:230
     230        prepare_to_copy(orig);
     (gdb) s
     prepare_to_copy (tsk=tsk@entry=0xc1692440 <init_task>)
         at arch/x86/kernel/process_32.c:238
     238    {
     (gdb) p tsk
     $2 = (struct task_struct *) 0xc1692440 <init_task>   
     (gdb) p tsk->pid
     $3 = 0
     (gdb) n
     239        unlazy_fpu(tsk);
     (gdb) s
     unlazy_fpu (tsk=0xc1692440 <init_task>) at arch/x86/kernel/process_32.c:239
     239        unlazy_fpu(tsk);
     (gdb) s
     __unlazy_fpu (tsk=0xc1692440 <init_task>)
         at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/i387.h:274
     274        if (task_thread_info(tsk)->status & TS_USEDFPU) {
     (gdb) p ((struct thread_info *)(tsk)->stack)->status
     $4 = 1
     (gdb) n
     prepare_to_copy (tsk=tsk@entry=0xc1692440 <init_task>)
         at arch/x86/kernel/process_32.c:239
     239        unlazy_fpu(tsk);
     (gdb) s
     unlazy_fpu (tsk=0xc1692440 <init_task>) at arch/x86/kernel/process_32.c:239
     239        unlazy_fpu(tsk);
     (gdb) s
     __unlazy_fpu (tsk=0xc1692440 <init_task>) at arch/x86/kernel/process_32.c:239
     239        unlazy_fpu(tsk);
     (gdb) s
     __save_init_fpu (tsk=0xc1692440 <init_task>)
         at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/i387.h:211
     211        if (task_thread_info(tsk)->status & TS_XSAVE) {
     (gdb) n
     234        alternative_input(
     (gdb) 
     245        if (unlikely(boot_cpu_has(X86_FEATURE_FXSAVE_LEAK))) {
     (gdb) 
     253        task_thread_info(tsk)->status &= ~TS_USEDFPU;
     (gdb) 
     __unlazy_fpu (tsk=0xc1692440 <init_task>)
         at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/i387.h:276
     276            stts();
     (gdb) 
     prepare_to_copy (tsk=tsk@entry=0xc1692440 <init_task>)
         at arch/x86/kernel/process_32.c:240
     240    }
     (gdb) 
     dup_task_struct (orig=0xc1692440 <init_task>) at kernel/fork.c:232
     232        tsk = alloc_task_struct();
     (gdb) s
     kmem_cache_alloc (s=0xc70024b0, gfpflags=gfpflags@entry=208) at mm/slub.c:1749
     1749        void *ret = slab_alloc(s, gfpflags, -1, _RET_IP_);
     (gdb) n
     1751        trace_kmem_cache_alloc(_RET_IP_, ret, s->objsize, s->size, gfpflags);
     (gdb) 
     1754    }
     (gdb) 
     dup_task_struct (orig=0xc1692440 <init_task>) at kernel/fork.c:233
     233        if (!tsk)
     (gdb) 
     236        ti = alloc_thread_info(tsk);
     (gdb) s
     __get_free_pages (gfp_mask=gfp_mask@entry=32976, order=order@entry=1)
         at mm/page_alloc.c:1994
     1994        VM_BUG_ON((gfp_mask & __GFP_HIGHMEM) != 0);
     (gdb) n
     1996        page = alloc_pages(gfp_mask, order);
     (gdb) s
     alloc_pages_node (order=1, gfp_mask=32976, nid=0) at mm/page_alloc.c:1996
     1996        page = alloc_pages(gfp_mask, order);
     (gdb) s
     __alloc_pages (zonelist=0xc16dca60 <contig_page_data+5376>, order=1, 
         gfp_mask=32976) at include/linux/gfp.h:276
     276        return __alloc_pages_nodemask(gfp_mask, order, zonelist, NULL);
     (gdb) n
     __get_free_pages (gfp_mask=gfp_mask@entry=32976, order=order@entry=1)
         at mm/page_alloc.c:1997

     1997        if (!page)
     (gdb) 
     1999        return (unsigned long) page_address(page);
     (gdb) 
     2000    }
     (gdb) 
     dup_task_struct (orig=0xc1692440 <init_task>) at kernel/fork.c:237
     237        if (!ti) {
     (gdb) 
     242	 	err = arch_dup_task_struct(tsk, orig);
     (gdb) s
     arch_dup_task_struct (dst=0xc7070000, src=0xc1692440 <init_task>)
         at arch/x86/kernel/process.c:30
     30		*dst = *src;
     (gdb) 
     31		if (src->thread.xstate) {
     (gdb) p src->thread.xstate 
     $5 = (union thread_xstate *) 0xc7028000
     (gdb) n
     32			dst->thread.xstate = kmem_cache_alloc(task_xstate_cachep,
     (gdb) 
     34			if (!dst->thread.xstate)
     (gdb) 
     36			WARN_ON((unsigned long)dst->thread.xstate & 15);
     (gdb) 
     37			memcpy(dst->thread.xstate, src->thread.xstate, xstate_size);
     (gdb) 
     39		return 0;
     (gdb) 
     40	}
     (gdb) 
     dup_task_struct (orig=0xc1692440 <init_task>) at kernel/fork.c:243
     243		if (err)
     (gdb) 
     246		tsk->stack = ti;
     (gdb) 
     248		err = prop_local_init_single(&tsk->dirties);
     (gdb) s
     prop_local_init_single (pl=pl@entry=0xc7070d74) at lib/proportions.c:327
     327		spin_lock_init(&pl->lock);
     (gdb) n
     328		pl->shift = 0;
     (gdb) 
     329		pl->period = 0;
     (gdb) 
     330		pl->events = 0;
     (gdb) 
     332	}
     (gdb) 
     dup_task_struct (orig=0xc1692440 <init_task>) at kernel/fork.c:249
     249		if (err)
     (gdb) 
     252		setup_thread_stack(tsk, orig);
     (gdb) s
     setup_thread_stack (org=0xc1692440 <init_task>, p=0xc7070000)
         at include/linux/sched.h:2280
     2280		*task_thread_info(p) = *task_thread_info(org);
     (gdb) n
     2281		task_thread_info(p)->task = p;
     (gdb) 
     dup_task_struct (orig=<optimized out>) at kernel/fork.c:253
     253		stackend = end_of_stack(tsk);
     (gdb) 
     254		*stackend = STACK_END_MAGIC;	/* for overflow detection */
     (gdb) 
     257		tsk->stack_canary = get_random_int();
     (gdb) s
     get_random_int () at drivers/char/random.c:1481
     1481		if (arch_get_random_int(&ret))
     (gdb) s
     1484		hash = get_cpu_var(get_random_int_hash);
     (gdb) n
     1486		hash[0] += current->pid + jiffies + get_cycles();
     (gdb) 
     1487		md5_transform(hash, random_int_secret);
     (gdb) 
     1488		ret = hash[0];
     (gdb) p hash[0]
     $9 = 3166209240
     (gdb) n
     1492	}
     (gdb) 
     dup_task_struct (orig=<optimized out>) at kernel/fork.c:261
     261		atomic_set(&tsk->usage,2);
     (gdb) n
     262		atomic_set(&tsk->fs_excl, 0);
     (gdb) 
     264		tsk->btrace_seq = 0;
     (gdb) 
     266		tsk->splice_pipe = NULL;
     (gdb) 
     268		account_kernel_stack(ti, 1);
     (gdb) s
     account_kernel_stack (ti=ti@entry=0xc706a000, account=account@entry=1)
         at kernel/fork.c:144
     144		struct zone *zone = page_zone(virt_to_page(ti));
     (gdb) n
     146		mod_zone_page_state(zone, NR_KERNEL_STACK, account);
     (gdb) s
     mod_zone_page_state (zone=0xc16dbaa0 <contig_page_data+1344>, 
         item=item@entry=NR_KERNEL_STACK, delta=delta@entry=1) at mm/vmstat.c:184
     184	{
     (gdb) n
     187		local_irq_save(flags);
     (gdb) 
     188		__mod_zone_page_state(zone, item, delta);
     (gdb) s
     __mod_zone_page_state (zone=zone@entry=0xc16dbaa0 <contig_page_data+1344>, 
         item=item@entry=NR_KERNEL_STACK, delta=delta@entry=1) at mm/vmstat.c:165
     165		struct per_cpu_pageset *pcp = zone_pcp(zone, smp_processor_id());
     (gdb) n
     169		x = delta + *p;
     (gdb) 
     171		if (unlikely(x > pcp->stat_threshold || x < -pcp->stat_threshold)) {
     (gdb) p x
     $10 = 1
     (gdb) n
     172			zone_page_state_add(x, zone, item);
     (gdb) 
     173			x = 0;
     (gdb) 
     175		*p = x;
     (gdb) 
     176	}
     (gdb) 
     mod_zone_page_state (zone=0xc16dbaa0 <contig_page_data+1344>, 
         item=item@entry=NR_KERNEL_STACK, delta=delta@entry=1) at mm/vmstat.c:189
     189		local_irq_restore(flags);
     (gdb) 
     190	}
     (gdb) 
     account_kernel_stack (ti=ti@entry=0xc706a000, account=account@entry=1)
         at kernel/fork.c:147
     147	}
     (gdb) 
     ```
     
     Allocates return stack for newly created task if `ftrace_graph_active` is true; initializes lock for pi data.
     
     ```
     (gdb) 
     copy_process (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
         pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1030
     1030		ftrace_graph_init_task(p);
     (gdb) s
     ftrace_graph_init_task (t=t@entry=0xc7070000) at kernel/trace/ftrace.c:3313
     3313		t->ret_stack = NULL;
     (gdb) n
     3314		t->curr_ret_stack = -1;
     (gdb) 
     3316		if (ftrace_graph_active) {
     (gdb) 
     3326	}
     (gdb) 
     copy_process (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
         pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1032
     1032		rt_mutex_init_task(p);
     (gdb) s
     rt_mutex_init_task (p=<optimized out>) at kernel/fork.c:946
     946		spin_lock_init(&p->pi_lock);
     (gdb) n
     948		plist_head_init(&p->pi_waiters, &p->pi_lock);
     (gdb) s
     plist_head_init (lock=0xc7070468, head=<optimized out>)
         at include/linux/plist.h:133
     133		INIT_LIST_HEAD(&head->prio_list);
     (gdb) n
     134		INIT_LIST_HEAD(&head->node_list);
     (gdb) 
     136		head->lock = lock;
     (gdb) 
     rt_mutex_init_task (p=<optimized out>) at kernel/fork.c:949
     949		p->pi_blocked_on = NULL;
     (gdb) 
     copy_process (clone_flags=clone_flags@entry=8391424, 
         stack_start=stack_start@entry=0, 
         regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
         stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
         pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1035
     1035		DEBUG_LOCKS_WARN_ON(!p->hardirqs_enabled);
     (gdb) 
     1036		DEBUG_LOCKS_WARN_ON(!p->softirqs_enabled);
     (gdb) 
     1039		if (atomic_read(&p->real_cred->user->processes) >=
     (gdb) 
     1040				p->signal->rlim[RLIMIT_NPROC].rlim_cur) {
     (gdb) 
     1039		if (atomic_read(&p->real_cred->user->processes) >=
     (gdb) 

     ```

     Copies credentials for the new process
     
     ```
     1046		retval = copy_creds(p, clone_flags);
     (gdb) s
     copy_creds (p=p@entry=0xc7070000, clone_flags=clone_flags@entry=8391424)
         at kernel/cred.c:444
     444		mutex_init(&p->cred_guard_mutex);
     (gdb) s
     437	{
     (gdb) n
     444		mutex_init(&p->cred_guard_mutex);
     (gdb) s
     __mutex_init (lock=lock@entry=0xc70702e0, 
         name=name@entry=0xc15d671b "&p->cred_guard_mutex", 
         key=key@entry=0xc190e45c <__key.26889>) at kernel/mutex.c:50
     50	{
     (gdb) n
     51		atomic_set(&lock->count, 1);
     (gdb) 
     52		spin_lock_init(&lock->wait_lock);
     (gdb) 
     53		INIT_LIST_HEAD(&lock->wait_list);
     (gdb) 
     54		mutex_clear_owner(lock);
     (gdb) 
     56		debug_mutex_init(lock, name, key);
     (gdb) 
     57	}
     (gdb) 
     copy_creds (p=p@entry=0xc7070000, clone_flags=clone_flags@entry=8391424)
         at kernel/cred.c:446
     446		p->replacement_session_keyring = NULL;
     (gdb) 
     448		if (
     (gdb) 
     450			!p->cred->thread_keyring &&
     (gdb) 
     464		new = prepare_creds();
     (gdb) s
     prepare_creds () at kernel/cred.c:292
     292		validate_process_creds();
     (gdb) n
     288		struct task_struct *task = current;
     (gdb) 
     292		validate_process_creds();
     (gdb) 
     294		new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
     (gdb) 
     295		if (!new)
     (gdb) 
     300		old = task->cred;
     (gdb) 
     301		memcpy(new, old, sizeof(struct cred));
     (gdb) 
     303		atomic_set(&new->usage, 1);
     (gdb) 
     305		get_group_info(new->group_info);
     (gdb) 
     306		get_uid(new->user);
     (gdb) 
     309		key_get(new->thread_keyring);
     (gdb) 
     310		key_get(new->request_key_auth);
     (gdb) 
     311		atomic_inc(&new->tgcred->usage);
     (gdb) 
     315		new->security = NULL;
     (gdb) 
     318		if (security_prepare_creds(new, old, GFP_KERNEL) < 0)
     (gdb) 
     320		validate_creds(new);
     (gdb) 
     326	}
     (gdb) 
     copy_creds (p=p@entry=0xc7070000, clone_flags=clone_flags@entry=8391424)
         at kernel/cred.c:465
     465		if (!new)
     (gdb) 
     468		if (clone_flags & CLONE_NEWUSER) {
     (gdb) 
     477		if (new->thread_keyring) {
     (gdb) p new->thread_keyring 
     $11 = (struct key *) 0x0
     (gdb) n
     487		if (!(clone_flags & CLONE_THREAD)) {
     (gdb) 
     488			tgcred = kmalloc(sizeof(*tgcred), GFP_KERNEL);
     (gdb) 
     489			if (!tgcred) {
     (gdb) 
     493			atomic_set(&tgcred->usage, 1);
     (gdb) 
     494			spin_lock_init(&tgcred->lock);
     (gdb) 
     495			tgcred->process_keyring = NULL;
     (gdb) 
     496			tgcred->session_keyring = key_get(new->tgcred->session_keyring);
     (gdb) 
     498			release_tgcred(new);
     (gdb) 
     499			new->tgcred = tgcred;
     (gdb) 
     503		atomic_inc(&new->user->processes);
     (gdb) 
     504		p->cred = p->real_cred = get_cred(new);
     (gdb) 
     505		alter_cred_subscribers(new, 2);
     (gdb) 
     506		validate_creds(new);
     (gdb) 
     507		return 0;
     (gdb) 
     512	}
     ```

 

# Links

* [User-space lockdep](https://lwn.net/Articles/536363/)
* [The kernel lock validator](https://lwn.net/Articles/185666/)
* [Using the TRACE\_EVENT\(\) macro \(Part 1\)](https://lwn.net/Articles/379903/)
* [Using the TRACE\_EVENT\(\) macro \(Part 2\)](https://lwn.net/Articles/381064/)
* [Using the TRACE\_EVENT\(\) macro \(Part 3\)](https://lwn.net/Articles/383362/)



