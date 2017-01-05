# grub file system

In order to read from disk, grub implemets several filesystems to support it. ext2 is the most classic one in all filesystems, ext2 is the filesystem in grub startup of this book. Let's study it in grub startup when loading linux module.

grub_fs is filesystem descriptor, includes basic filesystem properties and functions of file operation, it's similar with linux filesystem but less complex. Every specific filesystem described with this data structure, all filesystems saved in link list grub_fs_list, from below debug information, only one filesystem exists when loading linux module.

```
(gdb) p grub_fs_list 
$12 = (grub_fs_t) 0x7ffb728 <grub_ext2_fs>
(gdb) p grub_fs_list->next
$13 = (struct grub_fs *) 0x0
```

Here is the filesystem descriptor of grub.

```grub_fs

/* Filesystem descriptor.  */
struct grub_fs
{
  /* The next filesystem.  */
  struct grub_fs *next;
  struct grub_fs **prev;

  /* My name.  */
  const char *name;

  /* Call HOOK with each file under DIR.  */
  grub_err_t (*dir) (grub_device_t device, const char *path,
		     int (*hook) (const char *filename,
				  const struct grub_dirhook_info *info));

  /* Open a file named NAME and initialize FILE.  */
  grub_err_t (*open) (struct grub_file *file, const char *name);

  /* Read LEN bytes data from FILE into BUF.  */
  grub_ssize_t (*read) (struct grub_file *file, char *buf, grub_size_t len);

  /* Close the file FILE.  */
  grub_err_t (*close) (struct grub_file *file);

  /* Return the label of the device DEVICE in LABEL.  The label is
     returned in a grub_malloc'ed buffer and should be freed by the
     caller.  */
  grub_err_t (*label) (grub_device_t device, char **label);

  /* Return the uuid of the device DEVICE in UUID.  The uuid is
     returned in a grub_malloc'ed buffer and should be freed by the
     caller.  */
  grub_err_t (*uuid) (grub_device_t device, char **uuid);

  /* Get writing time of filesystem. */
  grub_err_t (*mtime) (grub_device_t device, grub_int32_t *timebuf);

#ifdef GRUB_UTIL
  /* Determine sectors available for embedding.  */
  grub_err_t (*embed) (grub_device_t device, unsigned int *nsectors,
		       unsigned int max_nsectors,
		       grub_embed_type_t embed_type,
		       grub_disk_addr_t **sectors);

  /* Whether this filesystem reserves first sector for DOS-style boot.  */
  int reserved_first_sector;

  /* Whether blocklist installs have a chance to work.  */
  int blocklist_install;
#endif
};
typedef struct grub_fs *grub_fs_t;
```

ext2 filesystem is one of the modules included in grub core image, it's added to grub_fs_list when loading ext2 module. Call context of loading ext2 filesystem module as below, the initialization of grub ext2 filesystem is very simple.

```load_ext2_module
grub_main
    |--grub_load_modules
        |--grub_dl_load_core
            |--grub_dl_call_init
                |--(mod->init) (mod) -> grub_mod_init (mod=0x7ffbe50) at fs/ext2.c:983
                    |--grub_fs_register
                        |--grub_list_push                //push ext2 to grub_fs_list                           
```                

Instance of ext2 filesystem as below, later file created with style of ext2 filesystem could be operated with all routines inclued in grub_ext2_fs. Other filesytems are similar with ext2.

```instance_ext2

grub-core/fs/ext2.c:964

static struct grub_fs grub_ext2_fs =
  {
    .name = "ext2",
    .dir = grub_ext2_dir,
    .open = grub_ext2_open,
    .read = grub_ext2_read,
    .close = grub_ext2_close,
    .label = grub_ext2_label,
    .uuid = grub_ext2_uuid,
    .mtime = grub_ext2_mtime,
#ifdef GRUB_UTIL
    .reserved_first_sector = 1,
    .blocklist_install = 1,
#endif
    .next = 0
  };
```

Other important data structures of ext2 filesystem listed: super block, glock group, inode and directory entry. It's similar with linux.

```basic_ext2_filesystem_structures

grub-core/fs/ext2.c:134

/* The ext2 superblock.  */
struct grub_ext2_sblock
{
  grub_uint32_t total_inodes;
  grub_uint32_t total_blocks;
  grub_uint32_t reserved_blocks;
  grub_uint32_t free_blocks;
  grub_uint32_t free_inodes;
  grub_uint32_t first_data_block;
  grub_uint32_t log2_block_size;
  grub_uint32_t log2_fragment_size;
  grub_uint32_t blocks_per_group;
  grub_uint32_t fragments_per_group;
  grub_uint32_t inodes_per_group;
  grub_uint32_t mtime;
  grub_uint32_t utime;
  grub_uint16_t mnt_count;
  grub_uint16_t max_mnt_count;
  grub_uint16_t magic;
  grub_uint16_t fs_state;
  grub_uint16_t error_handling;
  grub_uint16_t minor_revision_level;
  grub_uint32_t lastcheck;
  grub_uint32_t checkinterval;
  grub_uint32_t creator_os;
  grub_uint32_t revision_level;
  grub_uint16_t uid_reserved;
  grub_uint16_t gid_reserved;
  grub_uint32_t first_inode;
  grub_uint16_t inode_size;
  grub_uint16_t block_group_number;
  grub_uint32_t feature_compatibility;
  grub_uint32_t feature_incompat;
  grub_uint32_t feature_ro_compat;
  grub_uint16_t uuid[8];
  char volume_name[16];
  char last_mounted_on[64];
  grub_uint32_t compression_info;
  grub_uint8_t prealloc_blocks;
  grub_uint8_t prealloc_dir_blocks;
  grub_uint16_t reserved_gdt_blocks;
  grub_uint8_t journal_uuid[16];
  grub_uint32_t journal_inum;
  grub_uint32_t journal_dev;
  grub_uint32_t last_orphan;
  grub_uint32_t hash_seed[4];
  grub_uint8_t def_hash_version;
  grub_uint8_t jnl_backup_type;
  grub_uint16_t reserved_word_pad;
  grub_uint32_t default_mount_opts;
  grub_uint32_t first_meta_bg;
  grub_uint32_t mkfs_time;
  grub_uint32_t jnl_blocks[17];
};

/* The ext2 blockgroup.  */
struct grub_ext2_block_group
{
  grub_uint32_t block_id;
  grub_uint32_t inode_id;
  grub_uint32_t inode_table_id;
  grub_uint16_t free_blocks;
  grub_uint16_t free_inodes;
  grub_uint16_t used_dirs;
  grub_uint16_t pad;
  grub_uint32_t reserved[3];
};

/* The ext2 inode.  */
struct grub_ext2_inode
{
  grub_uint16_t mode;
  grub_uint16_t uid;
  grub_uint32_t size;
  grub_uint32_t atime;
  grub_uint32_t ctime;
  grub_uint32_t mtime;
  grub_uint32_t dtime;
  grub_uint16_t gid;
  grub_uint16_t nlinks;
  grub_uint32_t blockcnt;  /* Blocks of 512 bytes!! */
  grub_uint32_t flags;
  grub_uint32_t osd1;
  union
  {
    struct datablocks
    {
      grub_uint32_t dir_blocks[INDIRECT_BLOCKS];
      grub_uint32_t indir_block;
      grub_uint32_t double_indir_block;
      grub_uint32_t triple_indir_block;
    } blocks;
    char symlink[60];
  };
  grub_uint32_t version;
  grub_uint32_t acl;
  grub_uint32_t size_high;
  grub_uint32_t fragment_addr;
  grub_uint32_t osd2[3];
};

/* The header of an ext2 directory entry.  */
struct ext2_dirent
{
  grub_uint32_t inode;
  grub_uint16_t direntlen;
  grub_uint8_t namelen;
  grub_uint8_t filetype;
};
```

grub_dl_load_file routine includes a typical file opertions procedure: grub_file_open, grub_file_read, grub_file_close. Let's study it one by one.

```ext2_file_operations
grub_dl_load_file
    |--grub_file_open
        |--grub_file_get_device_name                    //get device from configuration of grub.cfg
        |--grub_device_open
        |--grub_fs_probe                                //find proper filesystem for disk  device
            |--grub_ext2_dir
                |--grub_ext2_mount
                |--grub_fshelp_find_file
                |--grub_ext2_iterate_dir
        |--grub_ext2_open
            |--grub_ext2_mount
            |--grub_fshelp_find_file
            |--grub_ext2_read_inode
    |--grub_file_size
    |--grub_file_read
        |--grub_ext2_read
            |--grub_ext2_read_file
                |--grub_fshelp_read_file
                    |--grub_disk_read
    |--grub_file_close
        |--grub_ext2_close
        |--grub_device_close
    |--grub_dl_load_core
```

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

```grub_file_open

grub_file_open (name=0x7fe17c0 "(hd0,msdos1)/boot/grub/i386-pc/linux.mod") at kern/file.c:70

grub-core/kern/file.c:61

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

Find proper filesystem for device disk, only one filesystem in grub_fs_list.

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

(gdb) p grub_fs_list 
$3 = (grub_fs_t) 0x7ffb728 <grub_ext2_fs>
(gdb) p grub_fs_list->next
$4 = (struct grub_fs *) 0x0
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



```grub_ext2_dir

grub_ext2_dir (device=0x7fe1780, path=0xe8b1 "/", hook=0xc557 <dummy_func>) at fs/ext2.c:828

grub-core/fs/ext2.c:827

static grub_err_t
grub_ext2_dir (grub_device_t device, const char *path,
	       int (*hook) (const char *filename,
			    const struct grub_dirhook_info *info))
{
  struct grub_ext2_data *data = 0;
  struct grub_fshelp_node *fdiro = 0;

  auto int NESTED_FUNC_ATTR iterate (const char *filename,
				     enum grub_fshelp_filetype filetype,
				     grub_fshelp_node_t node);

  int NESTED_FUNC_ATTR iterate (const char *filename,
				enum grub_fshelp_filetype filetype,
				grub_fshelp_node_t node)
    {
      struct grub_dirhook_info info;
      grub_memset (&info, 0, sizeof (info));
      if (! node->inode_read)
	{
	  grub_ext2_read_inode (data, node->ino, &node->inode);
	  if (!grub_errno)
	    node->inode_read = 1;
	  grub_errno = GRUB_ERR_NONE;
	}
      if (node->inode_read)
	{
	  info.mtimeset = 1;
	  info.mtime = grub_le_to_cpu32 (node->inode.mtime);
	}

      info.dir = ((filetype & GRUB_FSHELP_TYPE_MASK) == GRUB_FSHELP_DIR);
      grub_free (node);
      return hook (filename, &info);
    }

  grub_dl_ref (my_mod);

  data = grub_ext2_mount (device->disk);
  if (! data)
    goto fail;
(gdb) p data
$5 = (struct grub_ext2_data *) 0x7fe1450
(gdb) p *data
$6 = {sblock = {total_inodes = 3600, total_blocks = 14368,
reserved_blocks = 718, free_blocks = 3190, free_inodes = 3323,
first_data_block = 1, log2_block_size = 0, log2_fragment_size = 0,
blocks_per_group = 8192, fragments_per_group = 8192,
inodes_per_group = 1800, mtime = 1482137221, utime = 1482137224,
mnt_count = 1, max_mnt_count = 65535, magic = 61267, fs_state = 1,
error_handling = 1, minor_revision_level = 0, lastcheck = 1482137221,
checkinterval = 0, creator_os = 0, revision_level = 1, uid_reserved = 0,
gid_reserved = 0, first_inode = 11, inode_size = 128,
block_group_number = 0, feature_compatibility = 56, feature_incompat = 2, 
feature_ro_compat = 1, uuid = {43444, 52472, 62440, 63562, 31403, 21528, 
30361, 53694}, volume_name = '\000' <repeats 15 times>,
last_mounted_on = "/mnt/disk_img_mk", '\000' <repeats 47 times>,
compression_info = 0, prealloc_blocks = 0 '\000',
prealloc_dir_blocks = 0 '\000', reserved_gdt_blocks = 56,
journal_uuid = '\000' <repeats 15 times>, journal_inum = 0,
journal_dev = 0, last_orphan = 0, hash_seed = {3035430676, 3494066414,
116144551, 3742743110}, def_hash_version = 1 '\001',
jnl_backup_type = 0 '\000', reserved_word_pad = 0,
default_mount_opts = 12, first_meta_bg = 0, mkfs_time = 1482137221,
jnl_blocks = {0 <repeats 17 times>}}, disk = 0x7fe1740,
inode = 0x7fe15ac, diropen = {data = 0x7fe1450, inode = {mode = 16877,
uid = 0, size = 1024, atime = 1482137221, ctime = 1482137221,
mtime = 1482137221, dtime = 0, gid = 0, nlinks = 4, blockcnt = 2,
flags = 0, osd1 = 1, {blocks = {dir_blocks = {286,
0 <repeats 11 times>}, indir_block = 0, double_indir_block = 0,
triple_indir_block = 0},
symlink = "\036\001", '\000' <repeats 57 times>}, version = 0,
acl = 0, size_high = 0, fragment_addr = 0, osd2 = {0, 0, 0}}, ino = 2,
inode_read = 1}}

  grub_fshelp_find_file (path, &data->diropen, &fdiro, grub_ext2_iterate_dir,
			 grub_ext2_read_symlink, GRUB_FSHELP_DIR);
  if (grub_errno)
    goto fail;

  grub_ext2_iterate_dir (fdiro, iterate);

 fail:
  if (fdiro != &data->diropen)
    grub_free (fdiro);
  grub_free (data);

  grub_dl_unref (my_mod);

  return grub_errno;
}
```



```grub_ext2_open

grub_ext2_open (file=0x7fe1650, name=0x7fe17cc "/boot/grub/i386-pc/linux.mod") at fs/ext2.c:764

grub-core/fs/ext2.c:756

/* Open a file named NAME and initialize FILE.  */
static grub_err_t
grub_ext2_open (struct grub_file *file, const char *name)
{
  struct grub_ext2_data *data;
  struct grub_fshelp_node *fdiro = 0;
  grub_err_t err;

  grub_dl_ref (my_mod);

  data = grub_ext2_mount (file->device->disk);
  if (! data)
    {
      err = grub_errno;
      goto fail;
    }

  err = grub_fshelp_find_file (name, &data->diropen, &fdiro,
			       grub_ext2_iterate_dir,
			       grub_ext2_read_symlink, GRUB_FSHELP_REG);
  if (err)
    goto fail;

  if (! fdiro->inode_read)
    {
      err = grub_ext2_read_inode (data, fdiro->ino, &fdiro->inode);
      if (err)
	goto fail;
    }

  grub_memcpy (data->inode, &fdiro->inode, sizeof (struct grub_ext2_inode));
  grub_free (fdiro);

  file->size = grub_le_to_cpu32 (data->inode->size);
  file->size |= ((grub_off_t) grub_le_to_cpu32 (data->inode->size_high)) << 32;
  file->data = data;
  file->offset = 0;

  return 0;

 fail:
  if (fdiro != &data->diropen)
    grub_free (fdiro);
  grub_free (data);

  grub_dl_unref (my_mod);

  return err;
}
```



```grub_file_read

grub_file_read (file=file@entry=0x7fe1650, buf=buf@entry=0x7fa1c20, len=len@entry=11488) at kern/file.c:139

grub-core/kern/file.c:134

grub_ssize_t
grub_file_read (grub_file_t file, void *buf, grub_size_t len)
{
  grub_ssize_t res;

  if (file->offset > file->size)
    {
      grub_error (GRUB_ERR_OUT_OF_RANGE,
		  N_("attempt to read past the end of file"));
      return -1;
    }

  if (len == 0)
    return 0;

  if (len > file->size - file->offset)
    len = file->size - file->offset;

  /* Prevent an overflow.  */
  if ((grub_ssize_t) len < 0)
    len >>= 1;

  if (len == 0)
    return 0;
  res = (file->fs->read) (file, buf, len);
  if (res > 0)
    file->offset += res;

  return res;
}
```



```

grub_ext2_read (file=0x7fe1650, buf=0x7fa1c20 "", len=11488) at fs/ext2.c:819

grub-core/fs/ext2.c:816

/* Read LEN bytes data from FILE into BUF.  */
static grub_ssize_t
grub_ext2_read (grub_file_t file, char *buf, grub_size_t len)
{
  struct grub_ext2_data *data = (struct grub_ext2_data *) file->data;

  return grub_ext2_read_file (&data->diropen, file->read_hook,
                              file->offset, len, buf);
}
```



```grub_ext2_read_file

grub_ext2_read_file (node=0x7fe15a8, read_hook=0x0, pos=0, len=11488, buf=0x7fa1c20 "") at fs/ext2.c:523

grub-core/fs/ext2.c:511

/* Read LEN bytes from the file described by DATA starting with byte
   POS.  Return the amount of read bytes in READ.  */
static grub_ssize_t
grub_ext2_read_file (grub_fshelp_node_t node,
                     void NESTED_FUNC_ATTR (*read_hook) (grub_disk_addr_t sector,
                                        unsigned offset, unsigned length),
                     grub_off_t pos, grub_size_t len, char *buf)
{
  return grub_fshelp_read_file (node->data->disk, node, read_hook,
                                pos, len, buf, grub_ext2_read_block,
                                grub_cpu_to_le32 (node->inode.size)
                                | (((grub_off_t) grub_cpu_to_le32 (node->inode.size_high)) << 32),
                                LOG2_EXT2_BLOCK_SIZE (node->data), 0);

}

```



```grub_file_close

grub-core/fs/ext2.c:165

grub_err_t
grub_file_close (grub_file_t file)
{
  if (file->fs->close)
    (file->fs->close) (file);

  if (file->device)
    grub_device_close (file->device);
  grub_free (file);
  return grub_errno;
}
```


# LINKS

* [second extended filesystem](https://en.wikipedia.org/wiki/Ext2)
* [file system](https://en.wikipedia.org/wiki/File_system)



