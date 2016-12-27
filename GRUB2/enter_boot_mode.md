Enter Boot Mode
=============================================================================================================

The entry of boot mode is grub_cmd_boot, after end user selected an operating system, grub executes the menu entry and finally involves grub_command_execute with module name 'boot' as input parameter. grub_cmd_boot function as follow, it's very simple just involves grub_loader_boot.

```grub_cmd_boot
/* boot */
static grub_err_t
grub_cmd_boot (struct grub_command *cmd __attribute__ ((unused)),
                    int argc __attribute__ ((unused)),
                    char *argv[] __attribute__ ((unused)))
{
  return grub_loader_boot ();
}
```

Call context of boot mode in initialization:

```boot_mode_context
grub_cmd_boot
    |--grub_loader_boot
        |--(grub_loader_boot_func) () -> grub_linux_boot     //assigned with grub_linux_boot when linux mode initialization
```

```grub_loader_boot
grub-core/commands/boot.c:139

grub_err_t
grub_loader_boot (void)
{
  grub_err_t err = GRUB_ERR_NONE;
  struct grub_preboot *cur;

  if (! grub_loader_loaded)
(gdb) p grub_loader_loaded 
$69 = 1
    return grub_error (GRUB_ERR_NO_KERNEL,
                       N_("you need to load the kernel first"));

  if (grub_loader_flags & GRUB_LOADER_FLAG_NORETURN)
(gdb) p grub_loader_flags 
$70 = 0
    grub_machine_fini ();

  for (cur = preboots_head; cur; cur = cur->next)
(gdb) p preboots_head 
$71 = (struct grub_preboot *) 0x0
    {
      err = cur->preboot_func (grub_loader_flags);
      if (err)
        {
          for (cur = cur->prev; cur; cur = cur->prev)
            cur->preboot_rest_func ();
          return err;
        }
    }
  err = (grub_loader_boot_func) ();

  for (cur = preboots_tail; cur; cur = cur->prev)
    if (! err)
      err = cur->preboot_rest_func ();
    else
      cur->preboot_rest_func ();

  return err;
}
```

Next in grub_linux_boot before involving grub_relocator32_boot, what preparations are done before grub transfers control to linux source code?

1. Set video mode with grub_video_set_mode with configuration in environment parameter
2. Setup video mode with grub_linux_setup_video to linux_params
3. Initialize video parameters of linux_params
4. Allocate memory in specific memory area for real mode code
5. Save linux parameters and linux command line to real mode memory
6. Add system memory region to e820 memory map in real mode memory
5. Initialize registers and involve grub_relocator32_boot



Call context of grub_linux_boot

```context_grub_linux_boot
grub_linux_boot
    |--grub_video_set_mode
    |--grub_linux_setup_video
        |--grub_video_get_driver_id
    |--find_mmap_size
        |--grub_mmap_iterate(hook)                         //get number of regions of entire memory
            |--grub_machine_mmap_iterate (count_hook)      //count number of mapped memory
                |--grub_bios_interrupt
            |--grub_machine_mmap_iterate (fill_hook)       //with fill_hook to map all memory
                |--grub_bios_interrupt
    |--grub_mmap_iterate(hook)                             //find memory region for real mode code.
    |--grub_relocator_alloc_chunk_addr
    |--get_virtual_current_address
    |--grub_mmap_iterate(fill_hook)                        //add memory regions to e820 map 
    |--grub_relocator32_boot
        |--grub_relocator_alloc_chunk_align                //allocate memory for relocate code
        |--grub_relocator_prepare_relocs
        |--((void (*) (void)) relst) ()                    //involve real start of linux


```

```grub_linux_boot
grub-core/loader/i386/linux.c:372

static grub_err_t
grub_linux_boot (void)
{
  int e820_num;
  grub_err_t err = 0;
  const char *modevar;
  char *tmp;
  struct grub_relocator32_state state;
  void *real_mode_mem;
  grub_addr_t real_mode_target = 0;
  grub_size_t real_size, mmap_size;
  grub_size_t cl_offset;

#ifdef GRUB_MACHINE_IEEE1275
  {
    const char *bootpath;
    grub_ssize_t len;

    bootpath = grub_env_get ("root");
    if (bootpath)
      grub_ieee1275_set_property (grub_ieee1275_chosen,
                                  "bootpath", bootpath,
                                  grub_strlen (bootpath) + 1,
                                  &len);
    linux_params.ofw_signature = GRUB_LINUX_OFW_SIGNATURE;
    linux_params.ofw_num_items = 1;
    linux_params.ofw_cif_handler = (grub_uint32_t) grub_ieee1275_entry_fn;
    linux_params.ofw_idt = 0;
  }
#endif

  modevar = grub_env_get ("gfxpayload");

  /* Now all graphical modes are acceptable.
     May change in future if we have modes without framebuffer.  */
  if (modevar && *modevar != 0)
(gdb) p modevar 
$72 = 0x0
    {
      tmp = grub_xasprintf ("%s;" DEFAULT_VIDEO_MODE, modevar);
      if (! tmp)
        return grub_errno;
#if ACCEPTS_PURE_TEXT
      err = grub_video_set_mode (tmp, 0, 0);
#else
      err = grub_video_set_mode (tmp, GRUB_VIDEO_MODE_TYPE_PURE_TEXT, 0);
#endif
      grub_free (tmp);
    }
  else
    {
#if ACCEPTS_PURE_TEXT
      err = grub_video_set_mode (DEFAULT_VIDEO_MODE, 0, 0);
#else
      err = grub_video_set_mode (DEFAULT_VIDEO_MODE,
                                 GRUB_VIDEO_MODE_TYPE_PURE_TEXT, 0);
#endif
    }

  if (err)
(gdb) p err
$80 = GRUB_ERR_NONE
    {
      grub_print_error ();
      grub_puts_ (N_("Booting in blind mode"));
      grub_errno = GRUB_ERR_NONE;
    }

  if (grub_linux_setup_video (&linux_params))
    {
#if defined (GRUB_MACHINE_PCBIOS) || defined (GRUB_MACHINE_COREBOOT) || defined (GRUB_MACHINE_QEMU)
      linux_params.have_vga = GRUB_VIDEO_LINUX_TYPE_TEXT;
      linux_params.video_mode = 0x3;
#else
      linux_params.have_vga = 0;
      linux_params.video_mode = 0;
      linux_params.video_width = 0;
      linux_params.video_height = 0;
#endif
    }


#ifndef GRUB_MACHINE_IEEE1275
  if (linux_params.have_vga == GRUB_VIDEO_LINUX_TYPE_TEXT)
(gdb) p linux_params.have_vga
$84 = 1 '\001'
#endif
    {
      grub_term_output_t term;
      int found = 0;
      FOR_ACTIVE_TERM_OUTPUTS(term)
        if (grub_strcmp (term->name, "vga_text") == 0
(gdb) p term->name 
$85 = 0xe78e "console"
            || grub_strcmp (term->name, "console") == 0
            || grub_strcmp (term->name, "ofconsole") == 0)
          {
            grub_uint16_t pos = grub_term_getxy (term);
            linux_params.video_cursor_x = pos >> 8;
            linux_params.video_cursor_y = pos & 0xff;
            linux_params.video_width = grub_term_width (term);
            linux_params.video_height = grub_term_height (term);
            found = 1;
(gdb) p linux_params.video_cursor_x
$87 = 0 '\000'
(gdb) p linux_params.video_cursor_y
$88 = 0 '\000'
(gdb) p linux_params.video_width
$89 = 80 'P'
(gdb) p linux_params.video_height
$90 = 25 '\031'
            break;
          }
      if (!found)
        {
          linux_params.video_cursor_x = 0;
          linux_params.video_cursor_y = 0;
          linux_params.video_width = 80;
          linux_params.video_height = 25;
        }
    }

  mmap_size = find_mmap_size ();
  /* Make sure that each size is aligned to a page boundary.  */
  cl_offset = ALIGN_UP (mmap_size + sizeof (linux_params), 4096);
  if (cl_offset < ((grub_size_t) linux_params.setup_sects << GRUB_DISK_SECTOR_BITS))
(gdb) p sizeof(linux_params)
$92 = 1020
(gdb) p cl_offset 
$93 = 12288
    cl_offset = ALIGN_UP ((grub_size_t) (linux_params.setup_sects
                                         << GRUB_DISK_SECTOR_BITS), 4096);
  real_size = ALIGN_UP (cl_offset + maximal_cmdline_size, 4096);

#ifdef GRUB_MACHINE_EFI
  efi_mmap_size = find_efi_mmap_size ();
  if (efi_mmap_size == 0)
    return grub_errno;
#endif

  grub_dprintf ("linux", "real_size = %x, mmap_size = %x\n",
                (unsigned) real_size, (unsigned) mmap_size);
(gdb) p cl_offset 
$94 = 16384
(gdb) p real_size 
$97 = 20480

  auto int NESTED_FUNC_ATTR hook (grub_uint64_t, grub_uint64_t,
                                  grub_memory_type_t);
  int NESTED_FUNC_ATTR hook (grub_uint64_t addr, grub_uint64_t size,
                             grub_memory_type_t type)
    {
      /* We must put real mode code in the traditional space.  */
      if (type != GRUB_MEMORY_AVAILABLE || addr > 0x90000)
        return 0;

      if (addr + size < 0x10000)
        return 0;

      if (addr < 0x10000)
        {
          size += addr - 0x10000;
          addr = 0x10000;
        }

      if (addr + size > 0x90000)
        size = 0x90000 - addr;

      if (real_size + efi_mmap_size > size)
        return 0;

      grub_dprintf ("linux", "addr = %lx, size = %x, need_size = %x\n",
                    (unsigned long) addr,
                    (unsigned) size,
                    (unsigned) (real_size + efi_mmap_size));
      real_mode_target = ((addr + size) - (real_size + efi_mmap_size));
      return 1;
    }
#ifdef GRUB_MACHINE_EFI
  grub_efi_mmap_iterate (hook, 1);
  if (! real_mode_target)
    grub_efi_mmap_iterate (hook, 0);
#else
  grub_mmap_iterate (hook);
#endif
  grub_dprintf ("linux", "real_mode_target = %lx, real_size = %x, efi_mmap_size = %x\n",
                (unsigned long) real_mode_target,
                (unsigned) real_size,
                (unsigned) efi_mmap_size);
(gdb) p /x real_mode_target 
$3 = 0x8b000
(gdb) p /x real_size 
$4 = 0x5000
(gdb) p /x efi_mmap_size 
$5 = 0x0

  if (! real_mode_target)
    return grub_error (GRUB_ERR_OUT_OF_MEMORY, "cannot allocate real mode pages");

  {
    grub_relocator_chunk_t ch;
    err = grub_relocator_alloc_chunk_addr (relocator, &ch,
                                           real_mode_target,
                                           (real_size + efi_mmap_size));
    if (err)
(gdb) p ch.next 
$6 = (struct grub_relocator_chunk *) 0x7fda990
(gdb) p /x ch.size
$7 = 0x5000
(gdb) p /x ch.srcv
$8 = 0x7feae60
(gdb) p /x ch.target
$9 = 0x8b000
(gdb) p /x ch.nsubchunks
$10 = 0x1
(gdb) p /x ch.src
$11 = 0x7feae60
(gdb) p /x ch.subchunks
$12 = 0x7fa2d60
(gdb) p err
$13 = GRUB_ERR_NONE
     return err;
    real_mode_mem = get_virtual_current_address (ch);
  }
  efi_mmap_buf = (grub_uint8_t *) real_mode_mem + real_size;
(gdb) p real_mode_mem 
$228 = (void *) 0x7feae60
(gdb) p efi_mmap_buf 
$235 = (void *) 0x7fefe60
(gdb) p /x real_size 
$20 = 0x5000

  grub_dprintf ("linux", "real_mode_mem = %lx\n",
                (unsigned long) real_mode_mem);

  struct linux_kernel_params *params;

  params = real_mode_mem;

  *params = linux_params;
  params->cmd_line_ptr = real_mode_target + cl_offset;
  grub_memcpy ((char *) params + cl_offset, linux_cmdline,
               maximal_cmdline_size);

  grub_dprintf ("linux", "code32_start = %x\n",
                (unsigned) params->code32_start);
(gdb) p /x params->code32_start 
$236 = 0x1000000
(gdb) p linux_cmdline 
$237 = 0x7fda8b0 "BOOT_IMAGE=/boot/vmlinuz-2.6.32.69 root=/dev/sda"

  auto int NESTED_FUNC_ATTR hook_fill (grub_uint64_t, grub_uint64_t,
                                  grub_memory_type_t);
  int NESTED_FUNC_ATTR hook_fill (grub_uint64_t addr, grub_uint64_t size,
                                  grub_memory_type_t type)
    {
      grub_uint32_t e820_type;
      switch (type)
        {
        case GRUB_MEMORY_AVAILABLE:
          e820_type = GRUB_E820_RAM;
          break;

        case GRUB_MEMORY_ACPI:
          e820_type = GRUB_E820_ACPI;
          break;

        case GRUB_MEMORY_NVS:
          e820_type = GRUB_E820_NVS;
          break;

        case GRUB_MEMORY_BADRAM:
          e820_type = GRUB_E820_BADRAM;
          break;

        default:
          e820_type = GRUB_E820_RESERVED;
        }
      if (grub_e820_add_region (params->e820_map, &e820_num,
                                addr, size, e820_type))
        return 1;

      return 0;
    }

  e820_num = 0;
  if (grub_mmap_iterate (hook_fill))
    return grub_errno;
  params->mmap_size = e820_num;
(gdb) p e820_num 
$241 = 6

#ifdef GRUB_MACHINE_EFI
  {
    grub_efi_uintn_t efi_desc_size;
    grub_size_t efi_mmap_target;
    grub_efi_uint32_t efi_desc_version;
    err = grub_efi_finish_boot_services (&efi_mmap_size, efi_mmap_buf, NULL,
                                         &efi_desc_size, &efi_desc_version);
    if (err)
      return err;

    /* Note that no boot services are available from here.  */
    efi_mmap_target = real_mode_target
      + ((grub_uint8_t *) efi_mmap_buf - (grub_uint8_t *) real_mode_mem);
    /* Pass EFI parameters.  */
    if (grub_le_to_cpu16 (params->version) >= 0x0208)
      {
        params->v0208.efi_mem_desc_size = efi_desc_size;
        params->v0208.efi_mem_desc_version = efi_desc_version;
        params->v0208.efi_mmap = efi_mmap_target;
        params->v0208.efi_mmap_size = efi_mmap_size;

#ifdef __x86_64__
        params->v0208.efi_mmap_hi = (efi_mmap_target >> 32);
#endif
      }
    else if (grub_le_to_cpu16 (params->version) >= 0x0206)
      {
        params->v0206.efi_mem_desc_size = efi_desc_size;
        params->v0206.efi_mem_desc_version = efi_desc_version;
        params->v0206.efi_mmap = efi_mmap_target;
        params->v0206.efi_mmap_size = efi_mmap_size;
      }
    else if (grub_le_to_cpu16 (params->version) >= 0x0204)
      {
        params->v0204.efi_mem_desc_size = efi_desc_size;
        params->v0204.efi_mem_desc_version = efi_desc_version;
        params->v0204.efi_mmap = efi_mmap_target;
        params->v0204.efi_mmap_size = efi_mmap_size;
      }
  }
#endif

  /* FIXME.  */
  /*  asm volatile ("lidt %0" : : "m" (idt_desc)); */
  state.ebp = state.edi = state.ebx = 0;
  state.esi = real_mode_target;
  state.esp = real_mode_target;
  state.eip = params->code32_start;
  return grub_relocator32_boot (relocator, state, 0);
}
```

```vedio_parameter_intialization

grub_video_set_mode (modestring=modestring@entry=0x7f734d6 "text", 
    modemask=modemask@entry=0, modevalue=modevalue@entry=0)
    at video/video.c:493

grub-core/video/video.c:489

grub_err_t
grub_video_set_mode (const char *modestring,
                     unsigned int modemask,
                     unsigned int modevalue)
{
  char *tmp;
  char *next_mode;
  char *current_mode;
  char *modevar;

  modevalue &= modemask;

  /* Take copy of env.var. as we don't want to modify that.  */
  modevar = grub_strdup (modestring);

  /* Initialize next mode.  */
  next_mode = modevar;

  if (! modevar)
(gdb) p modevar
$74 = 0x7fe1980 "text"
    return grub_errno;

  if (grub_memcmp (next_mode, "keep", sizeof ("keep")) == 0
(gdb) p next_mode
$75 = 0x7fe1980 "text"
      || grub_memcmp (next_mode, "keep,", sizeof ("keep,") - 1) == 0
      || grub_memcmp (next_mode, "keep;", sizeof ("keep;") - 1) == 0)
    {
      int suitable = 1;
      grub_err_t err;

      if (grub_video_adapter_active)
        {
          struct grub_video_mode_info mode_info;
          grub_memset (&mode_info, 0, sizeof (mode_info));
          err = grub_video_get_info (&mode_info);
          if (err)
            {
              suitable = 0;
              grub_errno = GRUB_ERR_NONE;
            }
          if ((mode_info.mode_type & modemask) != modevalue)
            suitable = 0;
        }
      else if (((GRUB_VIDEO_MODE_TYPE_PURE_TEXT & modemask) != 0)
               && ((GRUB_VIDEO_MODE_TYPE_PURE_TEXT & modevalue) == 0))
        suitable = 0;

      if (suitable)
        {
          grub_free (modevar);
          return GRUB_ERR_NONE;
        }
      next_mode += sizeof ("keep") - 1;
      if (! *next_mode)
        {
          grub_free (modevar);

          /* TRANSLATORS: This doesn't imply that there is no available video
             mode at all. All modes may have been filtered out by some criteria.
           */
          return grub_error (GRUB_ERR_BAD_ARGUMENT,
                             N_("no suitable video mode found"));
        }

      /* Skip separator. */
      next_mode++;
    }

  /* De-activate last set video adapter.  */
  if (grub_video_adapter_active)
(gdb) p grub_video_adapter_active 
$76 = (grub_video_adapter_t) 0x0
    {
      /* Finalize adapter.  */
      grub_video_adapter_active->fini ();
      if (grub_errno != GRUB_ERR_NONE)
        grub_errno = GRUB_ERR_NONE;

      /* Mark active adapter as not set.  */
      grub_video_adapter_active = 0;
    }

  /* Loop until all modes has been tested out.  */
  while (next_mode != NULL)
    {
      int width = -1;
      int height = -1;
      int depth = -1;
      grub_err_t err;
      unsigned int flags = modevalue;
      unsigned int flagmask = modemask;

      /* Use last next_mode as current mode.  */
      tmp = next_mode;

      /* Save position of next mode and separate modes.  */
      for (; *next_mode; next_mode++)
        if (*next_mode == ',' || *next_mode == ';')
          break;
      if (*next_mode)
        {
          *next_mode = 0;
          next_mode++;
        }
      else
        next_mode = 0;

      /* Skip whitespace.  */
      while (grub_isspace (*tmp))
(gdb) p next_mode
$77 = 0x0
(gdb) p tmp
$78 = 0x7fe1980 "text"
       tmp++;

      /* Initialize token holders.  */
      current_mode = tmp;

      /* XXX: we assume that we're in pure text mode if
         no video mode is initialized. Is it always true? */
      if (grub_strcmp (current_mode, "text") == 0)
(gdb) p current_mode
$79 = 0x7fe1980 "text"
        {
          struct grub_video_mode_info mode_info;

          grub_memset (&mode_info, 0, sizeof (mode_info));
          if (((GRUB_VIDEO_MODE_TYPE_PURE_TEXT & modemask) == 0)
              || ((GRUB_VIDEO_MODE_TYPE_PURE_TEXT & modevalue) != 0))
            {
              /* Valid mode found from adapter, and it has been activated.
                 Specify it as active adapter.  */
              grub_video_adapter_active = NULL;

              /* Free memory.  */
              grub_free (modevar);

              return GRUB_ERR_NONE;
            }
        }

......

-------------------------------------------------------------------------------------------------------------

grub_linux_setup_video (params=0x7f73ad8 <linux_params>)
    at loader/i386/linux.c:280

grub-core/loader/i386/linux.c:273

static grub_err_t
grub_linux_setup_video (struct linux_kernel_params *params)
{
  struct grub_video_mode_info mode_info;
  void *framebuffer;
  grub_err_t err;
  grub_video_driver_id_t driver_id;
  const char *gfxlfbvar = grub_env_get ("gfxpayloadforcelfb");

  driver_id = grub_video_get_driver_id ();
(gdb) p gfxlfbvar 
$81 = 0x0

  if (driver_id == GRUB_VIDEO_DRIVER_NONE)
(gdb) p driver_id 
$83 = GRUB_VIDEO_DRIVER_NONE
    return 1;
    
  err = grub_video_get_info_and_fini (&mode_info, &framebuffer);

  if (err)
    {
      grub_errno = GRUB_ERR_NONE;
      return 1;
    }

-------------------------------------------------------------------------------------------------------------

grub-core/video/video.c:66

grub_video_driver_id_t
grub_video_get_driver_id (void)
{
  if (! grub_video_adapter_active)
(gdb) p grub_video_adapter_active 
$82 = (grub_video_adapter_t) 0x0
    return GRUB_VIDEO_DRIVER_NONE;
  return grub_video_adapter_active->id;
}
```

Find size of memory used for mapping system memory with find_mmap_size.

```find_memory_size_for_mapping_system_memory

grub-core/loader/i386/linux.c:150

/* Find the optimal number of pages for the memory map. */
static grub_size_t
find_mmap_size (void)
{
  grub_size_t count = 0, mmap_size;

  auto int NESTED_FUNC_ATTR hook (grub_uint64_t, grub_uint64_t,
                                  grub_memory_type_t);
  int NESTED_FUNC_ATTR hook (grub_uint64_t addr __attribute__ ((unused)),
                             grub_uint64_t size __attribute__ ((unused)),
                             grub_memory_type_t type __attribute__ ((unused)))
    {
      count++;
      return 0;
    }

  grub_mmap_iterate (hook);

  mmap_size = count * sizeof (struct grub_e820_mmap);
(gdb) p count
$1 = 6


  /* Increase the size a bit for safety, because GRUB allocates more on
     later.  */
  mmap_size += (1 << 12);

  return page_align (mmap_size);
(gdb) p mmap_size
$2 = 4216
(gdb) p sizeof (struct grub_e820_mmap)
$3 = 20
}
```

grub_mmap_iterate is used several times in grub_linux_boot, not only in find_mmap_size. Inside grub_mmap_iterate, it queries system memory with BIOS call in grub_machine_mmap_iterate to get number of mapped memory regions in current system, then allocates memory to save memory information reflexing all memory in the system and sort them. Different purposes are satisfied with dedicated hook function to grub_mmap_iterate.

```grub_mmap_iterate

grub-core/mmap/mmap.c:38

grub_err_t
grub_mmap_iterate (grub_memory_hook_t hook)
{

  /* This function resolves overlapping regions and sorts the memory map.
     It uses scanline (sweeping) algorithm.
  */
  /* If same page is used by multiple types it's resolved
     according to priority:
     1 - free memory
     2 - memory usable by firmware-aware code
     3 - unusable memory
     4 - a range deliberately empty
  */
  int priority[] =
    {
      [GRUB_MEMORY_AVAILABLE] = 1,
      [GRUB_MEMORY_RESERVED] = 3,
      [GRUB_MEMORY_ACPI] = 2,
      [GRUB_MEMORY_CODE] = 3,
      [GRUB_MEMORY_NVS] = 3,
      [GRUB_MEMORY_HOLE] = 4,
    };

  int i, done;

  /* Scanline events. */
  struct grub_mmap_scan
  {
    /* At which memory address. */
    grub_uint64_t pos;
    /* 0 = region starts, 1 = region ends. */
    int type;
    /* Which type of memory region? */
    int memtype;
  };
  struct grub_mmap_scan *scanline_events;
  struct grub_mmap_scan t;

  /* Previous scanline event. */
  grub_uint64_t lastaddr;
  int lasttype;
  /* Current scanline event. */
  int curtype;
  /* How many regions of given type overlap at current location? */
  int present[ARRAY_SIZE (priority)];
  /* Number of mmap chunks. */
  int mmap_num;

#ifndef GRUB_MMAP_REGISTER_BY_FIRMWARE
  struct grub_mmap_region *cur;
#endif

  auto int NESTED_FUNC_ATTR count_hook (grub_uint64_t, grub_uint64_t,
                                        grub_uint32_t);
  int NESTED_FUNC_ATTR count_hook (grub_uint64_t addr __attribute__ ((unused)),
                                   grub_uint64_t size __attribute__ ((unused)),
                                   grub_memory_type_t type __attribute__ ((unused)))
  {
    mmap_num++;
    return 0;
  }

  auto int NESTED_FUNC_ATTR fill_hook (grub_uint64_t, grub_uint64_t,
                                        grub_uint32_t);
  int NESTED_FUNC_ATTR fill_hook (grub_uint64_t addr,
                                  grub_uint64_t size,
                                  grub_memory_type_t type)
  {
    scanline_events[i].pos = addr;
    scanline_events[i].type = 0;
    if (type < ARRAY_SIZE (priority) && priority[type])
      scanline_events[i].memtype = type;
    else
      {
        grub_dprintf ("mmap", "Unknown memory type %d. Assuming unusable\n",
                      type);
        scanline_events[i].memtype = GRUB_MEMORY_RESERVED;
      }
    i++;

    scanline_events[i].pos = addr + size;
    scanline_events[i].type = 1;
    scanline_events[i].memtype = scanline_events[i - 1].memtype;
    i++;

    return 0;
  }

  mmap_num = 0;

#ifndef GRUB_MMAP_REGISTER_BY_FIRMWARE
  for (cur = grub_mmap_overlays; cur; cur = cur->next)
    mmap_num++;
#endif

  grub_machine_mmap_iterate (count_hook);

  /* Initialize variables. */
  grub_memset (present, 0, sizeof (present));
  scanline_events = (struct grub_mmap_scan *)
    grub_malloc (sizeof (struct grub_mmap_scan) * 2 * mmap_num);
(gdb) p mmap_num 
$1 = 6

  if (! scanline_events)
    return grub_errno;

  i = 0;
#ifndef GRUB_MMAP_REGISTER_BY_FIRMWARE
  /* Register scanline events. */
  for (cur = grub_mmap_overlays; cur; cur = cur->next)
    {
      scanline_events[i].pos = cur->start;
      scanline_events[i].type = 0;
      if (cur->type < ARRAY_SIZE (priority) && priority[cur->type])
        scanline_events[i].memtype = cur->type;
      else
        scanline_events[i].memtype = GRUB_MEMORY_RESERVED;
      i++;

      scanline_events[i].pos = cur->end;
      scanline_events[i].type = 1;
      scanline_events[i].memtype = scanline_events[i - 1].memtype;
      i++;
    }
#endif /* ! GRUB_MMAP_REGISTER_BY_FIRMWARE */

  grub_machine_mmap_iterate (fill_hook);

  /* Primitive bubble sort. It has complexity O(n^2) but since we're
     unlikely to have more than 100 chunks it's probably one of the
     fastest for one purpose. */
  done = 1;
  while (done)
    {
      done = 0;
      for (i = 0; i < 2 * mmap_num - 1; i++)
(gdb) print_scanline_events_in_grub_mmap_iterate 
$3 = 0x0
$4 = 0x0
$5 = 0x1
$6 = 0x9fc00
$7 = 0x1
$8 = 0x1
$9 = 0x9fc00
$10 = 0x0
$11 = 0x2
$12 = 0xa0000
$13 = 0x1
$14 = 0x2
$15 = 0xf0000
$16 = 0x0
$17 = 0x2
$18 = 0x100000
$19 = 0x1
$20 = 0x2
$21 = 0x100000
$22 = 0x0
$23 = 0x1
$24 = 0x7ffe000
$25 = 0x1
$26 = 0x1
$27 = 0x7ffe000
$28 = 0x0
$29 = 0x2
$30 = 0x8000000
$31 = 0x1
$32 = 0x2
$33 = 0xfffc0000
$34 = 0x0
$35 = 0x2
$36 = 0x100000000
$37 = 0x1
$38 = 0x2

        if (scanline_events[i + 1].pos < scanline_events[i].pos
            || (scanline_events[i + 1].pos == scanline_events[i].pos
                && scanline_events[i + 1].type == 0
                && scanline_events[i].type == 1))
          {
            t = scanline_events[i + 1];
            scanline_events[i + 1] = scanline_events[i];
            scanline_events[i] = t;
            done = 1;
          }
    }

  lastaddr = scanline_events[0].pos;
(gdb) print_scanline_events_in_grub_mmap_iterate 
$39 = 0x0
$40 = 0x0
$41 = 0x1
$42 = 0x9fc00
$43 = 0x0
$44 = 0x2
$45 = 0x9fc00
$46 = 0x1
$47 = 0x1
$48 = 0xa0000
$49 = 0x1
$50 = 0x2
$51 = 0xf0000
$52 = 0x0
$53 = 0x2
$54 = 0x100000
$55 = 0x0
$56 = 0x1
$57 = 0x100000
$58 = 0x1
$59 = 0x2
$60 = 0x7ffe000
$61 = 0x0
$62 = 0x2
$63 = 0x7ffe000
$64 = 0x1
$65 = 0x1
$66 = 0x8000000
$67 = 0x1
$68 = 0x2
$69 = 0xfffc0000
$70 = 0x0
$71 = 0x2
$72 = 0x100000000
$73 = 0x1
$74 = 0x2

  lasttype = scanline_events[0].memtype;
  for (i = 0; i < 2 * mmap_num; i++)
    {
      unsigned k;
      /* Process event. */
      if (scanline_events[i].type)
        present[scanline_events[i].memtype]--;
      else
        present[scanline_events[i].memtype]++;

      /* Determine current region type. */
      curtype = -1;
      for (k = 0; k < ARRAY_SIZE (priority); k++)
        if (present[k] && (curtype == -1 || priority[k] > priority[curtype]))
          curtype = k;

      /* Announce region to the hook if necessary. */
      if ((curtype == -1 || curtype != lasttype)
          && lastaddr != scanline_events[i].pos
          && lasttype != -1
          && lasttype != GRUB_MEMORY_HOLE
          && hook (lastaddr, scanline_events[i].pos - lastaddr, lasttype))
        {
          grub_free (scanline_events);
          return GRUB_ERR_NONE;
        }

      /* Update last values if necessary. */
      if (curtype == -1 || curtype != lasttype)
        {
          lasttype = curtype;
          lastaddr = scanline_events[i].pos;
        }
    }

  grub_free (scanline_events);
  return GRUB_ERR_NONE;
}

```

Add print_scanline_events_in_grub_mmap_iterate in gdb_grub.in, it helps print content of entire scanline_events.

```print_scanline_events_in_grub_mmap_iterate
define print_scanline_events_in_grub_mmap_iterate
        set $idx = 0
        set $all_events = 2*mmap_num
        while ($idx < $all_events)
                print /x scanline_events[$idx].pos
                print /x scanline_events[$idx].type
                print /x scanline_events[$idx].memtype
                set $idx = $idx + 1
        end
end
document print_scanline_events_in_grub_mmap_iterate
        Print scanline_events in grub_mmap_iterate
end
```

The result of real_mode_target

```result_of_real_mode_target
(gdb) p /x real_mode_target 
$3 = 0x8b000
(gdb) p /x real_size 
$4 = 0x5000
(gdb) p /x efi_mmap_size 
$5 = 0x0
```

The region of real mode code is betwen 0x10000 and 0x90000

```real_mode_code_space

grub-core/loader/i386/linux.c:495

  auto int NESTED_FUNC_ATTR hook (grub_uint64_t, grub_uint64_t,
                                  grub_memory_type_t);
  int NESTED_FUNC_ATTR hook (grub_uint64_t addr, grub_uint64_t size,
                             grub_memory_type_t type)
    {
      /* We must put real mode code in the traditional space.  */
      if (type != GRUB_MEMORY_AVAILABLE || addr > 0x90000)
        return 0;

      if (addr + size < 0x10000)
        return 0;

      if (addr < 0x10000)
        {
          size += addr - 0x10000;
          addr = 0x10000;
        }

      if (addr + size > 0x90000)
        size = 0x90000 - addr;

      if (real_size + efi_mmap_size > size)
        return 0;

      grub_dprintf ("linux", "addr = %lx, size = %x, need_size = %x\n",
                    (unsigned long) addr,
                    (unsigned) size,
                    (unsigned) (real_size + efi_mmap_size));
      real_mode_target = ((addr + size) - (real_size + efi_mmap_size));
      return 1;
    }
```

Allocates memory start from real_mode_target.

```grub_relocator_alloc_chunk_addr

grub_relocator_alloc_chunk_addr (rel=0x7fdb100, out=out@entry=0x7fc84, 
    target=569344, size=20480) at lib/relocator.c:1219

grub-core/lib/relocator.c:1211

grub_err_t
grub_relocator_alloc_chunk_addr (struct grub_relocator *rel,
                                 grub_relocator_chunk_t *out,
                                 grub_phys_addr_t target, grub_size_t size)
{
  struct grub_relocator_chunk *chunk;
  grub_phys_addr_t min_addr = 0, max_addr;

  if (target > ~size)
    return grub_error (GRUB_ERR_BUG, "address is out of range");

  adjust_limits (rel, &min_addr, &max_addr, target, target);

  for (chunk = rel->chunks; chunk; chunk = chunk->next)
    if ((chunk->target <= target && target < chunk->target + chunk->size)
        || (target <= chunk->target && chunk->target < target + size))
      return grub_error (GRUB_ERR_BUG, "overlap detected");

  chunk = grub_malloc (sizeof (struct grub_relocator_chunk));
  if (!chunk)
    return grub_errno;

  grub_dprintf ("relocator",
                "min_addr = 0x%llx, max_addr = 0x%llx, target = 0x%llx\n",
                (unsigned long long) min_addr, (unsigned long long) max_addr,
                (unsigned long long) target);
(gdb) p /x min_addr 
$179 = 0x0
(gdb) p /x max_addr 
$180 = 0x100000
(gdb) p /x target 
$181 = 0x8b000

  do
    {
      /* A trick to improve Linux allocation.  */
#if defined (__i386__) || defined (__x86_64__)
      if (target < 0x100000)
        if (malloc_in_range (rel, rel->highestnonpostaddr, ~(grub_addr_t)0, 1,
                             size, chunk, 0, 1))
          {
            if (rel->postchunks > chunk->src)
              rel->postchunks = chunk->src;
            break;
          }
#endif
      if (malloc_in_range (rel, target, max_addr, 1, size, chunk, 1, 0))
        break;

      if (malloc_in_range (rel, min_addr, target, 1, size, chunk, 0, 0))
        break;

      if (malloc_in_range (rel, rel->highestnonpostaddr, ~(grub_addr_t)0, 1,
                           size, chunk, 0, 1))
        {
          if (rel->postchunks > chunk->src)
            rel->postchunks = chunk->src;
          break;
        }

      grub_dprintf ("relocator", "not allocated\n");
      grub_free (chunk);
      return grub_error (GRUB_ERR_OUT_OF_MEMORY, N_("out of memory"));
    }
  while (0);

  grub_dprintf ("relocator", "allocated 0x%llx/0x%llx\n",
                (unsigned long long) chunk->src, (unsigned long long) target);

  if (rel->highestaddr < target + size)
    rel->highestaddr = target + size;

  if (rel->highestaddr < chunk->src + size)
(gdb) p /x rel->highestaddr
$206 = 0x90000
(gdb) p /x chunk->src
$208 = 0x7feae60
(gdb) p /x size
$209 = 0x5000
    rel->highestaddr = chunk->src + size;

  if (chunk->src < rel->postchunks)
(gdb) p /x rel->highestaddr 
$211 = 0x7fefe60
(gdb) p /x rel->postchunks 
$213 = 0x7feae60
    {
      if (rel->highestnonpostaddr < target + size)
        rel->highestnonpostaddr = target + size;

      if (rel->highestnonpostaddr < chunk->src + size)
        rel->highestnonpostaddr = chunk->src + size;
    }

  grub_dprintf ("relocator", "relocators_size=%ld\n",
                (unsigned long) rel->relocators_size);

  if (chunk->src < target)
    rel->relocators_size += grub_relocator_backward_size;
  if (chunk->src > target)
    rel->relocators_size += grub_relocator_forward_size;

  grub_dprintf ("relocator", "relocators_size=%ld\n",
                (unsigned long) rel->relocators_size);
(gdb) p rel->relocators_size 
$215 = 61

  chunk->target = target;
  chunk->size = size;
  chunk->next = rel->chunks;
  rel->chunks = chunk;
  grub_dprintf ("relocator", "cur = %p, next = %p\n", rel->chunks,
                rel->chunks->next);

  chunk->srcv = grub_map_memory (chunk->src, chunk->size);
  *out = chunk;
#ifdef DEBUG_RELOCATOR
  grub_memset (chunk->srcv, 0xfa, chunk->size);
  grub_mm_check ();
#endif
  return GRUB_ERR_NONE;
(gdb) p /x chunk->target 
$216 = 0x8b000
(gdb) p /x chunk->size
$217 = 0x5000
(gdb) p /x rel->chunks 
$218 = 0x7fe1860
(gdb) p /x chunk->next 
$219 = 0x7fda880
(gdb) p *out
$224 = (grub_relocator_chunk_t) 0x7fe1860
}

-------------------------------------------------------------------------------------------------------------

adjust_limits (rel=rel@entry=0x7fdb100, min_addr=min_addr@entry=0x7fbbc, 
    max_addr=max_addr@entry=0x7fbc0, in_min=569344, in_max=569344)
    at lib/relocator.c:1195

grub-core/lib/relocator.c:1188

static void
adjust_limits (struct grub_relocator *rel,
               grub_phys_addr_t *min_addr, grub_phys_addr_t *max_addr,
               grub_phys_addr_t in_min, grub_phys_addr_t in_max)
{
  struct grub_relocator_chunk *chunk;

  *min_addr = 0;
  *max_addr = rel->postchunks;

(gdb) p /x *min_addr 
$154 = 0x0
(gdb) p /x *max_addr 
$155 = 0xffffffff

  /* Keep chunks in memory in the same order as they'll be after relocation.  */
  for (chunk = rel->chunks; chunk; chunk = chunk->next)
    {
      if (chunk->target > in_max && chunk->src < *max_addr
          && chunk->src < rel->postchunks)
        *max_addr = chunk->src;
      if (chunk->target + chunk->size <= in_min
          && chunk->src + chunk->size > *min_addr
          && chunk->src < rel->postchunks)
        *min_addr = chunk->src + chunk->size;
    }
(gdb) p /x *max_addr 
$168 = 0x100000
(gdb) p /x *min_addr 
$169 = 0x0
}

-------------------------------------------------------------------------------------------------------------

grub-core/lib/relocator.c:415

static int
malloc_in_range (struct grub_relocator *rel,
                 grub_addr_t start, grub_addr_t end, grub_addr_t align,
                 grub_size_t size, struct grub_relocator_chunk *res,
                 int from_low_priv, int collisioncheck)
```

Add memory regions to e820 map with hook_fill

```add_to_e820_map
  auto int NESTED_FUNC_ATTR hook_fill (grub_uint64_t, grub_uint64_t,
                                  grub_memory_type_t);
  int NESTED_FUNC_ATTR hook_fill (grub_uint64_t addr, grub_uint64_t size,
                                  grub_memory_type_t type)
    {
      grub_uint32_t e820_type;
      switch (type)
        {
        case GRUB_MEMORY_AVAILABLE:
          e820_type = GRUB_E820_RAM;
          break;

        case GRUB_MEMORY_ACPI:
          e820_type = GRUB_E820_ACPI;
          break;

        case GRUB_MEMORY_NVS:
          e820_type = GRUB_E820_NVS;
          break;

        case GRUB_MEMORY_BADRAM:
          e820_type = GRUB_E820_BADRAM;
          break;

        default:
          e820_type = GRUB_E820_RESERVED;
        }
      if (grub_e820_add_region (params->e820_map, &e820_num,
                                addr, size, e820_type))
        return 1;

      return 0;
    }

-------------------------------------------------------------------------------------------------------------

Breakpoint 12, hook_fill (addr=0, size=654336, type=GRUB_MEMORY_AVAILABLE)
    at loader/i386/linux.c:594
Breakpoint 12, hook_fill (addr=654336, size=1024, type=GRUB_MEMORY_RESERVED)
    at loader/i386/linux.c:594
Breakpoint 12, hook_fill (addr=983040, size=65536, type=GRUB_MEMORY_RESERVED)
    at loader/i386/linux.c:594
Breakpoint 12, hook_fill (addr=1048576, size=133160960, type=GRUB_MEMORY_AVAILABLE) 
    at loader/i386/linux.c:594
Breakpoint 12, hook_fill (addr=134209536, size=8192, type=GRUB_MEMORY_RESERVED)
    at loader/i386/linux.c:594
Breakpoint 12, hook_fill (addr=4294705152, size=262144, type=GRUB_MEMORY_RESERVED)
    at loader/i386/linux.c:594

grub-core/loader/i386/linux.c:253

static grub_err_t
grub_e820_add_region (struct grub_e820_mmap *e820_map, int *e820_num,
                      grub_uint64_t start, grub_uint64_t size,
                      grub_uint32_t type)
{
  int n = *e820_num;

  if ((n > 0) && (e820_map[n - 1].addr + e820_map[n - 1].size == start) &&
      (e820_map[n - 1].type == type))
      e820_map[n - 1].size += size;
  else
    {
      e820_map[n].addr = start;
      e820_map[n].size = size;
      e820_map[n].type = type;
      (*e820_num)++;
    }
  return GRUB_ERR_NONE;
}

```

In grub_relocator allocate memory for the destination of relocated code and move relocate code to destionation from grub_relocator32_start which defined in relocator32.S.

```grub_relocator32_boot

grub_relocator32_boot (rel=0x7fdb210, state=..., avoid_efi_bootservices=0)
    at lib/i386/relocator.c:167

grub-core/lib/i386/relocator.c:156

grub_err_t
grub_relocator32_boot (struct grub_relocator *rel,
                       struct grub_relocator32_state state,
                       int avoid_efi_bootservices)
{
  grub_err_t err;
  void *relst;
  grub_relocator_chunk_t ch;

  err = grub_relocator_alloc_chunk_align (rel, &ch, 0,
                                          (0xffffffff - RELOCATOR_SIZEOF (32))
                                          + 1, RELOCATOR_SIZEOF (32), 16,
                                          GRUB_RELOCATOR_PREFERENCE_NONE,
                                          avoid_efi_bootservices);
  if (err)
    return err;

  grub_relocator32_eax = state.eax;
  grub_relocator32_ebx = state.ebx;
  grub_relocator32_ecx = state.ecx;
  grub_relocator32_edx = state.edx;
  grub_relocator32_eip = state.eip;
  grub_relocator32_esp = state.esp;
  grub_relocator32_ebp = state.ebp;
  grub_relocator32_esi = state.esi;
  grub_relocator32_edi = state.edi;

  grub_memmove (get_virtual_current_address (ch), &grub_relocator32_start,
                RELOCATOR_SIZEOF (32));
(gdb) p ch.srcv 
$21 = (void *) 0x9df000
(gdb) p &grub_relocator32_start 
$22 = (grub_uint8_t *) 0x7f96600 <grub_relocator32_start> "\211\306\005\t"

  err = grub_relocator_prepare_relocs (rel, get_physical_target_address (ch),
                                       &relst, NULL);
  if (err)
    return err;

  asm volatile ("cli");
  ((void (*) (void)) relst) ();

  /* Not reached.  */
  return GRUB_ERR_NONE;
}

-------------------------------------------------------------------------------------------------------------

grub_relocator_alloc_chunk_align (rel=rel@entry=0x7fdb210, 
    out=out@entry=0x7fbb0, avoid_efi_boot_services=0) at lib/relocator.c:1317

grub-core/lib/relocator.c:1316

grub_err_t
grub_relocator_alloc_chunk_align (struct grub_relocator *rel,
				  grub_relocator_chunk_t *out,
				  grub_phys_addr_t min_addr,
				  grub_phys_addr_t max_addr,
				  grub_size_t size, grub_size_t align,
				  int preference,
				  int avoid_efi_boot_services)
{
  grub_addr_t min_addr2 = 0, max_addr2;
  struct grub_relocator_chunk *chunk;

  if (max_addr > ~size)
(gdb) p /x min_addr
$1 = 0x0
(gdb) p /x max_addr
$2 = 0xffffff30
(gdb) p /x size
$3 = 0xd0
(gdb) p /x align 
$4 = 0x10
(gdb) p preference 
$5 = 0
    max_addr = ~size;

#ifdef GRUB_MACHINE_PCBIOS
  if (min_addr < 0x1000)
    min_addr = 0x1000;
#endif

  grub_dprintf ("relocator", "chunks = %p\n", rel->chunks);
(gdb) p /x max_addr
$6 = 0xffffff2f
(gdb) p /x min_addr
$7 = 0x1000

  chunk = grub_malloc (sizeof (struct grub_relocator_chunk));
  if (!chunk)
    return grub_errno;

  if (malloc_in_range (rel, min_addr, max_addr, align,
		       size, chunk,
		       preference != GRUB_RELOCATOR_PREFERENCE_HIGH, 1))
    {
      grub_dprintf ("relocator", "allocated 0x%llx/0x%llx\n",
		    (unsigned long long) chunk->src,
		    (unsigned long long) chunk->src);
      grub_dprintf ("relocator", "chunks = %p\n", rel->chunks);
(gdb) p /x chunk->src
$9 = 0x9df000
(gdb) p /x rel->chunks 
$10 = 0x7fe1970
      chunk->target = chunk->src;
      chunk->size = size;
      chunk->next = rel->chunks;
      rel->chunks = chunk;
      chunk->srcv = grub_map_memory (chunk->src, chunk->size);
      *out = chunk;
      return GRUB_ERR_NONE;
(gdb) p /x chunk->target 
$11 = 0x9df000
(gdb) p /x chunk->size
size      size_t    sizetype  
(gdb) p /x chunk->size
$12 = 0xd0
(gdb) p /x chunk->next 
$13 = 0x7fe1970
(gdb) p /x rel->chunks 
$14 = 0x7fa46f0
(gdb) p /x chunk
$15 = 0x7fa46f0
(gdb) p /x chunk->srcv 
$16 = 0x9df000
    }

  adjust_limits (rel, &min_addr2, &max_addr2, min_addr, max_addr);
  grub_dprintf ("relocator", "Adjusted limits from %lx-%lx to %lx-%lx\n",
		(unsigned long) min_addr, (unsigned long) max_addr,
		(unsigned long) min_addr2, (unsigned long) max_addr2);

  do
    {
      if (malloc_in_range (rel, min_addr2, max_addr2, align,
			   size, chunk, 1, 1))
	break;

      if (malloc_in_range (rel, rel->highestnonpostaddr, ~(grub_addr_t)0, 1,
			   size, chunk, 0, 1))
	{
	  if (rel->postchunks > chunk->src)
	    rel->postchunks = chunk->src;
	  break;
	}

      return grub_error (GRUB_ERR_OUT_OF_MEMORY, N_("out of memory"));
    }
  while (0);

  {
    int found = 0;
    auto int NESTED_FUNC_ATTR hook (grub_uint64_t, grub_uint64_t,
				    grub_memory_type_t);
    int NESTED_FUNC_ATTR hook (grub_uint64_t addr, grub_uint64_t sz,
			       grub_memory_type_t type)
    {
      grub_uint64_t candidate;
      if (type != GRUB_MEMORY_AVAILABLE)
	return 0;
      candidate = ALIGN_UP (addr, align);
      if (candidate < min_addr)
	candidate = ALIGN_UP (min_addr, align);
      if (candidate + size > addr + sz
	  || candidate > ALIGN_DOWN (max_addr, align))
	return 0;
      if (preference == GRUB_RELOCATOR_PREFERENCE_HIGH)
	candidate = ALIGN_DOWN (min (addr + sz - size, max_addr), align);
      if (!found || (preference == GRUB_RELOCATOR_PREFERENCE_HIGH
		     && candidate > chunk->target))
	chunk->target = candidate;
      if (!found || (preference == GRUB_RELOCATOR_PREFERENCE_LOW
		     && candidate < chunk->target))
	chunk->target = candidate;
      found = 1;
      return 0;
    }

#ifdef GRUB_MACHINE_EFI
    grub_efi_mmap_iterate (hook, avoid_efi_boot_services);
#elif defined (__powerpc__)
    (void) avoid_efi_boot_services;
    grub_machine_mmap_iterate (hook);
#else
    (void) avoid_efi_boot_services;
    grub_mmap_iterate (hook);
#endif
    if (!found)
      return grub_error (GRUB_ERR_BAD_OS, "couldn't find suitable memory target");
  }
  while (1)
    {
      struct grub_relocator_chunk *chunk2;
      for (chunk2 = rel->chunks; chunk2; chunk2 = chunk2->next)
	if ((chunk2->target <= chunk->target
	     && chunk->target < chunk2->target + chunk2->size)
	    || (chunk->target <= chunk2->target && chunk2->target
		< chunk->target + size))
	  {
	    if (preference == GRUB_RELOCATOR_PREFERENCE_HIGH)
	      chunk->target = ALIGN_DOWN (chunk2->target, align);
	    else
	      chunk->target = ALIGN_UP (chunk2->target + chunk2->size, align);
	    break;
	  }
      if (!chunk2)
	break;
    }

  grub_dprintf ("relocator", "relocators_size=%ld\n",
		(unsigned long) rel->relocators_size);

  if (chunk->src < chunk->target)
    rel->relocators_size += grub_relocator_backward_size;
  if (chunk->src > chunk->target)
    rel->relocators_size += grub_relocator_forward_size;

  grub_dprintf ("relocator", "relocators_size=%ld\n",
		(unsigned long) rel->relocators_size);

  chunk->size = size;
  chunk->next = rel->chunks;
  rel->chunks = chunk;
  grub_dprintf ("relocator", "cur = %p, next = %p\n", rel->chunks,
		rel->chunks->next);
  chunk->srcv = grub_map_memory (chunk->src, chunk->size);
  *out = chunk;
#ifdef DEBUG_RELOCATOR
  grub_memset (chunk->srcv, 0xfa, chunk->size);
  grub_mm_check ();
#endif
  return GRUB_ERR_NONE;
}
```

```grub_memmove
grub_memmove (dest=0x9df000, src=0x7f96600 <grub_relocator32_start>, 
    n=n@entry=208) at kern/misc.c:52

grub-core/kern/misc.c:46

void *
grub_memmove (void *dest, const void *src, grub_size_t n)
{
  char *d = (char *) dest;
  const char *s = (const char *) src;

  if (d < s)
    while (n--)
      *d++ = *s++;
  else
    {
      d += n;
      s += n;

      while (n--)
        *--d = *--s;
    }

  return dest;
}
```

Result of grub_memmove in destination, address start from 0x9df000 size 0xd0

![grub_memmove_in_grub_relocator32_boot](/GRUB2/grub_memmove_in_grub_relocator32_boot.png)

Continue routines in grub_relocator32_boot

```grub_relocator_prepare_relocs
grub_relocator_prepare_relocs (rel=rel@entry=0x7fdb210, addr=10350592, 
    relstart=relstart@entry=0x7fbac, relsize=0x0) at lib/relocator.c:1488

grub-core/lib/relocator.c:1485

grub_err_t
grub_relocator_prepare_relocs (struct grub_relocator *rel, grub_addr_t addr,
                               void **relstart, grub_size_t *relsize)
{
  grub_uint8_t *rels;
  grub_uint8_t *rels0;
  struct grub_relocator_chunk *sorted;
  grub_size_t nchunks = 0;
  unsigned j;
  struct grub_relocator_chunk movers_chunk;

  grub_dprintf ("relocator", "Preparing relocs (size=%ld)\n",
                (unsigned long) rel->relocators_size);
(gdb) p /x rel->relocators_size 
$26 = 0x3d
(gdb) p grub_relocator_align 
$27 = 1

  if (!malloc_in_range (rel, 0, ~(grub_addr_t)0 - rel->relocators_size + 1,
                        grub_relocator_align,
                        rel->relocators_size, &movers_chunk, 1, 1))
    return grub_error (GRUB_ERR_OUT_OF_MEMORY, N_("out of memory"));
  movers_chunk.srcv = rels = rels0
    = grub_map_memory (movers_chunk.src, movers_chunk.size);

  if (relsize)
(gdb) p /x movers_chunk.src
$30 = 0x9df0d0
(gdb) p /x movers_chunk.size
$31 = 0x3d
(gdb) p movers_chunk.srcv
$33 = (void *) 0x9df0d0
    *relsize = rel->relocators_size;
(gdb) p /x rel->relocators_size 
$37 = 0x3d

  grub_dprintf ("relocator", "Relocs allocated at %p\n", movers_chunk.srcv);

  {
    unsigned i;
    grub_size_t count[257];
    struct grub_relocator_chunk *from, *to, *tmp;

    grub_memset (count, 0, sizeof (count));

    {
        struct grub_relocator_chunk *chunk;
        for (chunk = rel->chunks; chunk; chunk = chunk->next)
          {
            grub_dprintf ("relocator", "chunk %p->%p, 0x%lx\n",
                          (void *) chunk->src, (void *) chunk->target,
                          (unsigned long) chunk->size);
            nchunks++;
            count[(chunk->src & 0xff) + 1]++;
          }
    }
    from = grub_malloc (nchunks * sizeof (sorted[0]));
(gdb) p nchunks 
$38 = 4
(gdb) p sizeof (sorted[0])
$39 = 28
    to = grub_malloc (nchunks * sizeof (sorted[0]));
    if (!from || !to)
      {
        grub_free (from);
        grub_free (to);
        return grub_errno;
      }

    for (j = 0; j < 256; j++)
      count[j+1] += count[j];

    {
      struct grub_relocator_chunk *chunk;
      for (chunk = rel->chunks; chunk; chunk = chunk->next)
        from[count[chunk->src & 0xff]++] = *chunk;
    }

    for (i = 1; i < GRUB_CPU_SIZEOF_VOID_P; i++)
      {
        grub_memset (count, 0, sizeof (count));
        for (j = 0; j < nchunks; j++)
          count[((from[j].src >> (8 * i)) & 0xff) + 1]++;
        for (j = 0; j < 256; j++)
          count[j+1] += count[j];
        for (j = 0; j < nchunks; j++)
          to[count[(from[j].src >> (8 * i)) & 0xff]++] = from[j];
        tmp = to;
        to = from;
        from = tmp;
      }
    sorted = from;
    grub_free (to);
  }

  for (j = 0; j < nchunks; j++)
    {
      grub_dprintf ("relocator", "sorted chunk %p->%p, 0x%lx\n",
                    (void *) sorted[j].src, (void *) sorted[j].target,
                    (unsigned long) sorted[j].size);
      if (sorted[j].src < sorted[j].target)
        {
          grub_cpu_relocator_backward ((void *) rels,
                                       sorted[j].srcv,
                                       grub_map_memory (sorted[j].target,
                                                        sorted[j].size),
                                       sorted[j].size);
          rels += grub_relocator_backward_size;
        }
      if (sorted[j].src > sorted[j].target)
        {
          grub_cpu_relocator_forward ((void *) rels,
                                      sorted[j].srcv,
                                      grub_map_memory (sorted[j].target,
                                                       sorted[j].size),
                                      sorted[j].size);
          rels += grub_relocator_forward_size;
        }
      if (sorted[j].src == sorted[j].target)
        grub_arch_sync_caches (sorted[j].srcv, sorted[j].size);
    }
  grub_cpu_relocator_jumper ((void *) rels, (grub_addr_t) addr);
(gdb) p /x addr
$64 = 0x9df000
  *relstart = rels0;
  grub_free (sorted);
(gdb) p *relstart 
$67 = (void *) 0x9df0d0
  return GRUB_ERR_NONE;
}

```

LINKS:
=============================================================================================================

 * [Sweep line algorithm](https://en.wikipedia.org/wiki/Sweep_line_algorithm)
 * [Scanline rendering](https://en.wikipedia.org/wiki/Scanline_rendering)