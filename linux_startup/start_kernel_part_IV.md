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
  
  Dump pic devices with `early_dump_pci_devices` if `pci_early_dump_regs` set with nonzero. `pci_early_dump_regs` defined in `arch/x86/pci/common.c` line 22, it set as 1 in routine `pcibios_setup` defined in `arch/x86/pci/common.c` line 442 if we take `pci=earlydump` in boot command line.
  
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



# Links
  * [setup_data](https://lwn.net/Articles/632528/)
  * [MultiProcessor Specification](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf)