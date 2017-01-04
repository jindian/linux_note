# load linux mode part II

This chapter focus on grub_cmd_linux.

grub_cmd_linux
   |--grub_file_open                   //open linux kernel executable and read linux header
   |--grub_file_read
   |--allocate_pages                   //allocate pages for linux and the memory map
       |--free_pages                   //reloacor and protect mode code intialization
       |--grub_relocator_new           //allocate memory for relocator and do initialization for relocator
           |--grub_cpu_relocator_init
           |--grub_zalloc
       |--grub_relocator_alloc_chunk_align   //allocate memory for protect mode code
           |--grub_malloc                    //allocate memory for chunk information
           |--malloc_in_range
           |--grub_mmap_iterate
   |--grub_memset                      //initialize linux prameters
   |--grub_memcpy                      //copy linux header information to linux parameters
   |--grub_file_close

```grub_cmd_linux

grub_cmd_linux (cmd=0x7f71f00, argc=2, argv=0x7fe1884) at loader/i386/linux.c:682

grub-core/loader/i386/linux.c:667

static grub_err_t
grub_cmd_linux (grub_command_t cmd __attribute__ ((unused)),
		int argc, char *argv[])
{
  grub_file_t file = 0;
  struct linux_kernel_header lh;
  struct linux_kernel_params *params;
  grub_uint8_t setup_sects;
  grub_size_t real_size, prot_size, prot_file_size;
  grub_ssize_t len;
  int i;
  grub_size_t align, min_align;
  int relocatable;
  grub_uint64_t preffered_address = GRUB_LINUX_BZIMAGE_ADDR;

  grub_dl_ref (my_mod);

(gdb) p argv[0]
$7 = 0x7fe17e0 "/boot/vmlinuz-2.6.32.69"
(gdb) p argv[1]
$8 = 0x7fe17c0 "root=/dev/sda"
  if (argc == 0)
    {
      grub_error (GRUB_ERR_BAD_ARGUMENT, N_("filename expected"));
      goto fail;
    }

  file = grub_file_open (argv[0]);
  if (! file)
    goto fail;

  if (grub_file_read (file, &lh, sizeof (lh)) != sizeof (lh))
    {
      if (!grub_errno)
	grub_error (GRUB_ERR_BAD_OS, N_("premature end of file %s"),
		    argv[0]);
      goto fail;
    }
(gdb) p lh
$10 = {
  code1 = "\352\005\000\300\a\214Ȏ؎\300\216\320\061\344\373\374\276-\000\254 \300t\t\264\016\273\a\000\315\020", cl_magic = 62187, cl_offset = 49201, 
  code2 = "\315\026\315\031\352\360\377\000\360Direct booting from floppy is no longer supported.\r\nPlease use a boot loader program instead.\r\n\nRemove disk and press any key to reboot . . .\r\n", '\000' <repeats 308 times>, 
  setup_sects = 29 '\035', root_flags = 1, syssize = 47475, swap_dev = 3, 
  ram_size = 0, vid_mode = 65535, root_dev = 2055, boot_flag = 43605, 
  jump = 25323, header = 1400005704, version = 522, realmode_swtch = 0, 
  start_sys = 4096, kernel_version = 12256, type_of_loader = 0 '\000', 
  loadflags = 1 '\001', setup_move_size = 32768, code32_start = 1048576, 
  ramdisk_image = 0, ramdisk_size = 0, bootsect_kludge = 0, 
  heap_end_ptr = 19776, pad1 = 0, cmd_line_ptr = 0, 
  initrd_addr_max = 2147483647, kernel_alignment = 16777216, 
  relocatable = 1 '\001', min_alignment = 13 '\r', pad = "\000", 
  cmdline_size = 2047, hardware_subarch = 0, hardware_subarch_data = 0, 
  payload_offset = 108, payload_length = 3888088, setup_data = 0, 
  pref_address = 16777216, init_size = 9302016}

  if (lh.boot_flag != grub_cpu_to_le16 (0xaa55))
    {
      grub_error (GRUB_ERR_BAD_OS, "invalid magic number");
      goto fail;
    }

  if (lh.setup_sects > GRUB_LINUX_MAX_SETUP_SECTS)
    {
      grub_error (GRUB_ERR_BAD_OS, "too many setup sectors");
      goto fail;
    }

  /* FIXME: 2.03 is not always good enough (Linux 2.4 can be 2.03 and
     still not support 32-bit boot.  */
  if (lh.header != grub_cpu_to_le32 (GRUB_LINUX_MAGIC_SIGNATURE)
      || grub_le_to_cpu16 (lh.version) < 0x0203)
    {
      grub_error (GRUB_ERR_BAD_OS, "version too old for 32-bit boot"
#ifdef GRUB_MACHINE_PCBIOS
		  " (try with `linux16')"
#endif
		  );
      goto fail;
    }

  if (! (lh.loadflags & GRUB_LINUX_FLAG_BIG_KERNEL))
    {
      grub_error (GRUB_ERR_BAD_OS, "zImage doesn't support 32-bit boot"
#ifdef GRUB_MACHINE_PCBIOS
		  " (try with `linux16')"
#endif
		  );
      goto fail;
    }

  if (grub_le_to_cpu16 (lh.version) >= 0x0206)
    maximal_cmdline_size = grub_le_to_cpu32 (lh.cmdline_size) + 1;
  else
    maximal_cmdline_size = 256;
(gdb) p maximal_cmdline_size
$11 = 2048

  if (maximal_cmdline_size < 128)
    maximal_cmdline_size = 128;

  setup_sects = lh.setup_sects;

  /* If SETUP_SECTS is not set, set it to the default (4).  */
  if (! setup_sects)
    setup_sects = GRUB_LINUX_DEFAULT_SETUP_SECTS;

  real_size = setup_sects << GRUB_DISK_SECTOR_BITS;
  prot_file_size = grub_file_size (file) - real_size - GRUB_DISK_SECTOR_SIZE;
(gdb) p real_size 
$18 = 14848
(gdb) p prot_file_size 
$19 = 3905328

  if (grub_le_to_cpu16 (lh.version) >= 0x205
      && lh.kernel_alignment != 0
      && ((lh.kernel_alignment - 1) & lh.kernel_alignment) == 0)
    {
      for (align = 0; align < 32; align++)
	if (grub_le_to_cpu32 (lh.kernel_alignment) & (1 << align))
	  break;
(gdb) p align
$21 = 24
      relocatable = grub_le_to_cpu32 (lh.relocatable);
(gdb) p relocatable
$22 = 1
    }
  else
    {
      align = 0;
      relocatable = 0;
    }
    
  if (grub_le_to_cpu16 (lh.version) >= 0x020a)
    {
      min_align = lh.min_alignment;
      prot_size = grub_le_to_cpu32 (lh.init_size);
      prot_init_space = page_align (prot_size);
      if (relocatable)
	preffered_address = grub_le_to_cpu64 (lh.pref_address);
      else
	preffered_address = GRUB_LINUX_BZIMAGE_ADDR;
(gdb) p min_align
$23 = 13
(gdb) p prot_size 
$24 = 9302016
(gdb) p prot_init_space 
$25 = 9302016
(gdb) p preffered_address 
$26 = 1048576
    }
  else
    {
      min_align = align;
      prot_size = prot_file_size;
      preffered_address = GRUB_LINUX_BZIMAGE_ADDR;
      /* Usually, the compression ratio is about 50%.  */
      prot_init_space = page_align (prot_size) * 3;
    }

  if (allocate_pages (prot_size, &align,
		      min_align, relocatable,
		      preffered_address))
    goto fail;

  params = (struct linux_kernel_params *) &linux_params;
  grub_memset (params, 0, sizeof (*params));
  grub_memcpy (&params->setup_sects, &lh.setup_sects, sizeof (lh) - 0x1F1);

  params->code32_start = prot_mode_target + lh.code32_start - GRUB_LINUX_BZIMAGE_ADDR;
  params->kernel_alignment = (1 << align);
  params->ps_mouse = params->padding10 =  0;
(gdb) p params
$9 = (struct linux_kernel_params *) 0x7f73ad8 <linux_params>
(gdb) p *params
$10 = {video_cursor_x = 0 '\000', video_cursor_y = 0 '\000', ext_mem = 0, 
  video_page = 0, video_mode = 0 '\000', video_width = 0 '\000', 
  padding1 = "\000", video_ega_bx = 0, padding2 = "\000", 
  video_height = 0 '\000', have_vga = 0 '\000', font_size = 0, lfb_width = 0, 
  lfb_height = 0, lfb_depth = 0, lfb_base = 0, lfb_size = 0, cl_magic = 0, 
  cl_offset = 0, lfb_line_len = 0, red_mask_size = 0 '\000', 
  red_field_pos = 0 '\000', green_mask_size = 0 '\000', 
  green_field_pos = 0 '\000', blue_mask_size = 0 '\000', 
  blue_field_pos = 0 '\000', reserved_mask_size = 0 '\000', 
  reserved_field_pos = 0 '\000', vesapm_segment = 0, vesapm_offset = 0, 
  lfb_pages = 0, vesa_attrib = 0, capabilities = 0, 
  padding3 = "\000\000\000\000\000", apm_version = 0, apm_code_segment = 0, 
  apm_entry = 0, apm_16bit_code_segment = 0, apm_data_segment = 0, 
  apm_flags = 0, apm_code_len = 0, apm_data_len = 0, 
  padding4 = '\000' <repeats 11 times>, ist_signature = 0, ist_command = 0, 
  ist_event = 0, ist_perf_level = 0, padding5 = '\000' <repeats 15 times>, 
  hd0_drive_info = '\000' <repeats 15 times>, 
  hd1_drive_info = '\000' <repeats 15 times>, rom_config_len = 0, 
  padding6 = '\000' <repeats 13 times>, ofw_signature = 0, ofw_num_items = 0, 
  ofw_cif_handler = 0, ofw_idt = 0, padding7 = '\000' <repeats 247 times>, {
    v0204 = {efi_system_table = 0, padding7_1 = 0, efi_signature = 0, 
      efi_mem_desc_size = 0, efi_mem_desc_version = 0, efi_mmap_size = 0, 
      efi_mmap = 0}, v0206 = {padding7_1 = 0, padding7_2 = 0, 
      efi_signature = 0, efi_system_table = 0, efi_mem_desc_size = 0, 
      efi_mem_desc_version = 0, efi_mmap = 0, efi_mmap_size = 0}, v0208 = {
      padding7_1 = 0, padding7_2 = 0, efi_signature = 0, 
      efi_system_table = 0, efi_mem_desc_size = 0, efi_mem_desc_version = 0, 
      efi_mmap = 0, efi_mmap_size = 0, efi_system_table_hi = 0, 
      efi_mmap_hi = 0}}, alt_mem = 0, padding8 = "\000\000\000", 
  mmap_size = 0 '\000', padding9 = "\000\000\000\000\000\000\000", 
  setup_sects = 29 '\035', root_flags = 1, syssize = 47475, swap_dev = 3, 
  ram_size = 0, vid_mode = 65535, root_dev = 2055, padding10 = 85 'U', 
  ps_mouse = 170 '\252', jump = 25323, header = 1400005704, version = 522, 
  realmode_swtch = 0, start_sys = 4096, kernel_version = 12256, 
  type_of_loader = 0 '\000', loadflags = 1 '\001', setup_move_size = 32768, 
  code32_start = 1048576, ramdisk_image = 0, ramdisk_size = 0, 
  bootsect_kludge = 0, heap_end_ptr = 19776, ext_loader_ver = 0 '\000', 
  ext_loader_type = 0 '\000', cmd_line_ptr = 0, initrd_addr_max = 2147483647, 
  kernel_alignment = 16777216, relocatable_kernel = 1 '\001', 
  pad1 = "\r\000", cmdline_size = 2047, hardware_subarch = 0, 
  hardware_subarch_data = 0, payload_offset = 108, payload_length = 3888088, 
  setup_data = 0, 
  pad2 = "\000\000\000\001\000\000\000\000\000\360\215", '\000' <repeats 108 times>, e820_map = {{addr = 0, size = 0, type = 0} <repeats 15 times>}}

  len = sizeof (*params) - sizeof (lh);
  if (grub_file_read (file, (char *) params + sizeof (lh), len) != len)
    {
      if (!grub_errno)
	grub_error (GRUB_ERR_BAD_OS, N_("premature end of file %s"),
		    argv[0]);
      goto fail;
    }

  params->type_of_loader = GRUB_LINUX_BOOT_LOADER_TYPE;

  /* These two are used (instead of cmd_line_ptr) by older versions of Linux,
     and otherwise ignored.  */
  params->cl_magic = GRUB_LINUX_CL_MAGIC;
  params->cl_offset = 0x1000;

  params->ramdisk_image = 0;
  params->ramdisk_size = 0;

  params->heap_end_ptr = GRUB_LINUX_HEAP_END_OFFSET;
  params->loadflags |= GRUB_LINUX_FLAG_CAN_USE_HEAP;

  /* These are not needed to be precise, because Linux uses these values
     only to raise an error when the decompression code cannot find good
     space.  */
  params->ext_mem = ((32 * 0x100000) >> 10);
  params->alt_mem = ((32 * 0x100000) >> 10);

  /* Ignored by Linux.  */
  params->video_page = 0;

  /* Only used when `video_mode == 0x7', otherwise ignored.  */
  params->video_ega_bx = 0;

  params->font_size = 16; /* XXX */

#ifdef GRUB_MACHINE_EFI
#ifdef __x86_64__
  if (grub_le_to_cpu16 (params->version < 0x0208) &&
      ((grub_addr_t) grub_efi_system_table >> 32) != 0)
    return grub_error(GRUB_ERR_BAD_OS,
		      "kernel does not support 64-bit addressing");
#endif

  if (grub_le_to_cpu16 (params->version) >= 0x0208)
    {
      params->v0208.efi_signature = GRUB_LINUX_EFI_SIGNATURE;
      params->v0208.efi_system_table = (grub_uint32_t) (unsigned long) grub_efi_system_table;
#ifdef __x86_64__
      params->v0208.efi_system_table_hi = (grub_uint32_t) ((grub_uint64_t) grub_efi_system_table >> 32);
#endif
    }
  else if (grub_le_to_cpu16 (params->version) >= 0x0206)
    {
      params->v0206.efi_signature = GRUB_LINUX_EFI_SIGNATURE;
      params->v0206.efi_system_table = (grub_uint32_t) (unsigned long) grub_efi_system_table;
    }
  else if (grub_le_to_cpu16 (params->version) >= 0x0204)
    {
      params->v0204.efi_signature = GRUB_LINUX_EFI_SIGNATURE_0204;
      params->v0204.efi_system_table = (grub_uint32_t) (unsigned long) grub_efi_system_table;
    }
#endif

  /* The other parameters are filled when booting.  */

  grub_file_seek (file, real_size + GRUB_DISK_SECTOR_SIZE);

  grub_dprintf ("linux", "bzImage, setup=0x%x, size=0x%x\n",
		(unsigned) real_size, (unsigned) prot_size);

  /* Look for memory size and video mode specified on the command line.  */
  linux_mem_size = 0;
  for (i = 1; i < argc; i++)
#ifdef GRUB_MACHINE_PCBIOS
    if (grub_memcmp (argv[i], "vga=", 4) == 0)
      {
	/* Video mode selection support.  */
	char *val = argv[i] + 4;
	unsigned vid_mode = GRUB_LINUX_VID_MODE_NORMAL;
	struct grub_vesa_mode_table_entry *linux_mode;
	grub_err_t err;
	char *buf;

	grub_dl_load ("vbe");

	if (grub_strcmp (val, "normal") == 0)
	  vid_mode = GRUB_LINUX_VID_MODE_NORMAL;
	else if (grub_strcmp (val, "ext") == 0)
	  vid_mode = GRUB_LINUX_VID_MODE_EXTENDED;
	else if (grub_strcmp (val, "ask") == 0)
	  {
	    grub_puts_ (N_("Legacy `ask' parameter no longer supported."));

	    /* We usually would never do this in a loader, but "vga=ask" means user
	       requested interaction, so it can't hurt to request keyboard input.  */
	    grub_wait_after_message ();

	    goto fail;
	  }
	else
	  vid_mode = (grub_uint16_t) grub_strtoul (val, 0, 0);

	switch (vid_mode)
	  {
	  case 0:
	  case GRUB_LINUX_VID_MODE_NORMAL:
	    grub_env_set ("gfxpayload", "text");
	    grub_printf_ (N_("%s is deprecated. "
			     "Use set gfxpayload=%s before "
			     "linux command instead.\n"), "text",
			  argv[i]);
	    break;

	  case 1:
	  case GRUB_LINUX_VID_MODE_EXTENDED:
	    /* FIXME: support 80x50 text. */
	    grub_env_set ("gfxpayload", "text");
	    grub_printf_ (N_("%s is deprecated. "
			     "Use set gfxpayload=%s before "
			     "linux command instead.\n"), "text",
			  argv[i]);
	    break;
	  default:
	    /* Ignore invalid values.  */
	    if (vid_mode < GRUB_VESA_MODE_TABLE_START ||
		vid_mode > GRUB_VESA_MODE_TABLE_END)
	      {
		grub_env_set ("gfxpayload", "text");
		/* TRANSLATORS: "x" has to be entered in, like an identifier,
		   so please don't use better Unicode codepoints.  */
		grub_printf_ (N_("%s is deprecated. VGA mode %d isn't recognized. "
				 "Use set gfxpayload=WIDTHxHEIGHT[xDEPTH] "
				 "before linux command instead.\n"),
			     argv[i], vid_mode);
		break;
	      }

	    linux_mode = &grub_vesa_mode_table[vid_mode
					       - GRUB_VESA_MODE_TABLE_START];

	    buf = grub_xasprintf ("%ux%ux%u,%ux%u",
				 linux_mode->width, linux_mode->height,
				 linux_mode->depth,
				 linux_mode->width, linux_mode->height);
	    if (! buf)
	      goto fail;

	    grub_printf_ (N_("%s is deprecated. "
			     "Use set gfxpayload=%s before "
			     "linux command instead.\n"),
			 argv[i], buf);
	    err = grub_env_set ("gfxpayload", buf);
	    grub_free (buf);
	    if (err)
	      goto fail;
	  }
      }
    else
#endif /* GRUB_MACHINE_PCBIOS */
    if (grub_memcmp (argv[i], "mem=", 4) == 0)
      {
	char *val = argv[i] + 4;

	linux_mem_size = grub_strtoul (val, &val, 0);

	if (grub_errno)
	  {
	    grub_errno = GRUB_ERR_NONE;
	    linux_mem_size = 0;
	  }
	else
	  {
	    int shift = 0;

	    switch (grub_tolower (val[0]))
	      {
	      case 'g':
		shift += 10;
	      case 'm':
		shift += 10;
	      case 'k':
		shift += 10;
	      default:
		break;
	      }

	    /* Check an overflow.  */
	    if (linux_mem_size > (~0UL >> shift))
	      linux_mem_size = 0;
	    else
	      linux_mem_size <<= shift;
	  }
      }
    else if (grub_memcmp (argv[i], "quiet", sizeof ("quiet") - 1) == 0)
      {
	params->loadflags |= GRUB_LINUX_FLAG_QUIET;
      }

  /* Create kernel command line.  */
  linux_cmdline = grub_zalloc (maximal_cmdline_size + 1);
  if (!linux_cmdline)
    goto fail;
  grub_memcpy (linux_cmdline, LINUX_IMAGE, sizeof (LINUX_IMAGE));
  grub_create_loader_cmdline (argc, argv,
			      linux_cmdline
			      + sizeof (LINUX_IMAGE) - 1,
			      maximal_cmdline_size
			      - (sizeof (LINUX_IMAGE) - 1));

  len = prot_file_size;
  if (grub_file_read (file, prot_mode_mem, len) != len && !grub_errno)
    grub_error (GRUB_ERR_BAD_OS, N_("premature end of file %s"),
		argv[0]);

  if (grub_errno == GRUB_ERR_NONE)
    {
      grub_loader_set (grub_linux_boot, grub_linux_unload,
		       0 /* set noreturn=0 in order to avoid grub_console_fini() */);
      loaded = 1;
    }

 fail:

  if (file)
    grub_file_close (file);

  if (grub_errno != GRUB_ERR_NONE)
    {
      grub_dl_unref (my_mod);
      loaded = 0;
    }

  return grub_errno;
}
```

Initialize relocator, it will be used in relocator later, allocate memory for real mode, protect mode code and memory map.

```allocate_pages

allocate_pages (prefered_address=<optimized out>, 
    relocatable=<optimized out>, min_align=<optimized out>, 
    align=<optimized out>, prot_size=<optimized out>)
    at loader/i386/linux.c:788

grub-core/loader/i386/linux.c:186

/* Allocate pages for the real mode code and the protected mode code
   for linux as well as a memory map buffer.  */
static grub_err_t
allocate_pages (grub_size_t prot_size, grub_size_t *align,
		grub_size_t min_align, int relocatable,
		grub_uint64_t prefered_address)
{
  grub_err_t err;

  prot_size = page_align (prot_size);
(gdb) p prot_size 
$1 = 9302016

  /* Initialize the memory pointers with NULL for convenience.  */
  free_pages ();

  relocator = grub_relocator_new ();
  if (!relocator)
    {
      err = grub_errno;
      goto fail;
    }
(gdb) p relocator
$16 = (struct grub_relocator *) 0x7fdb100
(gdb) p* relocator
$17 = {chunks = 0x0, postchunks = 4294967295, highestaddr = 0, highestnonpostaddr = 0, relocators_size = 7}

  /* FIXME: Should request low memory from the heap when this feature is
     implemented.  */

  {
    grub_relocator_chunk_t ch;
    if (relocatable)
      {
	err = grub_relocator_alloc_chunk_align (relocator, &ch,
						prefered_address,
						prefered_address,
						prot_size, 1,
						GRUB_RELOCATOR_PREFERENCE_LOW,
						1);
grub_relocator_alloc_chunk_align (rel=0x7fdb100, out=out@entry=0x7faac, avoid_efi_boot_services=1) at lib/relocator.c:1317
# below parameters didn't show in the input, add it here
(gdb) p /x min_addr
$20 = 0x1000000
(gdb) p /x max_addr
$21 = 0x1000000
(gdb) p align 
$22 = 1
(gdb) p /x size
$24 = 0x8df000
(gdb) p /x preference 
$25 = 0x1

(gdb) print_chunks relocator
chunk: 0x7fdb0d0, src = 0x100000, target = 0x1000000, srcv = 0x100000, size = 0x8df000

	for (; err && *align + 1 > min_align; (*align)--)
	  {
	    grub_errno = GRUB_ERR_NONE;
	    err = grub_relocator_alloc_chunk_align (relocator, &ch,
						    0x1000000,
						    0xffffffff & ~prot_size,
						    prot_size, 1 << *align,
						    GRUB_RELOCATOR_PREFERENCE_LOW,
						    1);
	  }
	if (err)
	  goto fail;
      }
    else
      err = grub_relocator_alloc_chunk_addr (relocator, &ch,
					     prefered_address,
					     prot_size);
    if (err)
      goto fail;
    prot_mode_mem = get_virtual_current_address (ch);
    prot_mode_target = get_physical_target_address (ch);
  }

  grub_dprintf ("linux", "prot_mode_mem = %lx, prot_mode_target = %lx, prot_size = %x\n",
                (unsigned long) prot_mode_mem, (unsigned long) prot_mode_target,
		(unsigned) prot_size);
(gdb) p prot_mode_mem 
$5 = (void *) 0x100000
(gdb) p /x prot_mode_target 
$8 = 0x1000000

  return GRUB_ERR_NONE;

 fail:
  free_pages ();
  return err;
}
```




```grub_relocator_alloc_chunk_align

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
    max_addr = ~size;

#ifdef GRUB_MACHINE_PCBIOS
  if (min_addr < 0x1000)
    min_addr = 0x1000;
#endif

  grub_dprintf ("relocator", "chunks = %p\n", rel->chunks);

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
      chunk->target = chunk->src;
      chunk->size = size;
      chunk->next = rel->chunks;
      rel->chunks = chunk;
      chunk->srcv = grub_map_memory (chunk->src, chunk->size);
      *out = chunk;
      return GRUB_ERR_NONE;
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


# LINKS
  * [Vmlinux](https://en.wikipedia.org/wiki/Vmlinux)