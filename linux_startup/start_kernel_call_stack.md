# start_kernl call stack

```start_kernl_call_stack

    start_kernel                                                         # init/main.c:520
        |--smp_setup_processor_id                                        # init/main.c:496
        |--lockdep_init                                                  # kernel/lockdep.c:3570
        |--debug_objects_early_init                                      # lib/debugobjects.c:905
        |--boot_init_stack_canary                                        # arch/x86/include/asm/stackprotector.h:59
        |--cgroup_init_early                                             # kernel/cgroup.c:3214
        |--local_irq_disable                                             # include/linux/irqflags.h:61
            |--raw_local_irq_disable                                     # arch/x86/include/asm/irqflags.h:74
                |--native_irq_disable                                    # arch/x86/include/asm/irqflags.h:37
            |--trace_hardirqs_off                                        # kernel/lockdep.c:2380
                |--trace_hardirqs_off_caller                             # kernel/lockdep.c::2355
        |--early_boot_irqs_off                                           # kernel/lockdep.c:2289
        |--early_init_irq_lock_class                                     # kernel/irq/handle.c:544
            |--irq_to_desc                                               # kernel/irq/handle.c:192
        |--lock_kernel                                                   # lib/kernel_lock.c:116
            |--__lock_kernel                                             # lib/kernel_lock.c:94
                |--_raw_spin_lock                                        # lib/spinlock_debug.c:128
                    |--__raw_spin_trylock                                # lib/spinlock_debug.c:193
                        |--__ticket_spin_trylock                         # arch/x86/include/asm/spinlock.h:80
        |--tick_init                                                     # kernel/time/tick-common.c:417
            |--clockevents_register_notifier                             # kernel/time/clockevents.c:140
                |--spin_lock_irqsave
                |--raw_notifier_chain_register                           # kernel/notifier.c:344
                    |--notifier_chain_register                           # kernel/notifier.c:21
                        |--rcu_assign_pointer                            # include/linux/rcupdate.h:256
                |--spin_unlock_irqrestore
        |--boot_cpu_init                                                 # init/main.c:486
            |--smp_processor_id
            |--set_cpu_online                                            # kernel/cpu.c:607
            |--set_cpu_active
            |--set_cpu_present
            |--set_cpu_possible
```