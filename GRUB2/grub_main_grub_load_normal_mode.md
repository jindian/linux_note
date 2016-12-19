grub_main: grub_load_normal_mode
==============================================================================================================

In grub_load_normal_mode, grub initialization load normal module and execute the normal mode.

Before loading normal module, let's check loaded modules again:
```loaded_modules_before_normal
234	  grub_load_normal_mode ();
(gdb) s
grub_load_normal_mode () at kern/main.c:197
197	  grub_dl_load ("normal");
(gdb) print_all_modules 
$1 = 0x7ff8fd0 "search_fs_uuid"
$2 = (void (*)(struct grub_dl *)) 0x7ff8977
$3 = 0x7ffa0a0 "biosdisk"
$4 = (void (*)(struct grub_dl *)) 0x7ff9929
$5 = 0x7ffa920 "part_msdos"
$6 = (void (*)(struct grub_dl *)) 0x7ffa558
$7 = 0x7ffbe30 "ext2"
$8 = (void (*)(struct grub_dl *)) 0x7ffb629
$9 = 0x7ffc990 "fshelp"
$10 = (void (*)(struct grub_dl *)) 0x0
```

Call context of grub_load_normal_mode:

```grub_load_normal_mode
grub_load_normal_mode:
    |--grub_dl_load
        |--grub_env_get
        |--grub_dl_get
        |--grub_dl_load_file

```

```grub_load_normal_mode:

grub-core/kern/main.c:192

/* Load the normal mode module and execute the normal mode if possible.  */
static void
grub_load_normal_mode (void)
{
  /* Load the module.  */
  grub_dl_load ("normal");

  /* Print errors if any.  */
  grub_print_error ();
  grub_errno = 0;

  grub_command_execute ("normal", 0, 0);
}

-------------------------------------------------------------------------------------------------------------

grub-core/kern/dl.c:704

/* Load a module using a symbolic name.  */
grub_dl_t
grub_dl_load (const char *name)
{
  char *filename;
  grub_dl_t mod;
  const char *grub_dl_dir = grub_env_get ("prefix");

  mod = grub_dl_get (name);
(gdb) p grub_dl_dir
$11 = 0x7ff8030 "(hd0,msdos1)/boot/grub"

  if (mod)
gdb) p mod
$12 = (grub_dl_t) 0x0
    return mod;

  if (! grub_dl_dir) {
    grub_error (GRUB_ERR_FILE_NOT_FOUND, N_("variable `%s' isn't set"), "prefix");
    return 0;
  }

  filename = grub_xasprintf ("%s/" GRUB_TARGET_CPU "-" GRUB_PLATFORM "/%s.mod",
                             grub_dl_dir, name);
  if (! filename)
(gdb) p filename
$14 = 0x7ff7f20 "(hd0,msdos1)/boot/grub/i386-pc/normal.mod"
    return 0;

  mod = grub_dl_load_file (filename);
(gdb) p mod
$15 = (grub_dl_t) 0x7ff7ee0
  grub_free (filename);

  if (! mod)
    return 0;

  if (grub_strcmp (mod->name, name) != 0)
(gdb) p mod->name
$16 = 0x7ff7ec0 "normal"
    grub_error (GRUB_ERR_BAD_MODULE, "mismatched names");

  return mod;
}
```

After loaded normal module let's check modules again, several modules were loaded besides normal. Check these modules if you have intereting.

```loaded_modules_after_normal
(gdb) print_all_modules 
$17 = 0x7ff7ec0 "normal"
$18 = (void (*)(struct grub_dl *)) 0x7f06c68
$19 = 0x7f02bf0 "gzio"
$20 = (void (*)(struct grub_dl *)) 0x7f00199
$21 = 0x7ff30d0 "gettext"
$22 = (void (*)(struct grub_dl *)) 0x7ff2692
$23 = 0x7ff42c0 "terminal"
$24 = (void (*)(struct grub_dl *)) 0x7ff39c6
$25 = 0x7ff5c50 "crypto"
$26 = (void (*)(struct grub_dl *)) 0x0
$27 = 0x7ff70a0 "extcmd"
$28 = (void (*)(struct grub_dl *)) 0x0
$29 = 0x7ff7d50 "boot"
$30 = (void (*)(struct grub_dl *)) 0x7ff77ca
$31 = 0x7ff8fd0 "search_fs_uuid"
$32 = (void (*)(struct grub_dl *)) 0x7ff8977
$33 = 0x7ffa0a0 "biosdisk"
$34 = (void (*)(struct grub_dl *)) 0x7ff9929
$35 = 0x7ffa920 "part_msdos"
$36 = (void (*)(struct grub_dl *)) 0x7ffa558
$37 = 0x7ffbe30 "ext2"
$38 = (void (*)(struct grub_dl *)) 0x7ffb629
$39 = 0x7ffc990 "fshelp"
$40 = (void (*)(struct grub_dl *)) 0x0
```