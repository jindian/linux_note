grub_main: set environment, register command, load configuration
==============================================================================================================

Before get to grub_load_normal_module, there are several routines, let's check these routines one by one.

```before_grub_load_normal_module
  grub_set_prefix_and_root ();
  grub_env_export ("root");
  grub_env_export ("prefix");

  grub_register_core_commands ();

  grub_load_config ();
```

Get values from grub modules combined with grub core image, set root and prefix environment variables.

1. Get prefix from grub_modebase.
2. Set write hook for root environment variable with grub_register_variable_hook, the hook will be involved when update root environment variable, inside grub_register_variable_hook, first get environment variable from hash table, if not exists, allocate memory storing environment variable and insert it to hash table: grub_current_context. Every enviroment variable has a hash key which calculated with its name, with the key to find its information from hash table.
3. Get boot location with grub_machine_get_bootlocation.
4. Set prefix and root environment variables.

Call context of grub_set_prefix_and_root:
```grub_set_prefix_and_root
grub_set_prefix_and_root
    |--grub_register_variable_hook
    |--grub_machine_get_bootlocation
    |--grub_env_set
```

```grub_set_prefix_and_root
grub-core/kern/main.c:101

static void
grub_set_prefix_and_root (void)
{
  char *device = NULL;
  char *path = NULL;
  char *fwdevice = NULL;
  char *fwpath = NULL;
  char *prefix = NULL;
  struct grub_module_header *header;

  FOR_MODULES (header)
    if (header->type == OBJ_TYPE_PREFIX)
      prefix = (char *) header + sizeof (struct grub_module_header);

  grub_register_variable_hook ("root", 0, grub_env_write_root);

  if (prefix)
(gdb) p prefix
$1 = 0x10b7c4 "/boot/grub"
    {
      char *pptr = NULL;
      if (prefix[0] == '(')
        {
          pptr = grub_strrchr (prefix, ')');
          if (pptr)
            {
              device = grub_strndup (prefix + 1, pptr - prefix - 1);
              pptr++;
            }
        }
      if (!pptr)
        pptr = prefix;
      if (pptr[0])
        path = grub_strdup (pptr);
(gdb) p path
$2 = 0x7ff8320 "/boot/grub"
    }
  if ((!device || device[0] == ',' || !device[0]) || !path)
    grub_machine_get_bootlocation (&fwdevice, &fwpath);
(gdb) p fwdevice 
$3 = 0x7ff82a0 "hd0"
(gdb) p fwpath 
$4 = 0x0

  if (!device && fwdevice)
    device = fwdevice;
  else if (fwdevice && (device[0] == ',' || !device[0]))
    {
      /* We have a partition, but still need to fill in the drive.  */
      char *comma, *new_device;

      for (comma = fwdevice; *comma; )
        {
          if (comma[0] == '\\' && comma[1] == ',')
            {
              comma += 2;
              continue;
            }
          if (*comma == ',')
            break;
          comma++;
        }
      if (*comma)
        {
          char *drive = grub_strndup (fwdevice, comma - fwdevice);
          new_device = grub_xasprintf ("%s%s", drive, device);
          grub_free (drive);
        }
      else
        new_device = grub_xasprintf ("%s%s", fwdevice, device);

      grub_free (fwdevice);
      grub_free (device);
      device = new_device;
    }
  else
    grub_free (fwdevice);
  if (fwpath && !path)
    path = fwpath;
  else
    grub_free (fwpath);
  if (device)
    {
      char *prefix_set;

      prefix_set = grub_xasprintf ("(%s)%s", device, path ? : "");
      if (prefix_set)
(gdb) p prefix_set
$5 = 0x7ff8190 "(hd0)/boot/grub"
        {
          grub_env_set ("prefix", prefix_set);
          grub_free (prefix_set);
        }
      grub_env_set ("root", device);
(gdb) p device
$6 = 0x7ff82a0 "hd0"
    }

  grub_free (device);
  grub_free (path);
  grub_print_error ();
}


```

Export environment variable root and prefix with grub_env_export in which we can find global of a specific variable set as 1.

```export_environment_variable
grub-core/kern/env.c:241

grub_err_t
grub_env_export (const char *name)
{
  struct grub_env_var *var;

  var = grub_env_find (name);
(gdb) p name
$1 = 0xe9b8 "root"
  if (! var)
    {
      grub_err_t err;

      err = grub_env_set (name, "");
      if (err)
        return err;
      var = grub_env_find (name);
    }
  var->global = 1;

  return GRUB_ERR_NONE;
}

```

Register command set, unset, ls, insmod command to grub commmand, every command has a dedicated response function to process specific operation from console. All grub commands stored in grub_command_list, allocate memory for new command and insert it to the list.

Call context of grub_register_core_command:
```grub_register_core_commands
grub_register_core_commands
    |--grub_register_command
        |--grub_register_command_prio
```

```grub_register_core_commands

grub-core/kern/corecmd.c:178

void
grub_register_core_commands (void)
{
  grub_command_t cmd;
  cmd = grub_register_command ("set", grub_core_cmd_set,
                               N_("[ENVVAR=VALUE]"),
                               N_("Set an environment variable."));
  if (cmd)
    cmd->flags |= GRUB_COMMAND_FLAG_EXTRACTOR;
  grub_register_command ("unset", grub_core_cmd_unset,
                         N_("ENVVAR"),
                         N_("Remove an environment variable."));
  grub_register_command ("ls", grub_core_cmd_ls,
                         N_("[ARG]"), N_("List devices or files."));
  grub_register_command ("insmod", grub_core_cmd_insmod,
                         N_("MODULE"), N_("Insert a module."));
}

-------------------------------------------------------------------------------------------------------------

include/grub/command.h:90

static inline grub_command_t
grub_register_command (const char *name,
                       grub_command_func_t func,
                       const char *summary,
                       const char *description)
{
  return grub_register_command_prio (name, func, summary, description, 0);
}

-------------------------------------------------------------------------------------------------------------

grub-core/kern/command.c

grub_command_t
grub_register_command_prio (const char *name,
                            grub_command_func_t func,
                            const char *summary,
                            const char *description,
                            int prio)
{
  grub_command_t cmd;
  int inactive = 0;

  grub_command_t *p, q;

  cmd = (grub_command_t) grub_zalloc (sizeof (*cmd));
  if (! cmd)
    return 0;

  cmd->name = name;
  cmd->func = func;
  cmd->summary = (summary) ? summary : "";
  cmd->description = description;

  cmd->flags = 0;
  cmd->prio = prio;

  for (p = &grub_command_list, q = *p; q; p = &(q->next), q = q->next)
    {
      int r;

      r = grub_strcmp (cmd->name, q->name);
      if (r < 0)
        break;
      if (r > 0)
        continue;

      if (cmd->prio >= (q->prio & GRUB_COMMAND_PRIO_MASK))
        {
          q->prio &= ~GRUB_COMMAND_FLAG_ACTIVE;
          break;
        }

      inactive = 1;
    }

  *p = cmd;
  cmd->next = q;
  if (q)
    q->prev = &cmd->next;
  cmd->prev = p;

  if (! inactive)
    cmd->prio |= GRUB_COMMAND_FLAG_ACTIVE;

  return cmd;
}
```

Get comand and arguments from configuration in grub core image, execute the command with configured arguments. We have two commands:

1. search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root
2. set prefix=($root)/boot/grub

Call context of grub_load_config
```grub_load_config
grub_load_config
    |--grub_parser_execute
        |--getline
        |--grub_rescue_parse_line
            |--grub_parser_split_cmdline
            |--grub_command_find
            |--cmd->func                            //involve command function if exists
```

```grub_load_config
grub-core/kern/main.c:70

static void
grub_load_config (void)
{
  struct grub_module_header *header;
  FOR_MODULES (header)
  {
    /* Not an embedded config, skip.  */
    if (header->type != OBJ_TYPE_CONFIG)
      continue;

    grub_parser_execute ((char *) header +
                         sizeof (struct grub_module_header));
    break;
  }
}

-------------------------------------------------------------------------------------------------------------

grub-core/kern/parser.c:235

grub_err_t
grub_parser_execute (char *source)
{
  auto grub_err_t getline (char **line, int cont);
  grub_err_t getline (char **line, int cont __attribute__ ((unused)))
  {
    char *p;

    if (!source)
      {
        *line = 0;
        return 0;
      }

    p = grub_strchr (source, '\n');

    if (p)
      *line = grub_strndup (source, p - source);
    else
      *line = grub_strdup (source);
    source = p ? p + 1 : 0;
    return 0;
  }

  while (source)
(gdb) p source 
$1 = 0x10b760 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  \nset prefix=($root)/boot/grub\n"
    {
      char *line;

      getline (&line, 0);
      grub_rescue_parse_line (line, getline);
      grub_free (line);
    }

  return grub_errno;
}

-------------------------------------------------------------------------------------------------------------

grub-core/kern/rescue_parser.c:28

grub_err_t
grub_rescue_parse_line (char *line, grub_reader_getline_t getline)
{
  char *name;
  int n;
  grub_command_t cmd;
  char **args;

  if (grub_parser_split_cmdline (line, getline, &n, &args) || n < 0)
    return grub_errno;

  if (n == 0)
    return GRUB_ERR_NONE;

  /* In case of an assignment set the environment accordingly
     instead of calling a function.  */
  if (n == 1 && grub_strchr (line, '='))
    {
      char *val = grub_strchr (args[0], '=');
      val[0] = 0;
      grub_env_set (args[0], val + 1);
      val[0] = '=';
      goto quit;
    }

  /* Get the command name.  */
  name = args[0];

  /* If nothing is specified, restart.  */
  if (*name == '\0')
    goto quit;

  cmd = grub_command_find (name);
  if (cmd)
    {
      (cmd->func) (cmd, n - 1, &args[1]);
    }
  else
    {
      grub_printf_ (N_("Unknown command `%s'.\n"), name);
      if (grub_command_find ("help"))
        grub_printf ("Try `help' for usage\n");
    }

 quit:
  grub_free (args[0]);
  grub_free (args);

  return grub_errno;
}

-------------------------------------------------------------------------------------------------------------

grub-core/kern/parser.c:99

grub_err_t
grub_parser_split_cmdline (const char *cmdline, grub_reader_getline_t getline,
                           int *argc, char ***argv)
{
  grub_parser_state_t state = GRUB_PARSER_STATE_TEXT;
  /* XXX: Fixed size buffer, perhaps this buffer should be dynamically
     allocated.  */
  char buffer[1024];
  char *bp = buffer;
  char *rd = (char *) cmdline;
  char varname[200];
  char *vp = varname;
  char *args;
  int i;

  auto int check_varstate (grub_parser_state_t s);

  int check_varstate (grub_parser_state_t s)
  {
    return (s == GRUB_PARSER_STATE_VARNAME
            || s == GRUB_PARSER_STATE_VARNAME2
            || s == GRUB_PARSER_STATE_QVARNAME
            || s == GRUB_PARSER_STATE_QVARNAME2);
  }

  auto void add_var (grub_parser_state_t newstate);

  void add_var (grub_parser_state_t newstate)
  {
    const char *val;

    /* Check if a variable was being read in and the end of the name
       was reached.  */
    if (!(check_varstate (state) && !check_varstate (newstate)))
      return;

    *(vp++) = '\0';
    val = grub_env_get (varname);
    vp = varname;
    if (!val)
      return;

    /* Insert the contents of the variable in the buffer.  */
    for (; *val; val++)
      *(bp++) = *val;
  }

  *argc = 0;
  do
    {
      if (!rd || !*rd)
        {
          if (getline)
            getline (&rd, 1);
          else
            break;
        }

      if (!rd)
        break;

      for (; *rd; rd++)
        {
          grub_parser_state_t newstate;
          char use;

          newstate = grub_parser_cmdline_state (state, *rd, &use);

          /* If a variable was being processed and this character does
             not describe the variable anymore, write the variable to
             the buffer.  */
          add_var (newstate);

          if (check_varstate (newstate))
            {
              if (use)
                *(vp++) = use;
            }
          else
            {
              if (newstate == GRUB_PARSER_STATE_TEXT
                  && state != GRUB_PARSER_STATE_ESC && grub_isspace (use))
                {
                  /* Don't add more than one argument if multiple
                     spaces are used.  */
                  if (bp != buffer && *(bp - 1))
                    {
                      *(bp++) = '\0';
                      (*argc)++;
                    }
                }
              else if (use)
                *(bp++) = use;
            }
          state = newstate;
        }
    }
  while (state != GRUB_PARSER_STATE_TEXT && !check_varstate (state));

  /* A special case for when the last character was part of a
     variable.  */
  add_var (GRUB_PARSER_STATE_TEXT);

  if (bp != buffer && *(bp - 1))
    {
      *(bp++) = '\0';
      (*argc)++;
    }

  /* Reserve memory for the return values.  */
  args = grub_malloc (bp - buffer);
  if (!args)
    return grub_errno;
  grub_memcpy (args, buffer, bp - buffer);

  *argv = grub_malloc (sizeof (char *) * (*argc + 1));
  if (!*argv)
    {
      grub_free (args);
      return grub_errno;
    }

  /* The arguments are separated with 0's, setup argv so it points to
     the right values.  */
  bp = args;
  for (i = 0; i < *argc; i++)
    {
      (*argv)[i] = bp;
      while (*bp)
        bp++;
      bp++;
    }

  return 0;
}

```