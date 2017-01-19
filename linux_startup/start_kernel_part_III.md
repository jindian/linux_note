# start_kernel part III

Continue `setup_arch` of x86 architecture.

  5. `early_ioremap_init` initialize early ioremap for early initialization code before normal mapping functions are ready.
   
   transfer fixed bitmap physical address to virtual address and put it into `slot_virt`, the result of `slot_virt` after this procedure completed: `{0xffd00000, 0xffd40000, 0xffd80000, 0xffdc0000}`, length of the virtual address put into `slot_virt` is 1M.
   
   get pmd of the start address of fixed bitmap with `early_ioremap_pmd`.
   
   `bm_pte` is the entry of early ioremap page table, after initializing it's set as the first page table of the pmd.
   
   check if start address and end address of fixed bitmap are in same pmd, the result is yes.
   
   
   
   
   
   
  

# Links
  * [control register CR3](https://en.wikipedia.org/wiki/Control_register#CR3)
  * [Using I/O Memory](http://www.makelinux.net/ldd3/?u=chp-9-sect-4)