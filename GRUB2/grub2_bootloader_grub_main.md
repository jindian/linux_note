grub_main
================================

Here we are at the entrance of grub_main, next we will check functions involved in grub_main one by one.

```grub_main
grub-core/kern/main.c:206


/* The main routine.  */
void __attribute__ ((noreturn))
grub_main (void)
{
  /* First of all, initialize the machine.  */
  grub_machine_init ();

  /* Hello.  */
  grub_setcolorstate (GRUB_TERM_COLOR_HIGHLIGHT);
  grub_printf ("Welcome to GRUB!\n\n");
  grub_setcolorstate (GRUB_TERM_COLOR_STANDARD);

  /* Load pre-loaded modules and free the space.  */
  grub_register_exported_symbols ();
#ifdef GRUB_LINKER_HAVE_INIT
  grub_arch_dl_init_linker ();
#endif
  grub_load_modules ();

  /* It is better to set the root device as soon as possible,
     for convenience.  */
  grub_set_prefix_and_root ();
  grub_env_export ("root");
  grub_env_export ("prefix");

  grub_register_core_commands ();

  grub_load_config ();
  grub_load_normal_mode ();
  grub_rescue_run ();
}

```

In grub_machine_init:

1. grub_modbase stores start address of grub modules, remember only part of decompressed image copied back to address start at 0x9000, grub modules not included. 

2. Initialize the console with grub_console_init in which registers console input and output with intialized data struct, the struct initialized with callback functions and other values, detail information checks grub_console_term_input and grub_console_term_output in grub-core/term/i386/pc/console.c

3.  Check mapped memory region with grub_machine_mmap_iterate.

```grub_machine_init
grub-core/kern/i386/pc/init.c:157


void
grub_machine_init (void)
{
  int i;
#if 0
  int grub_lower_mem;
#endif

  grub_modbase = GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR + (_edata - _start);
(gdb) p /x grub_modbase
$2 = 0x106e54

  /* Initialize the console as early as possible.  */
  grub_console_init ();

  /* This sanity check is useless since top of GRUB_MEMORY_MACHINE_RESERVED_END
     is used for stack and if it's unavailable we wouldn't have gotten so far.
   */
#if 0
  grub_lower_mem = grub_get_conv_memsize () << 10;

  /* Sanity check.  */
  if (grub_lower_mem < GRUB_MEMORY_MACHINE_RESERVED_END)
    grub_fatal ("too small memory");
#endif

/* FIXME: This prevents loader/i386/linux.c from using low memory.  When our
   heap implements support for requesting a chunk in low memory, this should
   no longer be a problem.  */
#if 0
  /* Add the lower memory into free memory.  */
  if (grub_lower_mem >= GRUB_MEMORY_MACHINE_RESERVED_END)
    add_mem_region (GRUB_MEMORY_MACHINE_RESERVED_END,
                    grub_lower_mem - GRUB_MEMORY_MACHINE_RESERVED_END);
#endif

  auto int NESTED_FUNC_ATTR hook (grub_uint64_t, grub_uint64_t,
                                  grub_memory_type_t);
  int NESTED_FUNC_ATTR hook (grub_uint64_t addr, grub_uint64_t size,
                             grub_memory_type_t type)
    {
      /* Avoid the lower memory.  */
      if (addr < 0x100000)
        {
          if (size <= 0x100000 - addr)
            return 0;

          size -= 0x100000 - addr;
          addr = 0x100000;
        }

      /* Ignore >4GB.  */
      if (addr <= 0xFFFFFFFF && type == GRUB_MEMORY_AVAILABLE)
        {
          grub_size_t len;

          len = (grub_size_t) ((addr + size > 0xFFFFFFFF)
                 ? 0xFFFFFFFF - addr
                 : size);
          add_mem_region (addr, len);
        }

      return 0;
    }

  grub_machine_mmap_iterate (hook);

  compact_mem_regions ();

  for (i = 0; i < num_regions; i++)
      grub_mm_init_region ((void *) mem_regions[i].addr, mem_regions[i].size);

  grub_tsc_init ();
}

-----------------------------------------------------------------------

grub-core/term/i386/pc/console.c:275

grub_console_init (void)
{
  grub_term_register_output ("console", &grub_console_term_output);
  grub_term_register_input ("console", &grub_console_term_input);
}

-----------------------------------------------------------------------
The context of entry in loop
1. entry->addr: 0x0, entry->len: 0x9fc00, entry->size: 0x14, entry->type: 0x1
2. entry->addr: 0x9fc00, entry->len: 0x400, entry->size: 0x14, entry->type: 0x2
3. entry->addr: 0xf0000, entry->len: 0x10000, entry->size: 0x14, entry->type: 0x2
4. entry->addr: 0x100000, entry->len: 0x7efe000, entry->size: 0x14, entry->type: 0x1
5. entry->addr: 0x7ffe000, entry->len: 0x2000, entry->size: 0x14, entry->type: 0x2
6. entry->addr: 0xfffc0000, entry->len: 0x40000, entry->size: 0x14, entry->type: 0x2
-----------------------------------------------------------------------
grub-core/kern/i386/pc/mmap.c:141

grub_err_t
grub_machine_mmap_iterate (grub_memory_hook_t hook)
{
  grub_uint32_t cont;
  struct grub_machine_mmap_entry *entry
    = (struct grub_machine_mmap_entry *) GRUB_MEMORY_MACHINE_SCRATCH_ADDR;

  grub_memset (entry, 0, sizeof (entry));

  /* Check if grub_get_mmap_entry works.  */
  cont = grub_get_mmap_entry (entry, 0);

  if (entry->size)
    do
      {
        if (hook (entry->addr, entry->len,
                  /* GRUB mmaps have been defined to match with the E820 definition.
                     Therefore, we can just pass type through.  */
                  ((entry->type <= GRUB_MACHINE_MEMORY_BADRAM) && (entry->type >= GRUB_MACHINE_MEMORY_AVAILABLE)) ? entry->type : GRUB_MEMORY_RESERVED))
          break;

        if (! cont)
          break;

        grub_memset (entry, 0, sizeof (entry));

        cont = grub_get_mmap_entry (entry, cont);
      }
    while (entry->size);
  else
    {
      grub_uint32_t eisa_mmap = grub_get_eisa_mmap ();

      if (hook (0x0, ((grub_uint32_t) grub_get_conv_memsize ()) << 10,
                GRUB_MEMORY_AVAILABLE))
        return 0;

      if (eisa_mmap)
        {
          if (hook (0x100000, (eisa_mmap & 0xFFFF) << 10,
                    GRUB_MEMORY_AVAILABLE) == 0)
            hook (0x1000000, eisa_mmap & ~0xFFFF, GRUB_MEMORY_AVAILABLE);
        }
      else
        hook (0x100000, ((grub_uint32_t) grub_get_ext_memsize ()) << 10,
              GRUB_MEMORY_AVAILABLE);
    }

  return 0;
}
```