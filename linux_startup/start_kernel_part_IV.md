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

```dmi_ident

(gdb) p dmi_ident 
$25 = {0x0, 0xc1f43000 <.brk.pagetables+8192> "Bochs", 
  0xc1f43008 <.brk.pagetables+8200> "Bochs", 
  0xc1f43010 <.brk.pagetables+8208> "01/01/2011", 
  0xc1f4301c <.brk.pagetables+8220> "QEMU", 
  0xc1f43024 <.brk.pagetables+8228> "Standard PC (i440FX + PIIX, 1996)", 
  0xc1f43048 <.brk.pagetables+8264> "pc-i440fx-trusty", 
  0xc1f4305c <.brk.pagetables+8284> "", 0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 
  0xc1f43060 <.brk.pagetables+8288> "Bochs", 
  0xc1f43068 <.brk.pagetables+8296> "1", 
  0xc1f4306c <.brk.pagetables+8300> "", 0xc1f43070 <.brk.pagetables+8304> "", 
  0xc1f43074 <.brk.pagetables+8308> ""}
```

## _detect if kernel is running in Virtual Machine_

  `init_hypervisor_platform` initialize hypervisor if kernel is running in virtual machine.
  
  `init_hypervisor` is involved by `init_hypervisor_platform`, in which it first detect hypervisor exits or not with `detect_hypervisor_vendor`.
  
  In our kernel only VMware platform is checked, but it's not enough, in lator version of linux kernel, more types of virtual machine are checked.

```

detect_hypervisor_vendor(struct cpuinfo_x86 *c)
{
	if (vmware_platform())
		c->x86_hyper_vendor = X86_HYPER_VENDOR_VMWARE;
	else
		c->x86_hyper_vendor = X86_HYPER_VENDOR_NONE;
}

static inline void __cpuinit
hypervisor_set_feature_bits(struct cpuinfo_x86 *c)
{
	if (boot_cpu_data.x86_hyper_vendor == X86_HYPER_VENDOR_VMWARE) {
		vmware_set_feature_bits(c);
		return;
	}
}

void __cpuinit init_hypervisor(struct cpuinfo_x86 *c)
{
	detect_hypervisor_vendor(c);
	hypervisor_set_feature_bits(c);
}

void __init init_hypervisor_platform(void)
{
	init_hypervisor(&boot_cpu_data);
	if (boot_cpu_data.x86_hyper_vendor == X86_HYPER_VENDOR_VMWARE)
		vmware_platform_setup();
}
```

  How to detect the hypervisor? For example VMware:
  
  It uses cpuid instruction in cpuid routine to get vendor information, results stored in registers eax, ebx, ecx, edx, appends the result to a single string.
  
```vmware_platform

int vmware_platform(void)
{
	if (cpu_has_hypervisor) {
		unsigned int eax, ebx, ecx, edx;
		char hyper_vendor_id[13];

		cpuid(CPUID_VMWARE_INFO_LEAF, &eax, &ebx, &ecx, &edx);
		memcpy(hyper_vendor_id + 0, &ebx, 4);
		memcpy(hyper_vendor_id + 4, &ecx, 4);
		memcpy(hyper_vendor_id + 8, &edx, 4);
		hyper_vendor_id[12] = '\0';
		if (!strcmp(hyper_vendor_id, "VMwareVMware"))
			return 1;
	} else if (dmi_available && dmi_name_in_serial("VMware") &&
		   __vmware_platform())
		return 1;

	return 0;
}
```

  But we don't run linux kernel in VMware, above routine finally return 1.
  
  If linux kernel run in VMware, after above routine executing complete, specified bit of cpu mask set as enable to represent the capacity of the cpu. After that, get cpu frequency of VMware and initlize specific value in `x86_platform`.

## _probe roms_
 
  `probe_roms` probe roms include video, PCI, system, extension and adapter.
  
  All rom resources defined in file arch/x86/kernel/probe_roms_32.c
  
##### _video rom_
  From [physical memory layout of the PC](http://files.osdev.org/mirrors/geezer/osd/ram/index.htm#layout), video rom start from 0xc0000, read video rom signature at address `0xc00c0000` which already converted to virtual address.
  
```romsignature

romsignature (rom=rom@entry=0xc00c0000 "U\252G\351\036K\201")
    at arch/x86/kernel/probe_roms_32.c:83
```

  From debug information, the half word start from `0xc00c0000` is 0xaa55 and we find the correct start address of video rom, otherwise continue iterating memory block until find the result.
  
```

(gdb) info registers eax
eax            0xc00c0000	-1072955392
(gdb) x/w 0xc00c0000
0xc00c0000:	0xe947aa55
```
  
  Find end address of video rom after checking the checksum correctly.
  
```romchecksum

(gdb) p length
$11 = 0
(gdb) p sum
$12 = 0 '\000'
```
  
  Request I/O resource for video rom with `request_resource`
  
```__request_resource

(gdb) p /x *new
$15 = {start = 0xc0000, end = 0xc8dff, name = 0xc15cfa8e, flags = 0x80002200, 
  parent = 0xc16997c0, sibling = 0x0, child = 0x0}
(gdb) p (char*)0xc15cfa8e
$16 = 0xc15cfa8e "Video ROM"
```

##### _system rom_

  The subprocedure of system rom is much simpler, just request I/O resource for system rom.
  
##### _extension rom_

  Similar with video rom.

##### adapter rom

  The number of adapter resources is much more than above resources, every adapter resource should be checked on 2k boundaries.

## insert resources to resource tree

  Insert kernel code, kernel data and kernel bss to resource tree.

## check the processor, if bad cpu detected, take extra action

  Information of the `boot_cpu_data` as follow, our cpu should not be the bad processor.
  
  If it's the bad processor, update memory range in e820 map and sanitize the map again.

```boot_cpu_data

(gdb) p boot_cpu_data.x86_vendor
$2 = 0 '\000'
(gdb) p boot_cpu_data.x86
$3 = 6 '\006'
(gdb) p boot_cpu_data.x86_model
$4 = 6 '\006'
(gdb) p boot_cpu_data.x86_mask 
$5 = 3 '\003'
```

## find the highest page frame number we have available

  Some of the debug information when involving routine e820_end_pfn:

```e820_end_pfn

(gdb) p e820.nr_map 
$6 = 6
(gdb) p e820
$7 = {nr_map = 6, map = {{addr = 0, size = 654336, type = 1}, {addr = 654336, 
      size = 1024, type = 2}, {addr = 983040, size = 65536, type = 2}, {
      addr = 1048576, size = 133160960, type = 1}, {addr = 134209536, 
      size = 8192, type = 2}, {addr = 4294705152, size = 262144, type = 2}, {
      addr = 0, size = 0, type = 0} <repeats 125 times>}}
(gdb) p type
$8 = 1
(gdb) p limit_pfn
$9 = 16777216
```

  The result of highest page frame number:

```result_of_highest_pfn

(gdb) p last_pfn
$12 = 32766
```

## preallocate 4k for mptable mpc

  Preallocate 4k for mptable mpc if take with parameter `alloc_mptable`.
  
  The [MP Configuration Table](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf). This table is optional. The table is composed of a base section and an extended section. The base section contains entries that are completely backwards compatible with previous versions of this specification. The extended section contains additional entry types. The MP configuration table contains explicit configuration information about APICs, processors, buses, and interrupts. The table consists of a header, followed by a number of entries of various types. The format and length of each entry depends on its type. When present, this configuration table must be stored either in a non-reported system RAM or within the BIOS read-only memory space.

# Links
  * [setup_data](https://lwn.net/Articles/632528/)
  * [MultiProcessor Specification](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf)
  * [Add efi e820 memory mapping on x86](http://yarchive.net/comp/linux/efi.html)
  * [Converting EFI memory Map to E820 map](http://stackoverflow.com/questions/17591351/converting-efi-memory-map-to-e820-map)
  * [Why do we pass memory map both in e820 and efi_info](http://lists.infradead.org/pipermail/kexec/2014-May/011764.html)
  * [Unified Extensible Firmware Interface](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
  * [System Management BIOS](http://wiki.osdev.org/System_Management_BIOS)
  * [Desktop Management Interface](https://en.wikipedia.org/wiki/Desktop_Management_Interface)
  * [LowMemoryCorruption](https://wiki.ubuntu.com/Kernel/Quirks/LowMemoryCorruption)
  * [How the linux kernel knows it's running in a Virtual Machine](http://perfolys.io/2016/09/06/how-the-linux-kernel-knows-its-running-in-a-virtual-machine/)
  * `Memory Map (x86): http://wiki.osdev.org/Memory_Map_(x86)` //gitbook bug
  * [Physical memory layout of the PC](http://files.osdev.org/mirrors/geezer/osd/ram/index.htm#layout)
  * [BDA - BIOS Data Area - PC Memory Map](http://stanislavs.org/helppc/bios_data_area.html)
  * [data segment](https://en.wikipedia.org/wiki/Data_segment)
  * [MP Configuration Table](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf)
  

  