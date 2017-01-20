# start_kernel part III

Continue `setup_arch` of x86 architecture.

  5. `early_ioremap_init` initialize early ioremap for early initialization code before normal mapping functions are ready.
   
   transfer fixed bitmap physical address to virtual address and put it into `slot_virt`, the result of `slot_virt` after this procedure completed: `{0xffd00000, 0xffd40000, 0xffd80000, 0xffdc0000}`, length of the virtual address put into `slot_virt` is 1M.
   
   get pmd of the start address of fixed bitmap with `early_ioremap_pmd`.
   
   `bm_pte` is the entry of early ioremap page table, after initializing it's set as the first page table of the pmd.
   
   check if start address and end address of fixed bitmap are in same pmd, the result is yes.
   
 6. initialize linux global parameters with `boot_params`, the `boot_params` as follow:

```boot_params

(gdb) p boot_params
$14 = {screen_info = {orig_x = 0 '\000', orig_y = 3 '\003', 
    ext_mem_k = 32768, orig_video_page = 0, orig_video_mode = 3 '\003', 
    orig_video_cols = 80 'P', unused2 = 0, orig_video_ega_bx = 0, 
    unused3 = 0, orig_video_lines = 25 '\031', orig_video_isVGA = 1 '\001', 
    orig_video_points = 16, lfb_width = 0, lfb_height = 0, lfb_depth = 0, 
    lfb_base = 0, lfb_size = 0, cl_magic = 41791, cl_offset = 4096, 
    lfb_linelength = 0, red_size = 0 '\000', red_pos = 0 '\000', 
    green_size = 0 '\000', green_pos = 0 '\000', blue_size = 0 '\000', 
    blue_pos = 0 '\000', rsvd_size = 0 '\000', rsvd_pos = 0 '\000', 
    vesapm_seg = 0, vesapm_off = 0, pages = 0, vesa_attributes = 0, 
    capabilities = 0, _reserved = "\000\000\000\000\000"}, apm_bios_info = {
    version = 0, cseg = 0, offset = 0, cseg_16 = 0, dseg = 0, flags = 0, 
    cseg_len = 0, cseg_16_len = 0, dseg_len = 0}, _pad2 = "\000\000\000", 
  tboot_addr = 0, ist_info = {signature = 0, command = 0, event = 0, 
    perf_level = 0}, _pad3 = '\000' <repeats 15 times>, 
  hd0_info = '\000' <repeats 15 times>, hd1_info = '\000' <repeats 15 times>, 
  sys_desc_table = {length = 0, table = '\000' <repeats 13 times>}, 
  _pad4 = '\000' <repeats 143 times>, edid_info = {
    dummy = '\000' <repeats 127 times>}, efi_info = {
    efi_loader_signature = 0, efi_systab = 0, efi_memdesc_size = 0, 
    efi_memdesc_version = 0, efi_memmap = 0, efi_memmap_size = 0, 
    efi_systab_hi = 0, efi_memmap_hi = 0}, alt_mem_k = 32768, 
  scratch = 16777253, e820_entries = 6 '\006', eddbuf_entries = 0 '\000', 
  edd_mbr_sig_buf_entries = 0 '\000', _pad6 = "\000\000\000\000\000", hdr = {
    setup_sects = 29 '\035', root_flags = 1, syssize = 255531, ram_size = 0, 
    vid_mode = 65535, root_dev = 256, boot_flag = 0, jump = 25323, 
    header = 1400005704, version = 522, realmode_swtch = 0, start_sys = 4096, 
    kernel_version = 12256, type_of_loader = 114 'r', loadflags = 129 '\201', 
    setup_move_size = 32768, code32_start = 16777216, 
    ramdisk_image = 125521920, ramdisk_size = 2688152, bootsect_kludge = 0, 
    heap_end_ptr = 36352, ext_loader_ver = 0 '\000', 
    ext_loader_type = 0 '\000', cmd_line_ptr = 585728, 
    initrd_addr_max = 2147483647, kernel_alignment = 16777216, 
    relocatable_kernel = 1 '\001', _pad2 = "\r\000", cmdline_size = 2047, 
    hardware_subarch = 0, hardware_subarch_data = 0, payload_offset = 108, 
    payload_length = 4071132, setup_data = 0}, 
  _pad7 = "\000\000\000\001\000\000\000\000\000`\005\001\214؎\300\374\214\322\071\302\211\342t\026\272@M\366\006\021\002\200t\004\213\026$\002\201\302\000\002s\002\061҃\342\374u\003\272\374\377\216", edd_mbr_sig_buffer = {3071239888, 
    1746861026, 1724580507, 981220993, 1515891285, 2160007029, 1296283962, 
    700461414, 48873977, 1722509043, 2785768, 1085826560, 1711276035, 23528, 
    4243190784, 1716741887}, e820_map = {{addr = 0, size = 654336, type = 1}, 
    {addr = 654336, size = 1024, type = 2}, {addr = 983040, size = 65536, 
      type = 2}, {addr = 1048576, size = 133160960, type = 1}, {
      addr = 134209536, size = 8192, type = 2}, {addr = 4294705152, 
      size = 262144, type = 2}, {addr = 8386105371570413680, 
      size = 7959390400868086389, type = 774778468}, {addr = 429504151814154, 
      size = 5360972406433298790, type = 2162564980}, {
      addr = 5288919717233059044, size = 9288118376811991924, 
      type = 3874185444}, {addr = 7403972105352839848, 
      size = 7404875461093975747, type = 3526452819}, {
      addr = 16901727696821084814, size = 9900603030087886436, 
      type = 2372298698}, {addr = 7356195521970962716, 
      size = 16573809675471316034, type = 2707842432}, {
      addr = 16998683756064014864, size = 1047480971247576579, 
      type = 1533411840}, {addr = 9468346965594391398, size = 1102655663340, 
      type = 2215594854}, {addr = 140532646082, size = 9612651929639184486, 
      type = 1711699136}, {addr = 0, size = 0, 
      type = 0} <repeats 113 times>}, _pad8 = '\000' <repeats 47 times>, 
  eddbuf = {{device = 0 '\000', version = 0 '\000', interface_support = 0, 
      legacy_max_cylinder = 0, legacy_max_head = 0 '\000', 
      legacy_sectors_per_track = 0 '\000', params = {length = 0, 
        info_flags = 0, num_default_cylinders = 0, num_default_heads = 0, 
        sectors_per_track = 0, number_of_sectors = 0, bytes_per_sector = 0, 
        dpte_ptr = 0, key = 0, device_path_info_length = 0 '\000', 
        reserved2 = 0 '\000', reserved3 = 0, host_bus_type = "\000\000\000", 
        interface_type = "\000\000\000\000\000\000\000", interface_path = {
          isa = {base_address = 0, reserved1 = 0, reserved2 = 0}, pci = {
            bus = 0 '\000', slot = 0 '\000', function = 0 '\000', 
            channel = 0 '\000', reserved = 0}, ibnd = {reserved = 0}, xprs = {
            reserved = 0}, htpt = {reserved = 0}, unknown = {reserved = 0}}, 
        device_path = {ata = {device = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0, reserved3 = 0, reserved4 = 0}, atapi = {
            device = 0 '\000', lun = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0 '\000', reserved3 = 0, reserved4 = 0}, scsi = {
            id = 0, lun = 0, reserved1 = 0, reserved2 = 0}, usb = {
            serial_number = 0, reserved = 0}, i1394 = {eui = 0, 
            reserved = 0}, fibre = {wwid = 0, lun = 0}, i2o = {
            identity_tag = 0, reserved = 0}, raid = {array_number = 0, 
            reserved1 = 0, reserved2 = 0}, sata = {device = 0 '\000', 
            reserved1 = 0 '\000', reserved2 = 0, reserved3 = 0, 
            reserved4 = 0}, unknown = {reserved1 = 0, reserved2 = 0}}, 
        reserved4 = 0 '\000', checksum = 0 '\000'}}, {device = 0 '\000', 
      version = 0 '\000', interface_support = 0, legacy_max_cylinder = 0, 
      legacy_max_head = 0 '\000', legacy_sectors_per_track = 0 '\000', 
      params = {length = 0, info_flags = 0, num_default_cylinders = 0, 
        num_default_heads = 0, sectors_per_track = 0, number_of_sectors = 0, 
        bytes_per_sector = 0, dpte_ptr = 0, key = 0, 
        device_path_info_length = 0 '\000', reserved2 = 0 '\000', 
        reserved3 = 0, host_bus_type = "\000\000\000", 
        interface_type = "\000\000\000\000\000\000\000", interface_path = {
          isa = {base_address = 0, reserved1 = 0, reserved2 = 0}, pci = {
            bus = 0 '\000', slot = 0 '\000', function = 0 '\000', 
            channel = 0 '\000', reserved = 0}, ibnd = {reserved = 0}, xprs = {
            reserved = 0}, htpt = {reserved = 0}, unknown = {reserved = 0}}, 
        device_path = {ata = {device = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0, reserved3 = 0, reserved4 = 0}, atapi = {
            device = 0 '\000', lun = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0 '\000', reserved3 = 0, reserved4 = 0}, scsi = {
            id = 0, lun = 0, reserved1 = 0, reserved2 = 0}, usb = {
            serial_number = 0, reserved = 0}, i1394 = {eui = 0, 
            reserved = 0}, fibre = {wwid = 0, lun = 0}, i2o = {
            identity_tag = 0, reserved = 0}, raid = {array_number = 0, 
            reserved1 = 0, reserved2 = 0}, sata = {device = 0 '\000', 
            reserved1 = 0 '\000', reserved2 = 0, reserved3 = 0, 
            reserved4 = 0}, unknown = {reserved1 = 0, reserved2 = 0}}, 
        reserved4 = 0 '\000', checksum = 0 '\000'}}, {device = 0 '\000', 
      version = 0 '\000', interface_support = 0, legacy_max_cylinder = 0, 
      legacy_max_head = 0 '\000', legacy_sectors_per_track = 0 '\000', 
      params = {length = 0, info_flags = 0, num_default_cylinders = 0, 
        num_default_heads = 0, sectors_per_track = 0, number_of_sectors = 0, 
        bytes_per_sector = 0, dpte_ptr = 0, key = 0, 
        device_path_info_length = 0 '\000', reserved2 = 0 '\000', 
        reserved3 = 0, host_bus_type = "\000\000\000", 
        interface_type = "\000\000\000\000\000\000\000", interface_path = {
          isa = {base_address = 0, reserved1 = 0, reserved2 = 0}, pci = {
            bus = 0 '\000', slot = 0 '\000', function = 0 '\000', 
            channel = 0 '\000', reserved = 0}, ibnd = {reserved = 0}, xprs = {
            reserved = 0}, htpt = {reserved = 0}, unknown = {reserved = 0}}, 
        device_path = {ata = {device = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0, reserved3 = 0, reserved4 = 0}, atapi = {
            device = 0 '\000', lun = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0 '\000', reserved3 = 0, reserved4 = 0}, scsi = {
            id = 0, lun = 0, reserved1 = 0, reserved2 = 0}, usb = {
            serial_number = 0, reserved = 0}, i1394 = {eui = 0, 
            reserved = 0}, fibre = {wwid = 0, lun = 0}, i2o = {
            identity_tag = 0, reserved = 0}, raid = {array_number = 0, 
            reserved1 = 0, reserved2 = 0}, sata = {device = 0 '\000', 
            reserved1 = 0 '\000', reserved2 = 0, reserved3 = 0, 
            reserved4 = 0}, unknown = {reserved1 = 0, reserved2 = 0}}, 
        reserved4 = 0 '\000', checksum = 0 '\000'}}, {device = 0 '\000', 
      version = 0 '\000', interface_support = 0, legacy_max_cylinder = 0, 
      legacy_max_head = 0 '\000', legacy_sectors_per_track = 0 '\000', 
      params = {length = 0, info_flags = 0, num_default_cylinders = 0, 
        num_default_heads = 0, sectors_per_track = 0, number_of_sectors = 0, 
        bytes_per_sector = 0, dpte_ptr = 0, key = 0, 
        device_path_info_length = 0 '\000', reserved2 = 0 '\000', 
        reserved3 = 0, host_bus_type = "\000\000\000", 
        interface_type = "\000\000\000\000\000\000\000", interface_path = {
          isa = {base_address = 0, reserved1 = 0, reserved2 = 0}, pci = {
            bus = 0 '\000', slot = 0 '\000', function = 0 '\000', 
            channel = 0 '\000', reserved = 0}, ibnd = {reserved = 0}, xprs = {
            reserved = 0}, htpt = {reserved = 0}, unknown = {reserved = 0}}, 
        device_path = {ata = {device = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0, reserved3 = 0, reserved4 = 0}, atapi = {
            device = 0 '\000', lun = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0 '\000', reserved3 = 0, reserved4 = 0}, scsi = {
            id = 0, lun = 0, reserved1 = 0, reserved2 = 0}, usb = {
            serial_number = 0, reserved = 0}, i1394 = {eui = 0, 
            reserved = 0}, fibre = {wwid = 0, lun = 0}, i2o = {
            identity_tag = 0, reserved = 0}, raid = {array_number = 0, 
            reserved1 = 0, reserved2 = 0}, sata = {device = 0 '\000', 
            reserved1 = 0 '\000', reserved2 = 0, reserved3 = 0, 
            reserved4 = 0}, unknown = {reserved1 = 0, reserved2 = 0}}, 
        reserved4 = 0 '\000', checksum = 0 '\000'}}, {device = 0 '\000', 
      version = 0 '\000', interface_support = 0, legacy_max_cylinder = 0, 
      legacy_max_head = 0 '\000', legacy_sectors_per_track = 0 '\000', 
      params = {length = 0, info_flags = 0, num_default_cylinders = 0, 
        num_default_heads = 0, sectors_per_track = 0, number_of_sectors = 0, 
        bytes_per_sector = 0, dpte_ptr = 0, key = 0, 
        device_path_info_length = 0 '\000', reserved2 = 0 '\000', 
        reserved3 = 0, host_bus_type = "\000\000\000", 
        interface_type = "\000\000\000\000\000\000\000", interface_path = {
          isa = {base_address = 0, reserved1 = 0, reserved2 = 0}, pci = {
            bus = 0 '\000', slot = 0 '\000', function = 0 '\000', 
            channel = 0 '\000', reserved = 0}, ibnd = {reserved = 0}, xprs = {
            reserved = 0}, htpt = {reserved = 0}, unknown = {reserved = 0}}, 
        device_path = {ata = {device = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0, reserved3 = 0, reserved4 = 0}, atapi = {
            device = 0 '\000', lun = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0 '\000', reserved3 = 0, reserved4 = 0}, scsi = {
            id = 0, lun = 0, reserved1 = 0, reserved2 = 0}, usb = {
            serial_number = 0, reserved = 0}, i1394 = {eui = 0, 
            reserved = 0}, fibre = {wwid = 0, lun = 0}, i2o = {
            identity_tag = 0, reserved = 0}, raid = {array_number = 0, 
            reserved1 = 0, reserved2 = 0}, sata = {device = 0 '\000', 
            reserved1 = 0 '\000', reserved2 = 0, reserved3 = 0, 
            reserved4 = 0}, unknown = {reserved1 = 0, reserved2 = 0}}, 
        reserved4 = 0 '\000', checksum = 0 '\000'}}, {device = 0 '\000', 
      version = 0 '\000', interface_support = 0, legacy_max_cylinder = 0, 
      legacy_max_head = 0 '\000', legacy_sectors_per_track = 0 '\000', 
      params = {length = 0, info_flags = 0, num_default_cylinders = 0, 
        num_default_heads = 0, sectors_per_track = 0, number_of_sectors = 0, 
        bytes_per_sector = 0, dpte_ptr = 0, key = 0, 
        device_path_info_length = 0 '\000', reserved2 = 0 '\000', 
        reserved3 = 0, host_bus_type = "\000\000\000", 
        interface_type = "\000\000\000\000\000\000\000", interface_path = {
          isa = {base_address = 0, reserved1 = 0, reserved2 = 0}, pci = {
            bus = 0 '\000', slot = 0 '\000', function = 0 '\000', 
            channel = 0 '\000', reserved = 0}, ibnd = {reserved = 0}, xprs = {
            reserved = 0}, htpt = {reserved = 0}, unknown = {reserved = 0}}, 
        device_path = {ata = {device = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0, reserved3 = 0, reserved4 = 0}, atapi = {
            device = 0 '\000', lun = 0 '\000', reserved1 = 0 '\000', 
            reserved2 = 0 '\000', reserved3 = 0, reserved4 = 0}, scsi = {
            id = 0, lun = 0, reserved1 = 0, reserved2 = 0}, usb = {
            serial_number = 0, reserved = 0}, i1394 = {eui = 0, 
            reserved = 0}, fibre = {wwid = 0, lun = 0}, i2o = {
            identity_tag = 0, reserved = 0}, raid = {array_number = 0, 
            reserved1 = 0, reserved2 = 0}, sata = {device = 0 '\000', 
            reserved1 = 0 '\000', reserved2 = 0, reserved3 = 0, 
            reserved4 = 0}, unknown = {reserved1 = 0, reserved2 = 0}}, 
        reserved4 = 0 '\000', checksum = 0 '\000'}}}, 
  _pad9 = '\000' <repeats 275 times>}
```

  7. `x86_init_noop` initialize x86, but the defination of the function is null.
  
  x86 platform setup functions preset as follow
  
```

arch/x86/kernel/x86_init.c:22

/*
 * The platform setup functions are preset with the default functions
 * for standard PC hardware.
 */
struct x86_init_ops x86_init __initdata = {

	.resources = {
		.probe_roms		= x86_init_noop,
		.reserve_resources	= reserve_standard_io_resources,
		.memory_setup		= default_machine_specific_memory_setup,
	},

	.mpparse = {
		.mpc_record		= x86_init_uint_noop,
		.setup_ioapic_ids	= x86_init_noop,
		.mpc_apic_id		= default_mpc_apic_id,
		.smp_read_mpc_oem	= default_smp_read_mpc_oem,
		.mpc_oem_bus_info	= default_mpc_oem_bus_info,
		.find_smp_config	= default_find_smp_config,
		.get_smp_config		= default_get_smp_config,
	},

	.irqs = {
		.pre_vector_init	= init_ISA_irqs,
		.intr_init		= native_init_IRQ,
		.trap_init		= x86_init_noop,
	},

	.oem = {
		.arch_setup		= x86_init_noop,
		.banner			= default_banner,
	},

	.paging = {
		.pagetable_setup_start	= native_pagetable_setup_start,
		.pagetable_setup_done	= native_pagetable_setup_done,
	},

	.timers = {
		.setup_percpu_clockev	= setup_boot_APIC_clock,
		.tsc_pre_init		= x86_init_noop,
		.timer_init		= hpet_time_init,
	},
};  
```

  8. `setup_memory_map` sanitize BOIS [e820](https://en.wikipedia.org/wiki/E820) map which responsed by e820 as some e820 responses include overlapping entries, replaces the original e820 map with a new one, removing overlaps, and resolving conflicting memory types in favor of highest numbered type. After sanitized setup kernel e820 memory map with the updated BIOS e820 map.
  
```BIOS_e820_map

(gdb) p *pnr_map 
$18 = 6

addr       size       type
0,         9fc00,     1
9fc00,     400,       2
f0000,     10000,     2
100000,    7efe000,   1
7ffe000,   2000,      2
fffc0000,  40000,     2
```



# Links
  * [control register CR3](https://en.wikipedia.org/wiki/Control_register#CR3)
  * [Using I/O Memory](http://www.makelinux.net/ldd3/?u=chp-9-sect-4)
  * [e820](https://en.wikipedia.org/wiki/E820)
  * [INT 15h, AX=E820h - Query System Address Map](http://www.uruk.org/orig-grub/mem64mb.html)