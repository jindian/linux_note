# start_kernel part III

Continue `setup_arch` of x86 architecture.

  5. `early_ioremap_init` initialize early ioremap for early initialization code before normal mapping functions are ready.
   
   transfers fixed bitmap physical address to virtual address and put it into `slot_virt`, the result of `slot_virt` after this procedure completed: `{0xffd00000, 0xffd40000, 0xffd80000, 0xffdc0000}`, length of the virtual address put into `slot_virt` is 1M.
   
   
   
   
  

# Links
  * [control register CR3](https://en.wikipedia.org/wiki/Control_register#CR3)