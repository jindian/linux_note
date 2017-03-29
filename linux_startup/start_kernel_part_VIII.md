# start kernel part VIII

## parse early command line parameter

```parse_early_param

/* Arch code calls this early on, or if not, just before other parsing. */
void __init parse_early_param(void)
{
	static __initdata int done = 0;
	static __initdata char tmp_cmdline[COMMAND_LINE_SIZE];

	if (done)
		return;

	/* All fall through to do_early_param. */
	strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
	parse_early_options(tmp_cmdline);
	done = 1;
}
```

  This function already involved in `setup_arch`, when checking value `done` we can find it was set as 1.
  
```parse_early_param_done

565		parse_early_param();
(gdb) s
parse_early_param () at init/main.c:473
473		if (done)
(gdb) p done
$13 = 1
```

## parse kernel boot arguments

```parse_args

566		parse_args("Booting kernel", static_command_line, __start___param,
(gdb) s
parse_args (name=name@entry=0xc15c5350 "Booting kernel", 
    args=0xc2126ca0 "BOOT_IMAGE=/boot/vmlinuz-2.6.32.69 root=/dev/sda", 
    params=0xc1687b18 <__param_initcall_debug>, num=168, 
    unknown=unknown@entry=0xc16fa21a <unknown_bootoption>)
    at kernel/params.c:136
136	{
```

  Parse kernel boot parameter, same with it did in `parse_early_param` when it involved in `setup_arch`

