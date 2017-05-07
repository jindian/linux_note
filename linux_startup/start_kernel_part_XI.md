# start\_kernel part XI

### _**initialize cgoup**_

[Control Groups](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt) provide a mechanism for aggregating/partitioning sets of tasks, and all their future children, into hierarchical groups with specialized behaviour.

`cgroup_init`

* initializes `cgroup_backing_devinfo`\_, \_the routine `bdi_init` already introduced in `mount_init`

```
start_kernel () at init/main.c:681
681        cgroup_init();
(gdb) s
cgroup_init () at kernel/cgroup.c:3267
3267        err = bdi_init(&cgroup_backing_dev_info);
(gdb) n
3268        if (err)
```

* initializes any cgroup subsystems that don't need early init, all configured cgroup subsystems are defined in linux/cgroup\_subsys.h.

```
(gdb) 
3272            struct cgroup_subsys *ss = subsys[i];
(gdb) p subsys
$1 = {0xc16a1500 <cpuset_subsys>, 0xc1697d80 <cpuacct_subsys>, 
  0xc16a0c60 <freezer_subsys>}
(gdb) n
3273            if (!ss->early_init)
(gdb) p subsys[i]->early_init 
$4 = 1
(gdb) p subsys[i]->use_id 
$5 = false
(gdb) n
3275            if (ss->use_id)
(gdb) 
3271        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3272            struct cgroup_subsys *ss = subsys[i];
(gdb) 
3273            if (!ss->early_init)
(gdb) p subsys[i]->early_init 
$6 = 0
(gdb) p subsys[i]->use_id 
$7 = false
(gdb) n
3274                cgroup_init_subsys(ss);
(gdb) s
cgroup_init_subsys (ss=ss@entry=0xc1697d80 <cpuacct_subsys>)
    at kernel/cgroup.c:3180
3180        printk(KERN_INFO "Initializing cgroup subsys %s\n", ss->name);
(gdb) n
3183        list_add(&ss->sibling, &rootnode.subsys_list);
(gdb) 
3184        ss->root = &rootnode;
(gdb) n
3185        css = ss->create(ss, dummytop);
(gdb) s
cpuacct_create (ss=0xc1697d80 <cpuacct_subsys>, cgrp=0xc1e30b78 <rootnode+24>)
    at kernel/sched.c:10842
10842        struct cpuacct *ca = kzalloc(sizeof(*ca), GFP_KERNEL);
(gdb) n
10845        if (!ca)
(gdb) 
10848        ca->cpuusage = alloc_percpu(u64);
(gdb) 
10849        if (!ca->cpuusage)
(gdb) 
10853            if (percpu_counter_init(&ca->cpustat[i], 0))
(gdb) 
10856        if (cgrp->parent)
(gdb) 
10859        return &ca->css;
(gdb) 
10869    }
(gdb) 
cgroup_init_subsys (ss=ss@entry=0xc1697d80 <cpuacct_subsys>)
    at kernel/cgroup.c:3187
3187        BUG_ON(IS_ERR(css));
(gdb) 
3188        init_cgroup_css(css, ss, dummytop);
(gdb) 
3194        init_css_set.subsys[ss->subsys_id] = dummytop->subsys[ss->subsys_id];
(gdb) 
3196        need_forkexit_callback |= ss->fork || ss->exit;
(gdb) 
3201        BUG_ON(!list_empty(&init_task.tasks));
(gdb) 
3203        mutex_init(&ss->hierarchy_mutex);
(gdb) 
3204        lockdep_set_class(&ss->hierarchy_mutex, &ss->subsys_key);
(gdb) 
3205        ss->active = 1;
(gdb) 
3206    }
(gdb) 
cgroup_init () at kernel/cgroup.c:3275
3275            if (ss->use_id)
(gdb) 
3271        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
(gdb) 
3272            struct cgroup_subsys *ss = subsys[i];
(gdb) 
3273            if (!ss->early_init)
(gdb) p subsys[i]->early_init 
$8 = 0
(gdb) p subsys[i]->use_id 
$9 = false
(gdb) n
3274                cgroup_init_subsys(ss);
(gdb) s
cgroup_init_subsys (ss=ss@entry=0xc16a0c60 <freezer_subsys>)
    at kernel/cgroup.c:3180
3180        printk(KERN_INFO "Initializing cgroup subsys %s\n", ss->name);
(gdb) n
3183        list_add(&ss->sibling, &rootnode.subsys_list);
(gdb) 
3184        ss->root = &rootnode;
(gdb) 
3185        css = ss->create(ss, dummytop);
(gdb) s
freezer_create (ss=0xc16a0c60 <freezer_subsys>, 
    cgroup=0xc1e30b78 <rootnode+24>) at kernel/cgroup_freezer.c:136
136        freezer = kzalloc(sizeof(struct freezer), GFP_KERNEL);
(gdb) n
137        if (!freezer)
(gdb) 
140        spin_lock_init(&freezer->lock);
(gdb) 
141        freezer->state = CGROUP_THAWED;
(gdb) 
142        return &freezer->css;
(gdb) 
143    }
(gdb) 
cgroup_init_subsys (ss=ss@entry=0xc16a0c60 <freezer_subsys>)
    at kernel/cgroup.c:3187
3187        BUG_ON(IS_ERR(css));
(gdb) 
3188        init_cgroup_css(css, ss, dummytop);
(gdb) 
3194        init_css_set.subsys[ss->subsys_id] = dummytop->subsys[ss->subsys_id];
(gdb) 
3196        need_forkexit_callback |= ss->fork || ss->exit;
(gdb) 
3201        BUG_ON(!list_empty(&init_task.tasks));
(gdb) 
3203        mutex_init(&ss->hierarchy_mutex);
(gdb) 
3204        lockdep_set_class(&ss->hierarchy_mutex, &ss->subsys_key);
(gdb) 
3205        ss->active = 1;
(gdb) 
3206    }
(gdb) 
cgroup_init () at kernel/cgroup.c:3275
3275            if (ss->use_id)
(gdb) 
3271        for (i = 0; i < CGROUP_SUBSYS_COUNT; i++) {
```

* Add `init_css_set` to the hash table

```
3280        hhead = css_set_hash(init_css_set.subsys);
(gdb) p init_css_set.subsys
$10 = {0xc16a14a0 <top_cpuset>, 0xc701b780, 0xc7001150}
(gdb) s
css_set_hash (css=<optimized out>) at kernel/cgroup.c:262
262            tmp += (unsigned long)css[i];
(gdb) n
263        tmp = (tmp >> 16) ^ tmp;
(gdb) 
265        index = hash_long(tmp, CSS_SET_HASH_BITS);
(gdb) 
cgroup_init () at kernel/cgroup.c:3281
3281        hlist_add_head(&init_css_set.hlist, hhead);
(gdb) 
3282        BUG_ON(!init_root_id(&rootnode));
(gdb) p init_css_set.subsys[0]
$12 = (struct cgroup_subsys_state *) 0xc16a14a0 <top_cpuset>
(gdb) p init_css_set.subsys[1]
$13 = (struct cgroup_subsys_state *) 0xc701b780
(gdb) p init_css_set.subsys[2]
$14 = (struct cgroup_subsys_state *) 0xc7001150
(gdb) p css_set_table 
$15 = {{first = 0x0} <repeats 46 times>, {
    first = 0xc1e30b24 <init_css_set+4>}, {first = 0x0} <repeats 81 times>}
```

* Registers cgroup filesystem

```
(gdb) 
3283        err = register_filesystem(&cgroup_fs_type);
(gdb) 
3284        if (err < 0)
```

* Creates /proc/cgroups

```
3287        proc_create("cgroups", 0, NULL, &proc_cgroupstats_operations);
(gdb) 
3290        if (err)
(gdb) 
3294    }
```

### _**initialize cpusets**_

Cpusets provide a mechanism for assigning a set of CPUs and Memory Nodes to a set of tasks.

Cpusets constrain the CPU and Memory placement of tasks to only the resources within a tasks current cpuset.  They form a nested hierarchy visible in a virtual file system.  These are the essential hooks, beyond what is already present, required to manage dynamic job placement on large systems.

Each task has a pointer to a cpuset.  Multiple tasks may reference the same cpuset.  Requests by a task, using the sched\_setaffinity\(2\) system call to include CPUs in its CPU affinity mask, and using the mbind\(2\) and set\_mempolicy\(2\) system calls to include Memory Nodes in its memory policy, are both filtered through that tasks cpuset, filtering out any CPUs or Memory Nodes not in that cpuset.  The scheduler will not schedule a task on a CPU that is not allowed in its cpus\_allowed vector, and the kernel page allocator will not allocate a page on a node that is not allowed in the requesting tasks mems\_allowed vector.

If a cpuset is cpu or mem exclusive, no other cpuset, other than a direct ancestor or descendent, may share any of the same CPUs or Memory Nodes.

User level code may create and destroy cpusets by name in the cpuset virtual file system, manage the attributes and permissions of these cpusets and which CPUs and Memory Nodes are assigned to each cpuset, specify and query to which cpuset a task is assigned, and list the task pids assigned to a cpuset.

## Links

* [CGROUPS](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [CPUSETS](https://lwn.net/Articles/127936/)


