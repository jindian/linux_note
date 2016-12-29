# load linux mode.

In previous section, one routine in grub_menu_execute_entry was not introduced, it's grub_script_execute_sourcecode. 

Input parameters of grub_script_execute_sourcecode

```input_parameters_of_grub_script_execute_sourcecode
Breakpoint 6, grub_script_execute_sourcecode (argc=1, args=0x7fefef0)
    at script/execute.c:789

(gdb) p args
$1 = (char **) 0x7fefef0
(gdb) p args[0]
$2 = 0x7fefed0 "linux OS"
```

All loaded modules at this point

```print_all_modules
normal: normal->init: 0x7f0ec78
gzio: gzio->init: 0x7f081a9
gettext: gettext->init: 0x7ff2692
terminal: terminal->init: 0x7ff39c6
crypto: crypto->init: 0x0
extcmd: extcmd->init: 0x0
boot: boot->init: 0x7ff77ca
search_fs_uuid: search_fs_uuid->init: 0x7ff8977
biosdisk: biosdisk->init: 0x7ff9929
part_msdos: part_msdos->init: 0x7ffa558
ext2: ext2->init: 0x7ffb629
fshelp: fshelp->init: 0x0
```