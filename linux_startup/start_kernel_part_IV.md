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

## Check if ACPI is disabled and MPS code is not built-in
  
  `acpi_mps_check` return directly.



# Links
  * [setup_data](https://lwn.net/Articles/632528/)