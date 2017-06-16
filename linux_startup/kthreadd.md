# kthreadd

Let's check the callstack of function of `kthreadd`, it's invoked by routine `kernel_thread_helper` shown as follow

```
Breakpoint 2, kthreadd (unused=0x0) at kernel/kthread.c:192
192	{
(gdb) bt
#0  kthreadd (unused=0x0) at kernel/kthread.c:192
#1  0xc1003ad7 in kernel_thread_helper ()
    at arch/x86/kernel/entry_32.S:1000
```

Go back to `kernel_thread`, when trying to create kthreadd whose `pid=2`, function pointer of `kthreadd` is assigned `regs.bx`. register ip is assigned with `kernel_thread_helper`, if you familiar with [x86 registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html), ip holds the offset of the next instruction

```
/*
 * Create a kernel thread
 */
int kernel_thread(int (*fn)(void *), void *arg, unsigned long flags)
{
	struct pt_regs regs;

	memset(&regs, 0, sizeof(regs));

	regs.bx = (unsigned long) fn;
	regs.dx = (unsigned long) arg;

	regs.ds = __USER_DS;
	regs.es = __USER_DS;
	regs.fs = __KERNEL_PERCPU;
	regs.gs = __KERNEL_STACK_CANARY;
	regs.orig_ax = -1;
	regs.ip = (unsigned long) kernel_thread_helper;
	regs.cs = __KERNEL_CS | get_kernel_rpl();
	regs.flags = X86_EFLAGS_IF | X86_EFLAGS_SF | X86_EFLAGS_PF | 0x2;

	/* Ok, create the new process.. */
	return do_fork(flags | CLONE_VM | CLONE_UNTRACED, 0, &regs, 0, NULL, NULL);
}
EXPORT_SYMBOL(kernel_thread);
```

The defination of function `kernel_thread_helper` shown as follow, `ktreadd` invoked by instruction `call *%ebx`

```
ENTRY(kernel_thread_helper)
	pushl $0		# fake return address for unwinder
	CFI_STARTPROC
	movl %edx,%eax
	push %edx
	CFI_ADJUST_CFA_OFFSET 4
	call *%ebx
	push %eax
	CFI_ADJUST_CFA_OFFSET 4
	call do_exit
	ud2			# padding for call trace
	CFI_ENDPROC
ENDPROC(kernel_thread_helper)
```

Continue the proceduce of routine `kthreadd`

```kthreadd
int kthreadd(void *unused)
{
	struct task_struct *tsk = current;

	/* Setup a clean context for our children to inherit. */
	set_task_comm(tsk, "kthreadd");
	ignore_signals(tsk);
	set_cpus_allowed_ptr(tsk, cpu_all_mask);
	set_mems_allowed(node_states[N_HIGH_MEMORY]);

	current->flags |= PF_NOFREEZE | PF_FREEZER_NOSIG;

	for (;;) {
		set_current_state(TASK_INTERRUPTIBLE);
		if (list_empty(&kthread_create_list))			# if no kthread to create, swap out let other thread to execute
			schedule();
		__set_current_state(TASK_RUNNING);

		spin_lock(&kthread_create_lock);
		while (!list_empty(&kthread_create_list)) {		# create all kthreads in list
			struct kthread_create_info *create;

			create = list_entry(kthread_create_list.next,
					    struct kthread_create_info, list);
			list_del_init(&create->list);
			spin_unlock(&kthread_create_lock);

			create_kthread(create);
(gdb) s
create_kthread (create=0xc706bf14) at kernel/kthread.c:89
89		pid = kernel_thread(kthread, create, CLONE_FS | CLONE_FILES | SIGCHLD);
(gdb) n
90		if (pid < 0) {
(gdb) 
kthreadd (unused=<optimized out>) at kernel/kthread.c:209

			spin_lock(&kthread_create_lock);
		}
		spin_unlock(&kthread_create_lock);
	}

	return 0;
}
```

invoke `kthread_create` to create kthread, in which it add create thread information to list `kthread_create_list` and wake up kthreadd to create kernel thread.

```kthread_create
/**
 * kthread_create - create a kthread.
 * @threadfn: the function to run until signal_pending(current).
 * @data: data ptr for @threadfn.
 * @namefmt: printf-style name for the thread.
 *
 * Description: This helper function creates and names a kernel
 * thread.  The thread will be stopped: use wake_up_process() to start
 * it.  See also kthread_run(), kthread_create_on_cpu().
 *
 * When woken, the thread will run @threadfn() with @data as its
 * argument. @threadfn() can either call do_exit() directly if it is a
 * standalone thread for which noone will call kthread_stop(), or
 * return when 'kthread_should_stop()' is true (which means
 * kthread_stop() has been called).  The return value should be zero
 * or a negative error number; it will be passed to kthread_stop().
 *
 * Returns a task_struct or ERR_PTR(-ENOMEM).
 */
struct task_struct *kthread_create(int (*threadfn)(void *data),
				   void *data,
				   const char namefmt[],
				   ...)
{
	struct kthread_create_info create;

	create.threadfn = threadfn;
	create.data = data;
	init_completion(&create.done);

	spin_lock(&kthread_create_lock);
	list_add_tail(&create.list, &kthread_create_list);
	spin_unlock(&kthread_create_lock);

	wake_up_process(kthreadd_task);
	wait_for_completion(&create.done);

	if (!IS_ERR(create.result)) {
		struct sched_param param = { .sched_priority = 0 };
		va_list args;

		va_start(args, namefmt);
		vsnprintf(create.result->comm, sizeof(create.result->comm),
			  namefmt, args);
		va_end(args);
		/*
		 * root may have changed our (kthreadd's) priority or CPU mask.
		 * The kernel thread should not inherit these properties.
		 */
		sched_setscheduler_nocheck(create.result, SCHED_NORMAL, &param);
		set_cpus_allowed_ptr(create.result, cpu_all_mask);
	}
	return create.result;
}
EXPORT_SYMBOL(kthread_create);
```

# Links

* [x86 registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html)
