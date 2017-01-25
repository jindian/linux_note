# start_kernel part IV

## _initialize parameters of kernel sections_

  All sections defined in arch/x86/kernel/vmlinux.lds.S
  
## _copy command line_

  Copy command line from `boot_command_line`, `boot_command_line` is initialized before start_kernel in arch/x86/kernel/head_32.S,  the content of `boot_command_line` is:
  
```boot_command_line

p boot_command_line 
$73 = 0xc17304c0 <boot_command_line> "BOOT_IMAGE=/boot/vmlinuz-2.6.32.69 root=/dev/sda"
```

## _parse early boot command line_

  Parse command line `BOOT_IMAGE=/boot/vmlinuz-2.6.32.69 root=/dev/sda` with `parse_early_param`, finally involved routine is `parse_args`.

##  _initialize reserve early setup data_

  Value of setup_data in linux boot protocol in our image is not set, we could ignore `reserve_early_setup_data`.
  PCI device details are recorded in a singly-linked list at boot_params.hdr.setup_data, but we have no such device here, so ignore it.

```reserve_early_setup_data

(gdb) p /x boot_params.hdr.version 
$83 = 0x20a
(gdb) p /x boot_params.hdr.setup_data
$84 = 0x0
```

## _check acpi_
  
  `acpi_mps_check` If CONFIG_X86_LOCAL_APIC is set and CONFIG_x86_MPPAARSE is
not set, acpi_mps_check prints warning message if the one of the command line options: acpi=off , acpi=noirq or pci=noacpi passed to the kernel. If acpi_mps_check returns 1 which means that we disable local APIC and clears X86_FEATURE_APIC bit in the of the current CPU with the setup_clear_cpu_cap macro.

## _dump pci devices_
  
  Dump pic devices with `early_dump_pci_devices` if `pci_early_dump_regs` set with nonzero. `pci_early_dump_regs` defined in `arch/x86/pci/common.c` line 22, it set as 1 in routine `pcibios_setup` defined in `arch/x86/pci/common.c` line 522 if we take `pci=earlydump` in boot command line.
  
```pcibios_setup

char * __devinit  pcibios_setup(char *str)
{

    ......
    
        } else if (!strcmp(str, "earlydump")) {
                pci_early_dump_regs = 1;
                return NULL;
        }
        
    ......
```

  Every architecture has its own implementation of routine `pcibios_setup`. `pcibios_setup` involved by `pci_setup` in file `drivers/pci/pci.c` line 2763, `pci_setup` is the response function of early parameter `pci`.

  All early parameters defined with same macro `#define early_param(str, fn)` in file  `linux/init.h` line 241, for example: `early_param("pci", pci_setup)`.
  
```early_param

/*
 * Only for really core code.  See moduleparam.h for the normal way.
 *
 * Force the alignment so the compiler doesn't space elements of the
 * obs_kernel_param "array" too far apart in .init.setup.
 */
#define __setup_param(str, unique_id, fn, early)                        \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id, fn, early }

#define __setup(str, fn)                                        \
        __setup_param(str, fn, fn, 0)

/* NOTE: fn is as per module_param, not __setup!  Emits warning if fn
 * returns non-zero. */
#define early_param(str, fn)                                    \
        __setup_param(str, fn, fn, 1)
```
  
  From above defination, early parameters are stored in section `.init.setup` with uniqure identifier. Early parameters are parsed by routine `parse_early_param`, we just introduced it in `setup_arch` which will in involved in start_kernel again after `setup_arch`.

  In our `boot_command_line` we haven't set `pci=earlydump`, let's set value of `pci_early_dump_regs` as 1 mannually to check the detail of routine `early_dump_pci_devices`.
  
  `early_dump_pci_devices` reads pci configuration and print it to kernel log.

## _finish e820 parsing_
  
  Sanitize e820 map if `userdef` set as 1, the value of `userdef` is set as 1 in `parse_memopt` or `parse_memmap_opt`, both of the routines are response function of memory early parameters.

```finish_e820_parsing

void __init finish_e820_parsing(void)
{
	if (userdef) {
		u32 nr = e820.nr_map;

		if (sanitize_e820_map(e820.map, ARRAY_SIZE(e820.map), &nr) < 0)
			early_panic("Invalid user supplied memory map");
		e820.nr_map = nr;

		printk(KERN_INFO "user-defined physical RAM map:\n");
		e820_print_map("user");
	}
}
```

  `sanitize_e820_map` already introduced in setup_memory_map.

## _initialize efi_
  
  `efi_init` is used to map physical memory to EFI memory map if `efi_enabled` set with nonzero which is ignored as we don't use efi boot here.

## _scan memory of SMBIOS to get the DMI table_

  DMI is the abbreviated terms of [Desktop Management Interface](https://en.wikipedia.org/wiki/Desktop_Management_Interface)
  
  System Management BIOS (SMBIOS) is a standard developed by DMTF. The purpose of this standard is to allow the operating system to retrieve information about the PC. On booting the SMBIOS will put a table somewhere in memory. By parsing this table it is possible to access information about the computer and its capabilities. The SMBIOS Entry Point Table is located somewhere between the addresses 0xF0000 and 0xFFFFF, and must be on a 16-byte boundary. To find the specific location of the start of the table it is necessary to search that region of memory for the string "_DMI_", and then check the structure's checksum (add all bytes and see if the lowest 8 bits of the result are zero).
  
  There are two approaches to scan SMBIOS, one with efi table and the other with e820 memory map, here we use the second approach.
  
  `dmi_scan_machine` firstly mappes address 0xF0000 size 0x10000 with `dmi_ioremap` which expands to `early_ioremap`, `early_ioremap` involves `__early_ioremap` inside which we could find above memory block is devided into 16 pages for mapping and set to page table with `early_set_fixmap`.

```

(gdb) p nrpages 
$1 = 16
```

  After mapping memory region completed, iterates mapped memory to check if `_DMI_` string presents with routine `dmi_present`.

```dmi_present

(gdb) p p
$9 = 0xffd00000 ""
(gdb) p q
$10 = 0xffd00b10 "_DMI_.3\001\300\t\017"
```

  If find dmi table, decode the table and save the dmi string.

## _check system DMI data_

  There is a known issue about low memory corruption BIOS problem with specific BIOS vender or board name, all these information are saved in a list named `bad_bios_dmi_table`, check the system DMI data, if matches, working around the BIOS low RAM curruption problem with `dmi_low_memory_corruption`.

# Links
  * [setup_data](https://lwn.net/Articles/632528/)
  * [MultiProcessor Specification](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf)
  * [Add efi e820 memory mapping on x86](http://yarchive.net/comp/linux/efi.html)
  * [Converting EFI memory Map to E820 map](http://stackoverflow.com/questions/17591351/converting-efi-memory-map-to-e820-map)
  * [Why do we pass memory map both in e820 and efi_info](http://lists.infradead.org/pipermail/kexec/2014-May/011764.html)
  * [Unified Extensible Firmware Interface](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
  * [System Management BIOS](http://wiki.osdev.org/System_Management_BIOS)
  * [Desktop Management Interface](https://en.wikipedia.org/wiki/Desktop_Management_Interface)
  
  