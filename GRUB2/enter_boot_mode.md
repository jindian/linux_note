Enter Boot Mode
==============================================================================================================

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

1. Set video mode with grub_video_set_mode.
2. Setup video mode with grub_linux_setup_video
3. Initialize video parameters



Call context of grub_linux_boot

```context_grub_linux_boot
grub_linux_boot
    |--grub_video_set_mode
    |--grub_linux_setup_video
        |--grub_video_get_driver_id
    |--find_mmap_size
        |--grub_mmap_iterate

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
(gdb) s
grub_video_set_mode (modestring=modestring@entry=0x7f734d6 "text", 
    modemask=modemask@entry=0, modevalue=modevalue@entry=0)
    at video/video.c:493
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

  if (! real_mode_target)
    return grub_error (GRUB_ERR_OUT_OF_MEMORY, "cannot allocate real mode pages");

  {
    grub_relocator_chunk_t ch;
    err = grub_relocator_alloc_chunk_addr (relocator, &ch,
                                           real_mode_target,
                                           (real_size + efi_mmap_size));
    if (err)
     return err;
    real_mode_mem = get_virtual_current_address (ch);
  }
  efi_mmap_buf = (grub_uint8_t *) real_mode_mem + real_size;

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

------------------------------------------------------------------------------------------------------------------

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

------------------------------------------------------------------------------------------------------------------

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

```memory_map
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

  /* Increase the size a bit for safety, because GRUB allocates more on
     later.  */
  mmap_size += (1 << 12);

  return page_align (mmap_size);
}

```

LINKS:
==================================================================================================================

 * [Sweep line algorithm](https://en.wikipedia.org/wiki/Sweep_line_algorithm)
 * [Scanline rendering](https://en.wikipedia.org/wiki/Scanline_rendering)