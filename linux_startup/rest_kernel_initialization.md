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

1. `kernel_thread` initializes register parameters and invokes `do_fork` to create init process
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

Initializes rest of task variables

```rest_of_task_var

(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1047
1047		if (retval < 0)
(gdb) 
1056		if (nr_threads >= max_threads)
(gdb) 
1059		if (!try_module_get(task_thread_info(p)->exec_domain->module))
(gdb) 
1062		p->did_exec = 0;
(gdb) 
1063		delayacct_tsk_init(p);	/* Must remain after dup_task_struct() */
(gdb) s
delayacct_tsk_init (tsk=<optimized out>) at include/linux/delayacct.h:67
67		tsk->delays = NULL;
(gdb) n
68		if (delayacct_on)
(gdb) 
69			__delayacct_tsk_init(tsk);
(gdb) s
__delayacct_tsk_init (tsk=tsk@entry=0xc7070000) at kernel/delayacct.c:41
41		tsk->delays = kmem_cache_zalloc(delayacct_cache, GFP_KERNEL);
(gdb) n
42		if (tsk->delays)
(gdb) 
43			spin_lock_init(&tsk->delays->lock);
(gdb) 
44	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1064
1064		copy_flags(clone_flags, p);
(gdb) s
copy_flags (clone_flags=<optimized out>, p=<optimized out>)
	 at kernel/fork.c:928
928		unsigned long new_flags = p->flags;
(gdb) n
930		new_flags &= ~PF_SUPERPRIV;
(gdb) 
932		new_flags |= PF_STARTING;
(gdb) 
934		clear_freeze_flag(p);
(gdb) s
clear_freeze_flag (p=<optimized out>) at kernel/fork.c:934
934		clear_freeze_flag(p);
(gdb) s
clear_tsk_thread_flag (flag=23, tsk=<optimized out>) at kernel/fork.c:934
934		clear_freeze_flag(p);
(gdb) s
clear_ti_thread_flag (flag=23, ti=0xc706a000) at kernel/fork.c:934
934		clear_freeze_flag(p);
(gdb) s
clear_bit (addr=0xc706a008, nr=23)
	 at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/bitops.h:101
101			asm volatile(LOCK_PREFIX "andb %1,%0"
(gdb) n
102				: CONST_MASK_ADDR(nr, addr)
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1065
1065		INIT_LIST_HEAD(&p->children);
(gdb) 
1066		INIT_LIST_HEAD(&p->sibling);
(gdb) 
1069		spin_lock_init(&p->alloc_lock);
(gdb) 
1068		p->vfork_done = NULL;
(gdb) 
1069		spin_lock_init(&p->alloc_lock);
(gdb) 
1071		init_sigpending(&p->pending);
(gdb) 
1073		p->utime = cputime_zero;
(gdb) 
1074		p->stime = cputime_zero;
(gdb) 
1075		p->gtime = cputime_zero;
(gdb) 
1076		p->utimescaled = cputime_zero;
(gdb) 
1077		p->stimescaled = cputime_zero;
(gdb) 
1078		p->prev_utime = cputime_zero;
(gdb) 
1079		p->prev_stime = cputime_zero;
(gdb) 
1081		p->default_timer_slack_ns = current->timer_slack_ns;
(gdb) 
1083		task_io_accounting_init(&p->ioac);
(gdb) s
task_io_accounting_init (ioac=<optimized out>) at kernel/fork.c:1083
1083		task_io_accounting_init(&p->ioac);
(gdb) s
	__constant_c_and_count_memset (count=56, pattern=0, s=0xc7070c90)
	 at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/string_32.h:291
291				COMMON("");
(gdb) n
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1084
1084		acct_clear_integrals(p);
(gdb) s
acct_clear_integrals (tsk=tsk@entry=0xc7070000) at kernel/tsacct.c:151
151		tsk->acct_timexpd = 0;
(gdb) n
152		tsk->acct_rss_mem1 = 0;
(gdb) 
153		tsk->acct_vm_mem1 = 0;
(gdb) 
154	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1086
1086		posix_cpu_timers_init(p);
(gdb) s
   posix_cpu_timers_init (tsk=<optimized out>) at kernel/fork.c:965
   965		tsk->cputime_expires.prof_exp = cputime_zero;
(gdb) n
966		tsk->cputime_expires.virt_exp = cputime_zero;
(gdb) 
967		tsk->cputime_expires.sched_exp = 0;
(gdb) 
968		INIT_LIST_HEAD(&tsk->cpu_timers[0]);
(gdb) n
969		INIT_LIST_HEAD(&tsk->cpu_timers[1]);
(gdb) 
970		INIT_LIST_HEAD(&tsk->cpu_timers[2]);
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1088
1088		p->lock_depth = -1;		/* -1 = no lock */
(gdb) 
1089		do_posix_clock_monotonic_gettime(&p->start_time);
(gdb) s
ktime_get_ts (ts=ts@entry=0xc7070298) at kernel/time/timekeeping.c:313
313		WARN_ON(timekeeping_suspended);
(gdb) s
308	{
(gdb) n
313		WARN_ON(timekeeping_suspended);
(gdb) 
316			seq = read_seqbegin(&xtime_lock);
(gdb) 
317			*ts = xtime;
(gdb) 
319			nsecs = timekeeping_get_ns();
(gdb) 
317			*ts = xtime;
(gdb) 
318			tomono = wall_to_monotonic;
(gdb) 
319			nsecs = timekeeping_get_ns();
(gdb) 
323		} while (read_seqretry(&xtime_lock, seq));

......

325		set_normalized_timespec(ts, ts->tv_sec + tomono.tv_sec,
(gdb) 
326					ts->tv_nsec + tomono.tv_nsec + nsecs);
(gdb) 
(gdb) 
327	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1090
1090		p->real_start_time = p->start_time;
1091		monotonic_to_bootbased(&p->real_start_time);
(gdb) 
1092		p->io_context = NULL;
(gdb) n
1093		p->audit_context = NULL;
(gdb) 
1094		cgroup_fork(p);
(gdb) s
cgroup_fork (child=child@entry=0xc7070000) at kernel/cgroup.c:3421
3421		task_lock(current);
(gdb) n
3422		child->cgroups = current->cgroups;
(gdb) 
3423		get_css_set(child->cgroups);
(gdb) 
3424		task_unlock(current);
(gdb) 
3425		INIT_LIST_HEAD(&child->cg_list);
(gdb) 
3426	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1105
1105		p->irq_events = 0;
(gdb) 
1109		p->hardirqs_enabled = 0;
(gdb) 
1111		p->hardirq_enable_ip = 0;
(gdb) 
1112		p->hardirq_enable_event = 0;
(gdb) 
1113		p->hardirq_disable_ip = _THIS_IP_;
(gdb) 
1114		p->hardirq_disable_event = 0;
(gdb) 
1115		p->softirqs_enabled = 1;
(gdb) 
1116		p->softirq_enable_ip = _THIS_IP_;
(gdb) 
1117		p->softirq_enable_event = 0;
(gdb) 
1118		p->softirq_disable_ip = 0;
(gdb) 
1119		p->softirq_disable_event = 0;
(gdb) 
1120		p->hardirq_context = 0;
(gdb) 
1121		p->softirq_context = 0;
(gdb) 
1124		p->lockdep_depth = 0; /* no locks held yet */
(gdb) 
1125		p->curr_chain_key = 0;
(gdb) 
1126		p->lockdep_recursion = 0;
(gdb) 
1130		p->blocked_on = NULL; /* not blocked yet */
(gdb) 
1133		p->bts = NULL;
(gdb) 
1136		sched_fork(p, clone_flags);
sched_fork (p=p@entry=0xc7070000, 
	 clone_flags=clone_flags@entry=8391424) at kernel/sched.c:2696
2696		int cpu = get_cpu();
(gdb) n
2698		__sched_fork(p);
(gdb) 
2704		p->state = TASK_RUNNING;
(gdb) 
2709		if (unlikely(p->sched_reset_on_fork)) {
(gdb) 
2731		p->prio = current->normal_prio;
(gdb) 
2733		if (!rt_prio(p->prio))
(gdb) 
2734			p->sched_class = &fair_sched_class;
(gdb) 
2736		if (p->sched_class->task_fork)
(gdb) 
2737			p->sched_class->task_fork(p);
(gdb) 
2739		set_task_cpu(p, cpu);
(gdb) 
2743			memset(&p->sched_info, 0, sizeof(p->sched_info));
(gdb) 
2752		plist_node_init(&p->pushable_tasks, MAX_PRIO);
(gdb) 
2755	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1138
1138		retval = perf_event_init_task(p);
(gdb) s
perf_event_init_task (child=child@entry=0xc7070000) at kernel/perf_event.c:4900
4900		mutex_init(&child->perf_event_mutex);
(gdb) n
4901		INIT_LIST_HEAD(&child->perf_event_list);
(gdb) 
4903		if (likely(!parent->perf_event_ctxp))
(gdb) 
4904			return 0;
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1139
1139		if (retval)
(gdb) 
1142		if ((retval = audit_alloc(p)))
(gdb) s
audit_alloc (tsk=tsk@entry=0xc7070000) at kernel/auditsc.c:887
887		if (likely(!audit_ever_enabled))
(gdb) n
903		return 0;
(gdb) 
904	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1145
1145		if ((retval = copy_semundo(clone_flags, p)))
(gdb) s
copy_semundo (clone_flags=clone_flags@entry=8391424, tsk=tsk@entry=0xc7070000)
	 at ipc/sem.c:1254
1254		if (clone_flags & CLONE_SYSVSEM) {
(gdb) n
1261			tsk->sysvsem.undo_list = NULL;
(gdb) 
1263		return 0;
(gdb) 
1264	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1147
1147		if ((retval = copy_files(clone_flags, p)))
(gdb) 
1149		if ((retval = copy_fs(clone_flags, p)))
(gdb) 
1151		if ((retval = copy_sighand(clone_flags, p)))
(gdb) 
1153		if ((retval = copy_signal(clone_flags, p)))
(gdb) 
1155		if ((retval = copy_mm(clone_flags, p)))
(gdb) 
1157		if ((retval = copy_namespaces(clone_flags, p)))
(gdb) 
1157		if ((retval = copy_namespaces(clone_flags, p)))
(gdb) 
1159		if ((retval = copy_io(clone_flags, p)))
(gdb) s
copy_io (tsk=0xc7070000, clone_flags=8391424) at kernel/fork.c:1159
1159		if ((retval = copy_io(clone_flags, p)))
(gdb) s
get_current ()
	 at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/current.h:14
14		return percpu_read_stable(current_task);
(gdb) n
copy_io (tsk=0xc7070000, clone_flags=8391424) at kernel/fork.c:778
778		struct io_context *ioc = current->io_context;
(gdb) n
780		if (!ioc)
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1161
1161		retval = copy_thread(clone_flags, stack_start, stack_size, p, regs);
(gdb) s
copy_thread (clone_flags=clone_flags@entry=8391424, sp=sp@entry=0, 
	 unused=unused@entry=0, p=p@entry=0xc7070000, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>)
	 at arch/x86/kernel/process_32.c:250
250		childregs = task_pt_regs(p);
(gdb) n
251		*childregs = *regs;
(gdb) 
252		childregs->ax = 0;
(gdb) 
253		childregs->sp = sp;
(gdb) 
256		p->thread.sp0 = (unsigned long) (childregs+1);
(gdb) 
258		p->thread.ip = (unsigned long) ret_from_fork;
(gdb) 
260		task_user_gs(p) = get_user_gs(regs);
(gdb) 
262		tsk = current;
(gdb) 
263		if (unlikely(test_tsk_thread_flag(tsk, TIF_IO_BITMAP))) {
(gdb) 
273		err = 0;
(gdb) 
278		if (clone_flags & CLONE_SETTLS)
(gdb) 
287		clear_tsk_thread_flag(p, TIF_DS_AREA_MSR);
(gdb) 
288		p->thread.ds_ctx = NULL;
(gdb) 
290		clear_tsk_thread_flag(p, TIF_DEBUGCTLMSR);
(gdb) 
291		p->thread.debugctlmsr = 0;
(gdb) 
293		return err;
(gdb) 
294	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1162
1162		if (retval)
(gdb) 
1165		if (pid != &init_struct_pid) {
(gdb) p init_struct_pid 
$1 = {count = {counter = 1}, level = 0, tasks = {{
	   first = 0xc169267c <init_task+572>}, {
	   first = 0xc1692688 <init_task+584>}, {
	   first = 0xc1692694 <init_task+596>}}, rcu = {next = 0x0, func = 0x0}, 
  numbers = {{nr = 0, ns = 0xc169dba0 <init_pid_ns>, pid_chain = {next = 0x0, 
		 pprev = 0x0}}}}
(gdb) p pid
$2 = (struct pid *) 0x0
(gdb) p &init_struct_pid
$3 = (struct pid *) 0xc169dbe0 <init_struct_pid>
(gdb) n
1167			pid = alloc_pid(p->nsproxy->pid_ns);
(gdb) 
1168			if (!pid)
(gdb) 
1171			if (clone_flags & CLONE_NEWPID) {
(gdb) 
1178		p->pid = pid_nr(pid);
(gdb) 
1179		p->tgid = p->pid;
(gdb) 
1180		if (clone_flags & CLONE_THREAD)
(gdb) 
1189		p->set_child_tid = (clone_flags & CLONE_CHILD_SETTID) ? child_tidptr : NULL;
(gdb) 
1193		p->clear_child_tid = (clone_flags & CLONE_CHILD_CLEARTID) ? child_tidptr: NULL;
(gdb) 
1195		p->robust_list = NULL;
(gdb) 
1199		INIT_LIST_HEAD(&p->pi_state_list);
(gdb) 
1200		p->pi_state_cache = NULL;
(gdb) 
1205		if ((clone_flags & (CLONE_VM|CLONE_VFORK)) == CLONE_VM)
(gdb) 
1206			p->sas_ss_sp = p->sas_ss_size = 0;
(gdb) 
1212		clear_tsk_thread_flag(p, TIF_SYSCALL_TRACE);
(gdb) 
1214		clear_tsk_thread_flag(p, TIF_SYSCALL_EMU);
(gdb) 
1216		clear_all_latency_tracing(p);
(gdb) s
clear_all_latency_tracing (p=p@entry=0xc7070000) at kernel/latencytop.c:70
70	{
(gdb) n
73		if (!latencytop_enabled)
(gdb) 
80	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=0xc7066000, pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1219
1219		p->exit_signal = (clone_flags & CLONE_THREAD) ? -1 : (clone_flags & CSIGNAL);
(gdb) 
1220		p->pdeath_signal = 0;
(gdb) 
1221		p->exit_state = 0;
(gdb) 
1227		p->group_leader = p;
(gdb)
1228		INIT_LIST_HEAD(&p->thread_group);
(gdb) 
1233		cgroup_fork_callbacks(p);
(gdb) s
cgroup_fork_callbacks (child=child@entry=0xc7070000) at kernel/cgroup.c:3438
3438		if (need_forkexit_callback) {
(gdb) n
3437	{
(gdb) 
3438		if (need_forkexit_callback) {
(gdb) 
3441				struct cgroup_subsys *ss = subsys[i];
(gdb) 
3442				if (ss->fork)
(gdb) p subsys
$4 = {0xc16a1500 <cpuset_subsys>, 0xc1697d80 <cpuacct_subsys>, 
  0xc16a0c60 <freezer_subsys>}
(gdb) n
3440			for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3441				struct cgroup_subsys *ss = subsys[i];
(gdb) 
3442				if (ss->fork)
(gdb) 
3440			for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3441				struct cgroup_subsys *ss = subsys[i];
(gdb) 
3442				if (ss->fork)
(gdb) 
3443					ss->fork(ss, child);
(gdb) 
3440			for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3446	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=0xc7066000, pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1237
1237		write_lock_irq(&tasklist_lock);
(gdb) 
1240		if (clone_flags & (CLONE_PARENT|CLONE_THREAD)) {
(gdb) 
1244			p->real_parent = current;
(gdb) 
1245			p->parent_exec_id = current->self_exec_id;
(gdb) 
1248		spin_lock(&current->sighand->siglock);
(gdb) 
1258		recalc_sigpending();
(gdb) 
1259		if (signal_pending(current)) {
(gdb) 
1266		if (clone_flags & CLONE_THREAD) {
(gdb) 
1273		if (likely(p->pid)) {
(gdb) 
1274			list_add_tail(&p->sibling, &p->real_parent->children);
(gdb) 
1275			tracehook_finish_clone(p, clone_flags, trace);
(gdb) 
1277			if (thread_group_leader(p)) {
(gdb) 
1278				if (clone_flags & CLONE_NEWPID)
(gdb) 
1281				p->signal->leader_pid = pid;
(gdb) 
1282				tty_kref_put(p->signal->tty);
(gdb) 
1283				p->signal->tty = tty_kref_get(current->signal->tty);
(gdb) 
1284				attach_pid(p, PIDTYPE_PGID, task_pgrp(current));
(gdb) 
1285				attach_pid(p, PIDTYPE_SID, task_session(current));
(gdb) 
1286				list_add_tail_rcu(&p->tasks, &init_task.tasks);
(gdb) 
1287				__get_cpu_var(process_counts)++;
(gdb) 
1289			attach_pid(p, PIDTYPE_PID, pid);
(gdb) 
1290			nr_threads++;
(gdb) 
1293		total_forks++;
(gdb) 
1294		spin_unlock(&current->sighand->siglock);
(gdb) p nr_threads 
$5 = 1
(gdb) p total_forks 
$7 = 1
(gdb) n
1295		write_unlock_irq(&tasklist_lock);
(gdb) 
1296		proc_fork_connector(p);
(gdb) s
proc_fork_connector (task=task@entry=0xc7070000)
	 at drivers/connector/cn_proc.c:51
51	{
(gdb) n
57		if (atomic_read(&proc_event_num_listeners) < 1)
(gdb) 
78	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=0xc7066000, pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1297
1297		cgroup_post_fork(p);
(gdb) s
cgroup_post_fork (child=child@entry=0xc7070000) at kernel/cgroup.c:3458
3458	{
(gdb) n
3459		if (use_task_css_set_links) {
(gdb) 
3467	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=0xc7066000, pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1298
1298		perf_event_fork(p);
(gdb) s
perf_event_fork (task=task@entry=0xc7070000) at kernel/perf_event.c:3310
3310		perf_event_task(task, NULL, 1);
(gdb) s
perf_event_task (task=task@entry=0xc7070000, task_ctx=task_ctx@entry=0x0, 
	 new=new@entry=1) at kernel/perf_event.c:3284
3284		if (!atomic_read(&nr_comm_events) &&
(gdb) n
3285		    !atomic_read(&nr_mmap_events) &&
(gdb) 
3286		    !atomic_read(&nr_task_events))
(gdb) 
3306	}
(gdb) 
perf_event_fork (task=task@entry=0xc7070000) at kernel/perf_event.c:3311
3311	}
(gdb) 
copy_process (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, child_tidptr=child_tidptr@entry=0x0, 
	 pid=0xc7066000, pid@entry=0x0, trace=trace@entry=0) at kernel/fork.c:1341
1341	}
(gdb) 
do_fork (clone_flags=clone_flags@entry=8391424, 
	 stack_start=stack_start@entry=0, 
	 regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
	 stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
	 child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1423
1423		if (!IS_ERR(p)) {
```

4. After new task created successfully, wake it up if clone flag doesn't configured as start in stopped state.

```
1423		if (!IS_ERR(p)) {
(gdb) 
1426			trace_sched_process_fork(current, p);
(gdb) 
1428			nr = task_pid_vnr(p);
(gdb) s
task_pid_vnr (tsk=<optimized out>) at include/linux/sched.h:1639
1639		return __task_pid_nr_ns(tsk, PIDTYPE_PID, NULL);
(gdb) s
__task_pid_nr_ns (task=task@entry=0xc7070000, type=type@entry=PIDTYPE_PID, 
    ns=ns@entry=0x0) at kernel/pid.c:449
449	{
(gdb) n
452		rcu_read_lock();
(gdb) 
453		if (!ns)
(gdb) 
454			ns = current->nsproxy->pid_ns;
(gdb) 
455		if (likely(pid_alive(task))) {
(gdb) p ns
$1 = (struct pid_namespace *) 0xc169dba0 <init_pid_ns>
(gdb) p task->pids[PIDTYPE_PID].pid
$2 = (struct pid *) 0xc7066000
(gdb) n
456			if (type != PIDTYPE_PID)
(gdb) 
458			nr = pid_nr_ns(task->pids[type].pid, ns);
(gdb) s
pid_nr_ns (ns=0xc169dba0 <init_pid_ns>, pid=0xc7066000) at kernel/pid.c:433
433		if (pid && ns->level <= pid->level) {
(gdb) p ns
$3 = (struct pid_namespace *) 0xc169dba0 <init_pid_ns>
(gdb) p ns->level 
$4 = 0
(gdb) p pid->level 
$5 = 0
(gdb) n
434			upid = &pid->numbers[ns->level];
(gdb) 
435			if (upid->ns == ns)
(gdb) p upid
$6 = (struct upid *) 0xc706601c
(gdb) p *upid
$7 = {nr = 1, ns = 0xc169dba0 <init_pid_ns>, pid_chain = {next = 0x0, 
    pprev = 0xc2129278}}
(gdb) n
436				nr = upid->nr;
(gdb) n
__task_pid_nr_ns (task=<optimized out>, task@entry=0xc7070000, 
    type=type@entry=PIDTYPE_PID, ns=0xc169dba0 <init_pid_ns>, ns@entry=0x0)
    at kernel/pid.c:460
460		rcu_read_unlock();
(gdb) 
463	}
do_fork (clone_flags=clone_flags@entry=8391424, 
    stack_start=stack_start@entry=0, 
    regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
    stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
    child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1430
1430			if (clone_flags & CLONE_PARENT_SETTID)
(gdb) n
1433			if (clone_flags & CLONE_VFORK) {
(gdb) 
1438			audit_finish_fork(p);
(gdb) s
audit_finish_fork (child=child@entry=0xc7070000) at kernel/auditsc.c:1661
1661		struct audit_context *ctx = current->audit_context;
(gdb) n
1662		struct audit_context *p = child->audit_context;
(gdb) 
1663		if (!p || !ctx)
(gdb) 
1677	}
(gdb) 
do_fork (clone_flags=clone_flags@entry=8391424, 
    stack_start=stack_start@entry=0, 
    regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
    stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
    child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1439
1439			tracehook_report_clone(regs, clone_flags, nr, p);
(gdb) 
1447			p->flags &= ~PF_STARTING;
(gdb) 
1449			if (unlikely(clone_flags & CLONE_STOPPED)) {
(gdb) 
1457				wake_up_new_task(p, clone_flags);
(gdb) s
wake_up_new_task (p=p@entry=0xc7070000, clone_flags=clone_flags@entry=8391424)
    at kernel/sched.c:2771
2771		rq = task_rq_lock(p, &flags);
(gdb) n
2772		p->state = TASK_WAKING;
(gdb) 
2782		cpu = select_task_rq(rq, p, SD_BALANCE_FORK, 0);
(gdb) 
2783		set_task_cpu(p, cpu);
(gdb) 
2785		p->state = TASK_RUNNING;
(gdb) 
2786		task_rq_unlock(rq, &flags);
(gdb) 
2789		rq = task_rq_lock(p, &flags);
(gdb) 
2790		update_rq_clock(rq);
(gdb) 
2791		activate_task(rq, p, 0);
(gdb) 
2792		trace_sched_wakeup_new(rq, p, 1);
(gdb) 
2793		check_preempt_curr(rq, p, WF_FORK);
(gdb) 
2795		if (p->sched_class->task_woken)
(gdb) 
2798		task_rq_unlock(rq, &flags);
(gdb) 
2800	}
(gdb) 
do_fork (clone_flags=clone_flags@entry=8391424, 
    stack_start=stack_start@entry=0, 
    regs=regs@entry=0xc168bf64 <init_thread_union+8036>, 
    stack_size=stack_size@entry=0, parent_tidptr=parent_tidptr@entry=0x0, 
    child_tidptr=child_tidptr@entry=0x0) at kernel/fork.c:1460
1460			tracehook_report_clone_complete(trace, regs,
(gdb) 
1463			if (clone_flags & CLONE_VFORK) {
(gdb) 
1473	}
(gdb) 
kernel_thread (fn=fn@entry=0xc16fa7e8 <kernel_init>, arg=arg@entry=0x0, 
    flags=flags@entry=2560) at arch/x86/kernel/process_32.c:224
224	}
(gdb) 
```

* NUMA configuration doesn't not enable in my enviroment, defination of `numa_default_policy` is NULL

* Creates kthreadd process, [kthreadd] is the kernel thread daemon.  All kthreads are forked off this thread.  To demonstrate this fact, run a ps -ef.  The other kthreads on the server will have a PPID of [kthreadd] which is usually PPID 2.
  Creation of new kernel threads is done via kthreadd so that a clean environment is obtained even if this were to be invoked by userspace by way of modprobe, hotplug cpu, etc.
  For more detail, the interface for the creation of kernel threads is declared in <linux/kthread.h>.
  Create kthreadd is similar with the procedure of init process creation.
  
```kthreadd
425		pid = kernel_thread(kthreadd, NULL, CLONE_FS | CLONE_FILES);
(gdb) 
426		kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
(gdb) p pid
$1 = 2
```

* Find kthreadd task by pid and pdi namespace

```find_task_by_pid_ns
426		kthreadd_task = find_task_by_pid_ns(pid, &init_pid_ns);
(gdb) p pid
$1 = 2
(gdb) s
find_task_by_pid_ns (nr=2, ns=0xc169dba0 <init_pid_ns>) at kernel/pid.c:386
386		return pid_task(find_pid_ns(nr, ns), PIDTYPE_PID);
(gdb) s
find_pid_ns (nr=2, ns=0xc169dba0 <init_pid_ns>) at kernel/pid.c:299
299		hlist_for_each_entry_rcu(pnr, elem,
(gdb) n
301			if (pnr->nr == nr && pnr->ns == ns)
(gdb) 
302				return container_of(pnr, struct pid,
(gdb) p pnr->nr
$2 = 2
(gdb) p pnr->ns
$3 = (struct pid_namespace *) 0xc169dba0 <init_pid_ns>
(gdb) n
306	}
(gdb) 
find_task_by_pid_ns (nr=<optimized out>, ns=<optimized out>)
    at kernel/pid.c:386
386		return pid_task(find_pid_ns(nr, ns), PIDTYPE_PID);
(gdb) s
pid_task (type=PIDTYPE_PID, pid=0xc7066060) at kernel/pid.c:371
371		if (pid) {
(gdb) p pid
$4 = (struct pid *) 0xc7066060
(gdb) p *pid
$5 = {count = {counter = 1}, level = 0, tasks = {{first = 0xc70717bc}, {
      first = 0x0}, {first = 0x0}}, rcu = {next = 0x6b6b6b6b, 
    func = 0x6b6b6b6b}, numbers = {{nr = 2, ns = 0xc169dba0 <init_pid_ns>, 
      pid_chain = {next = 0x0, pprev = 0xc2129768}}}}
(gdb) n
373			first = rcu_dereference(pid->tasks[type].first);
(gdb) 
374			if (first)
(gdb) p first
$6 = (struct hlist_node *) 0xc70717bc
(gdb) n
375				result = hlist_entry(first, struct task_struct, pids[(type)].node);
(gdb) 
find_task_by_pid_ns (nr=<optimized out>, ns=<optimized out>)
    at kernel/pid.c:387
387	}
(gdb) 
rest_init () at init/main.c:427
427		complete(&kthreadd_done);
(gdb) p *kthreadd_task
$9 = {state = 0, stack = 0xc706e000, usage = {counter = 2}, flags = 2105408, 
  ptrace = 0, lock_depth = -1, prio = 120, static_prio = 120, 
  normal_prio = 120, rt_priority = 0, 
  sched_class = 0xc1497b20 <fair_sched_class>, se = {load = {weight = 1024, 
      inv_weight = 4194304}, run_node = {rb_parent_color = 3339124788, 
      rb_right = 0x0, rb_left = 0x0}, group_node = {next = 0xc7070040, 
      prev = 0xc234a35c}, on_rq = 1, exec_start = 0, sum_exec_runtime = 0, 
    vruntime = 1450203, prev_sum_exec_runtime = 0, last_wakeup = 0, 
    avg_overlap = 0, nr_migrations = 0, start_runtime = 0, 
    avg_wakeup = 1000000, avg_running = 0, wait_start = 211460260, 
    wait_max = 0, wait_count = 0, wait_sum = 0, iowait_count = 0, 
    iowait_sum = 0, sleep_start = 0, sleep_max = 0, sum_sleep_runtime = 0, 
    block_start = 0, block_max = 0, exec_max = 0, slice_max = 0, 
    nr_migrations_cold = 0, nr_failed_migrations_affine = 0, 
    nr_failed_migrations_running = 0, nr_failed_migrations_hot = 0, 
    nr_forced_migrations = 0, nr_wakeups = 0, nr_wakeups_sync = 0, 
    nr_wakeups_migrate = 0, nr_wakeups_local = 0, nr_wakeups_remote = 0, 
    nr_wakeups_affine = 0, nr_wakeups_affine_attempts = 0, 
    nr_wakeups_passive = 0, nr_wakeups_idle = 0}, rt = {run_list = {
      next = 0xc70716f4, prev = 0xc70716f4}, timeout = 0, time_slice = 250, 
    nr_cpus_allowed = 8, back = 0x0}, preempt_notifiers = {first = 0x0}, 
  fpu_counter = 0 '\000', btrace_seq = 0, policy = 0, cpus_allowed = {bits = {
      1}}, sched_info = {pcount = 0, run_delay = 0, last_arrival = 0, 
---Type <return> to continue, or q <return> to quit---
    last_queued = 211460260, bkl_count = 0}, tasks = {
    next = 0xc1692600 <init_task+448>, prev = 0xc70701c0}, pushable_tasks = {
    prio = 140, plist = {prio_list = {next = 0xc707174c, prev = 0xc707174c}, 
      node_list = {next = 0xc7071754, prev = 0xc7071754}, lock = 0x0}}, 
  mm = 0x0, active_mm = 0x0, exit_state = 0, exit_code = 0, exit_signal = 0, 
  pdeath_signal = 0, personality = 0, did_exec = 0, in_execve = 0, 
  in_iowait = 0, sched_reset_on_fork = 0, pid = 2, tgid = 2, 
  stack_canary = 1354951921, real_parent = 0xc1692440 <init_task>, 
  parent = 0xc1692440 <init_task>, children = {next = 0xc7071794, 
    prev = 0xc7071794}, sibling = {next = 0xc1692654 <init_task+532>, 
    prev = 0xc707021c}, group_leader = 0xc7071580, ptraced = {
    next = 0xc70717a8, prev = 0xc70717a8}, ptrace_entry = {next = 0xc70717b0, 
    prev = 0xc70717b0}, bts = 0x0, pids = {{node = {next = 0x0, 
        pprev = 0xc7066068}, pid = 0xc7066060}, {node = {next = 0xc7070248, 
        pprev = 0xc169dbec <init_struct_pid+12>}, 
      pid = 0xc169dbe0 <init_struct_pid>}, {node = {next = 0xc7070254, 
        pprev = 0xc169dbf0 <init_struct_pid+16>}, 
      pid = 0xc169dbe0 <init_struct_pid>}}, thread_group = {next = 0xc70717e0, 
    prev = 0xc70717e0}, vfork_done = 0x0, set_child_tid = 0x0, 
  clear_child_tid = 0x0, utime = 0, stime = 0, utimescaled = 0, 
  stimescaled = 0, gtime = 0, prev_utime = 0, prev_stime = 0, nvcsw = 0, 
  nivcsw = 0, start_time = {tv_sec = 0, tv_nsec = 208013000}, 
  real_start_time = {tv_sec = 0, tv_nsec = 208013000}, min_flt = 0, 
---Type <return> to continue, or q <return> to quit---
  maj_flt = 0, cputime_expires = {utime = 0, stime = 0, sum_exec_runtime = 0}, 
  cpu_timers = {{next = 0xc7071840, prev = 0xc7071840}, {next = 0xc7071848, 
      prev = 0xc7071848}, {next = 0xc7071850, prev = 0xc7071850}}, 
  real_cred = 0xc70640c0, cred = 0xc70640c0, cred_guard_mutex = {count = {
      counter = 1}, wait_lock = {raw_lock = {slock = 0}, magic = 3735899821, 
      owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
        key = 0xc190e388 <__key.15367>, class_cache = 0x0, 
        name = 0xc15d6430 "&lock->wait_lock", cpu = 0, ip = 0}}, wait_list = {
      next = 0xc7071888, prev = 0xc7071888}, owner = 0x0, name = 0x0, 
    magic = 0xc7071860, dep_map = {key = 0xc190e45c <__key.26889>, 
      class_cache = 0x0, name = 0xc15d671b "&p->cred_guard_mutex", cpu = 0, 
      ip = 0}}, replacement_session_keyring = 0x0, 
  comm = "swapper\000\000\000\000\000\000\000\000", link_count = 0, 
  total_link_count = 0, sysvsem = {undo_list = 0x0}, last_switch_count = 0, 
  thread = {tls_array = {{{{a = 0, b = 0}, {limit0 = 0, base0 = 0, base1 = 0, 
            type = 0, s = 0, dpl = 0, p = 0, limit = 0, avl = 0, l = 0, d = 0, 
            g = 0, base2 = 0}}}, {{{a = 0, b = 0}, {limit0 = 0, base0 = 0, 
            base1 = 0, type = 0, s = 0, dpl = 0, p = 0, limit = 0, avl = 0, 
            l = 0, d = 0, g = 0, base2 = 0}}}, {{{a = 0, b = 0}, {limit0 = 0, 
            base0 = 0, base1 = 0, type = 0, s = 0, dpl = 0, p = 0, limit = 0, 
            avl = 0, l = 0, d = 0, g = 0, base2 = 0}}}}, sp0 = 3339124728, 
    sp = 3339124660, sysenter_cs = 96, ip = 3238014304, gs = 0, debugreg0 = 0, 
    debugreg1 = 0, debugreg2 = 0, debugreg3 = 0, debugreg6 = 0, debugreg7 = 0, 
---Type <return> to continue, or q <return> to quit---
    cr2 = 0, trap_no = 0, error_code = 0, xstate = 0xc7028480, 
    vm86_info = 0x0, screen_bitmap = 0, v86flags = 0, v86mask = 0, 
    saved_sp0 = 0, saved_fs = 0, saved_gs = 0, io_bitmap_ptr = 0x0, iopl = 0, 
    io_bitmap_max = 0, debugctlmsr = 0, ds_ctx = 0x0}, 
  fs = 0xc16aa620 <init_fs>, files = 0xc16a9480 <init_files>, 
  nsproxy = 0xc169e2c8 <init_nsproxy>, signal = 0xc706c2c0, 
  sighand = 0xc7078000, blocked = {sig = {0, 0}}, real_blocked = {sig = {0, 
      0}}, saved_sigmask = {sig = {0, 0}}, pending = {list = {
      next = 0xc7071984, prev = 0xc7071984}, signal = {sig = {0, 0}}}, 
  sas_ss_sp = 0, sas_ss_size = 0, notifier = 0x0, notifier_data = 0x0, 
  notifier_mask = 0x0, audit_context = 0x0, loginuid = 4294967295, 
  sessionid = 4294967295, seccomp = {mode = 0}, parent_exec_id = 0, 
  self_exec_id = 0, alloc_lock = {raw_lock = {slock = 0}, magic = 3735899821, 
    owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
      key = 0xc18ec404 <__key.36775>, class_cache = 0x0, 
      name = 0xc15d3e40 "&p->alloc_lock", cpu = 0, ip = 3238286233}}, 
  irqaction = 0x0, pi_lock = {raw_lock = {slock = 0}, magic = 3735899821, 
    owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
      key = 0xc18ec3fc <__key.36748>, class_cache = 0x0, 
      name = 0xc15d3e34 "&p->pi_lock", cpu = 0, ip = 0}}, pi_waiters = {
    prio_list = {next = 0xc7071a0c, prev = 0xc7071a0c}, node_list = {
      next = 0xc7071a14, prev = 0xc7071a14}, lock = 0xc70719e8}, 
  pi_blocked_on = 0x0, blocked_on = 0x0, irq_events = 0, hardirqs_enabled = 0, 
---Type <return> to continue, or q <return> to quit---
  hardirq_enable_ip = 0, hardirq_enable_event = 0, 
  hardirq_disable_ip = 3238284417, hardirq_disable_event = 0, 
  softirqs_enabled = 1, softirq_disable_ip = 0, softirq_disable_event = 0, 
  softirq_enable_ip = 3238284417, softirq_enable_event = 0, 
  hardirq_context = 0, softirq_context = 0, curr_chain_key = 0, 
  lockdep_depth = 0, lockdep_recursion = 0, held_locks = {{prev_chain_key = 0, 
      acquire_ip = 0, instance = 0xc190cf50 <boot_tvec_bases+16>, 
      nest_lock = 0x0, waittime_stamp = 0, holdtime_stamp = 208057924, 
      class_idx = 0, irq_context = 0, trylock = 0, read = 0, check = 2, 
      hardirqs_off = 1, references = 0}, {prev_chain_key = 0, acquire_ip = 0, 
      instance = 0xc1694890 <i8259A_lock+16>, nest_lock = 0x0, 
      waittime_stamp = 0, holdtime_stamp = 208049271, class_idx = 0, 
      irq_context = 0, trylock = 0, read = 0, check = 2, hardirqs_off = 1, 
      references = 0}, {prev_chain_key = 0, acquire_ip = 0, 
      instance = 0xc1694890 <i8259A_lock+16>, nest_lock = 0x0, 
      waittime_stamp = 0, holdtime_stamp = 188132819, class_idx = 0, 
      irq_context = 0, trylock = 0, read = 0, check = 2, hardirqs_off = 1, 
      references = 0}, {prev_chain_key = 0, acquire_ip = 0, 
      instance = 0xc1eb61b4 <obj_hash+299860>, nest_lock = 0x0, 
      waittime_stamp = 0, holdtime_stamp = 25522519, class_idx = 0, 
      irq_context = 0, trylock = 0, read = 0, check = 2, hardirqs_off = 1, 
      references = 0}, {prev_chain_key = 0, acquire_ip = 0, 
      instance = 0xc1694890 <i8259A_lock+16>, nest_lock = 0x0, 
---Type <return> to continue, or q <return> to quit---
      waittime_stamp = 0, holdtime_stamp = 16016691, class_idx = 0, 
      irq_context = 0, trylock = 0, read = 0, check = 2, hardirqs_off = 1, 
      references = 0}, {prev_chain_key = 0, acquire_ip = 0, instance = 0x0, 
      nest_lock = 0x0, waittime_stamp = 0, holdtime_stamp = 0, class_idx = 0, 
      irq_context = 0, trylock = 0, read = 0, check = 0, hardirqs_off = 0, 
      references = 0} <repeats 43 times>}, lockdep_reclaim_gfp = 0, 
  journal_info = 0x0, bio_list = 0x0, bio_tail = 0x0, reclaim_state = 0x0, 
  backing_dev_info = 0x0, io_context = 0x0, ptrace_message = 0, 
  last_siginfo = 0x0, ioac = {rchar = 0, wchar = 0, syscr = 0, syscw = 0, 
    read_bytes = 0, write_bytes = 0, cancelled_write_bytes = 0}, 
  acct_rss_mem1 = 0, acct_vm_mem1 = 0, acct_timexpd = 0, mems_allowed = {
    bits = {1}}, cpuset_mem_spread_rotor = 0, 
  cgroups = 0xc1e30b20 <init_css_set>, cg_list = {next = 0xc7072268, 
    prev = 0xc7072268}, robust_list = 0x0, pi_state_list = {next = 0xc7072274, 
    prev = 0xc7072274}, pi_state_cache = 0x0, perf_event_ctxp = 0x0, 
  perf_event_mutex = {count = {counter = 1}, wait_lock = {raw_lock = {
        slock = 0}, magic = 3735899821, owner_cpu = 4294967295, 
      owner = 0xffffffff, dep_map = {key = 0xc190e388 <__key.15367>, 
        class_cache = 0x0, name = 0xc15d6430 "&lock->wait_lock", cpu = 0, 
        ip = 0}}, wait_list = {next = 0xc70722ac, prev = 0xc70722ac}, 
    owner = 0x0, name = 0x0, magic = 0xc7072284, dep_map = {
      key = 0xc1e5c5e4 <__key.25109>, class_cache = 0x0, 
      name = 0xc15dca01 "&child->perf_event_mutex", cpu = 0, ip = 0}}, 
---Type <return> to continue, or q <return> to quit---
  perf_event_list = {next = 0xc70722d4, prev = 0xc70722d4}, fs_excl = {
    counter = 0}, rcu = {next = 0x0, func = 0x0}, splice_pipe = 0x0, 
  delays = 0xc7026130, make_it_fail = 0, dirties = {events = 0, period = 0, 
    shift = 0, lock = {raw_lock = {slock = 0}, magic = 3735899821, 
      owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
        key = 0xc1e6cca0 <__key.9654>, class_cache = 0x0, 
        name = 0xc1612cec "&pl->lock", cpu = 0, ip = 0}}}, 
  latency_record_count = 0, latency_record = {{backtrace = {
        0 <repeats 12 times>}, count = 0, time = 0, 
      max = 0} <repeats 32 times>}, timer_slack_ns = 50000, 
  default_timer_slack_ns = 50000, scm_work_list = 0x0, curr_ret_stack = -1, 
  ret_stack = 0x0, ftrace_timestamp = 0, trace_overrun = {counter = 0}, 
  tracing_graph_pause = {counter = 0}, trace = 0, trace_recursion = 0}
```

* Signals `kernel_init` waiting on completion `kthreadd_done` which indicates all setup of kernel thread `kthreadd` is complete.

```complete
427		complete(&kthreadd_done);
(gdb) s
complete (x=x@entry=0xc1730480 <kthreadd_done>) at kernel/sched.c:6015
6015		spin_lock_irqsave(&x->wait.lock, flags);
(gdb) n
6016		x->done++;
(gdb) 
6017		__wake_up_common(&x->wait, TASK_NORMAL, 1, 0, NULL);
(gdb) s
__wake_up_common (q=q@entry=0xc1730484 <kthreadd_done+4>, mode=mode@entry=3, 
    nr_exclusive=nr_exclusive@entry=1, wake_flags=wake_flags@entry=0, 
    key=key@entry=0x0) at kernel/sched.c:5912
5912		list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
(gdb) n
5919	}
(gdb) 
complete (x=x@entry=0xc1730480 <kthreadd_done>) at kernel/sched.c:6018
6018		spin_unlock_irqrestore(&x->wait.lock, flags);
(gdb) 
6019	}
(gdb) 
rest_init () at init/main.c:428
428		unlock_kernel();
(gdb) p kthreadd_done.wait
$2 = {lock = {raw_lock = {slock = 257}, magic = 3735899821, 
    owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
      key = 0xc1730494 <kthreadd_done+20>, 
      class_cache = 0xc1b5f8d0 <lock_classes+151696>, 
      name = 0xc15c53c4 "(kthreadd_done).wait.lock", cpu = 0, 
      ip = 3238203194}}, task_list = {next = 0xc17304a8 <kthreadd_done+40>, 
    prev = 0xc17304a8 <kthreadd_done+40>}}
```

* Unlocks kernel
The 'big kernel lock': This spinlock is taken and released recursively by lock_kernel() and unlock_kernel().  It is transparently dropped and reacquired over schedule().  It is used to protect legacy code that hasn't been migrated to a proper locking design yet.

```
428		unlock_kernel();
(gdb) s
unlock_kernel () at lib/kernel_lock.c:126
126		BUG_ON(current->lock_depth < 0);
(gdb) n
127		if (likely(--current->lock_depth < 0))
(gdb) 
128			__unlock_kernel();
(gdb) s
__unlock_kernel () at lib/kernel_lock.c:106
106		_raw_spin_unlock(&kernel_flag);
(gdb) s
_raw_spin_unlock (lock=lock@entry=0xc1694880 <i8259A_lock>)
    at lib/spinlock_debug.c:152
152	{
(gdb) n
153		debug_spin_unlock(lock);
(gdb) 
154		__raw_spin_unlock(&lock->raw_lock);
(gdb) s
__ticket_spin_unlock (lock=0xc1694880 <i8259A_lock>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/spinlock.h:101
101		asm volatile(UNLOCK_LOCK_PREFIX "incb %0"
(gdb) n
_raw_spin_unlock (lock=lock@entry=0xc1694880 <i8259A_lock>)
    at lib/spinlock_debug.c:155
155	}
```

# Links

* [User-space lockdep](https://lwn.net/Articles/536363/)
* [The kernel lock validator](https://lwn.net/Articles/185666/)
* [Using the TRACE\_EVENT\(\) macro \(Part 1\)](https://lwn.net/Articles/379903/)
* [Using the TRACE\_EVENT\(\) macro \(Part 2\)](https://lwn.net/Articles/381064/)
* [Using the TRACE\_EVENT\(\) macro \(Part 3\)](https://lwn.net/Articles/383362/)



