# load linux mode

Load linux mode is part of grub_script_execute_sourcecode, besides linux module, there are several other modules loaded, so a new section created here. Let's continue read grub_script_execute_sourcecode routine.

grub_script_execute_sourcecode
    |--grub_script_parse
    |--grub_script_execute
        |--grub_script_execute_cmd
            |--grub_script_execute_cmdlist
                |--grub_script_execute_cmd
                    |--grub_script_execute_cmdline
                        |--grub_script_arglist_to_argv
                        |--grub_command_find
                        |--grub_extcmd_dispatcher
                            |--(ext->func) (&context, argc, args) -> grub_dyncmd_dispatcher
                                |--grub_dl_load
                                    |--grub_dl_get
                                    |--grub_dl_load_file
                                        |--grub_file_open
                                            |--grub_file_get_device_name
                                            |--grub_device_open
                                            |--grub_fs_probe



```grub_script_execute_sourcecode

grub-core/script/execute.c:786

/* Execute a source script.  */
grub_err_t
grub_script_execute_sourcecode (const char *source, int argc, char **args)
{
  .......

  while (source)
    {
      char *line;

      getline (&line, 0);
(gdb) p line
$3 = 0x7fe1a70 "    linux /boot/vmlinuz-2.6.32.69 root=/dev/sda"
      parsed_script = grub_script_parse (line, getline);
$4 = {refcnt = 0, mem = 0x7fb4c70, cmd = 0x7fb4c74, next_siblings = 0x0, children = 0x0}
      if (! parsed_script)
	{
	  ret = grub_errno;
	  break;
	}

      ret = grub_script_execute (parsed_script);
      grub_free (line);
    }

  scope = old_scope;
  return ret;
}
```

```grub_script_execute

grub_script_execute (script=0x7fe1a40) at script/execute.c:1080

grub-core/script/execute.c:1077

/* Execute any GRUB pre-parsed command or script.  */
grub_err_t
grub_script_execute (struct grub_script *script)
{
  if (script == 0)
    return 0;
  
  return grub_script_execute_cmd (script->cmd);
}
```



```grub_script_execute_cmd

grub_script_execute_cmd (cmd=0x7fb4c74) at script/execute.c:750

grub-core/script/execute.c:744

static grub_err_t
grub_script_execute_cmd (struct grub_script_cmd *cmd)
{   
  int ret;
  char errnobuf[ERRNO_DIGITS_MAX + 1];
  
(gdb) p *cmd
$6 = {exec = 0x7f179f5 <grub_script_execute_cmdlist>, next = 0x7fb4934}
  if (cmd == 0)
    return 0;

  ret = cmd->exec (cmd);

  grub_snprintf (errnobuf, sizeof (errnobuf), "%d", ret);
  grub_env_set ("?", errnobuf);
  return ret;
}

```


```grub_script_execute_cmdlist

grub_script_execute_cmdlist (list=0x7fb4c74) at script/execute.c:967

grub-core/script/execute.c:959

/* Execute a block of one or more commands.  */
grub_err_t
grub_script_execute_cmdlist (struct grub_script_cmd *list)
{
  int ret = 0;
  struct grub_script_cmd *cmd;

  /* Loop over every command and execute it.  */
  for (cmd = list->next; cmd; cmd = cmd->next)
    {
      if (active_breaks)
        break;

      ret = grub_script_execute_cmd (cmd);
(gdb) s
grub_script_execute_cmd (cmd=cmd@entry=0x7fb4934) at script/execute.c:750
750	  if (cmd == 0)

      if (function_return)
        break;
    }

  return ret;
}

```



```grub_script_execute_cmdline

grub_script_execute_cmdline (cmd=0x7fb4934) at script/execute.c:859

grub-core/script/execute.c:843

/* Execute a single command line.  */
grub_err_t
grub_script_execute_cmdline (struct grub_script_cmd *cmd)
{
  struct grub_script_cmdline *cmdline = (struct grub_script_cmdline *) cmd;
  grub_command_t grubcmd;
  grub_err_t ret = 0;
  grub_script_function_t func = 0;
  char errnobuf[18];
  char *cmdname;
  int argc;
  char **args;
  int invert;
  struct grub_script_argv argv = { 0, 0, 0 };

  /* Lookup the command.  */
  if (grub_script_arglist_to_argv (cmdline->arglist, &argv) || ! argv.args[0])
    return grub_errno;
(gdb) p argv
$7 = {argc = 3, args = 0x7fe1990, script = 0x0}
(gdb) p argv->args[0]
$8 = 0x7fe1950 "linux"
(gdb) p argv->args[1]
$9 = 0x7fe18f0 "/boot/vmlinuz-2.6.32.69"
(gdb) p argv->args[2]
$11 = 0x7fe18d0 "root=/dev/sda"

  invert = 0;
  argc = argv.argc - 1;
  args = argv.args + 1;
  cmdname = argv.args[0];
  if (grub_strcmp (cmdname, "!") == 0)
    {
      if (argv.argc < 2 || ! argv.args[1])
	{
	  grub_script_argv_free (&argv);
	  return grub_error (GRUB_ERR_BAD_ARGUMENT,
			     N_("no command is specified"));
	}

      invert = 1;
      argc = argv.argc - 2;
      args = argv.args + 2;
      cmdname = argv.args[1];
    }
  grubcmd = grub_command_find (cmdname);
  if (! grubcmd)
    {
      grub_errno = GRUB_ERR_NONE;

      /* It's not a GRUB command, try all functions.  */
      func = grub_script_function_find (cmdname);
      if (! func)
	{
	  /* As a last resort, try if it is an assignment.  */
	  char *assign = grub_strdup (cmdname);
	  char *eq = grub_strchr (assign, '=');

	  if (eq)
	    {
	      /* This was set because the command was not found.  */
	      grub_errno = GRUB_ERR_NONE;

	      /* Create two strings and set the variable.  */
	      *eq = '\0';
	      eq++;
	      grub_script_env_set (assign, eq);
	    }
	  grub_free (assign);

	  grub_snprintf (errnobuf, sizeof (errnobuf), "%d", grub_errno);
	  grub_script_env_set ("?", errnobuf);

	  grub_script_argv_free (&argv);
	  grub_print_error ();

	  return 0;
	}
    }

  /* Execute the GRUB command or function.  */
  if (grubcmd)
    {
      if (grub_extractor_level && !(grubcmd->flags
				    & GRUB_COMMAND_FLAG_EXTRACTOR))
	ret = grub_error (GRUB_ERR_EXTRACTOR,
			  "%s isn't allowed to execute in an extractor",
			  cmdname);
      else if ((grubcmd->flags & GRUB_COMMAND_FLAG_BLOCKS) &&
	       (grubcmd->flags & GRUB_COMMAND_FLAG_EXTCMD))
	ret = grub_extcmd_dispatcher (grubcmd, argc, args, argv.script);
      else
	ret = (grubcmd->func) (grubcmd, argc, args);
    }
  else
    ret = grub_script_function_call (func, argc, args);

  if (invert)
    {
      if (ret == GRUB_ERR_TEST_FAILURE)
	grub_errno = ret = GRUB_ERR_NONE;
      else if (ret == GRUB_ERR_NONE)
	ret = grub_error (GRUB_ERR_TEST_FAILURE, N_("false"));
      else
	{
	  grub_print_error ();
	  ret = GRUB_ERR_NONE;
	}
    }

  /* Free arguments.  */
  grub_script_argv_free (&argv);

  if (grub_errno == GRUB_ERR_TEST_FAILURE)
    grub_errno = GRUB_ERR_NONE;

  grub_print_error ();

  grub_snprintf (errnobuf, sizeof (errnobuf), "%d", ret);
  grub_env_set ("?", errnobuf);

  return ret;
}

```



```grub_extcmd_dispatcher

grub_extcmd_dispatcher (cmd=cmd@entry=0x7fe4100, argc=argc@entry=2, args=0x7fe1994, script=0x0) at commands/extcmd.c:38

grub-core/commands/extcmd.c:29

grub_err_t
grub_extcmd_dispatcher (struct grub_command *cmd, int argc, char **args,
			struct grub_script *script)
{
  int new_argc;
  char **new_args;
  struct grub_arg_list *state;
  struct grub_extcmd_context context;
  grub_err_t ret;
  grub_extcmd_t ext = cmd->data;

  context.state = 0;
  context.extcmd = ext;
  context.script = script;

(gdb) p ext->options 
$15 = (const struct grub_arg_option *) 0x0
  if (! ext->options)
    {
      ret = (ext->func) (&context, argc, args);
      return ret;
    }

  state = grub_arg_list_alloc (ext, argc, args);
  if (grub_arg_parse (ext, argc, args, state, &new_args, &new_argc))
    {
      context.state = state;
      ret = (ext->func) (&context, new_argc, new_args);
      grub_free (new_args);
      grub_free (state);
      return ret;
    }

  grub_free (state);
  return grub_errno;
}
```



```grub_dyncmd_dispatcher

grub_dyncmd_dispatcher (ctxt=0x7fd88, argc=2, args=0x7fe1994) at normal/dyncmd.c:63

grub-core/normal/dyncmd.c:56

static grub_err_t
grub_dyncmd_dispatcher (struct grub_extcmd_context *ctxt,
                        int argc, char **args)
{
  char *modname;
  grub_dl_t mod;
  grub_err_t ret;
  grub_extcmd_t extcmd = ctxt->extcmd;
  grub_command_t cmd = extcmd->cmd;
  char *name;

  modname = extcmd->data;
(gdb) p modname 
$16 = 0x7fe4160 "linux"
  mod = grub_dl_load (modname);
  if (!mod)
    return grub_errno;

  grub_free (modname);
  grub_dl_ref (mod);

  name = (char *) cmd->name;
  grub_unregister_extcmd (extcmd);

  cmd = grub_command_find (name);
  if (cmd)
    {
      if (cmd->flags & GRUB_COMMAND_FLAG_BLOCKS &&
          cmd->flags & GRUB_COMMAND_FLAG_EXTCMD)
        ret = grub_extcmd_dispatcher (cmd, argc, args, ctxt->script);
      else
        ret = (cmd->func) (cmd, argc, args);
    }
  else
    ret = grub_errno;

  grub_free (name);

  return ret;
}
```



```grub_dl_load

grub_dl_load (name=name@entry=0x7fe4160 "linux") at kern/dl.c:710

grub-core/kern/dl.c:704

/* Load a module using a symbolic name.  */
grub_dl_t
grub_dl_load (const char *name)
{
  char *filename;
  grub_dl_t mod;
  const char *grub_dl_dir = grub_env_get ("prefix");

  mod = grub_dl_get (name);
(gdb) p mod
$18 = (grub_dl_t) 0x0
  if (mod)
    return mod;

(gdb) p grub_dl_dir 
$19 = 0x7ff8030 "(hd0,msdos1)/boot/grub"
  if (! grub_dl_dir) {
    grub_error (GRUB_ERR_FILE_NOT_FOUND, N_("variable `%s' isn't set"), "prefix");
    return 0;
  }

  filename = grub_xasprintf ("%s/" GRUB_TARGET_CPU "-" GRUB_PLATFORM "/%s.mod",
                             grub_dl_dir, name);
(gdb) p filename 
$20 = 0x7fe17c0 "(hd0,msdos1)/boot/grub/i386-pc/linux.mod"
  if (! filename)
    return 0;

  mod = grub_dl_load_file (filename);
  grub_free (filename);

  if (! mod)
    return 0;

  if (grub_strcmp (mod->name, name) != 0)
    grub_error (GRUB_ERR_BAD_MODULE, "mismatched names");

  return mod;
}
```

module linux doesn't exist in grub_dl_head from result of print_all_modules

```grub_dl_get

grub_dl_get (name=name@entry=0x7fe4160 "linux") at kern/dl.c:85

grub-core/kern/dl.c:80

grub_dl_t
grub_dl_get (const char *name)
{
  grub_dl_t l;
(gdb) print_all_modules 
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

  for (l = grub_dl_head; l; l = l->next)
    if (grub_strcmp (name, l->name) == 0)
      return l;

  return 0;
}
```



```grub_dl_load_file

grub_dl_load_file (filename=0x7fe17c0 "(hd0,msdos1)/boot/grub/i386-pc/linux.mod") at kern/dl.c:671

/* Load a module from the file FILENAME.  */
grub_dl_t
grub_dl_load_file (const char *filename)
{
  grub_file_t file = NULL;
  grub_ssize_t size;
  void *core = 0;
  grub_dl_t mod = 0;

  file = grub_file_open (filename);
  if (! file)
    return 0;

  size = grub_file_size (file);
  core = grub_malloc (size);
  if (! core)
    {
      grub_file_close (file);
      return 0;
    }

  if (grub_file_read (file, core, size) != (int) size)
    {
      grub_file_close (file);
      grub_free (core);
      return 0;
    }

  /* We must close this before we try to process dependencies.
     Some disk backends do not handle gracefully multiple concurrent
     opens of the same device.  */
  grub_file_close (file);

  mod = grub_dl_load_core (core, size);
  grub_free (core);
  if (! mod)
    return 0;

  mod->ref_count--;
  return mod;
}
```



```grub_file_open

grub_file_open (name=0x7fe17c0 "(hd0,msdos1)/boot/grub/i386-pc/linux.mod") at kern/file.c:70

grub-core/kern/file.c:91

grub_file_t
grub_file_open (const char *name)
{
  grub_device_t device = 0;
  grub_file_t file = 0, last_file = 0;
  char *device_name;
  char *file_name;
  grub_file_filter_id_t filter;

  device_name = grub_file_get_device_name (name);
  if (grub_errno)
    goto fail;

  /* Get the file part of NAME.  */
  file_name = (name[0] == '(') ? grub_strchr (name, ')') : NULL;
(gdb) p file_name 
$26 = 0x7fe17cc "/boot/grub/i386-pc/linux.mod"
  if (file_name)
    file_name++;
  else
    file_name = (char *) name;

  device = grub_device_open (device_name);
  grub_free (device_name);
  if (! device)
    goto fail;

  file = (grub_file_t) grub_zalloc (sizeof (*file));
  if (! file)
    goto fail;

  file->device = device;
(gdb) p *device->disk 
$31 = {name = 0x7fe1700 "hd0", dev = 0x7ff9b18 <grub_biosdisk_dev>, 
  total_sectors = 28800, log_sector_size = 9, id = 128, 
  partition = 0x7fe1690, read_hook = 0x0, data = 0x7fe16d0}

  if (device->disk && file_name[0] != '/')
    /* This is a block list.  */
    file->fs = &grub_fs_blocklist;
  else
    {
      file->fs = grub_fs_probe (device);
      if (! file->fs)
	goto fail;
    }

  if ((file->fs->open) (file, file_name) != GRUB_ERR_NONE)
    goto fail;

  for (filter = 0; file && filter < ARRAY_SIZE (grub_file_filters_enabled);
       filter++)
    if (grub_file_filters_enabled[filter])
      {
	last_file = file;
	file = grub_file_filters_enabled[filter] (file);
      }
  if (!file)
    grub_file_close (last_file);
    
  grub_memcpy (grub_file_filters_enabled, grub_file_filters_all,
	       sizeof (grub_file_filters_enabled));

  return file;

 fail:
  if (device)
    grub_device_close (device);

  /* if (net) grub_net_close (net);  */

  grub_free (file);

  grub_memcpy (grub_file_filters_enabled, grub_file_filters_all,
	       sizeof (grub_file_filters_enabled));

  return 0;
}
```



```grub_fs_probe

grub_fs_probe (device=device@entry=0x7fe1780) at kern/fs.c:48

grub-core/kern/fs.c:35

grub_fs_t
grub_fs_probe (grub_device_t device)
{
  grub_fs_t p;
  auto int dummy_func (const char *filename,
		       const struct grub_dirhook_info *info);

  int dummy_func (const char *filename __attribute__ ((unused)),
		  const struct grub_dirhook_info *info  __attribute__ ((unused)))
    {
      return 1;
    }

  if (device->disk)
    {
      /* Make it sure not to have an infinite recursive calls.  */
      static int count = 0;

      for (p = grub_fs_list; p; p = p->next)
	{
	  grub_dprintf ("fs", "Detecting %s...\n", p->name);

	  /* This is evil: newly-created just mounted BtrFS after copying all
	     GRUB files has a very peculiar unrecoverable corruption which
	     will be fixed at sync but we'd rather not do a global sync and
	     syncing just files doesn't seem to help. Relax the check for
	     this time.  */
#ifdef GRUB_UTIL
	  if (grub_strcmp (p->name, "btrfs") == 0)
	    {
	      char *label = 0;
	      p->uuid (device, &label);
	      if (label)
		grub_free (label);
	    }
	  else
#endif
	    (p->dir) (device, "/", dummy_func);
	  if (grub_errno == GRUB_ERR_NONE)
	    return p;

	  grub_error_push ();
	  grub_dprintf ("fs", "%s detection failed.\n", p->name);
	  grub_error_pop ();

	  if (grub_errno != GRUB_ERR_BAD_FS
	      && grub_errno != GRUB_ERR_OUT_OF_RANGE)
	    return 0;

	  grub_errno = GRUB_ERR_NONE;
	}

      /* Let's load modules automatically.  */
      if (grub_fs_autoload_hook && count == 0)
	{
	  count++;

	  while (grub_fs_autoload_hook ())
	    {
	      p = grub_fs_list;

	      (p->dir) (device, "/", dummy_func);
	      if (grub_errno == GRUB_ERR_NONE)
		{
		  count--;
		  return p;
		}

	      if (grub_errno != GRUB_ERR_BAD_FS
		  && grub_errno != GRUB_ERR_OUT_OF_RANGE)
		{
		  count--;
		  return 0;
		}

	      grub_errno = GRUB_ERR_NONE;
	    }

	  count--;
	}
    }
  else if (device->net && device->net->fs)
    return device->net->fs;

  grub_error (GRUB_ERR_UNKNOWN_FS, N_("unknown filesystem"));
  return 0;
}
```

