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