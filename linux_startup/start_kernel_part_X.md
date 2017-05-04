# **start\_kernel part X**

Continue routine `vfs_caches_init` of start\_kernel part IX

* Involves routine `mnt_init`, which allocates slab cache for struct `vfsmount`named `mnt_cache`,  gets free pages for `mount_hashtable` and initializes  it, registers `sysfs`, this is needed later for actually finding root device, registers `rootfs`, creates the initial filesystem namespace, with rootfs mounted at `/`

  Let's check more about `sysfs_init`, `init_rootfs`and `init_mount_tree`

  With `sysfs_init` to learn how a new file system mounted to kernel, debug information of the function shown as below

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
920        if (!type)
(gdb) n
924        mnt = alloc_vfsmnt(name);
(gdb) s
alloc_vfsmnt (name=name@entry=0xc1616401 "sysfs")
    at fs/namespace.c:130
130        struct vfsmount *mnt = kmem_cache_zalloc(mnt_cache, GFP_KERNEL);
(gdb) n
131        if (mnt) {
(gdb) 
134            err = mnt_alloc_id(mnt);
(gdb) 
138            if (name) {
(gdb) 
139                mnt->mnt_devname = kstrdup(name, GFP_KERNEL);
(gdb) 
140                if (!mnt->mnt_devname)
(gdb) 
144            atomic_set(&mnt->mnt_count, 1);
(gdb) 
145            INIT_LIST_HEAD(&mnt->mnt_hash);
(gdb) 
146            INIT_LIST_HEAD(&mnt->mnt_child);
(gdb) 
147            INIT_LIST_HEAD(&mnt->mnt_mounts);
(gdb) 
148            INIT_LIST_HEAD(&mnt->mnt_list);
(gdb) 
149            INIT_LIST_HEAD(&mnt->mnt_expire);
(gdb) 
150            INIT_LIST_HEAD(&mnt->mnt_share);
(gdb) 
151            INIT_LIST_HEAD(&mnt->mnt_slave_list);
(gdb) 
152            INIT_LIST_HEAD(&mnt->mnt_slave);
(gdb) 
154            mnt->mnt_writers = alloc_percpu(int);
(gdb) 
155            if (!mnt->mnt_writers)
(gdb) 
172    }
(gdb) 
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=0xc1616401 "sysfs", 
    data=data@entry=0x0) at fs/super.c:925
925        if (!mnt)
(gdb) 
928        if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {
(gdb) 
938        error = type->get_sb(type, flags, name, data, mnt);
(gdb) 
939        if (error < 0)
(gdb) 
941        BUG_ON(!mnt->mnt_sb);
(gdb) 
943         error = security_sb_kern_mount(mnt->mnt_sb, flags, secdata);
(gdb) s
security_sb_kern_mount (sb=0xc7008430, flags=flags@entry=4194304, 
    data=data@entry=0x0) at security/security.c:273
273        return security_ops->sb_kern_mount(sb, flags, data);
(gdb) s
cap_sb_kern_mount (sb=0xc7008430, flags=4194304, data=0x0)
    at security/capability.c:65
65    }
(gdb) n
security_sb_kern_mount (sb=<optimized out>, 
    flags=flags@entry=4194304, data=data@entry=0x0)
    at security/security.c:274
274    }
(gdb) 
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=<optimized out>, 
    data=data@entry=0x0) at fs/super.c:944
944         if (error)
(gdb) 
954        WARN((mnt->mnt_sb->s_maxbytes < 0), "%s set sb->s_maxbytes to "
(gdb) 
957        mnt->mnt_mountpoint = mnt->mnt_root;
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
958        mnt->mnt_parent = mnt;
(gdb) 
959        up_write(&mnt->mnt_sb->s_umount);
(gdb) 
960        free_secdata(secdata);
(gdb) s
free_secdata (secdata=0x0) at include/linux/security.h:3133
3133        free_page((unsigned long)secdata);
(gdb) s
free_pages (addr=addr@entry=0, order=order@entry=0)
    at mm/page_alloc.c:2034
2034        if (addr != 0) {
(gdb) n
2038    }
(gdb) 
vfs_kern_mount (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    flags=flags@entry=4194304, name=<optimized out>, 
    data=data@entry=0x0) at fs/super.c:961
961        return mnt;
(gdb) n
971    }
(gdb) 
kern_mount_data (type=type@entry=0xc16ac520 <sysfs_fs_type>, 
    data=data@entry=0x0) at fs/super.c:1017
1017    }
(gdb) 
sysfs_init () at fs/sysfs/mount.c:105
105            if (IS_ERR(sysfs_mount)) {
(gdb) 
120    }
(gdb) 
mnt_init () at fs/namespace.c:2304
2304        if (err)
```

Debug information of \`init\_rootfs\` shown as follow, it just registers \`rootfs\`

```
2310        init_rootfs();
(gdb) s
init_rootfs () at fs/ramfs/inode.c:311
311        err = bdi_init(&ramfs_backing_dev_info);
(gdb) s
bdi_init (bdi=bdi@entry=0xc16ad4c0 <ramfs_backing_dev_info>)
    at mm/backing-dev.c:655
(gdb) n
655        bdi->dev = NULL;
(gdb) 
657        bdi->min_ratio = 0;
(gdb) 
658        bdi->max_ratio = 100;
(gdb) 
659        bdi->max_prop_frac = PROP_FRAC_BASE;
(gdb) 
660        spin_lock_init(&bdi->wb_lock);
(gdb) 
661        INIT_RCU_HEAD(&bdi->rcu_head);
(gdb) 
662        INIT_LIST_HEAD(&bdi->bdi_list);
(gdb) 
663        INIT_LIST_HEAD(&bdi->wb_list);
(gdb) 
663        INIT_LIST_HEAD(&bdi->wb_list);
(gdb) 
664        INIT_LIST_HEAD(&bdi->work_list);
(gdb) 
666        bdi_wb_init(&bdi->wb, bdi);
(gdb) 
666        bdi_wb_init(&bdi->wb, bdi);
(gdb) 
671        bdi->wb_mask = 1;
(gdb) 
672        bdi->wb_cnt = 1;
(gdb) 
675            err = percpu_counter_init(&bdi->bdi_stat[i], 0);
(gdb) 
676            if (err)
(gdb) 
680        bdi->dirty_exceeded = 0;
(gdb) 
681        err = prop_local_init_percpu(&bdi->completions);
(gdb) 
683        if (err) {
(gdb) 
685            while (i--)
(gdb) 
690    }
(gdb) 
init_rootfs () at fs/ramfs/inode.c:312
312        if (err)
(gdb) 
315        err = register_filesystem(&rootfs_fs_type);
(gdb) s
register_filesystem (fs=fs@entry=0xc16ad400 <rootfs_fs_type>)
    at fs/filesystems.c:74
74        BUG_ON(strchr(fs->name, '.'));
(gdb) n
75        if (fs->next)
(gdb) 
77        INIT_LIST_HEAD(&fs->fs_supers);
(gdb) 
81            res = -EBUSY;
(gdb) 
77        INIT_LIST_HEAD(&fs->fs_supers);
(gdb) 
78        write_lock(&file_systems_lock);
(gdb) 
79        p = find_filesystem(fs->name, strlen(fs->name));
(gdb) 
80        if (*p)
(gdb) 
83            *p = fs;
(gdb) 
71        int res = 0;
(gdb) p file_systems
$18 = (struct file_system_type *) 0xc16ac520 <sysfs_fs_type>
(gdb) p file_systems->next
$19 = (struct file_system_type *) 0xc16ad400 <rootfs_fs_type>
(gdb) p file_systems->next->next
$20 = (struct file_system_type *) 0x0
(gdb) n
84        write_unlock(&file_systems_lock);
(gdb) 
85        return res;
(gdb) 
86    }
(gdb) 
init_rootfs () at fs/ramfs/inode.c:316
316        if (err)
(gdb) 
320    }
(gdb) 
mnt_init () at fs/namespace.c:2311
2311        init_mount_tree();
```

Debug information of  \`init\_mount\_tree\` shown as follow

```
mnt_init () at fs/namespace.c:2311
2311        init_mount_tree();
(gdb) s
init_mount_tree () at fs/namespace.c:2266
2266        mnt = do_kern_mount("rootfs", 0, "rootfs", NULL);
(gdb) s
do_kern_mount (fstype=fstype@entry=0xc15ef45c "rootfs", 
    flags=flags@entry=0, name=name@entry=0xc15ef45c "rootfs", 
    data=data@entry=0x0) at fs/super.c:1000
1000    {
(gdb) n
1001        struct file_system_type *type = get_fs_type(fstype);
(gdb) 
1003        if (!type)
(gdb) 
1005        mnt = vfs_kern_mount(type, flags, name, data);
(gdb) p type
$21 = (struct file_system_type *) 0xc16ad400 <rootfs_fs_type>
(gdb) n
1006        if (!IS_ERR(mnt) && (type->fs_flags & FS_HAS_SUBTYPE) &&
(gdb) 
1009        put_filesystem(type);
(gdb) s
put_filesystem (fs=fs@entry=0xc16ad400 <rootfs_fs_type>)
    at fs/filesystems.c:43
43        module_put(fs->owner);
(gdb) s
module_put (module=0x0) at kernel/module.c:951
951        if (module) {
(gdb) n
961    }
(gdb) 
put_filesystem (fs=fs@entry=0xc16ad400 <rootfs_fs_type>)
    at fs/filesystems.c:44
44    }
(gdb) 
do_kern_mount (fstype=fstype@entry=0xc15ef45c "rootfs", 
    flags=flags@entry=0, name=name@entry=0xc15ef45c "rootfs", 
    data=data@entry=0x0) at fs/super.c:1010
1010        return mnt;
(gdb) 
1011    }
(gdb) 
init_mount_tree () at fs/namespace.c:2267
2267        if (IS_ERR(mnt))
(gdb) 
2269        ns = create_mnt_ns(mnt);
(gdb) s
create_mnt_ns (mnt=0xc701d0c0) at fs/namespace.c:2079
2079    {
(gdb) n
2082        new_ns = alloc_mnt_ns();
(gdb) s
alloc_mnt_ns () at fs/namespace.c:1986
1986        new_ns = kmalloc(sizeof(struct mnt_namespace), GFP_KERNEL);
(gdb) n
1987        if (!new_ns)
(gdb) 
1989        atomic_set(&new_ns->count, 1);
(gdb) 
1990        new_ns->root = NULL;
(gdb) 
1991        INIT_LIST_HEAD(&new_ns->list);
(gdb) 
1992        init_waitqueue_head(&new_ns->poll);
(gdb) s
__init_waitqueue_head (q=q@entry=0xc7001240, 
    key=key@entry=0xc1e6266c <__key.25018>) at kernel/wait.c:15
15        spin_lock_init(&q->lock);
(gdb) n
16        lockdep_set_class(&q->lock, key);
(gdb) s
lockdep_init_map (lock=lock@entry=0xc7001250, 
    name=name@entry=0xc16237be "key", 
    key=key@entry=0xc1e6266c <__key.25018>, 
    subclass=subclass@entry=0) at kernel/lockdep.c:2678
2678        lock->class_cache = NULL;
(gdb) n
2680        lock->cpu = raw_smp_processor_id();
(gdb) 
2683        if (DEBUG_LOCKS_WARN_ON(!name)) {
(gdb) 
2688        lock->name = name;
(gdb) 
2690        if (DEBUG_LOCKS_WARN_ON(!key))
(gdb) 
2695        if (!static_obj(key)) {
(gdb) 
2700        lock->key = key;
(gdb) 
2702        if (unlikely(!debug_locks))
(gdb) 
2705        if (subclass)
(gdb) 
2707    }
(gdb) 
__init_waitqueue_head (q=q@entry=0xc7001240, 
    key=key@entry=0xc1e6266c <__key.25018>) at kernel/wait.c:17
17        INIT_LIST_HEAD(&q->task_list);
(gdb) 
18    }
(gdb) 
alloc_mnt_ns () at fs/namespace.c:1993
1993        new_ns->event = 0;
(gdb) 
1994        return new_ns;
(gdb) 
1995    }
(gdb) 
create_mnt_ns (mnt=0xc701d0c0) at fs/namespace.c:2083
2083        if (!IS_ERR(new_ns)) {
(gdb) 
2084            mnt->mnt_ns = new_ns;
(gdb) 
2085            new_ns->root = mnt;
(gdb) 
2086            list_add(&new_ns->list, &new_ns->root->mnt_list);
(gdb) 
2089    }
(gdb) 
init_mount_tree () at fs/namespace.c:2270
2270        if (IS_ERR(ns))
(gdb) 
2273        init_task.nsproxy->mnt_ns = ns;
(gdb) p ns
$22 = (struct mnt_namespace *) 0xc7001230
(gdb) p *ns
$23 = {count = {counter = 1}, root = 0xc701d0c0, list = {
    next = 0xc701d0f0, prev = 0xc701d0f0}, poll = {lock = {
      raw_lock = {slock = 0}, magic = 3735899821, 
      owner_cpu = 4294967295, owner = 0xffffffff, dep_map = {
        key = 0xc1e6266c <__key.25018>, class_cache = 0x0, 
        name = 0xc16237be "key", cpu = 0, ip = 1802201963}}, 
    task_list = {next = 0xc7001264, prev = 0xc7001264}}, event = 0}
(gdb) n
2274        get_mnt_ns(ns);
(gdb) 
2276        root.mnt = ns->root;
(gdb) 
2279        set_fs_pwd(current->fs, &root);
(gdb) s
get_current ()
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/current.h:14
14        return percpu_read_stable(current_task);
(gdb) n
init_mount_tree () at fs/namespace.c:2279
2279        set_fs_pwd(current->fs, &root);
(gdb) s
set_fs_pwd (fs=0xc16aa620 <init_fs>, 
    path=path@entry=0xc168bf88 <init_thread_union+8072>)
    at fs/fs_struct.c:33
33        write_lock(&fs->lock);
(gdb) n
34        old_pwd = fs->pwd;
(gdb) p fs->pwd
$24 = {mnt = 0x0, dentry = 0x0}
(gdb) n
35        fs->pwd = *path;
(gdb) p path
$25 = (struct path *) 0xc168bf88 <init_thread_union+8072>
(gdb) n
36        path_get(path);
(gdb) p fs->pwd
$26 = {mnt = 0xc701d0c0, dentry = 0x0}
(gdb) s
path_get (path=path@entry=0xc168bf88 <init_thread_union+8072>)
    at fs/namei.c:363
363        mntget(path->mnt);
(gdb) p path->mnt 
$27 = (struct vfsmount *) 0xc701d0c0
(gdb) p path->mnt->mnt_count
$28 = {counter = 1}
(gdb) n
364        dget(path->dentry);
(gdb) p path->mnt->mnt_count
$29 = {counter = 2}
(gdb) s
dget (dentry=0xc6c020d0) at include/linux/dcache.h:335
335        if (dentry) {
(gdb) n
336            BUG_ON(!atomic_read(&dentry->d_count));
(gdb) 
337            atomic_inc(&dentry->d_count);
(gdb) 
path_get (path=path@entry=0xc168bf88 <init_thread_union+8072>)
    at fs/namei.c:365
365    }
(gdb) 
set_fs_pwd (fs=0xc16aa620 <init_fs>, 
    path=path@entry=0xc168bf88 <init_thread_union+8072>)
    at fs/fs_struct.c:37
37        write_unlock(&fs->lock);
(gdb) 
39        if (old_pwd.dentry)
(gdb) p old_pwd.dentry 
$30 = (struct dentry *) 0x0
(gdb) n
41    }
(gdb) 
init_mount_tree () at fs/namespace.c:2280
2280        set_fs_root(current->fs, &root);
(gdb) s
set_fs_root (fs=0xc16aa620 <init_fs>, 
    path=path@entry=0xc168bf88 <init_thread_union+8072>)
    at fs/fs_struct.c:16
16        write_lock(&fs->lock);
(gdb) n
17        old_root = fs->root;
(gdb) 
18        fs->root = *path;
(gdb) p fs->root 
$31 = {mnt = 0x0, dentry = 0x0}
(gdb) n
19        path_get(path);
(gdb) p fs->root 
$32 = {mnt = 0xc701d0c0, dentry = 0x0}
(gdb) n
20        write_unlock(&fs->lock);
(gdb) 
21        if (old_root.dentry)
(gdb) 
23    }
(gdb) 
mnt_init () at fs/namespace.c:2312
2312    }
(gdb)
```

* Involves routine `bdev_cache_init` to initialize block device. Inside `bdev_cache_init`, it creates slab cache for struct `bdev_inode`, registers file system type `bd_type` and mounts the new regisgered file system, with routine `kmemleak_not_leak` to tell kmemleak not to report it, finnaly initializes block device super block data.
* Involves routine chrdev\_init to initialize charactor device.

```
Breakpoint 2, chrdev_init () at fs/char_dev.c:569
569    {
(gdb) n
570        cdev_map = kobj_map_init(base_probe, &chrdevs_lock);
(gdb) s
kobj_map_init (base_probe=base_probe@entry=0xc1116760 <base_probe>, 
    lock=lock@entry=0xc16a7a20 <chrdevs_lock>)
    at drivers/base/map.c:137
137    {
(gdb) n
138        struct kobj_map *p = kmalloc(sizeof(struct kobj_map), GFP_KERNEL);
(gdb) 
139        struct probe *base = kzalloc(sizeof(*base), GFP_KERNEL);
(gdb) 
142        if ((p == NULL) || (base == NULL)) {
(gdb) 
148        base->dev = 1;
(gdb) 
149        base->range = ~0;
(gdb) 
150        base->get = base_probe;
(gdb) 
151        for (i = 0; i < 255; i++)
(gdb) 
152            p->probes[i] = base;
(gdb) break if i==254
Breakpoint 3 at 0xc12f5218: file drivers/base/map.c, line 152.
(gdb) c
Continuing.

Breakpoint 3, kobj_map_init (
    base_probe=base_probe@entry=0xc1116760 <base_probe>, 
    lock=lock@entry=0xc16a7a20 <chrdevs_lock>)
    at drivers/base/map.c:152
152            p->probes[i] = base;
(gdb) p i
$1 = 254
(gdb) n
151        for (i = 0; i < 255; i++)
(gdb) 
153        p->lock = lock;
(gdb) 
154        return p;
(gdb) 
155    }
(gdb) 
chrdev_init () at fs/char_dev.c:571
571        bdi_init(&directly_mappable_cdev_bdi);
chrdev_init () at fs/char_dev.c:572
572    }
(gdb) 
vfs_caches_init (mempages=27097) at fs/dcache.c:2339
2339    }
(gdb) 
start_kernel () at init/main.c:674
674        radix_tree_init();
```

## _initialize radix tree_

`radix_tree_init` allocates slab cache for struct `radix_tree_node`, initializes max index of radix tree, registers notifier `radix_tree_callback` when cpu hot plugin.

## _initialize signals_

`signals_init` just allocates slab cache for struct `sigqueue`.

## _initialize page writeback_

The linux kernel implements a primary disk cache called the page cache. The goal of this cache is to minimize disk I/O by storing in physical memory data that would otherwise be accessed from disk.

Disk caches are beneficial for two reasons. First, disk access is magnitudes slower than memory access. Accessing data from memory rather than the disk is much faster. Second, data accessed once will, with a high likelihood, find itself accessed again in the near future. This principle, that access to a particular piece of data tends to be clustered in time, is called temporal localityz.Temporal locality ensures that if data is cached on its first access, there is a high probability of a cache hit \(access to data that is in the cache\) in the near future.

The page cache consists of physical pages in RAM. Each page in the cache corresponds to multiple blocks on the disk. Whenever the kernel begins a page I/O operation \(a disk operation in page-size chunks, usually to a regular file\), it first checks whether the requisite data is in the page cache. If it is, the kernel can forego accessing the disk and use the data straight from the page cache.

Individual disk blocks can also tie into the page cache, by way of block I/O buffers. A buffer is the in-memory representation of a single physical disk block. Buffers act as descriptors that map pages in memory to disk blocks; thus, the page cache also reduces disk access during block I/O operations by both caching disk blocks and buffering block I/O operations until later. This caching is often referred to as the "buffer cache," although in reality it is not a separate cache and is part of the page cache.

`page_writeback_init` is called early on to tune the page writeback dirty limits.

* Involves `writeback_set_ratelimit` to set `ratelimit_pages`,  If `ratelimit_pages` is too high then we can get into dirty-data overload if a large number of processes all perform writes at the same time. If it is too low then SMP machines will call the \(expensive\) `get_writeback_state` too often
* Register notifier when cpu going up/down
* Calculate the couple the period to the dirty\_ratio

* Initialzes prop\_descriptor `vm_completions` and `vm_dirties`

```
page_writeback_init () at mm/page-writeback.c:801
801        shift = calc_period_shift();
(gdb) 
802        prop_descriptor_init(&vm_completions, shift);
(gdb) p shift
$8 = 14
(gdb) s
prop_descriptor_init (pd=pd@entry=0xc1e5c820 <vm_completions>, 
    shift=shift@entry=14) at lib/proportions.c:83
83        pd->index = 0;
(gdb) n
84        pd->pg[0].shift = shift;
(gdb) 
85        mutex_init(&pd->mutex);
(gdb) 
86        err = percpu_counter_init(&pd->pg[0].events, 0);
(gdb) 
87        if (err)
(gdb) 
90        err = percpu_counter_init(&pd->pg[1].events, 0);
(gdb) 
91        if (err)
(gdb) 
96    }
page_writeback_init () at mm/page-writeback.c:803
803        prop_descriptor_init(&vm_dirties, shift);
(gdb) s
prop_descriptor_init (pd=pd@entry=0xc1e5c740 <vm_dirties>, 
    shift=shift@entry=14) at lib/proportions.c:83
83        pd->index = 0;
(gdb) n
84        pd->pg[0].shift = shift;
(gdb) 
85        mutex_init(&pd->mutex);
(gdb) 
86        err = percpu_counter_init(&pd->pg[0].events, 0);
(gdb) 
87        if (err)
(gdb) 
90        err = percpu_counter_init(&pd->pg[1].events, 0);
(gdb) 
91        if (err)
(gdb) 
96    }
(gdb) 
page_writeback_init () at mm/page-writeback.c:804
804    }
(gdb) 
start_kernel () at init/main.c:679
679        proc_root_init();
(gdb)
```

## _initialize proc file system_

/proc is very special in that it is also a virtual filesystem. It's sometimes referred to as a process information pseudo-file system. It doesn't contain 'real' files but runtime system information \(e.g. system memory, devices mounted, hardware configuration, etc\). For this reason it can be regarded as a control and information centre for the kernel. In fact, quite a lot of system utilities are simply calls to files in this directory. For example, 'lsmod' is the same as 'cat /proc/modules' while 'lspci' is a synonym for 'cat /proc/pci'. By altering files located in this directory you can even read/change kernel parameters \(sysctl\) while the system is running.

`proc_root_init`

* Create slab caches for struct `proc_inode` named `proc_inode_cache`

```
Breakpoint 2, proc_root_init () at fs/proc/root.c:105
105    {
(gdb) n
108        proc_init_inodecache();
(gdb) s
proc_init_inodecache () at fs/proc/inode.c:105
105        proc_inode_cachep = kmem_cache_create("proc_inode_cache",
(gdb) n
110    }
(gdb) 
proc_root_init () at fs/proc/root.c:109
```

* Registers proc filesystem and checks the result

```
109        err = register_filesystem(&proc_fs_type);
(gdb) p proc_fs_type 
$1 = {name = 0xc15dbf10 "proc", fs_flags = 0, 
  get_sb = 0xc115e4f0 <proc_get_sb>, kill_sb = 0xc115e4c0 <proc_kill_sb>, 
  owner = 0x0, next = 0x0, fs_supers = {next = 0x0, prev = 0x0}, s_lock_key = {
    subkeys = {{__one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, s_umount_key = {
    subkeys = {{__one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, i_lock_key = {
    subkeys = {{__one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, i_mutex_key = {
    subkeys = {{__one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, i_mutex_dir_key = {
    subkeys = {{__one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}, i_alloc_sem_key = {
---Type <return> to continue, or q <return> to quit---
    subkeys = {{__one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}, {
        __one_byte = 0 '\000'}, {__one_byte = 0 '\000'}}}}
(gdb) n
110        if (err)
```

* Mounts proc filesystem and checks the result

```
(gdb) 
112        proc_mnt = kern_mount_data(&proc_fs_type, &init_pid_ns);
(gdb) 
114        if (IS_ERR(proc_mnt)) {
```

* Create symlink `mounts` under `proc_root` for the real destination directory `self/mounts`

```
119		proc_symlink("mounts", NULL, "self/mounts");
(gdb) s
proc_symlink (name=name@entry=0xc15f0aec "mounts", parent=parent@entry=0x0, 
    dest=dest@entry=0xc15f0ae7 "self/mounts") at fs/proc/generic.c:643
643	{
(gdb) n
646		ent = __proc_create(&parent, name,
(gdb) s
__proc_create (parent=parent@entry=0xc168bfa8 <init_thread_union+8104>, 
    name=name@entry=0xc15f0aec "mounts", mode=mode@entry=41471, 
    nlink=nlink@entry=1) at fs/proc/generic.c:606
606	{
(gdb) n
608		const char *fn = name;
(gdb) 
612		if (!name || !strlen(name)) goto out;
(gdb) 
614		if (xlate_proc_name(name, parent, &fn) != 0)
(gdb) s
xlate_proc_name (name=name@entry=0xc15f0aec "mounts", 
    ret=ret@entry=0xc168bfa8 <init_thread_union+8104>, 
    residual=residual@entry=0xc168bf8c <init_thread_union+8076>)
    at fs/proc/generic.c:302
302		de = *ret;
(gdb) n
304			de = &proc_root;
(gdb) 
306		spin_lock(&proc_subdir_lock);
(gdb) 
308			next = strchr(cp, '/');
(gdb) 
309			if (!next)
(gdb) 
323		*residual = cp;
(gdb) 
324		*ret = de;
(gdb) 
326		spin_unlock(&proc_subdir_lock);
(gdb) 
328	}
(gdb) p residual 
$2 = (const char **) 0xc168bf8c <init_thread_union+8076>
(gdb) p *residual 
$3 = 0xc15f0aec "mounts"
(gdb) p ret
$4 = (struct proc_dir_entry **) 0xc168bfa8 <init_thread_union+8104>
(gdb) p *ret
$5 = (struct proc_dir_entry *) 0xc16abe20 <proc_root>
(gdb) n
__proc_create (parent=parent@entry=0xc168bfa8 <init_thread_union+8104>, 
    name=name@entry=0xc15f0aec "mounts", mode=mode@entry=41471, 
    nlink=nlink@entry=1) at fs/proc/generic.c:618
618		if (strchr(fn, '/'))
(gdb) 
621		len = strlen(fn);
(gdb) 
623		ent = kmalloc(sizeof(struct proc_dir_entry) + len + 1, GFP_KERNEL);
(gdb) p len
$6 = 6
(gdb) n
624		if (!ent) goto out;
(gdb) 
626		memset(ent, 0, sizeof(struct proc_dir_entry));
(gdb) 
627		memcpy(((char *) ent) + sizeof(struct proc_dir_entry), fn, len + 1);
(gdb) 
628		ent->name = ((char *) ent) + sizeof(*ent);
(gdb) 
629		ent->namelen = len;
(gdb) 
630		ent->mode = mode;
(gdb) 
631		ent->nlink = nlink;
(gdb) 
632		atomic_set(&ent->count, 1);
(gdb) 
633		ent->pde_users = 0;
(gdb) 
634		spin_lock_init(&ent->pde_unload_lock);
(gdb) 
635		ent->pde_unload_completion = NULL;
(gdb) 
636		INIT_LIST_HEAD(&ent->pde_openers);
(gdb) 
639	}
(gdb) p ent
$7 = (struct proc_dir_entry *) 0xc701f420
(gdb) p *ent
$8 = {low_ino = 0, namelen = 6, name = 0xc701f498 "mounts", mode = 41471, 
  nlink = 1, uid = 0, gid = 0, size = 0, proc_iops = 0x0, proc_fops = 0x0, 
  next = 0x0, parent = 0x0, subdir = 0x0, data = 0x0, read_proc = 0x0, 
  write_proc = 0x0, count = {counter = 1}, pde_users = 0, pde_unload_lock = {
    raw_lock = {slock = 0}, magic = 3735899821, owner_cpu = 4294967295, 
    owner = 0xffffffff, dep_map = {key = 0xc1e62b00 <__key.14684>, 
      class_cache = 0x0, name = 0xc15f0d6f "&ent->pde_unload_lock", cpu = 0, 
      ip = 0}}, pde_unload_completion = 0x0, pde_openers = {next = 0xc701f490, 
    prev = 0xc701f490}}
(gdb) n
639	}
(gdb) 
proc_symlink (name=name@entry=0xc15f0aec "mounts", 
    parent=0xc16abe20 <proc_root>, parent@entry=0x0, 
    dest=dest@entry=0xc15f0ae7 "self/mounts") at fs/proc/generic.c:649
649		if (ent) {
(gdb) 
650			ent->data = kmalloc((ent->size=strlen(dest))+1, GFP_KERNEL);
(gdb) 
651			if (ent->data) {
(gdb) 
652				strcpy((char*)ent->data,dest);
(gdb) 
653				if (proc_register(parent, ent) < 0) {
(gdb) s
proc_register (dir=0xc16abe20 <proc_root>, dp=dp@entry=0xc701f420)
    at fs/proc/generic.c:560
560	{
(gdb) n
564		i = get_inode_number();
(gdb) 
569		if (S_ISDIR(dp->mode)) {
(gdb) 
575		} else if (S_ISLNK(dp->mode)) {
(gdb) 
576			if (dp->proc_iops == NULL)
(gdb) 
577				dp->proc_iops = &proc_link_inode_operations;
(gdb) 
585		spin_lock(&proc_subdir_lock);
(gdb) 
587		for (tmp = dir->subdir; tmp; tmp = tmp->next)
(gdb) 
594		dp->next = dir->subdir;
(gdb) 
595		dp->parent = dir;
(gdb) 
596		dir->subdir = dp;
(gdb) 
597		spin_unlock(&proc_subdir_lock);
(gdb) 
599		return 0;
(gdb) 
600	}
(gdb) 
proc_symlink (name=name@entry=0xc15f0aec "mounts", 
    parent=0xc16abe20 <proc_root>, parent@entry=0x0, 
    dest=dest@entry=0xc15f0ae7 "self/mounts") at fs/proc/generic.c:664
664	}
(gdb) 

```



# Links：

* [How Is The Root File System Found?](https://kernelnewbies.org/RootFileSystem)
* [The Page Cache and Page Writeback](http://www.makelinux.net/books/lkd2/ch15)
* [Linux Page Cache Basics](https://www.thomas-krenn.com/en/wiki/Linux_Page_Cache_Basics)
* [/proc](http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html)



