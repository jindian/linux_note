# start_kernel part IV

## initialize parameters of kernel sections

  All sections defined in arch/x86/kernel/vmlinux.lds.S
  
## copy command line

  Copy command line from `boot_command_line`, `boot_command_line` is initialized before start_kernel in arch/x86/kernel/head_32.S,  the content of `boot_command_line` is:
  
```boot_command_line

p boot_command_line 
$73 = 0xc17304c0 <boot_command_line> "BOOT_IMAGE=/boot/vmlinuz-2.6.32.69 root=/dev/sda"
```