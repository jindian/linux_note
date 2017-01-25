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
        |--page_address_init                                             # mm/highmem.c:409
            |--list_add                                                  # include/linux/list.h:64
                |--__list_add                                            # lib/list_debug.c:19
            |--spin_lock_init                                            # include/linux/spinlock.h:96
                |---__spin_lock_init                                     # lib/spinlock_debug.c:16
        |--printk
        |--setup_arch                                                    # arch/x86/kernel/setup.c:720
            |--visws_early_detect        # We don't use visual workstation, this routine could be ingnored
            |--vmi_init                  # Same as visws_early_detect because of no configuration in linux config file
            |--early_cpu_init                                            # arch/x86/kernel/cpu/common.c:659
                |--early_identify_cpu                                    # arch/x86/kernel/cpu/common.c:621
                    |--have_cpuid_p                                      # arch/x86/kernel/cpu/common.c:204
                        |--flag_is_changeable_p                          # arch/x86/kernel/cpu/common.c:175
                    |--cpu_detect                                        # arch/x86/kernel/cpu/common.c:514
                        |--cpuid                                         # arch/x86/include/asm/processor.h:648
                            |--__cpuid                                   # arch/x86/include/asm/processor.h:577
                                |--native_cpuid                          # arch/x86/include/asm/processor.h:179
                    |--get_cpu_vendor                                    # arch/x86/kernel/cpu/common.c:487
                    |--get_cpu_cap                                       # arch/x86/kernel/cpu/common.c:544
                        |--cpuid
                        |--cpuid_eax                                     # arch/x86/include/asm/processor.h:670
                            |--cpuid
                                |--__cpuid
                                    |--native_cpuid
                        |--cpuid_edx
                        |--cpuid_ecx
                        |--cpu_has                                      # arch/x86/include/asm/cpufeature.h:184
                        |--init_scattered_cpuid_features                # arch/x86/kernel/cpu/addon_cpuid_features.c:
                    |--early_init_intel                                 # arch/x86/kernel/cpu/intel.c:31
                        |--clear_cpu_cap                                # arch/x86/include/asm/cpufeature.h:200
                            |--clear_bit                                # arch/x86/include/asm/bitops.h:97
            |--early_ioremap_init                                       # arch/x86/mm/ioremap.c:430
                |--early_ioremap_pmd                                    # arch/x86/mm/ioremap.c:412
                    |--__fix_to_virt                                    # arch/x86/include/asm/fixmap.h:179
                    |--read_cr3                                         # arch/x86/include/asm/system.h:310
                        |--native_read_cr3                              # arch/x86/include/asm/system.h:244
                |--pmd_populate_kernel                                  # arch/x86/include/asm/pgalloc.h:62
                    |--__pa                                             # arch/x86/include/asm/page.h:36
                        |--__phys_addr                                  # arch/x86/mm/physaddr.c:48
                    |--set_pmd                                          # arch/x86/include/asm/pgtable.h:38
                        |--native_set_pmd                               # arch/x86/include/asm/pgtable-3level.h:39
                            set_64bit                                   # arch/x86/include/asm/cmpxchg_32.h:33
            |--old_decode_dev                                           # include/linux/kdev_t.h:31
                |--MKDEV                                                # include/linux/kdev_t.h:9
            |--x86_init.oem.arch_setup -> x86_init_noop                 # arch/x86/kernel/x86_init.c:18
            |--setup_memory_map                                         # arch/x86/kernel/e820.c:1464
                |--default_machine_specific_memory_setup                # arch/x86/kernel/e820.c:1426
                    |--sanitize_e820_map                                # arch/x86/kernel/e820.c:235
                    |--append_e820_map                                  # arch/x86/kernel/e820.c:423
                        |--__append_e820_map                            # arch/x86/kernel/e820.c:394
                            |--e820_add_region                          # arch/x86/kernel/e820.c:129
                                |--__e820_add_region                    # arch/x86/kernel/e820.c:113
            |--parse_setup_data                                         # arch/x86/kernel/setup.c:419
            |--e820_reserve_setup_data                                  # arch/x86/kernel/setup.c:441
            |--copy_edd                                                 # arch/x86/kernel/setup.c:252
            |--parse_early_param                                        # init/main.c:468
                |--parse_early_options                                  # init/main.c:462
                    |--parse_args                                       # kernel/params.c:131
            |--vmi_activate                                             # arch/x86/include/asm/vmi.h:232
            |--reserve_early_setup_data                                 # arch/x86/kernel/setup.c:467
            |--acpi_mps_check                                           # arch/x86/kernel/acpi/boot.c:1673
            |--setup_clear_cpu_cap
            |--early_dump_pci_devices                                   # arch/x86/pci/early.c:88
                |--read_pci_config                                      # arch/x86/pci/early.c:10
                |--early_dump_pci_device                                # arch/x86/pci/early.c:66
                    |--read_pci_config
                |--read_pci_config_byte                                 # arch/x86/pci/early.c:20
            |--finish_e820_parsing                                      # arch/x86/kernel/e820.c:1304
            |--efi_init                                                 # arch/x86/kernel/efi.c:317
            |--dmi_scan_machine                                         # drivers/firmware/dmi_scan.c:372
                |--dmi_ioremap -> early_ioremap                         # arch/x86/include/asm/dmi.h:16
                    |--__early_ioremap                                  # arch/x86/mm/ioremap.c:529
                        |--early_set_fixmap                             # arch/x86/mm/ioremap.c:488
                            |--__early_set_fixmap                       # arch/x86/mm/ioremap.c:469
                                |--early_ioremap_pte                    # arch/x86/mm/ioremap.c:423
                                    |--set_pte
                |--dmi_present                                          # drivers/firmware/dmi_scan.c:346
                    |--dmi_walk_early                                   # drivers/firmware/dmi_scan.c:104
                        |--dmi_ioremap
                        |--dmi_table                                    # drivers/firmware/dmi_scan.c:71
                            |--dmi_decode                               # drivers/firmware/dmi_scan.c:303
                        |--add_device_randomness                        # drivers/char/random.c"632
                        |--dmi_iounmap
                |--dmi_iounmap
            |--dmi_check_system                                         # drivers/firmware/dmi_scan.c:467
                |--dmi_matches                                          # drivers/firmware/dmi_scan.c:426
                |--dmi_low_memory_corruption                            # arch/x86/kernel/setup.c:634
            |--init_hypervisor_platform                                 # arch/x86/kernel/cpu/hypervisor.c:52
                |--init_hypervisor                                      # arch/x86/kernel/cpu/hypervisor.c:46
                    |--detect_hypervisor_vendor                         # arch/x86/kernel/cpu/hypervisor.c:28
                        |--vmware_platform                              # arch/x86/kernel/cpu/vmware.c:93
                            |--cpuid                                    # arch/x86/include/asm/processor.h:648
                                |--__cpuid -> native_cpuid              # arch/x86/include/asm/processor.h:577
                                                                        # arch/x86/include/asm/processor.h:179
:
                            
```


# Links
  * [SMP Boot](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/smpboot.html)
  * [Linux Kernel 2.4 Internals](http://www.tldp.org/LDP/lki/lki.html#toc4)
