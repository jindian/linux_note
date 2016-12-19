grub_main: grub_load_normal_mode
==============================================================================================================

In grub_load_normal_mode, grub initialization load normal module and execute it.

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
    |--grub_print_error
    |--grub_command_execute
        |--grub_command_find
            |--grub_named_list_find
        |--cmd->func: grub_cmd_normal                   //callback function of normal mode: grub_cmd_normal
            |--grub_enter_normal_mode
                |--grub_normal_execute
                    |--read_lists
                        |--read_command_list
                        |--read_fs_list
                        |--read_crypto_list
                        |--read_terminal_list
                        |--grub_gettext_reread_prefix
                            |--grub_gettext_init_ext
                    |--grub_register_variable_hook
                    |--read_config_file
                        |--grub_env_get_menu
                        |--grub_env_set_menu
                        |--grub_normal_parse_line
                    |--grub_show_menu
                        |--show_menu
                            |--run_menu                 //wait for response from end user or timeout
                            |--grub_menu_get_entry      //get selected entry after run menu
                            |--grub_menu_execute_entry  //execute selected entry
```

Result of grub_show_menu:
![result of grub_show_menu](/GRUB2/result_of_grub_show_menu.png)

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

Find normal command in grub_command_list and involve callback function grub_cmd_normal.

```grub_load_normal_mode:

-------------------------------------------------------------------------------------------------------------

include/grub/command.h:114

static inline grub_err_t
grub_command_execute (const char *name, int argc, char **argv)
{
  grub_command_t cmd;

  cmd = grub_command_find (name);
  return (cmd) ? cmd->func (cmd, argc, argv) : GRUB_ERR_FILE_NOT_FOUND;
}

-------------------------------------------------------------------------------------------------------------

include/grub/command.h:108

static inline grub_command_t
grub_command_find (const char *name)
{
  return grub_named_list_find (GRUB_AS_NAMED_LIST (grub_command_list), name);
}

-------------------------------------------------------------------------------------------------------------

grub-core/kern/list.c:24

void *
grub_named_list_find (grub_named_list_t head, const char *name)
{
  grub_named_list_t item;

  FOR_LIST_ELEMENTS (item, head)
    if (grub_strcmp (item->name, name) == 0)
      return item;

  return NULL;
}

```

In grub_cmd_normal, autoload config lists in dedicated directory, get grub configuration from specify directory in first partition, enter normal mode. In normal mode, first read lists and auto-loading, then read configuration information from grub.cfg, show grub menu and wait for response from end user.

```grub_cmd_normal
grub-core/normal/main.c:326

/* Enter normal mode from rescue mode.  */
static grub_err_t
grub_cmd_normal (struct grub_command *cmd __attribute__ ((unused)),
                 int argc, char *argv[])
{
  if (argc == 0)
    {
      /* Guess the config filename. It is necessary to make CONFIG static,
         so that it won't get broken by longjmp.  */
      char *config;
      const char *prefix;

      prefix = grub_env_get ("prefix");
(gdb) p prefix
$45 = 0x7ff8030 "(hd0,msdos1)/boot/grub"
      if (prefix)
        {
          config = grub_xasprintf ("%s/grub.cfg", prefix);
(gdb) p config
$47 = 0x7ff7f20 "(hd0,msdos1)/boot/grub/grub.cfg"
          if (! config)
            goto quit;

          grub_enter_normal_mode (config);
          grub_free (config);
        }
      else
        grub_enter_normal_mode (0);
    }
  else
    grub_enter_normal_mode (argv[0]);

quit:
  return 0;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/main.c:314

/* This starts the normal mode.  */
void
grub_enter_normal_mode (const char *config)
{
  nested_level++;
  grub_normal_execute (config, 0, 0);
  grub_cmdline_run (0);
  nested_level--;
  if (grub_normal_exit_level)
    grub_normal_exit_level--;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/main.c:280

/* Read the config file CONFIG and execute the menu interface or
   the command line interface if BATCH is false.  */
void
grub_normal_execute (const char *config, int nested, int batch)
{
  grub_menu_t menu = 0;
  const char *prefix;

  if (! nested)
(gdb) p nested
$48 = 0
    {
      prefix = grub_env_get ("prefix");
      read_lists (prefix);
(gdb) p prefix
$49 = 0x7ff8030 "(hd0,msdos1)/boot/grub"
      grub_register_variable_hook ("prefix", NULL, read_lists_hook);
    }

  if (config)
    {
      menu = read_config_file (config);

      /* Ignore any error.  */
      grub_errno = GRUB_ERR_NONE;
    }

  if (! batch)
    {
      if (menu && menu->size)
(gdb) p menu
$4 = (grub_menu_t) 0x7ff19f0
(gdb) p menu->size 
$5 = 1
        {
          grub_show_menu (menu, nested, 0);
          if (nested)
            grub_normal_free_menu (menu);
        }
    }
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/dyncmd.c:95

/* Read the file command.lst for auto-loading.  */
void
read_command_list (const char *prefix)
{
  if (prefix)
    {
      char *filename;

      filename = grub_xasprintf ("%s/" GRUB_TARGET_CPU "-" GRUB_PLATFORM
                                 "/command.lst", prefix);
(gdb) p filename 
$60 = 0x7ff1c60 "(hd0,msdos1)/boot/grub/i386-pc/command.lst"
      if (filename)
        {
          grub_file_t file;

          file = grub_file_open (filename);
          if (file)
            {
              char *buf = NULL;
              grub_command_t ptr, last = 0, next;

              /* Override previous commands.lst.  */
              for (ptr = grub_command_list; ptr; ptr = next)
                {
                  next = ptr->next;
                  if (ptr->flags & GRUB_COMMAND_FLAG_DYNCMD)
                    {
                      if (last)
                        last->next = ptr->next;
                      else
                        grub_command_list = ptr->next;
                      grub_free (ptr->data); /* extcmd struct */
                      grub_free (ptr);
                    }
                  else
                    last = ptr;
                }

              for (;; grub_free (buf))
                {
                  char *p, *name, *modname;
                  grub_extcmd_t cmd;
                  int prio = 0;

                  buf = grub_file_getline (file);

                  if (! buf)
                    break;

                  name = buf;
                  while (grub_isspace (name[0]))
                    name++;

                  if (*name == '*')
                    {
                      name++;
                      prio++;
                    }

                  if (! grub_isgraph (name[0]))
                    continue;

                  p = grub_strchr (name, ':');
                  if (! p)
                    continue;

                  *p = '\0';
                  p++;
                  while (*p == ' ' || *p == '\t')
                    p++;

                  if (! grub_isgraph (*p))
                    continue;

                  if (grub_dl_get (p))
                    continue;

                  name = grub_strdup (name);
                  if (! name)
                    continue;

                  modname = grub_strdup (p);
                  if (! modname)
                    {
                      grub_free (name);
                      continue;
                    }

                  cmd = grub_register_extcmd_prio (name,
                                                   grub_dyncmd_dispatcher,
                                                   GRUB_COMMAND_FLAG_BLOCKS
                                                   | GRUB_COMMAND_FLAG_EXTCMD
                                                   | GRUB_COMMAND_FLAG_DYNCMD,
                                                   0, N_("module isn't loaded"),
                                                   0, prio);
                  if (! cmd)
                    {
                      grub_free (name);
                      grub_free (modname);
                      continue;
                    }
                  cmd->data = modname;

                  /* Update the active flag.  */
                  grub_command_find (name);
                }

              grub_file_close (file);
            }

          grub_free (filename);
        }
    }

  /* Ignore errors.  */
  grub_errno = GRUB_ERR_NONE;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/autofs.c:52

/* Read the file fs.lst for auto-loading.  */
void
read_fs_list (const char *prefix)
{
  if (prefix)
    {
      char *filename;

      filename = grub_xasprintf ("%s/" GRUB_TARGET_CPU "-" GRUB_PLATFORM
                                 "/fs.lst", prefix);
      if (filename)
(gdb) p filename 
$61 = 0x7ff1c60 "(hd0,msdos1)/boot/grub/i386-pc/fs.lst"
        {
          grub_file_t file;
          grub_fs_autoload_hook_t tmp_autoload_hook;

          /* This rules out the possibility that read_fs_list() is invoked
             recursively when we call grub_file_open() below.  */
          tmp_autoload_hook = grub_fs_autoload_hook;
          grub_fs_autoload_hook = NULL;

          file = grub_file_open (filename);
          if (file)
            {
              /* Override previous fs.lst.  */
              while (fs_module_list)
                {
                  grub_named_list_t tmp;
                  tmp = fs_module_list->next;
                  grub_free (fs_module_list);
                  fs_module_list = tmp;
                }

              while (1)
                {
                  char *buf;
                  char *p;
                  char *q;
                  grub_named_list_t fs_mod;

                  buf = grub_file_getline (file);
                  if (! buf)
                    break;

                  p = buf;
                  q = buf + grub_strlen (buf) - 1;

                  /* Ignore space.  */
                  while (grub_isspace (*p))
                    p++;

                  while (p < q && grub_isspace (*q))
                    *q-- = '\0';

                  /* If the line is empty, skip it.  */
                  if (p >= q)
                    {
                      grub_free (buf);
                      continue;
                    }

                  fs_mod = grub_malloc (sizeof (*fs_mod));
                  if (! fs_mod)
                    {
                      grub_free (buf);
                      continue;
                    }

                  fs_mod->name = grub_strdup (p);
                  grub_free (buf);
                  if (! fs_mod->name)
                    {
                      grub_free (fs_mod);
                      continue;
                    }

                  fs_mod->next = fs_module_list;
                  fs_module_list = fs_mod;
                }

              grub_file_close (file);
              grub_fs_autoload_hook = tmp_autoload_hook;
            }

          grub_free (filename);
        }
    }

  /* Ignore errors.  */
  grub_errno = GRUB_ERR_NONE;

  /* Set the hook.  */
  grub_fs_autoload_hook = autoload_fs_module;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/crypto.c:75

/* Read the file crypto.lst for auto-loading.  */
void
read_crypto_list (const char *prefix)
{
  char *filename;
  grub_file_t file;
  char *buf = NULL;

  if (!prefix)
    {
      grub_errno = GRUB_ERR_NONE;
      return;
    }

  filename = grub_xasprintf ("%s/" GRUB_TARGET_CPU "-" GRUB_PLATFORM
                             "/crypto.lst", prefix);
  if (!filename)
(gdb) p filename 
$62 = 0x7ff1c60 "(hd0,msdos1)/boot/grub/i386-pc/crypto.lst"
    {
      grub_errno = GRUB_ERR_NONE;
      return;
    }

  file = grub_file_open (filename);
  grub_free (filename);
  if (!file)
    {
      grub_errno = GRUB_ERR_NONE;
      return;
    }

  /* Override previous crypto.lst.  */
  grub_crypto_spec_free ();

  for (;; grub_free (buf))
    {
      char *p, *name;
      struct load_spec *cur;

      buf = grub_file_getline (file);

      if (! buf)
        break;

      name = buf;
      while (grub_isspace (name[0]))
        name++;

      p = grub_strchr (name, ':');
      if (! p)
        continue;

      *p = '\0';
      p++;
      while (*p == ' ' || *p == '\t')
        p++;

      cur = grub_malloc (sizeof (*cur));
      if (!cur)
        {
          grub_errno = GRUB_ERR_NONE;
          continue;
        }

      cur->name = grub_strdup (name);
      if (! name)
        {
          grub_errno = GRUB_ERR_NONE;
          grub_free (cur);
          continue;
        }

      if (! cur->modname)
        {
          grub_errno = GRUB_ERR_NONE;
          grub_free (cur);
          grub_free (cur->name);
          continue;
        }
      cur->next = crypto_specs;
      crypto_specs = cur;
    }

  grub_file_close (file);

  grub_errno = GRUB_ERR_NONE;

  grub_crypto_autoload_hook = grub_crypto_autoload;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/term.c

/* Read the file terminal.lst for auto-loading.  */
void
read_terminal_list (const char *prefix)
{
  char *filename;
  grub_file_t file;
  char *buf = NULL;

  if (!prefix)
    {
      grub_errno = GRUB_ERR_NONE;
      return;
    }

  filename = grub_xasprintf ("%s/" GRUB_TARGET_CPU "-" GRUB_PLATFORM
                             "/terminal.lst", prefix);
(gdb) p filename
$63 = 0x7ff1ab0 "(hd0,msdos1)/boot/grub/i386-pc/terminal.lst"
  if (!filename)
    {
      grub_errno = GRUB_ERR_NONE;
      return;
    }

  file = grub_file_open (filename);
  grub_free (filename);
  if (!file)
    {
      grub_errno = GRUB_ERR_NONE;
      return;
    }

  /* Override previous terminal.lst.  */
  grub_terminal_autoload_free ();

  for (;; grub_free (buf))
    {
      char *p, *name;
      struct grub_term_autoload *cur;
      struct grub_term_autoload **target = NULL;

      buf = grub_file_getline (file);

      if (! buf)
        break;

      p = buf;
      while (grub_isspace (p[0]))
        p++;

      switch (p[0])
        {
        case 'i':
          target = &grub_term_input_autoload;
          break;

        case 'o':
          target = &grub_term_output_autoload;
          break;
        }
      if (!target)
        continue;

      name = p + 1;

      p = grub_strchr (name, ':');
      if (! p)
        continue;
      *p = 0;

      p++;
      while (*p == ' ' || *p == '\t')
        p++;

      cur = grub_malloc (sizeof (*cur));
      if (!cur)
        {
          grub_errno = GRUB_ERR_NONE;
          continue;
        }

      cur->name = grub_strdup (name);
      if (! name)
        {
          grub_errno = GRUB_ERR_NONE;
          grub_free (cur);
          continue;
        }

      cur->modname = grub_strdup (p);
      if (! cur->modname)
        {
          grub_errno = GRUB_ERR_NONE;
          grub_free (cur->name);
          grub_free (cur);
          continue;
        }
      cur->next = *target;
      *target = cur;
    }

  grub_file_close (file);

  grub_errno = GRUB_ERR_NONE;
}

-------------------------------------------------------------------------------------------------------------

grub-core/gettext/gettext.c:436

void
grub_gettext_reread_prefix (const char *val)
{
  grub_err_t err;
  err = grub_gettext_init_ext (&main_context, grub_env_get ("lang"),
                               grub_env_get ("locale_dir"),
                               val);
  if (err)
    grub_print_error ();
}

-------------------------------------------------------------------------------------------------------------

grub-core/gettext/gettext.c:374

static grub_err_t
grub_gettext_init_ext (struct grub_gettext_context *ctx,
                       const char *locale,
                       const char *locale_dir, const char *prefix)
Breakpoint 4, grub_gettext_init_ext (ctx=0x7ff28d0 <main_context>, 
    locale=0x7ff1d70 "", locale_dir=0x7ff1ea0 "", 
    prefix=0x7ff8030 "(hd0,msdos1)/boot/grub") at gettext/gettext.c:378
{
  const char *part1, *part2;
  grub_err_t err;

  grub_gettext_delete_list (ctx);

  if (!locale || locale[0] == 0)
    return 0;

  part1 = locale_dir;
  part2 = "";
  if (!part1 || part1[0] == 0)
    {
      part1 = prefix;
      part2 = "/locale";
    }

  if (!part1 || part1[0] == 0)
    return 0;

  err = grub_mofile_open_lang (ctx, part1, part2, locale);

  /* ll_CC didn't work, so try ll.  */
  if (err)
    {
      char *lang = grub_strdup (locale);
      char *underscore = lang ? grub_strchr (lang, '_') : 0;

      if (underscore)
        {
          *underscore = '\0';
          grub_errno = GRUB_ERR_NONE;
          err = grub_mofile_open_lang (ctx, part1, part2, lang);
        }

      grub_free (lang);
    }
  return err;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/menu.c:748

grub_err_t
grub_show_menu (grub_menu_t menu, int nested, int autoboot)
{
  grub_err_t err1, err2;

  while (1)
    {
      err1 = show_menu (menu, nested, autoboot);
      autoboot = 0;
      grub_print_error ();

      if (grub_normal_exit_level)
        break;

      err2 = grub_auth_check_authentication (NULL);
      if (err2)
        {
          grub_print_error ();
          grub_errno = GRUB_ERR_NONE;
          continue;
        }

      break;
    }

  return err1;
}

-------------------------------------------------------------------------------------------------------------

grub-core/normal/menu.c:717

static grub_err_t
show_menu (grub_menu_t menu, int nested, int autobooted)
{
  while (1)
    {
      int boot_entry;
      grub_menu_entry_t e;
      int auto_boot;

      boot_entry = run_menu (menu, nested, &auto_boot);
      if (boot_entry < 0)
        break;

      e = grub_menu_get_entry (menu, boot_entry);
      if (! e)
        continue; /* Menu is empty.  */

      grub_cls ();

      if (auto_boot)
        grub_menu_execute_with_fallback (menu, e, autobooted,
                                         &execution_callback, 0);
      else
        grub_menu_execute_entry (e, 0);
      if (autobooted)
        break;
    }

  return GRUB_ERR_NONE;
}
```