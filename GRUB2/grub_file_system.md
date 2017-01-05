# grub file system

In order to read from disk, grub implemets several filesystems to support it. ext2 is the most classic one in all filesystems, ext2 is the filesystem in grub startup of this book. Let's study it in grub startup when loading linux module.

grub_dl_load_file routine includes a typical file opertions procedure: grub_file_open, grub_file_read, grub_file_close. Let's study it one by one.

```grub_dl_load_file

grub_dl_load_file (filename=0x7fe17c0 "(hd0,msdos1)/boot/grub/i386-pc/linux.mod") at kern/dl.c:671

grub-core/kern/dl.c:662

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

# LINKS

* [second extended filesystem](https://en.wikipedia.org/wiki/Ext2)
* [file system](https://en.wikipedia.org/wiki/File_system)



