# **start\_kernel part X**

Continue routine `vfs_caches_init` of start\_kernel part IX

* Involves routine `mnt_init`, which registers `sysfs`, this is needed later for actually finding root device, registers `rootfs`, creates the initial filesystem namespace, with rootfs mounted at `/`

# Links：

* [How Is The Root File System Found?](https://kernelnewbies.org/RootFileSystem)



