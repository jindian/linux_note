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