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

1. Get values from grub modules.
2. Set write hook for root environment variable with grub_register_variable_hook, the hook will be involved when update root environment variable, inside grub_register_variable_hook, first get environment variable from hash table, if not exists, allocate memory storing environment variable and insert it to hash table: grub_current_context. Every enviroment variable has a hash key which calculated with its name, with the key to find its information from hash table.
3. Get boot location with grub_machine_get_bootlocation
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
(gdb) p header->type 
$8 = 3
(gdb) p header
$9 = (struct grub_module_header *) 0x10b7bc

  grub_register_variable_hook ("root", 0, grub_env_write_root);
(gdb) s
grub_register_variable_hook (name=name@entry=0xe9b8 "root", 
    read_hook=read_hook@entry=0x0, 
    write_hook=write_hook@entry=0xc9c9 <grub_env_write_root>)
    at kern/env.c:224
224	  struct grub_env_var *var = grub_env_find (name);
(gdb) n
226	  if (! var)
(gdb) p var
$11 = (struct grub_env_var *) 0x0
(gdb) n
228	      if (grub_env_set (name, "") != GRUB_ERR_NONE)
(gdb) s
grub_env_set (name=0xe9b8 "root", val=0xee54 "") at kern/env.c:87
87	  var = grub_env_find (name);
(gdb) s
grub_env_find (name=name@entry=0xe9b8 "root") at kern/env.c:46
46	{
(gdb) n
48	  int idx = grub_env_hashval (name);
(gdb) p name
$12 = 0xe9b8 "root"
(gdb) n
51	  for (var = grub_current_context->vars[idx]; var; var = var->next)
(gdb) p idx
$13 = 11
(gdb) n
55	  return 0;
(gdb) p var
$14 = (struct grub_env_var *) 0x0
(gdb) n
56	}
(gdb) 
grub_env_set (name=0xe9b8 "root", val=0xee54 "") at kern/env.c:88
88	  if (var)
(gdb) n
108	  var = grub_zalloc (sizeof (*var));
(gdb) 
109	  if (! var)
(gdb) 
112	  var->name = grub_strdup (name);
(gdb) 
113	  if (! var->name)
(gdb) 
116	  var->value = grub_strdup (val);
(gdb) 
117	  if (! var->value)
(gdb) 
120	  grub_env_insert (grub_current_context, var);
(gdb) n
122	  return GRUB_ERR_NONE;
(gdb) 
130	}
(gdb) n
grub_register_variable_hook (name=name@entry=0xe9b8 "root", 
    read_hook=read_hook@entry=0x0, 
    write_hook=write_hook@entry=0xc9c9 <grub_env_write_root>)
    at kern/env.c:231
231	      var = grub_env_find (name);
(gdb) n
235	  var->read_hook = read_hook;
(gdb) p var
$15 = (struct grub_env_var *) 0x7ff8380
(gdb) n
236	  var->write_hook = write_hook;
(gdb) 
238	  return GRUB_ERR_NONE;
(gdb) 
239	}
grub_set_prefix_and_root () at kern/main.c:117
117	  if (prefix)

  if (prefix)
(gdb) p prefix 
$16 = 0x10b7c4 "/boot/grub"
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
(gdb) p pptr 
$17 = 0x0
        pptr = prefix;
      if (pptr[0])
        path = grub_strdup (pptr);
    }
(gdb) p path
$18 = 0x7ff8320 "/boot/grub"
(gdb) p device
$19 = 0x0
  if ((!device || device[0] == ',' || !device[0]) || !path)
    grub_machine_get_bootlocation (&fwdevice, &fwpath);
(gdb) s
grub_machine_get_bootlocation (device=device@entry=0x7ffcc, 
    path=path@entry=0x7ffd0) at kern/i386/pc/init.c:73
73	  boot_drive = (grub_boot_device >> 24);
(gdb) p /x grub_boot_device 
$20 = 0x80ffffff
(gdb) n
74	  dos_part = (grub_boot_device >> 16);
(gdb) p /x dos_part 
$21 = 0xff
(gdb) n
75	  bsd_part = (grub_boot_device >> 8);
(gdb) n
79	  if (boot_drive == GRUB_BOOT_MACHINE_PXE_DL)
(gdb) 
88	  *device = grub_malloc (DEV_SIZE);
(gdb) p *device
$22 = 0x7ff82a0 ""
90	  grub_snprintf (*device, DEV_SIZE,
(gdb) p *device
$23 = 0x7ff82a0 "hd0"
(gdb) n
93	  ptr += grub_strlen (ptr);
(gdb) p ptr
$24 = 0x7ff82a0 "hd0"
(gdb) n
95	  if (dos_part != 0xff)
(gdb) p ptr
$25 = 0x7ff82a3 ""
(gdb) n
98	  ptr += grub_strlen (ptr);
(gdb) n
100	  if (bsd_part != 0xff)
(gdb) 
103	  ptr += grub_strlen (ptr);
(gdb) 
104	  *ptr = 0;
(gdb) n
105	}
(gdb) 

  if (!device && fwdevice)
    device = fwdevice;
(gdb) p fwdevice 
$26 = 0x7ff82a0 "hd0"
(gdb) p device
$27 = 0x7ff82a0 "hd0"

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
(gdb) p fwpath 
$28 = 0x0
    path = fwpath;
  else
    grub_free (fwpath);
  if (device)
    {
      char *prefix_set;

      prefix_set = grub_xasprintf ("(%s)%s", device, path ? : "");
      if (prefix_set)
        {
          grub_env_set ("prefix", prefix_set);
(gdb) p prefix_set 
$29 = 0x7ff8190 "(hd0)/boot/grub"
          grub_free (prefix_set);
        }
      grub_env_set ("root", device);
(gdb) p device
$30 = 0x7ff82a0 "hd0"
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
$31 = 0xe9b8 "root"
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

Register command set, unset, ls, insmod command to grub commmand, every command has a dedicated response function to process specific operation from console.

```grub_register_core_commands
grub-core/kern/corecmd.c:178

void
grub_register_core_commands (void)
{
  grub_command_t cmd;
  cmd = grub_register_command ("set", grub_core_cmd_set,
                               N_("[ENVVAR=VALUE]"),
                               N_("Set an environment variable."));
(gdb) s
grub_register_command (description=0xe92f "Set an environment variable.", 
    summary=0xe920 "[ENVVAR=VALUE]", func=0xa352 <grub_core_cmd_set>, 
    name=0xe9d1 "set") at ../include/grub/command.h:96
96	  return grub_register_command_prio (name, func, summary, description, 0);
(gdb) s
grub_register_command_prio (name=name@entry=0xe9d1 "set", 
    func=func@entry=0xa352 <grub_core_cmd_set>, 
    summary=summary@entry=0xe920 "[ENVVAR=VALUE]", 
    description=0xe92f "Set an environment variable.", prio=0)
    at kern/command.c:37
37	  cmd = (grub_command_t) grub_zalloc (sizeof (*cmd));
(gdb) n
38	  if (! cmd)
(gdb) p cmd
$33 = (grub_command_t) 0x7ff8320
(gdb) n
41	  cmd->name = name;
(gdb) 
42	  cmd->func = func;
(gdb) 
43	  cmd->summary = (summary) ? summary : "";
44	  cmd->description = description;
(gdb) 
46	  cmd->flags = 0;
(gdb) 
47	  cmd->prio = prio;
(gdb) 
49	  for (p = &grub_command_list, q = *p; q; p = &(q->next), q = q->next)
(gdb) 
53	      r = grub_strcmp (cmd->name, q->name);
(gdb) 
54	      if (r < 0)
(gdb) 
56	      if (r > 0)
(gdb) 
49	  for (p = &grub_command_list, q = *p; q; p = &(q->next), q = q->next)
(gdb) 
70	  if (q)
(gdb) 
68	  *p = cmd;
(gdb) 
69	  cmd->next = q;
(gdb) 
70	  if (q)
(gdb) 
74	  if (! inactive)
(gdb) 
72	  cmd->prev = p;
(gdb) 
74	  if (! inactive)
(gdb) 
75	    cmd->prio |= GRUB_COMMAND_FLAG_ACTIVE;
(gdb) 
78	}
(gdb) 
grub_register_core_commands () at kern/corecmd.c:185

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
```



```grub_load_config
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
```