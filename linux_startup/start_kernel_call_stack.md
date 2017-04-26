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
                |--e820_search_gap                                       # arch/x86/kernel/e820.c:577
            |--x86_init.oem.banner -> x86_init_noop                      # arch/x86/kernel/x86_init.c:18
            |--mcheck_intel_therm_init                                   # arch/x86/kernel/cpu/mcheck/therm_throt.c:259
        |--mm_init_owner                                                 # include/linux/sched.h:2603
        |--setup_command_line                                            # init/main.c:393
        |--setup_nr_cpu_ids                                              # init/main.c:362
        |--setup_per_cpu_areas                                           # arch/x86/kernel/setup_percpu.c:171
            |--pcpu_embed_first_chunk                                    # mm/percpu.c:1862
                |--pcpu_build_alloc_info                                 # mm/percpu.c:1395
        |--smp_prepare_boot_cpu                                          # arch/x86/include/asm/smp.h:84
            |--native_smp_prepare_boot_cpu                               # arch/x86/kernel/smpboot.c:1155
                |--switch_to_new_gdt                                     # arch/x86/kernel/cpu/common.c:344
        |--build_all_zonelists                                           # mm/page_alloc.c:2768
            |--set_zonelist_order                                        # mm/page_alloc.c:2702
            |--__build_all_zonelists                                     # mm/page_alloc.c:2752
                |--build_zonelists                                       # mm/page_alloc.c:2707
                    |--build_zonelists_node                              # mm/page_alloc.c:2321
                |--build_zonelist_cache                                  # mm/page_alloc.c:2744
            |--mminit_verify_zonelist                                    # mm/mm_init.c:22
            |--cpuset_init_current_mems_allowed                          # kernel/cpuset.c:2199
                |--__nodes_setall                                        # include/linux/nodemask.h:114
        |--page_alloc_init                                               # mm/page_alloc.c:4511
            |--hotcpu_notifier -> cpu_notifier -> register_cpu_notifier  # kernel/cpu.c:128
                |--cpu_maps_update_begin                                 # kernel/cpu.c:76
                |--raw_notifier_chain_register                           # kernel/notifier.c:344
                    |--notifier_chain_register                           # kernel/notifier.c:21
                |--cpu_maps_update_done                                  # kernel/cpu.c:81
        |--parse_early_param                                             # init/main.c:468
        |--parse_args                                                    # kernel/params.c:131
        |--pidhash_init                                                  # kernel/pid.c:502
            |--alloc_large_system_hash                                   # mm/page_alloc.c:4844
                |--alloc_bootmem_nopanic -> __alloc_bootmem_nopanic      # mm/bootmem.c:607
                    |--___alloc_bootmem_nopanic                          # mm/bootmem.c:562
                        |--alloc_arch_preferred_bootmem                  # mm/bootmem.c:541
                        |--alloc_bootmem_core                            # mm/bootmem.c:434
        |--vfs_caches_init_early                                         # fs/dcache.c:2314
            |--dcache_init_early                                         # fs/dcache.c:2253
            |--inode_init_early                                          # fs/inode.c:1561
        |--sort_main_extable                                             # kernel/extable.c:39
            |--sort_extable                                              # lib/extable.c:36
                |--sort                                                  # lib/sort.c:47
        |--trap_init                                                     # arch/x86/kernel/traps.c:941
            |--set_intr_gate                                             # arch/x86/include/asm/desc.h:338
                |--_set_gate                                             # arch/x86/include/asm/desc.h:320
                    |--pack_gate                                         # arch/x86/include/asm/desc.h:64
                    |--write_idt_entry -> native_write_idt_entry         # arch/x86/include/asm/desc.h:115
            |--cpu_init                                                  # arch/x86/kernel/cpu/common.c:1200
                |--load_idt -> native_load_idt                           # arch/x86/include/asm/desc.h:222
                |--switch_to_new_gdt                                     # arch/x86/kernel/cpu/common.c:344
                    |--load_gdt -> native_load_gdt                       # arch/x86/include/asm/desc.h:217
                    |--load_percpu_segment                               # arch/x86/kernel/cpu/common.c:329
                |--enter_lazy_tlb                                        # arch/x86/include/asm/mmu_context.h
                |--load_sp0 -> native_load_sp0                           # arch/x86/include/asm/processor.h:555
                |--set_tss_desc -> __set_tss_desc                        # arch/x86/include/asm/desc.h:176
                    |--set_tssldt_descriptor                             # arch/x86/include/asm/desc.h:157
                    |--write_gdt_entry -> native_write_gdt_entry         # arch/x86/include/asm/desc.h:127
                |--load_TR_desc -> native_load_tr_desc                   # arch/x86/include/asm/desc.h:212
                |--load_LDT                                              # arch/x86/include/asm/desc.h:290
                    |--load_LDT_nolock                                   # arch/x86/include/asm/desc.h:285
                |--__set_tss_desc
                |--clear_all_debug_regs                                  # arch/x86/kernel/cpu/common.c:1074
                |--clear_used_math
                |--mxcsr_feature_mask_init                               # arch/x86/kernel/i387.c:45
                |--init_thread_xstate                                    # arch/x86/kernel/i387.c:61
                |--xsave_init                                            # arch/x86/kernel/xsave.c:289
            |--x86_init.irqs.trap_init -> x86_init_noop                  # arch/x86/kernel/x86_init.c:18
        |--mm_init                                                       # init/main.c:507
            |--mem_init                                                  # arch/x86/mm/init_32.c:860
                |--pci_iommu_alloc                                       # arch/x86/kernel/pci-dma.c:123
                |--free_all_bootmem                                      # mm/bootmem.c:224
                    |--free_all_bootmem_core                             # mm/bootmem.c:146
                |--page_is_ram                                           # arch/x86/mm/ioremap.c:27
                |--PageReserved ?
                |--set_highmem_pages_init                                # arch/x86/mm/highmem_32.c:109
                    |--is_highmem                                        # include/linux/mmzone.h:734
                    |--add_highpages_with_active_regions                 # arch/x86/mm/init_32.c:455
                        |--work_with_active_regions                      # mm/page_alloc.c:3441
                            |--work_fn -> add_highpages_work_fn          # arch/x86/mm/init_32.c:428
                                |--add_one_highpage_init                 # arch/x86/mm/init_32.c:415
                |--save_pg_dir                                           # arch/x86/mm/init_32.c:557
                |--zap_low_mappings                                      # arch/x86/mm/init_32.c:567
                    |--set_pgd
                    |--__native_flush_tlb
            |--kmem_cache_init                                           # mm/slub.c:3174
                |--init_alloc_cpu                                        # mm/slub.c:2204
                    |--init_alloc_cpu_cpu                                # mm/slub.c:2191
                |--create_kmalloc_cache                                  # mm/slub.c:2693
                    |--kmem_cache_open                                   # mm/slub.c:2464
                        |--kmem_cache_flags                              # mm/slub.c:1038
                        |--calculate_sizes                               # mm/slub.c:2344
                        |--set_min_partial                               # mm/slub.c:2331
                        |--init_kmem_cache_nodes                         # mm/slub.c:2324
                            |--init_kmem_cache_node                      # mm/slub.c:2086
                        |--alloc_kmem_cache_cpus                         # mm/slub.c:2168
                            |--get_cpu_slab                              # mm/slub.c:245
                            |--alloc_kmem_cache_cpu                      # mm/slub.c:2122
                            |--init_kmem_cache_cpu                       # mm/slub.c:2072
                    |--sysfs_slab_add                                    # mm/slub.c:4567
            |--vmalloc_init                                              # mm/vmalloc.c:1097
        |--sched_init                                                    # kernel/sched.c:9657
            |--init_defrootdomain                                        # kernel/sched.c:8282
            |--init_rt_bandwidth                                         # kernel/sched.c:176
            |--set_load_weight                                           # kernel/sched.c:1919
            |--open_softirq                                              # kernel/softirq.c:374
            |--enter_lazy_tlb                                            # arch/x86/include/asm/mmu_context.h:25
            |--init_idle                                                 # kernel/sched.c:7252
                |--__sched_fork                                          # kernel/sched.c:2637
                |--__set_task_cpu                                        # kernel/sched.c:1795
                |--ftrace_graph_init_idle_task                           # kernel/trace/ftrace.c:3283
            |--perf_event_init                                           # kernel/perf_event.c:5069
        |--early_irq_init                                                # kernel/irq/handle.c:150
            |--init_irq_default_affinity                                 # kernel/irq/handle.c:47
            |--arch_probe_nr_irqs                                        # arch/x86/kernel/apic/io_apic.c:3864
            |--arch_early_irq_init                                       # arch/x86/kernel/apic/io_apic.c:187
        |--init_IRQ                                                      # arch/x86/kernel/irqinit.c:143
        |--prio_tree_init                                                # lib/prio_tree.c:71
        |--init_timers                                                   # kernel/timer.c:1653
            |--timer_cpu_notify                                          # kernel/timer.c:1626
                |--init_timers_cpu                                       # kernel/timer.c:1520
            |--init_timer_stats                                          # kernel/time/timer_stats.c:406
            |--register_cpu_notifier                                     # kernel/cpu.c:128
            |--open_softirq
        |--hrtimers_init                                                 # kernel/hrtimer.c:1758
        |--softirq_init                                                  # kernel/softirq.c:710
            |--register_hotcpu_notifier
            |--open_softirq
        |--timekeeping_init                                              # kernel/time/timekeeping.c:566
        |--time_init                                                     # arch/x86/kernel/time.c:118
        |--profile_init                                                  # kernel/profile.c:105
        |--early_boot_irqs_on                                            # kernel/lockdep.c:2294
        |--local_irq_enable                                              # include/linux/irgflags.h:59
            -> |--trace_hardirqs_on                                      # kernel/lockdep.c:2346
                   |--trace_hardirqs_on_caller                           # kernel/lockdep.c:2302
               |--raw_local_irq_enable                                   # arch/x86/include/asm/irqflags.h:79
                   |--native_irq_enable                                  # arch/x86/include/asm/irqflags.h:42
       |--set_gfp_allowed_mask
       |--kmem_cache_init_late                                           # mm/slub.c:3278
       |--console_init                                                   # drivers/char/tty_io.c:3091
           |--tty_ldisc_begin                                            # drivers/char/tty_ldisc.c:918
               |--tty_register_ldisc                                     # drivers/char/tty_ldisc.c::101
           |--con_init                                                   # drivers/char/vt.c:2833
           |--hvc_console_init                                           # drivers/char/hvc_console.c:222
           |--serial8250_console_init                                    # drivers/serial/8250.c:2821
       |--lockdep_info
       |--locking_selftest                                               # lib/locking-selftest.c:1113
       |--page_cgroup_init                                               # include/linux/page_cgroup.h:29
       
        

        
        
                            
                        
                        
                

                    
                
                

            
                
            
            
            
                
            
                    
                
            
            
                
                    
```


# Links
  * [SMP Boot](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/smpboot.html)
  * [Linux Kernel 2.4 Internals](http://www.tldp.org/LDP/lki/lki.html#toc4)
