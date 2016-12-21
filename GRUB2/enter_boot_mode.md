Enter Boot Mode
==============================================================================================================

The entry of boot mode is grub_cmd_boot, after end user selected an operating system, grub executes the menu entry and finally involves grub_command_execute with module name 'boot' as input parameter. grub_cmd_boot function as follow, it's very simple just involves grub_loader_boot.

```grub_cmd_boot
/* boot */
static grub_err_t
grub_cmd_boot (struct grub_command *cmd __attribute__ ((unused)),
                    int argc __attribute__ ((unused)),
                    char *argv[] __attribute__ ((unused)))
{
  return grub_loader_boot ();
}
```

The context of boot mode in initialization:

```boot_mode_context
grub_cmd_boot
    |--grub_loader_boot
        |--(grub_loader_boot_func) () -> grub_linux_boot     //assigned with grub_linux_boot when linux mode initialization
```

```grub_loader_boot
grub-core/commands/boot.c:139

grub_err_t
grub_loader_boot (void)
{
  grub_err_t err = GRUB_ERR_NONE;
  struct grub_preboot *cur;

  if (! grub_loader_loaded)
(gdb) p grub_loader_loaded 
$69 = 1
    return grub_error (GRUB_ERR_NO_KERNEL,
                       N_("you need to load the kernel first"));

  if (grub_loader_flags & GRUB_LOADER_FLAG_NORETURN)
(gdb) p grub_loader_flags 
$70 = 0
    grub_machine_fini ();

  for (cur = preboots_head; cur; cur = cur->next)
(gdb) p preboots_head 
$71 = (struct grub_preboot *) 0x0
    {
      err = cur->preboot_func (grub_loader_flags);
      if (err)
        {
          for (cur = cur->prev; cur; cur = cur->prev)
            cur->preboot_rest_func ();
          return err;
        }
    }
  err = (grub_loader_boot_func) ();

  for (cur = preboots_tail; cur; cur = cur->prev)
    if (! err)
      err = cur->preboot_rest_func ();
    else
      cur->preboot_rest_func ();

  return err;
}
```

Next in grub_linux_boot before involving grub_relocator32_boot, what preparations are done before grub transfers control to linux source code?

