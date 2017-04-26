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
  
## _initialize interrupt call gate of interrupt description table_

  `native_init_IRQ` initialize interrupt call gates and register function of interrupts.

## _initialize data used for priority search tree_

  `prio_tree_init` initializes array `index_bits_to_maxindex` which is used to quickly find node in priority search tree. About priority search tree, we can find many references by google.
  
## _initialize timer_

  In the Linux kernel, time is measured by a global variable named jiffies, which identifies the number of ticks that have occurred since the system was booted. The manner in which ticks are counted depends, at its lowest level, on the particular hardware platform on which you're running; however, it is typically incremented through an interrupt. The tick rate (jiffies's least significant bit) is configurable, but in a recent 2.6 kernel for x86, a tick equals 4ms (250Hz). The jiffies global variable is used broadly in the kernel for a number of purposes, one of which is the current absolute time to calculate the time-out value for a timer (you'll see examples of this later).
  
  There are few different schemes for timers in recent 2.6 kernels. The simplest and least accurate of all timers (though suitable for most instances) is the timer API. This API permits the construction of timers that operate in the jiffies domain (minimum 4ms time-out). There's also the high-resolution timer API, which permits timer constructions in which time is defined in nanoseconds. Depending upon your processor and the speed at which it operates, your mileage may vary, but the API does offer a way to schedule time-outs below the jiffies tick interval.
  
  `init_timers`:
  
  * Initializes per cpu variable `boot_tvec_bases` for cpu 0
  * Initializes per cpu look up lock
  * Adds notifier to `cpu_chain`
  * Registers function for running timers and timer-tq in bootom half context for `TIMER_SOFTIRQ`

  The initialization of high resolution timers is similar, ignore it.

## _initialize soft interrupt_

  The softirq mechanism is meant to handle processing that is almost — but not quite — as important as the handling of hardware interrupts. Softirqs run at a high priority (though with an interesting exception, described below), but with hardware interrupts enabled. They thus will normally preempt any work except the response to a "real" hardware interrupt.

  Once upon a time, there were 32 hardwired software interrupt vectors, one assigned to each device driver or related task. Drivers have, for the most part, been detached from software interrupts for a long time — they still use softirqs, but that access has been laundered through intermediate APIs like tasklets and timers. In current kernels there are ten softirq vectors defined; two for tasklet processing, two for networking, two for the block layer, two for timers, and one each for the scheduler and read-copy-update processing. The kernel maintains a per-CPU bitmask indicating which softirqs need processing at any given time. So, for example, when a kernel subsystem calls tasklet_schedule(), the TASKLET_SOFTIRQ bit is set on the corresponding CPU and, when softirqs are processed, the tasklet will be run.

  There are two places where software interrupts can "fire" and preempt the current thread. One of them is at the end of the processing for a hardware interrupt; it is common for interrupt handlers to raise softirqs, so it makes sense (for latency and optimal cache use) to process them as soon as hardware interrupts can be re-enabled. The other possibility is anytime that kernel code re-enables softirq processing (via a call to functions like local_bh_enable() or spin_unlock_bh()). The end result is that the accumulated softirq work (which can be substantial) is executed in the context of whichever process happens to be running at the wrong time; that is the "randomly chosen victim" aspect that Thomas was talking about.

  `softirq_init`:
  
  * Initializes per cpu variables `tasklet_vec`, `tasklet_hi_vec` and `softirq_work_list` for each cpus
  * Registers notifier for cpu hot plugin
  * Opens soft interrupt for `TASKLET_SOFTIRQ` and `HI_SOFTIRQ`

## _initializes the clocksource and common timekeeping values_

  
  
  

# Links
  * [x86 Registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html)
  * [read-copy-update (RCU)](https://en.wikipedia.org/wiki/Read-copy-update)
  * [Interrupt Handlers](http://www.tldp.org/LDP/lkmpg/2.4/html/x1210.html)
  * [ISA interrupts versus PCI interrupts](https://www.safaribooksonline.com/library/view/pc-hardware-in/059600513X/ch01s03s01s01.html)
  * [Interrupts](http://wiki.osdev.org/Interrupts)
  * [Timers](www.cs.columbia.edu/~nahum/w6998/lectures/timers.ppt)
  * [Kernel Timers](http://www.makelinux.net/ldd3/chp-7-sect-4)
  * [Kernel Timer Systems](http://elinux.org/Kernel_Timer_Systems)
  * [Timers and lists in the 2.6 kernel](https://www.ibm.com/developerworks/linux/library/l-timers-list/)
  * [A new approach to kernel timers](https://lwn.net/Articles/152436/)
  * [The high-resolution timer API](https://lwn.net/Articles/167897/)
  * [Software interrupts and realtime](https://lwn.net/Articles/520076/)
