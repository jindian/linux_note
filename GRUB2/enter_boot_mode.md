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

The context of boot mode in initialization:

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

```grub_linux_boot
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
#endif
    {
      grub_term_output_t term;
      int found = 0;
      FOR_ACTIVE_TERM_OUTPUTS(term)
        if (grub_strcmp (term->name, "vga_text") == 0
            || grub_strcmp (term->name, "console") == 0
            || grub_strcmp (term->name, "ofconsole") == 0)
          {
            grub_uint16_t pos = grub_term_getxy (term);
            linux_params.video_cursor_x = pos >> 8;
            linux_params.video_cursor_y = pos & 0xff;
            linux_params.video_width = grub_term_width (term);
            linux_params.video_height = grub_term_height (term);
            found = 1;
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