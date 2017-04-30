# **start\_kernel part X**

Continue routine `vfs_caches_init` of start\_kernel part IX

* Involves routine `mnt_init`, which allocates slab cache for struct `vfsmount`named `mnt_cache`,  gets free pages for `mount_hashtable` and initializes  it, registers `sysfs`, this is needed later for actually finding root device, registers `rootfs`, creates the initial filesystem namespace, with rootfs mounted at `/`

  Let's check more about `sysfs_init`, `init_rootfs`and `init_mount_tree`

```
(gdb) break sysfs_init
Breakpoint 6 at 0xc1717b35: file fs/sysfs/mount.c, line 89.
(gdb) c
Continuing.

Breakpoint 6, sysfs_init () at fs/sysfs/mount.c:89
89    {
(gdb) n
92        sysfs_dir_cachep = kmem_cache_create("sysfs_dir_cache",
(gdb) 
95        if (!sysfs_dir_cachep)
(gdb) 
98        err = sysfs_inode_init();
(gdb) s
sysfs_inode_init () at fs/sysfs/inode.c:46
46        return bdi_init(&sysfs_backing_dev_info);
(gdb) s
bdi_init (bdi=bdi@entry=0xc16ac140 <sysfs_backing_dev_info>)
    at mm/backing-dev.c:660
660        spin_lock_init(&bdi->wb_lock);
(gdb) n
652    {
(gdb) 
655        bdi->dev = NULL;
(gdb) 
657        bdi->min_ratio = 0;
(gdb) 
658        bdi->max_ratio = 100;
(gdb) 
659        bdi->max_prop_frac = PROP_FRAC_BASE;
(gdb) 
660        spin_lock_init(&bdi->wb_lock);
(gdb) s
__spin_lock_init (
    lock=lock@entry=0xc16ac284 <sysfs_backing_dev_info+324>, 
    name=name@entry=0xc15ece57 "&bdi->wb_lock", 
    key=key@entry=0xc1e5c9e8 <__key.22236>)
    at lib/spinlock_debug.c:23
23        debug_check_no_locks_freed((void *)lock, sizeof(*lock));
(gdb) n
24        lockdep_init_map(&lock->dep_map, name, key, 0);
(gdb) 
26        lock->raw_lock = (raw_spinlock_t)__RAW_SPIN_LOCK_UNLOCKED;
(gdb) 
27        lock->magic = SPINLOCK_MAGIC;
(gdb) 
28        lock->owner = SPINLOCK_OWNER_INIT;
(gdb) 
29        lock->owner_cpu = -1;
(gdb) 
30    }
(gdb) 
bdi_init (bdi=bdi@entry=0xc16ac140 <sysfs_backing_dev_info>)
    at mm/backing-dev.c:661
661        INIT_RCU_HEAD(&bdi->rcu_head);
(gdb) 
662        INIT_LIST_HEAD(&bdi->bdi_list);
(gdb) 
663        INIT_LIST_HEAD(&bdi->wb_list);
(gdb) 
664        INIT_LIST_HEAD(&bdi->work_list);
(gdb) 
666        bdi_wb_init(&bdi->wb, bdi);
(gdb) 
671        bdi->wb_mask = 1;
(gdb) 
672        bdi->wb_cnt = 1;
(gdb) 
675            err = percpu_counter_init(&bdi->bdi_stat[i], 0);
(gdb) p i
$10 = 1
(gdb) n
676            if (err)
(gdb) 
680        bdi->dirty_exceeded = 0;
(gdb) 
681        err = prop_local_init_percpu(&bdi->completions);
(gdb) 
685            while (i--)
(gdb) p err
$12 = 0
(gdb) n
690    }
(gdb) 
sysfs_inode_init () at fs/sysfs/inode.c:47
47    }
(gdb) 
sysfs_init () at fs/sysfs/mount.c:99
99        if (err)
(gdb) 
102        err = register_filesystem(&sysfs_fs_type);
(gdb) p sysfs_fs_type 
$13 = {name = 0xc1616401 "sysfs", fs_flags = 0, 
  get_sb = 0xc11716c0 <sysfs_get_sb>, 
  kill_sb = 0xc1115230 <kill_anon_super>, owner = 0x0, next = 0x0, 
  fs_supers = {next = 0x0, prev = 0x0}, s_lock_key = {subkeys = {{
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, 
  s_umount_key = {subkeys = {{__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}}}, i_lock_key = {subkeys = {{
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, 
  i_mutex_key = {subkeys = {{__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}}}, i_mutex_dir_key = {subkeys = {{
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
---Type <return> to continue, or q <return> to quit---
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, 
  i_alloc_sem_key = {subkeys = {{__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}}}}
(gdb) s
register_filesystem (fs=fs@entry=0xc16ac520 <sysfs_fs_type>)
    at fs/filesystems.c:74
74        BUG_ON(strchr(fs->name, '.'));
(gdb) n
75        if (fs->next)
(gdb) 
77        INIT_LIST_HEAD(&fs->fs_supers);
(gdb) 
78        write_lock(&file_systems_lock);
(gdb) 
79        p = find_filesystem(fs->name, strlen(fs->name));
(gdb) s
strlen (s=0xc1616401 "sysfs") at arch/x86/lib/string_32.c:168
168        asm volatile("repne\n\t"
(gdb) n
176    }
(gdb) s
find_filesystem (name=0xc1616401 "sysfs", len=5)
    at fs/filesystems.c:49
49        for (p=&file_systems; *p; p=&(*p)->next)
(gdb) n
54    }
(gdb) 
register_filesystem (fs=fs@entry=0xc16ac520 <sysfs_fs_type>)
    at fs/filesystems.c:80
80        if (*p)
(gdb) 
83            *p = fs;
(gdb) 
71        int res = 0;
(gdb) 
84        write_unlock(&file_systems_lock);
(gdb) 
85        return res;
(gdb) 
86    }
(gdb) 
sysfs_init () at fs/sysfs/mount.c:103
103        if (!err) {
(gdb) p file_systems
$14 = (struct file_system_type *) 0xc16ac520 <sysfs_fs_type>
(gdb) p file_systems->next
$15 = (struct file_system_type *) 0x0
104            sysfs_mount = kern_mount(&sysfs_fs_type);
(gdb) s
kern_mount_data (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    data=data@entry=0x0) at fs/super.c:1016
1016        return vfs_kern_mount(type, MS_KERNMOUNT, type->name, data);
(gdb) s
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=0xc1616401 "sysfs", 
    data=data@entry=0x0) at fs/super.c:920
920		if (!type)
(gdb) n
924		mnt = alloc_vfsmnt(name);
(gdb) s
alloc_vfsmnt (name=name@entry=0xc1616401 "sysfs")
    at fs/namespace.c:130
130		struct vfsmount *mnt = kmem_cache_zalloc(mnt_cache, GFP_KERNEL);
(gdb) n
131		if (mnt) {
(gdb) 
134			err = mnt_alloc_id(mnt);
(gdb) 
138			if (name) {
(gdb) 
139				mnt->mnt_devname = kstrdup(name, GFP_KERNEL);
(gdb) 
140				if (!mnt->mnt_devname)
(gdb) 
144			atomic_set(&mnt->mnt_count, 1);
(gdb) 
145			INIT_LIST_HEAD(&mnt->mnt_hash);
(gdb) 
146			INIT_LIST_HEAD(&mnt->mnt_child);
(gdb) 
147			INIT_LIST_HEAD(&mnt->mnt_mounts);
(gdb) 
148			INIT_LIST_HEAD(&mnt->mnt_list);
(gdb) 
149			INIT_LIST_HEAD(&mnt->mnt_expire);
(gdb) 
150			INIT_LIST_HEAD(&mnt->mnt_share);
(gdb) 
151			INIT_LIST_HEAD(&mnt->mnt_slave_list);
(gdb) 
152			INIT_LIST_HEAD(&mnt->mnt_slave);
(gdb) 
154			mnt->mnt_writers = alloc_percpu(int);
(gdb) 
155			if (!mnt->mnt_writers)
(gdb) 
172	}
(gdb) 
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=0xc1616401 "sysfs", 
    data=data@entry=0x0) at fs/super.c:925
925		if (!mnt)
(gdb) 
928		if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {
(gdb) 
938		error = type->get_sb(type, flags, name, data, mnt);
(gdb) 
939		if (error < 0)
(gdb) 
941		BUG_ON(!mnt->mnt_sb);
(gdb) 
943	 	error = security_sb_kern_mount(mnt->mnt_sb, flags, secdata);
(gdb) s
security_sb_kern_mount (sb=0xc7008430, flags=flags@entry=4194304, 
    data=data@entry=0x0) at security/security.c:273
273		return security_ops->sb_kern_mount(sb, flags, data);
(gdb) s
cap_sb_kern_mount (sb=0xc7008430, flags=4194304, data=0x0)
    at security/capability.c:65
65	}
(gdb) n
security_sb_kern_mount (sb=<optimized out>, 
    flags=flags@entry=4194304, data=data@entry=0x0)
    at security/security.c:274
274	}
(gdb) 
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=<optimized out>, 
    data=data@entry=0x0) at fs/super.c:944
944	 	if (error)
(gdb) 
954		WARN((mnt->mnt_sb->s_maxbytes < 0), "%s set sb->s_maxbytes to "
(gdb) 
957		mnt->mnt_mountpoint = mnt->mnt_root;
(gdb) p mnt->mnt_root 
$16 = (struct dentry *) 0xc6c02000
(gdb) p *mnt->mnt_root 
$17 = {d_count = {counter = 2}, d_flags = 16, d_lock = {raw_lock = {
      slock = 514}, magic = 3735899821, owner_cpu = 4294967295, 
    owner = 0xffffffff, dep_map = {key = 0xc1e6248c <__key.26347>, 
      class_cache = 0xc1b5db20 <lock_classes+144096>, 
      name = 0xc15ef0b2 "&dentry->d_lock", cpu = 0, 
      ip = 3239351818}}, d_mounted = 0, d_inode = 0xc6c00000, 
  d_hash = {next = 0x0, pprev = 0x0}, d_parent = 0xc6c02000, 
  d_name = {hash = 0, len = 1, name = 0xc6c0207c "/"}, d_lru = {
    next = 0xc6c0204c, prev = 0xc6c0204c}, d_u = {d_child = {
      next = 0xc6c02054, prev = 0xc6c02054}, d_rcu = {
      next = 0xc6c02054, func = 0xc6c02054}}, d_subdirs = {
    next = 0xc6c0205c, prev = 0xc6c0205c}, d_alias = {
    next = 0xc6c00018, prev = 0xc6c00018}, d_time = 1802201963, 
  d_op = 0x0, d_sb = 0xc7008430, 
  d_fsdata = 0xc16ac580 <sysfs_root>, 
  d_iname = "/\000", 'k' <repeats 37 times>, "\245"}
(gdb) n
958		mnt->mnt_parent = mnt;
(gdb) 
959		up_write(&mnt->mnt_sb->s_umount);
(gdb) 
960		free_secdata(secdata);
(gdb) s
free_secdata (secdata=0x0) at include/linux/security.h:3133
3133		free_page((unsigned long)secdata);
(gdb) s
free_pages (addr=addr@entry=0, order=order@entry=0)
    at mm/page_alloc.c:2034
2034		if (addr != 0) {
(gdb) n
2038	}
(gdb) 
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=<optimized out>, 
    data=data@entry=0x0) at fs/super.c:961
961		return mnt;
(gdb) n
971	}
(gdb) 
kern_mount_data (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    data=data@entry=0x0) at fs/super.c:1017
1017	}
(gdb) 
sysfs_init () at fs/sysfs/mount.c:105
105			if (IS_ERR(sysfs_mount)) {
(gdb) 
120	}
(gdb) 
mnt_init () at fs/namespace.c:2304
2304		if (err)

```

# Links：

* [How Is The Root File System Found?](https://kernelnewbies.org/RootFileSystem)



