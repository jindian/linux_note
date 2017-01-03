# execute commands in grub configuration

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

Inside grub_script_execute_sourcecode, parse configuration with grub_script_parse and execute it one by one.

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

1. line: "setparams 'linux OS'"
   parsed_script: {refcnt = 0, mem = 0x7fb53c0, cmd = 0x7fb53c4, next_siblings = 0x0, children = 0x0}
   
   function call context
   grub_script_execute_sourcecode
       |--grub_script_parse                       //detail of how to parse a line ignored here
       |--grub_script_execute
           |--grub_script_execute_cmd
               |--grub_script_execute_cmdlist
                   |--grub_script_execute_cmd
                       |--grub_script_execute_cmdline
                           |--grub_script_arglist_to_argv
                           |--grub_command_find
                           |--(grubcmd->func) (grubcmd, argc, args) -> grub_script_setparams
                               |--replace_scope

2. line: ""
   parsed_script: {refcnt = 0, mem = 0x7fb5090, cmd = 0x0, next_siblings = 0x0, children = 0x0}

```grub_script_execute
grub_script_execute (script=0x7ff02b0) at script/execute.c:1080

grub-core/script/execute.c:1077

/* Execute any GRUB pre-parsed command or script.  */
grub_err_t
grub_script_execute (struct grub_script *script)
{
  if (script == 0)
    return 0;

  return grub_script_execute_cmd (script->cmd);
}

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmd (cmd=0x7fb53c4) at script/execute.c:750

grub-core/script/execute.c:744

static grub_err_t
grub_script_execute_cmd (struct grub_script_cmd *cmd)
{
  int ret;
  char errnobuf[ERRNO_DIGITS_MAX + 1];

  if (cmd == 0)
    return 0;

  ret = cmd->exec (cmd);

  grub_snprintf (errnobuf, sizeof (errnobuf), "%d", ret);
  grub_env_set ("?", errnobuf);
  return ret;
}

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmdlist (list=0x7fb53c4) at script/execute.c:967

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

(gdb) p function_return 
$17 = 0
      if (function_return)
        break;
    }

  return ret;
}

--------------------------------------------------------------------------------------------------------------------------------

Involve grub_script_execute_cmd again, only parameters listed here

grub_script_execute_cmd (cmd=cmd@entry=0x7fb50d4) at script/execute.c:750

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmdline (cmd=0x7fb50d4) at script/execute.c:859

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
$6 = {argc = 2, args = 0x7feffa0, script = 0x0}
(gdb) p argv->args[0]
$7 = 0x7feff30 "setparams"
(gdb) p argv->args[1]
$8 = 0x7fefe70 "linux OS"

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
(gdb) p cmdname 
$9 = 0x7feff30 "setparams"
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

--------------------------------------------------------------------------------------------------------------------------------

grub_script_setparams (cmd=<optimized out>, argc=1, args=0x7feffa4)
    at script/execute.c:191

grub-core/script/execute.c:184

grub_err_t
grub_script_setparams (grub_command_t cmd __attribute__((unused)),
		       int argc, char **args)
{
  struct grub_script_scope *new_scope;
  struct grub_script_argv argv = { 0, 0, 0 };

  if (! scope)
    return GRUB_ERR_INVALID_COMMAND;

  new_scope = grub_malloc (sizeof (*new_scope));
  if (! new_scope)
    return grub_errno;

  if (grub_script_argv_make (&argv, argc, args))
    {
      grub_free (new_scope);
      return grub_errno;
    }
(gdb) p argv
$11 = {argc = 1, args = 0x7ff0260, script = 0x0}
(gdb) p argv->args[0]
$12 = 0x7feff80 "linux OS"

  new_scope->shifts = 0;
  new_scope->argv = argv;
  new_scope->flags = GRUB_SCRIPT_SCOPE_MALLOCED |
    GRUB_SCRIPT_SCOPE_ARGS_MALLOCED;

  replace_scope (new_scope);
  return GRUB_ERR_NONE;
}

--------------------------------------------------------------------------------------------------------------------------------

replace_scope (new_scope=new_scope@entry=0x7ff0280) at script/execute.c:107

grub-core/script/execute.c:104

static void
replace_scope (struct grub_script_scope *new_scope)
{
$14 = {flags = 0, shifts = 134154864, argv = {argc = 1, args = 0x7fefef0, 
    script = 0x100000}}
(gdb) p scope->argv.args[0]
$16 = 0x7fefed0 "linux OS"
  if (scope)
    {
      scope->argv.argc += scope->shifts;
      scope->argv.args -= scope->shifts;

      if (scope->flags & GRUB_SCRIPT_SCOPE_ARGS_MALLOCED)
        grub_script_argv_free (&scope->argv);

      if (scope->flags & GRUB_SCRIPT_SCOPE_MALLOCED)
        grub_free (scope);
    }
  scope = new_scope;
}

```

3. line "    set root=(hd0,1)"
   parsed_script: {refcnt = 0, mem = 0x7fb4fc0, cmd = 0x7fb4fc4, next_siblings = 0x0, children = 0x0}

   grub_script_execute_sourcecode
       |--grub_script_parse
       |--grub_script_execute
           |--grub_script_execute_cmd
               |--grub_script_execute_cmdlist
                   |--grub_script_execute_cmd
                       |--grub_script_execute_cmdline
                           |--grub_script_arglist_to_argv
                           |--grub_command_find
                           |--(grubcmd->func) (grubcmd, argc, args) -> grub_core_cmd_set


```grub_script_execute

grub_script_execute (script=0x7feff50) at script/execute.c:1080

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmd (cmd=0x7fb4fc4) at script/execute.c:750

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmdlist (list=0x7fb4fc4) at script/execute.c:967

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmd (cmd=cmd@entry=0x7fb4ce4) at script/execute.c:750

--------------------------------------------------------------------------------------------------------------------------------

grub_script_execute_cmdline (cmd=0x7fb4ce4) at script/execute.c:859

/* Execute a single command line.  */
grub_err_t
grub_script_execute_cmdline (struct grub_script_cmd *cmd)
{

  ......

  /* Lookup the command.  */
  if (grub_script_arglist_to_argv (cmdline->arglist, &argv) || ! argv.args[0])
    return grub_errno;
(gdb) p argv
$24 = {argc = 2, args = 0x7fe7df0, script = 0x0}
(gdb) p argv->args[0]
$25 = 0x7fe1960 "set"
(gdb) p argv->args[1]
$26 = 0x7fe1940 "root=(hd0,1)"

  ......
  
  grubcmd = grub_command_find (cmdname);
(gdb) p cmdname
$23 = 0x7fe1960 "set"
  
  ......

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

  ......

}

--------------------------------------------------------------------------------------------------------------------------------

grub_core_cmd_set (cmd=0x7ff8320, argc=1, argv=0x7fe7df4) at kern/corecmd.c:35

grub-core/kern/corecmd.c:31

/* set ENVVAR=VALUE */
static grub_err_t
grub_core_cmd_set (struct grub_command *cmd __attribute__ ((unused)),
		   int argc, char *argv[])
{
  char *var;
  char *val;

  auto int print_env (struct grub_env_var *env);

  int print_env (struct grub_env_var *env)
    {
      grub_printf ("%s=%s\n", env->name, env->value);
      return 0;
    }

  if (argc < 1)
    {
      grub_env_iterate (print_env);
      return 0;
    }
(gdb) p argv[0]
$27 = 0x7fe1940 "root=(hd0,1)"

  var = argv[0];
  val = grub_strchr (var, '=');
  if (! val)
    return grub_error (GRUB_ERR_BAD_ARGUMENT, "not an assignment");

  val[0] = 0;
(gdb) p var
$30 = 0x7fe1940 "root"
(gdb) p val
$31 = 0x7fe1944 ""
(gdb) p val+1
$32 = 0x7fe1945 "(hd0,1)"
  grub_env_set (var, val + 1);
  val[0] = '=';

  return 0;
}
```