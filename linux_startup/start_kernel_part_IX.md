# start kernel part IX

  `CONFIG_PREEMPT` is not set in our environment, `preempt_disable` do nothing, ignore it.
  
  `irqs_disabled` is a macro, it could be found in include/linux/irqflags.h:114, as well as `raw_local_save_flags`, with all macros expanded, actual code of `irqs_disabled` shown as follow:

```irqs_disabled

unsigned long _flags;
do { (flags) = __raw_local_save_flags(); } while (0)
raw_irqs_disabled_flags(_flags);
```

  `__raw_local_save_flags` (arch/x86/include/asm/irqflags.h:64) invovles `native_save_fl` which reads the flags register.

```native_save_fl

static inline unsigned long native_save_fl(void)
{
	unsigned long flags;

	/*
	 * "=rm" is safe here, because "pop" adjusts the stack before
	 * it evaluates its effective address -- this is part of the
	 * documented behavior of the "pop" instruction.
	 */
	asm volatile("# __raw_save_flags\n\t"
		     "pushf ; pop %0"
		     : "=rm" (flags)
		     : /* no input */
		     : "memory");

	return flags;
}
```
  
  `raw_irqs_disabled_flags` checks if raw interrupt is disabled and returns the result.
  
  If interrupt is enabled, disable it with `local_irq_disable`.

## _initialize read-copy-update_

In computer science, [read-copy-update (RCU)](https://en.wikipedia.org/wiki/Read-copy-update) is a synchronization mechanism based on mutual exclusion.[note 1] It is used when performance of reads are crucial and is an example of space-time tradeoff, enabling fast operations at the cost of more space.

Read-copy-update allows multiple threads to efficiently read from shared memory by deferring updates after pre-existing reads to a later time while simultaneously marking the data, ensuring new readers will read the updated data. This makes all readers proceed as if there were no synchronization involved, hence they will be fast, but also making updates more difficult.

`rcu_init` involves `__rcu_init` which initializes rcu_state structure of `rcu_sched_state` and `rcu_bh_state`, as well as per cpu rcu date, after initialization completed, `__rcu_init` registers response routine for `RCU_SOFTIRQ`.

`rcu_init` register `rcu_barrier_cpu_hotplug` as the response routine when cpu up/down to create/delete rcu data.

`rcu_init` initialzes per cpu rcu data for current up CPU.
