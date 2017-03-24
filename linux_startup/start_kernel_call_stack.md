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
                        |--cpu_has                                       # arch/x86/include/asm/cpufeature.h:184
                        |--init_scattered_cpuid_features                 # arch/x86/kernel/cpu/addon_cpuid_features.c:
                    |--early_init_intel                                  # arch/x86/kernel/cpu/intel.c:31
                        |--clear_cpu_cap                                 # arch/x86/include/asm/cpufeature.h:200
                            |--clear_bit                                 # arch/x86/include/asm/bitops.h:97
            |--early_ioremap_init                                        # arch/x86/mm/ioremap.c:430
                |--early_ioremap_pmd                                     # arch/x86/mm/ioremap.c:412
                    |--__fix_to_virt                                     # arch/x86/include/asm/fixmap.h:179
                    |--read_cr3                                          # arch/x86/include/asm/system.h:310
                        |--native_read_cr3                               # arch/x86/include/asm/system.h:244
                |--pmd_populate_kernel                                   # arch/x86/include/asm/pgalloc.h:62
                    |--__pa                                              # arch/x86/include/asm/page.h:36
                        |--__phys_addr                                   # arch/x86/mm/physaddr.c:48
                    |--set_pmd                                           # arch/x86/include/asm/pgtable.h:38
                        |--native_set_pmd                                # arch/x86/include/asm/pgtable-3level.h:39
                            set_64bit                                    # arch/x86/include/asm/cmpxchg_32.h:33
            |--old_decode_dev                                            # include/linux/kdev_t.h:31
                |--MKDEV                                                 # include/linux/kdev_t.h:9
            |--x86_init.oem.arch_setup -> x86_init_noop                  # arch/x86/kernel/x86_init.c:18
            |--setup_memory_map                                          # arch/x86/kernel/e820.c:1464
                |--default_machine_specific_memory_setup                 # arch/x86/kernel/e820.c:1426
                    |--sanitize_e820_map                                 # arch/x86/kernel/e820.c:235
                    |--append_e820_map                                   # arch/x86/kernel/e820.c:423
                        |--__append_e820_map                             # arch/x86/kernel/e820.c:394
                            |--e820_add_region                           # arch/x86/kernel/e820.c:129
                                |--__e820_add_region                     # arch/x86/kernel/e820.c:113
            |--parse_setup_data                                          # arch/x86/kernel/setup.c:419
            |--e820_reserve_setup_data                                   # arch/x86/kernel/setup.c:441
            |--copy_edd                                                  # arch/x86/kernel/setup.c:252
            |--parse_early_param                                         # init/main.c:468
                |--parse_early_options                                   # init/main.c:462
                    |--parse_args                                        # kernel/params.c:131
            |--vmi_activate                                              # arch/x86/include/asm/vmi.h:232
            |--reserve_early_setup_data                                  # arch/x86/kernel/setup.c:467
            |--acpi_mps_check                                            # arch/x86/kernel/acpi/boot.c:1673
            |--setup_clear_cpu_cap
            |--early_dump_pci_devices                                    # arch/x86/pci/early.c:88
                |--read_pci_config                                       # arch/x86/pci/early.c:10
                |--early_dump_pci_device                                 # arch/x86/pci/early.c:66
                    |--read_pci_config
                |--read_pci_config_byte                                  # arch/x86/pci/early.c:20
            |--finish_e820_parsing                                       # arch/x86/kernel/e820.c:1304
            |--efi_init                                                  # arch/x86/kernel/efi.c:317
            |--dmi_scan_machine                                          # drivers/firmware/dmi_scan.c:372
                |--dmi_ioremap -> early_ioremap                          # arch/x86/include/asm/dmi.h:16
                    |--__early_ioremap                                   # arch/x86/mm/ioremap.c:529
                        |--early_set_fixmap                              # arch/x86/mm/ioremap.c:488
                            |--__early_set_fixmap                        # arch/x86/mm/ioremap.c:469
                                |--early_ioremap_pte                     # arch/x86/mm/ioremap.c:423
                                    |--set_pte
                |--dmi_present                                           # drivers/firmware/dmi_scan.c:346
                    |--dmi_walk_early                                    # drivers/firmware/dmi_scan.c:104
                        |--dmi_ioremap
                        |--dmi_table                                     # drivers/firmware/dmi_scan.c:71
                            |--dmi_decode                                # drivers/firmware/dmi_scan.c:303
                        |--add_device_randomness                         # drivers/char/random.c"632
                        |--dmi_iounmap
                |--dmi_iounmap
            |--dmi_check_system                                          # drivers/firmware/dmi_scan.c:467
                |--dmi_matches                                           # drivers/firmware/dmi_scan.c:426
                |--dmi_low_memory_corruption                             # arch/x86/kernel/setup.c:634
            |--init_hypervisor_platform                                  # arch/x86/kernel/cpu/hypervisor.c:52
                |--init_hypervisor                                       # arch/x86/kernel/cpu/hypervisor.c:46
                    |--detect_hypervisor_vendor                          # arch/x86/kernel/cpu/hypervisor.c:28
                        |--vmware_platform                               # arch/x86/kernel/cpu/vmware.c:93
                            |--cpuid                                     # arch/x86/include/asm/processor.h:648
                                |--__cpuid -> native_cpuid               # arch/x86/include/asm/processor.h:577
                                                                         # arch/x86/include/asm/processor.h:179
                    |--hypervisor_set_feature_bits                       # arch/x86/kernel/cpu/hypervisor.c:38
                |--vmware_platform_setup                                 # arch/x86/kernel/cpu/vmware.c:75
            |--probe_roms                                                # arch/x86/kernel/probe_roms_32.c:95
                |--isa_bus_to_virt -> phys_to_virt                       # arch/x86/include/asm/io.h:135
                                                                         # arch/x86/include/asm/io.h:115
                    |--__va                                              # arch/x86/include/asm/page.h:42
                |--romsignature                                          # arch/x86/kernel/probe_roms_32.c:78
                    |--probe_kernel_address -> ... __copy_from_user_inatomic ...                             
                                                                         # include/linux/uaccess.h:74
                                                                         # include/linux/uaccess.h:89
                |--probe_kernel_address
                |--romchecksum                                           # arch/x86/kernel/probe_roms_32.c:86
                    |--probe_kernel_address
                |--request_resource                                      # kernel/resource.c:198
                    |--write_lock
                    |--__request_resource                                # kernel/resource.c:144
                    |--write_unlock
            |--insert_resource                                           # kernel/resource.c:439
                |--write_lock
                    |--__insert_resource                                 # kernel/resource.c:379
                        |--__request_resource
                |--write_unlock
            |--ppro_with_ram_bug                                         # arch/x86/kernel/cpu/intel.c:143
            |--e820_end_of_ram_pfn                                       # arch/x86/kernel/e820.c:1145
                |--e820_end_pfn                                          # arch/x86/kernel/e820.c:1111
            |--early_reserve_e820_mpc_new                                # arch/x86/kernel/mpparse.c:957
                |--early_reserve_e820
            |--mtrr_bp_init                                              # arch/x86/kernel/cpu/mtrr/main.c:654
                |--init_ifs                                              # arch/x86/kernel/cpu/mtrr/main.c:592
                    |--amd_init_mtrr                                     # arch/x86/kernel/cpu/mtrr/amd.c:120
                    |--cyrix_init_mtrr                                   # arch/x86/kernel/cpu/mtrr/cyrix.c:278
                    |--centaur_init_mtrr                                 # arch/x86/kernel/cpu/mtrr/centaur.c:123
            |--mtrr_trim_uncached_memory                                 # arch/x86/kernel/cpu/mtrr/cleanup.c:992
            |--find_low_pfn_range                                        # arch/x86/mm/init_32.c:699
                |--lowmem_pfn_init                                       # arch/x86/mm/init_32.c:622
            |--setup_bios_corruption_check                               # arch/x86/kernel/check.c:60
                |--find_e820_area_size                                   # arch/x86/kernel/e820.c:1032
            |--reserve_brk                                               # arch/x86/kernel/setup.c:299
                |--reserve_early                                         # arch/x86/kernel/e820.c:894
                    |--drop_overlaps_that_are_ok                         # arch/x86/kernel/e820.c:781
                    |--__reserve_early                                   # arch/x86/kernel/e820.c:838
                        |--find_overlapped_early                         # arch/x86/kernel/e820.c:739
            |--init_gbpages
            |--init_memory_mapping                                       # arch/x86/mm/init.c:123
                |--set_nx                                                # arch/x86/mm/setup_nx.c:37
                |--set_in_cr4                                            # arch/x86/include/asm/processor.h:605
            |--reserve_initrd                                            # arch/x86/kernel/setup.c:374
            |--io_delay_init                                             # arch/x86/kernel/io_delay.c:105
                |--dmi_check_system                                      # drivers/firmware/dmi_scan.c:467
                    |--dmi_matches                                       # drivers/firmware/dmi_scan.c:426
            |--acpi_boot_table_init                                      # arch/x86/kernel/acpi/boot.c:1537
                |--acpi_table_init                                       # drivers/acpi/tables.c:338
                    |--acpi_initialize_tables                            # drivers/acpi/acpica/tbxface.c:107
                |--acpi_table_parse                                      # drivers/acpi/tables.c:278
                    |--acpi_get_table_with_size                          # drivers/acpi/acpica/tbxface.c:377
                |--acpi_blacklisted                                      # drivers/acpi/blacklist.c:112
            |--early_acpi_boot_init                                      # arch/x86/kernel/acpi/boot.c:1578
                |--early_acpi_process_madt                               # arch/x86/kernel/acpi/boot.c:1159
                    |--acpi_table_parse
                    |--early_acpi_parse_madt_lapic_addr_ovr              # arch/x86/kernel/acpi/boot.c:759
                        |--acpi_table_parse_madt
                        |--acpi_register_lapic_address
            |--initmem_init                                              # arch/x86/mm/init_32.c:706
                |--e820_register_active_regions                          # arch/x86/kernel/e820.c:1188
                    |--e820_find_active_region                           # arch/x86/kernel/e820.c:1158
                    |--add_active_range                                  # mm/page_alloc.c:3974
                |--sparse_memory_present_with_active_regions             # mm/page_alloc.c:3461
                |--setup_bootmem_allocator                               # arch/x86/mm/init_32.c
                    |--bootmem_bootmap_pages                             # mm/bootmem.c:66
                    |--find_e820_area                                    # arch/x86/kernel/e820.c:1000
                    |--reserve_early                                     # arch/x86/kernel/e820.c:894
                        |--drop_overlaps_that_are_ok                     # arch/x86/kernel/e820.c:781
                        |--__reserve_early                               # arch/x86/kernel/e820.c:838
            |--acpi_reserve_bootmem                                      # arch/x86/kernel/acpi/sleep.c:129
                |--__alloc_bootmem_low                                   # mm/bootmem.c:747
                    |--___alloc_bootmem                                  # mm/bootmem.c:613
                        |--___alloc_bootmem_nopanic                      # mm/bootmem.c:562
                            |--alloc_arch_preferred_bootmem              # mm/bootmem.c:541
                            |--alloc_bootmem_core                        # mm/bootmem.c:434
            |--find_smp_config                                           # arch/x86/include/asm/mpspec.h:72
                |--default_find_smp_config                               # arch/x86/kernel/mpparse.c:728
                    |--smp_scan_config                                   # arch/x86/kernel/mpparse.c:688
            |--reserve_crashkernel                                       # arch/x86/kernel/setup.c:531
                |--get_total_mem                                         # arch/x86/kernel/setup.c:519
                |--parse_crashkernel
                |--reserve_bootmem_generic
                |--insert_resource
            |--reserve_ibft_region                                       # drivers/firmware/iscsi_ibft_find.c:55
                |--reserve_bootmem
            |--x86_init.paging.pagetable_setup_start
                -> native_pagetable_setup_start                          # arch/x86/mm/init_32.c:472
                |--paravirt_alloc_pmd                                    # arch/x86/include/asm/pgalloc.h:16
            |--paging_init                                               # arch/x86/mm/init_32.c:818
                |--pagetable_init                                        # arch/x86/mm/init_32.c:542
                    |--permanent_kmaps_init                              # arch/x86/mm/init_32.c:397
                        |--page_table_range_init                         # arch/x86/mm/init_32.c:198
                            |--one_md_table_init                         # arch/x86/mm/init_32.c:79
                            |--one_page_table_init                       # arch/x86/mm/init_32.c:108
                            |--page_table_kmap_check
                    |--__flush_tlb_all                                   # arch/x86/include/asm/tlbflush.h:49
                        |--__native_flush_tlb_single                     # arch/x86/include/asm/tlbflush.h:44
                    |--kmap_init                                         # arch/x86/mm/init_32.c:383
                    |--zone_sizes_init                                   # arch/x86/mm/init_32.c:737
            |--x86_init.paging.pagetable_setup_done
                -> native_pagetable_setup_done                           # arch/x86/mm/init_32.c:504
            |--setup_trampoline_page_table                               # arch/x86/kernel/trampoline.c:44
                |--clone_pgd_range                                       # arch/x86/include/asm/pgtable.h:619
            |--generic_apic_probe                                        # arch/x86/kernel/apic/probe_32.c:240
            |--early_quirks                                              # arch/x86/kernel/early-quirks.c:279
                |--early_pci_allowed                                     # arch/x86/pci/early.c:60
                |--check_dev_quirk                                       # arch/x86/kernel/early-quirks.c:240
                    |--read_pci_config_16                                # arch/x86/pci/early.c:29
                    |--read_pci_config_byte                              # arch/x86/pci/early.c:20
            |--acpi_boot_init                                            # arch/x86/kernel/acpi/boot.c:1595
                |--dmi_check_system                                      # drivers/firmware/dmi_scan.c:467
                    |--dmi_matches                                       # drivers/firmware/dmi_scan.c:426
                |--acpi_table_parse                                      # drivers/acpi/tables.c:278
                |--acpi_process_madt                                     # arch/x86/kernel/acpi/boot.c:1186
            |--sfi_init                                                  # drivers/sfi/sfi_core.c:385
            |--get_smp_config                                            # arch/x86/include/asm/mpspec.h:60
                |--default_get_smp_config                                # arch/x86/kernel/mpparse.c:606
            |--prefill_possible_map                                      # arch/x86/kernel/smpboot.c:1202
                |--set_cpu_possible                                      # kernel/cpu.c:591
                    |--cpumask_set_cpu                                   # include/linux/cpumask.h:240
            |--init_apic_mappings                                        # arch/x86/kernel/apic/apic.c:1622
                |--read_apic_id                                          # arch/x86/include/asm/apic.h:480
                    |--apic_read                                         # arch/x86/include/asm/apic.h:376
                        |--native_apic_mem_read                          # arch/x86/include/asm/apic.h:110
                        |--default_get_apic_id                           # arch/x86/include/asm/apic.h:420
            |--ioapic_init_mappings                                      # arch/x86/kernel/apic/io_apic.c:4163
                |--ioapic_setup_resources                                # arch/x86/kernel/apic/io_apic.c:4133
            |--probe_nr_irqs_gsi                                         # arch/x86/kernel/apic/io_apic.c:3841
                |--acpi_probe_gsi                                        # arch/x86/kernel/acpi/boot.c:849
            |--e820_reserve_resources                                    # arch/x86/kernel/e820.c:1334
                |--firmware_map_add_early                                # arch/x86/kernel/e820.c:163
                    |--firmware_map_add_entry                            # drivers/firmware/memmap.c:108
            |--e820_mark_nosave_regions                                  # arch/x86/kernel/e820.c:681
                |--register_nosave_region                                # include/linux/suspend.h:233
                    |--__register_nosave_region                          # kernel/power/snapshot.c:599
            |--i386_reserve_resources                                    # arch/x86/kernel/setup.c:1070
                |--request_resource                                      # kernel/resource.c:198
                    |--__request_resource                                # kernel/resource.c:144
                |--reserve_standard_io_resources                         # arch/x86/kernel/setup.c:601
            |--e820_setup_gap                                            # arch/x86/kernel/e820.c:618
            
            
                
            
                    
                
            
            
                
                    
```


# Links
  * [SMP Boot](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/smpboot.html)
  * [Linux Kernel 2.4 Internals](http://www.tldp.org/LDP/lki/lki.html#toc4)
