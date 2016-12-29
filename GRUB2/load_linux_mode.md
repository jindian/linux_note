# load linux mode.

In previous section, one routine inside grub_menu_execute_entry was not introduced, it's grub_script_execute_sourcecode which parses all configurations of a selected menu entry and executes the configurations.

Input parameters of grub_script_execute_sourcecode, only too arguments are shown: number of arguments and arguments array, parameter source is missed, it's the configuration of specific menu entry.

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

```grub_script_execute_sourcecode

grub-core/script/execute.c:786

/* Execute a source script.  */
grub_err_t
grub_script_execute_sourcecode (const char *source, int argc, char **args)
{
  grub_err_t ret = 0;
  struct grub_script *parsed_script;
  struct grub_script_scope new_scope;
  struct grub_script_scope *old_scope;

  auto grub_err_t getline (char **line, int cont);
  grub_err_t getline (char **line, int cont __attribute__ ((unused)))
  {
    const char *p;

    if (! source)
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

  new_scope.argv.argc = argc;
  new_scope.argv.args = args;
  new_scope.flags = 0;

  old_scope = scope;
  scope = &new_scope;

(gdb) p source
$6 = 0x7ff0000 "setparams 'linux OS'\n\n    set root=(hd0,1)\n    linux /boot/vmlinuz-2.6.32.69 root=/dev/sda\n    initrd /boot/initrd.img-2.6.32.69\n"
  while (source)
    {
      char *line;

      getline (&line, 0);
      parsed_script = grub_script_parse (line, getline);
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

All parameters of routine grub_script_parse every time it involved

```parameter_of_grub_script_parse
1.  grub_script_parse (script=0x7ff02e0 "setparams 'linux OS'", 
        getline=getline@entry=0x7fe74) at script/script.c:350
```


```grub_script_parse

grub-core/script/script.c:340

/* Parse the script passed in SCRIPT and return the parsed
   datastructure that is ready to be interpreted.  */
struct grub_script *
grub_script_parse (char *script, grub_reader_getline_t getline)
{
  struct grub_script *parsed;
  struct grub_script_mem *membackup;
  struct grub_lexer_param *lexstate;
  struct grub_parser_param *parsestate;

  parsed = grub_script_create (0, 0);
  if (!parsed)
    return 0;
(gdb) p *parsed
$16 = {refcnt = 0, mem = 0x0, cmd = 0x0, next_siblings = 0x0, children = 0x0}

  parsestate = grub_zalloc (sizeof (*parsestate));
  if (!parsestate)
    {
      grub_free (parsed);
      return 0;
    }
$18 = {func_mem = 0x0, err = 0, memused = 0x0, scripts = 0x0, parsed = 0x0, 
  lexerstate = 0x0}

  /* Initialize the lexer.  */
  lexstate = grub_script_lexer_init (parsestate, script, getline);
  if (!lexstate)
    {
      grub_free (parsed);
      grub_free (parsestate);
      return 0;
    }

  parsestate->lexerstate = lexstate;

  membackup = grub_script_mem_record (parsestate);

  /* Parse the script.  */
  if (grub_script_yyparse (parsestate) || parsestate->err)
    {
      struct grub_script_mem *memfree;
      memfree = grub_script_mem_record_stop (parsestate, membackup);
      grub_script_mem_free (memfree);
      grub_script_lexer_fini (lexstate);
      grub_free (parsestate);
      grub_free (parsed);
      return 0;
    }

  parsed->mem = grub_script_mem_record_stop (parsestate, membackup);
  parsed->cmd = parsestate->parsed;
  parsed->children = parsestate->scripts;

  grub_script_lexer_fini (lexstate);
  grub_free (parsestate);

  return parsed;
}
```