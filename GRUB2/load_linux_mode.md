# load linux mode

Load linux mode is part of grub_script_execute_sourcecode, besides linux module, there are several other modules loaded, so a new section created here. Let's continue read grub_script_execute_sourcecode routine.

grub_script_execute_sourcecode
    |--grub_script_parse
    |--grub_script_execute
        |--grub_script_execute_cmd


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
/* Execute any GRUB pre-parsed command or script.  */
grub_err_t
grub_script_execute (struct grub_script *script)
{
  if (script == 0)
    return 0;
  
  return grub_script_execute_cmd (script->cmd);
}
```