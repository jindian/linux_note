# kthreadd

* kernel thread routine with pid=2 executed

```
Breakpoint 4, kthreadd (unused=0x0) at kernel/kthread.c:196
196		set_task_comm(tsk, "kthreadd");					# set executable name
(gdb) s
set_task_comm (tsk=tsk@entry=0xc7071580, buf=buf@entry=0xc15d63d2 "kthreadd")
    at fs/exec.c:982
982		task_lock(tsk);
(gdb) n
983		strlcpy(tsk->comm, buf, sizeof(tsk->comm));
(gdb) 
984		task_unlock(tsk);
(gdb) 
985		perf_event_comm(tsk);
(gdb) 
986	}
(gdb) 
kthreadd (unused=<optimized out>) at kernel/kthread.c:197
197		ignore_signals(tsk);						# ignore signals of current task with pid=2
(gdb) s
ignore_signals (t=t@entry=0xc7071580) at kernel/signal.c:304
304		for (i = 0; i < _NSIG; ++i)
(gdb) n
305			t->sighand->action[i].sa.sa_handler = SIG_IGN;
(gdb) break if i==63
Breakpoint 5 at 0xc105b180: file kernel/signal.c, line 305.
(gdb) c
Continuing.

Breakpoint 5, ignore_signals (t=t@entry=0xc7071580) at kernel/signal.c:305
305			t->sighand->action[i].sa.sa_handler = SIG_IGN;
(gdb) n
(gdb) 
304		for (i = 0; i < _NSIG; ++i)
(gdb) 
305			t->sighand->action[i].sa.sa_handler = SIG_IGN;
(gdb) p i
$2 = 64
(gdb) n
304		for (i = 0; i < _NSIG; ++i)
(gdb) 
307		flush_signals(t);
(gdb) s
flush_signals (t=t@entry=0xc7071580) at kernel/signal.c:258
258	{
(gdb) n
261		spin_lock_irqsave(&t->sighand->siglock, flags);
(gdb) 
262		__flush_signals(t);
(gdb) s
__flush_signals (t=t@entry=0xc7071580) at kernel/signal.c:251
251	{
(gdb) n
252		clear_tsk_thread_flag(t, TIF_SIGPENDING);
(gdb) 
253		flush_sigqueue(&t->pending);
(gdb) 
254		flush_sigqueue(&t->signal->shared_pending);
(gdb) 
255	}
(gdb) 
flush_signals (t=t@entry=0xc7071580) at kernel/signal.c:263
263		spin_unlock_irqrestore(&t->sighand->siglock, flags);
(gdb) 
264	}
(gdb) 
ignore_signals (t=t@entry=0xc7071580) at kernel/signal.c:308
308	}
(gdb) 
kthreadd (unused=<optimized out>) at kernel/kthread.c:198
198		set_cpus_allowed_ptr(tsk, cpu_all_mask);
(gdb) s
set_cpus_allowed_ptr (p=p@entry=0xc7071580, new_mask=0xc1497fa4 <cpu_all_bits>)
    at kernel/sched.c:7372
7372		while (task_is_waking(p))
(gdb) p p->state
$3 = 0
(gdb) n
7374		rq = task_rq_lock(p, &flags);
(gdb) s
task_rq_lock (p=p@entry=0xc7071580, flags=flags@entry=0xc706ff78)
    at kernel/sched.c:949
949	{
(gdb) n
953			local_irq_save(*flags);
(gdb) 
954			rq = task_rq(p);
(gdb) 
955			spin_lock(&rq->lock);
(gdb) p rq
$4 = (struct rq *) 0xc234a2e0
(gdb) n
956			if (likely(rq == task_rq(p)))
(gdb) 
960	}
(gdb) 
set_cpus_allowed_ptr (p=p@entry=0xc7071580, new_mask=0xc1497fa4 <cpu_all_bits>)
    at kernel/sched.c:7375
7375		if (task_is_waking(p)) {
(gdb) 
7380		if (!cpumask_intersects(new_mask, cpu_active_mask)) {
(gdb) 
7385		if (unlikely((p->flags & PF_THREAD_BOUND) && p != current &&
(gdb) p cpu_active_mask
$5 = (const struct cpumask * const) 0xc16f7258 <cpu_active_bits>
(gdb) p *cpu_active_mask
$6 = {bits = {1}}
(gdb) p *new_mask
$7 = {bits = {255}}
(gdb) n
7391		if (p->sched_class->set_cpus_allowed)
(gdb) p p->sched_class->set_cpus_allowed 
$8 = (void (*)(struct task_struct *, const struct cpumask *)) 0x0
(gdb) n
7394			cpumask_copy(&p->cpus_allowed, new_mask);
(gdb) 
7395			p->rt.nr_cpus_allowed = cpumask_weight(new_mask);
(gdb) 
7399		if (cpumask_test_cpu(task_cpu(p), new_mask))			# test if task could run on current CPU
(gdb) p p->rt.nr_cpus_allowed
$9 = 8
(gdb) n
7365		int ret = 0;
(gdb) 
7415		task_rq_unlock(rq, &flags);
(gdb) 
7417		return ret;
(gdb) 
7418	}
(gdb) 
kthreadd (unused=<optimized out>) at kernel/kthread.c:199
199		set_mems_allowed(node_states[N_HIGH_MEMORY]);
(gdb) s
set_mems_allowed (nodemask=...) at include/linux/cpuset.h:91
91		current->mems_allowed = nodemask;
(gdb) n
kthreadd (unused=<optimized out>) at kernel/kthread.c:201
201		current->flags |= PF_NOFREEZE | PF_FREEZER_NOSIG;
(gdb) 
204			set_current_state(TASK_INTERRUPTIBLE);			# enter for loop of kthreadd routine
(gdb) s
__xchg (size=4, ptr=0xc7071580, x=1)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/cmpxchg_32.h:69
69			asm volatile("xchgl %0,%1"
(gdb) n
kthreadd (unused=<optimized out>) at kernel/kthread.c:205
205			if (list_empty(&kthread_create_list))			# if kthread_create_list is empty, invoke schedule and  										# current thread go to sleep as the state of current
										# task was set as TASK_INTERRUPTIBLE
(gdb) p kthread_create_list 
$11 = {next = 0xc706bf50, prev = 0xc706bf50}
(gdb) n
207			__set_current_state(TASK_RUNNING);
(gdb) 
209			spin_lock(&kthread_create_lock);
(gdb) p &kthread_create_list
$12 = (struct list_head *) 0xc169e1c0 <kthread_create_list>
(gdb) n
210			while (!list_empty(&kthread_create_list)) {		# create all processes in kthread_create_list
(gdb) 
215				list_del_init(&create->list);
(gdb) 
216				spin_unlock(&kthread_create_lock);
(gdb) 
218				create_kthread(create);
(gdb) p create
$13 = (struct kthread_create_info *) 0xc706bf14
(gdb) s
create_kthread (create=0xc706bf14) at kernel/kthread.c:89
89		pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
(gdb) p create
$14 = (struct kthread_create_info *) 0xc706bf14
(gdb) p *create
$15 = {threadfn = 0xc1041100 <migration_thread>, data = 0x0, 
  result = 0xc7070000, done = {done = 0, wait = {lock = {raw_lock = {
          slock = 0}, magic = 3735899821, owner_cpu = 4294967295, 
        owner = 0xffffffff, dep_map = {key = 0xc190e35c <__key.5700>, 
          class_cache = 0x0, name = 0xc16237be "key", cpu = 0, ip = 642}}, 
      task_list = {next = 0xc706bf48, prev = 0xc706bf48}}}, list = {
    next = 0xc706bf50, prev = 0xc706bf50}}
(gdb) n
90		if (pid < 0) {
(gdb) 
kthreadd (unused=<optimized out>) at kernel/kthread.c:209
209			spin_lock(&kthread_create_lock);
(gdb) 
210			while (!list_empty(&kthread_create_list)) {
(gdb) 
222			spin_unlock(&kthread_create_lock);
(gdb) 
223		}
(gdb) 
204			set_current_state(TASK_INTERRUPTIBLE);
(gdb) 
205			if (list_empty(&kthread_create_list))
(gdb) 
206				schedule();
(gdb) 
```
