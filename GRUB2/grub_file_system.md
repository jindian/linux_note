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

Other important data structures of ext2 filesystem are super block, glock group, inode and directory entry, same as linux filesystem.

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

# LINKS

* [second extended filesystem](https://en.wikipedia.org/wiki/Ext2)
* [file system](https://en.wikipedia.org/wiki/File_system)



