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

## _initialize hardware interrupt_

  There are two types of interaction between the CPU and the rest of the computer's hardware. The first type is when the CPU gives orders to the hardware, the other is when the hardware needs to tell the CPU something. The second, called interrupts, is much harder to implement because it has to be dealt with when convenient for the hardware, not the CPU. Hardware devices typically have a very small amount of RAM, and if you don't read their information when available, it is lost.

  Under Linux, hardware interrupts are called IRQ's (Interrupt Requests)[1]. There are two types of IRQ's, short and long. A short IRQ is one which is expected to take a very short period of time, during which the rest of the machine will be blocked and no other interrupts will be handled. A long IRQ is one which can take longer, and during which other interrupts may occur (but not interrupts from the same device). If at all possible, it's better to declare an interrupt handler to be long.

  When the CPU receives an interrupt, it stops whatever it's doing (unless it's processing a more important interrupt, in which case it will deal with this one only when the more important one is done), saves certain parameters on the stack and calls the interrupt handler. This means that certain things are not allowed in the interrupt handler itself, because the system is in an unknown state. The solution to this problem is for the interrupt handler to do what needs to be done immediately, usually read something from the hardware or send something to the hardware, and then schedule the handling of the new information at a later time (this is called the "bottom half") and return. The kernel is then guaranteed to call the bottom half as soon as possible -- and when it does, everything allowed in kernel modules will be allowed.
  
  `early_irq_init`:
  
  * allocates cpu variables for all possible cpus and set all cpus in the allocated cpumask.
  * initialize nr_irqs based on nr_cpu_ids.

```arch_probe_nr_irqs

arch_probe_nr_irqs () at arch/x86/kernel/apic/io_apic.c:3868
3868		if (nr_irqs > (NR_VECTORS * nr_cpu_ids))
(gdb) p nr_irqs
$1 = 2304

......

3882	}
(gdb) p nr_irqs
$6 = 256

```
  
  * allocates memory for array `irq_desc_ptrs` based on `nr_irqs`.
  * allocates memory for array `kstat_irqs_legacy` based on `nr_cpu_ids`.
  * initializes array `irq_desc_ptrs`
  
## initialize interrupt call gate of interrupt description table

  `native_init_IRQ`:
  * initialize ISA interrupt call gates and register function to handle level interrupts.
  
  
  


  
  

# Links
  * [x86 Registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html)
  * [read-copy-update (RCU)](https://en.wikipedia.org/wiki/Read-copy-update)
  * [Interrupt Handlers](http://www.tldp.org/LDP/lkmpg/2.4/html/x1210.html)
  * [ISA interrupts versus PCI interrupts](https://www.safaribooksonline.com/library/view/pc-hardware-in/059600513X/ch01s03s01s01.html)
  * [Interrupts](http://wiki.osdev.org/Interrupts)
