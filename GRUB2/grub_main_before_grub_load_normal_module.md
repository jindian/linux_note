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

Get comand and arguments from configuration in grub core image, execute the command with configured arguments. We have two commands here with arguments here:

1. search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root
2. set prefix=($root)/boot/grub

```grub_load_config
static void
grub_load_config (void)
{
  struct grub_module_header *header;
  FOR_MODULES (header)
  {
    /* Not an embedded config, skip.  */
    if (header->type != OBJ_TYPE_CONFIG)
(gdb) break if header->type == 2
Breakpoint 3 at 0xcc76: file kern/main.c, line 77.
(gdb) c
Continuing.

Breakpoint 3, grub_load_config () at kern/main.c:77
77	    if (header->type != OBJ_TYPE_CONFIG)
(gdb) n
      continue;

    grub_parser_execute ((char *) header +
                         sizeof (struct grub_module_header));
(gdb) p (char *) header + sizeof (struct grub_module_header)
$2 = 0x10b760 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  \nset prefix=($root)/boot/grub\n"
(gdb) s
grub_parser_execute () at kern/parser.c:236
236	grub_parser_execute (char *source)
(gdb) n
259	  while (source)
(gdb) 
263	      getline (&line, 0);
(gdb) n
264	      grub_rescue_parse_line (line, getline);
(gdb) p line
$3 = 0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  "
(gdb) s
grub_rescue_parse_line (
    line=0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  ", getline=getline@entry=0x7ff98) at kern/rescue_parser.c:36
36	  if (grub_parser_split_cmdline (line, getline, &n, &args) || n < 0)
(gdb) s
grub_parser_split_cmdline (
    cmdline=cmdline@entry=0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  ", getline=getline@entry=0x7ff98, argc=argc@entry=0x7ff6c, 
    argv=0x7ff70) at kern/parser.c:102
102	{
(gdb) n
103	  grub_parser_state_t state = GRUB_PARSER_STATE_TEXT;
(gdb) 
107	  char *bp = buffer;
(gdb) n
108	  char *rd = (char *) cmdline;
(gdb) 
110	  char *vp = varname;
(gdb) 
146	  *argc = 0;
(gdb) n
149	      if (!rd || !*rd)
(gdb) n
157	      if (!rd)
(gdb) 
160	      for (; *rd; rd++)
(gdb) p rd
$4 = 0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  "
(gdb) n
165		  newstate = grub_parser_cmdline_state (state, *rd, &use);
(gdb) s
grub_parser_cmdline_state (state=GRUB_PARSER_STATE_TEXT, c=115 's', 
    result=result@entry=0x7fa5b "") at kern/parser.c:70
70	  for (transition = state_transitions; transition->from_state; transition++)
(gdb) p state_transitions 
$5 = {{from_state = GRUB_PARSER_STATE_TEXT, 
    to_state = GRUB_PARSER_STATE_QUOTE, input = 39 '\'', keep_value = 0}, {
    from_state = GRUB_PARSER_STATE_TEXT, to_state = GRUB_PARSER_STATE_DQUOTE, 
    input = 34 '"', keep_value = 0}, {from_state = GRUB_PARSER_STATE_TEXT, 
    to_state = GRUB_PARSER_STATE_VAR, input = 36 '$', keep_value = 0}, {
    from_state = GRUB_PARSER_STATE_TEXT, to_state = GRUB_PARSER_STATE_ESC, 
    input = 92 '\\', keep_value = 0}, {from_state = GRUB_PARSER_STATE_ESC, 
    to_state = GRUB_PARSER_STATE_TEXT, input = 0 '\000', keep_value = 1}, {
    from_state = GRUB_PARSER_STATE_QUOTE, to_state = GRUB_PARSER_STATE_TEXT, 
    input = 39 '\'', keep_value = 0}, {from_state = GRUB_PARSER_STATE_DQUOTE, 
    to_state = GRUB_PARSER_STATE_TEXT, input = 34 '"', keep_value = 0}, {
    from_state = GRUB_PARSER_STATE_DQUOTE, to_state = GRUB_PARSER_STATE_QVAR, 
    input = 36 '$', keep_value = 0}, {from_state = GRUB_PARSER_STATE_VAR, 
    to_state = GRUB_PARSER_STATE_VARNAME2, input = 123 '{', keep_value = 0}, {
    from_state = GRUB_PARSER_STATE_VAR, to_state = GRUB_PARSER_STATE_VARNAME, 
    input = 0 '\000', keep_value = 1}, {
    from_state = GRUB_PARSER_STATE_VARNAME, 
    to_state = GRUB_PARSER_STATE_TEXT, input = 32 ' ', keep_value = 1}, {
    from_state = GRUB_PARSER_STATE_VARNAME, 
    to_state = GRUB_PARSER_STATE_TEXT, input = 9 '\t', keep_value = 1}, {
    from_state = GRUB_PARSER_STATE_VARNAME2, 
    to_state = GRUB_PARSER_STATE_TEXT, input = 125 '}', keep_value = 0}, {
---Type <return> to continue, or q <return> to quit---
    from_state = GRUB_PARSER_STATE_QVAR, 
    to_state = GRUB_PARSER_STATE_QVARNAME2, input = 123 '{', keep_value = 0}, 
  {from_state = GRUB_PARSER_STATE_QVAR, 
    to_state = GRUB_PARSER_STATE_QVARNAME, input = 0 '\000', keep_value = 1}, 
  {from_state = GRUB_PARSER_STATE_QVARNAME, 
    to_state = GRUB_PARSER_STATE_TEXT, input = 34 '"', keep_value = 0}, {
    from_state = GRUB_PARSER_STATE_QVARNAME, 
    to_state = GRUB_PARSER_STATE_DQUOTE, input = 32 ' ', keep_value = 1}, {
    from_state = GRUB_PARSER_STATE_QVARNAME, 
    to_state = GRUB_PARSER_STATE_DQUOTE, input = 9 '\t', keep_value = 1}, {
    from_state = GRUB_PARSER_STATE_QVARNAME2, 
    to_state = GRUB_PARSER_STATE_DQUOTE, input = 125 '}', keep_value = 0}, {
    from_state = 0, to_state = 0, input = 0 '\000', keep_value = 0}}
(gdb) break kern/parser.c:88
Breakpoint 4 at 0xddb0: file kern/parser.c, line 88.
(gdb) c
Continuing.

Breakpoint 4, grub_parser_cmdline_state (state=GRUB_PARSER_STATE_TEXT, 
    c=115 's', result=result@entry=0x7fa5b "") at kern/parser.c:89
89	    transition = &default_transition;
(gdb) p transition->from_state 
$6 = 0
(gdb) n
92	    *result = c;
(gdb) p transition->keep_value 
$7 = 1
(gdb) n
95	  return transition->to_state;
(gdb) 
96	}
(gdb) 
grub_parser_split_cmdline (
    cmdline=cmdline@entry=0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  ", getline=getline@entry=0x7ff98, argc=argc@entry=0x7ff6c, 
    argv=0x7ff70) at kern/parser.c:170
170		  add_var (newstate);
(gdb) p newstate 
$9 = GRUB_PARSER_STATE_TEXT
(gdb) n
172		  if (check_varstate (newstate))
(gdb) n
179		      if (newstate == GRUB_PARSER_STATE_TEXT
(gdb) n
180			  && state != GRUB_PARSER_STATE_ESC && grub_isspace (use))
(gdb) 
190		      else if (use)
(gdb) 
191			*(bp++) = use;
(gdb) p bp
$10 = 0x7fb34 ""
(gdb) n
193		  state = newstate;
(gdb) p bp
$11 = 0x7fb35 ""
(gdb) p (char*)0x7fb34
$12 = 0x7fb34 "s"
-------------------------------------------------------------------------------------------------------------
......
-------------------------------------------------------------------------------------------------------------
(gdb) 
179		      if (newstate == GRUB_PARSER_STATE_TEXT
(gdb) 
180			  && state != GRUB_PARSER_STATE_ESC && grub_isspace (use))
(gdb) p use
$11 = 32 ' '
(gdb) n
184			  if (bp != buffer && *(bp - 1))
(gdb) p buffer 
$12 = "search.fs_uuid", '\000' <repeats 706 times>...
(gdb) n
186			      *(bp++) = '\0';
(gdb) 
187			      (*argc)++;
(gdb) p argc
$13 = (int *) 0x7ff6c
(gdb) p *argc
$14 = 0
(gdb) n
193		  state = newstate;
(gdb) p *argc
$15 = 1
(gdb) break kern/parser.c:200
Breakpoint 5 at 0xde29: file kern/parser.c, line 200.
(gdb) c
Continuing.

Breakpoint 5, grub_parser_split_cmdline (
    cmdline=cmdline@entry=0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  ", getline=getline@entry=0x7ff98, argc=argc@entry=0x7ff6c, 
    argv=0x7ff70) at kern/parser.c:200
200	  add_var (GRUB_PARSER_STATE_TEXT);
(gdb) p buffer 
$16 = "search.fs_uuid\000a6f72da4-5a32-4b43-9a02-d9447c833f94\000root", '\000' <repeats 664 times>...
(gdb) p *argc
$18 = 3
Breakpoint 6 at 0xdfdc: file kern/parser.c, line 232.
(gdb) c
Continuing.

Breakpoint 6, grub_parser_split_cmdline (
    cmdline=cmdline@entry=0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  ", getline=getline@entry=0x7ff98, argc=argc@entry=0x7ff6c, 
    argv=0x7ff70) at kern/parser.c:232
232	  return 0;
(gdb) p *argv[0]
$32 = 0x7ff81a0 "search.fs_uuid"
(gdb) p *argv[1]
$33 = 0x10b79b "set prefix=($root)/boot/grub\n"
(gdb) p *argv[2]
$34 = 0x10b79b "set prefix=($root)/boot/grub\n"
(gdb) p *argv[3]
$35 = 0x7ffec "\360\377\a"
(gdb) p *argc
$36 = 3
(gdb) n
233	}
(gdb) 
grub_rescue_parse_line (
    line=0x7ff81f0 "search.fs_uuid a6f72da4-5a32-4b43-9a02-d9447c833f94 root  ", getline=getline@entry=0x7ff98) at kern/rescue_parser.c:39
39	  if (n == 0)
(gdb) p n
$37 = 3
(gdb) n
44	  if (n == 1 && grub_strchr (line, '='))
(gdb) n
54	  name = args[0];
(gdb) 
57	  if (*name == '\0')
(gdb) p name
$38 = 0x7ff81a0 "search.fs_uuid"
(gdb) n
60	  cmd = grub_command_find (name);
(gdb) n
63	      (cmd->func) (cmd, n - 1, &args[1]);
(gdb) p args[1]
$41 = 0x7ff81af "a6f72da4-5a32-4b43-9a02-d9447c833f94"
(gdb) p args[2]
$42 = 0x7ff81d4 "root"
(gdb) 


    break;
  }
}
```