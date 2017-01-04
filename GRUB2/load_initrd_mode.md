# load initrd module

Because it still executing configuration in grub.cfg, before involing grub_cmd_initrd, the call context of load initrd module is same as load linux module as follow, let's start studying load initrd module from grub_cmd_initrd.

grub_script_execute_sourcecode
    |--grub_script_parse
    |--grub_script_execute
        |--grub_script_execute_cmd
            |--grub_script_execute_cmdlist
                |--grub_script_execute_cmd
                    |--grub_script_execute_cmdline
                        |--grub_cmd_initrd
                            |--grub_file_open              //open inird file
                            |--grub_file_size              //for size alignment
                            |--grub_relocator_alloc_chunk_align   //allocate memory for initrd protect mode code
                            |--grub_file_read              //read initrd file
                            |--grub_file_close             //close initrd file
                            

```grub_cmd_initrd

grub_cmd_initrd (cmd=0x7f71ec0, argc=1, argv=0x7fe1854) at loader/i386/linux.c:1050

grub-core/loader/i386/linux.c:1037

static grub_err_t
grub_cmd_initrd (grub_command_t cmd __attribute__ ((unused)),
		 int argc, char *argv[])
{
  grub_file_t *files = 0;
  grub_size_t size = 0;
  grub_addr_t addr_min, addr_max;
  grub_addr_t addr;
  grub_err_t err;
  int i;
  int nfiles = 0;
  grub_uint8_t *ptr;

(gdb) p argv[0]
$20 = 0x7fe17b0 "/boot/initrd.img-2.6.32.69"
  if (argc == 0)
    {
      grub_error (GRUB_ERR_BAD_ARGUMENT, N_("filename expected"));
      goto fail;
    }

  if (! loaded)
    {
      grub_error (GRUB_ERR_BAD_ARGUMENT, N_("you need to load the kernel first"));
      goto fail;
    }

  files = grub_zalloc (argc * sizeof (files[0]));
  if (!files)
    goto fail;

  for (i = 0; i < argc; i++)
    {
      grub_file_filter_disable_compression ();
      files[i] = grub_file_open (argv[i]);
      if (! files[i])
	goto fail;
      nfiles++;
      size += ALIGN_UP (grub_file_size (files[i]), 4);
    }
(gdb) p size
$23 = 2688716

  initrd_pages = (page_align (size) >> 12);
(gdb) p initrd_pages 
$29 = 657

  /* Get the highest address available for the initrd.  */
  if (grub_le_to_cpu16 (linux_params.version) >= 0x0203)
    {
      addr_max = grub_cpu_to_le32 (linux_params.initrd_addr_max);

      /* XXX in reality, Linux specifies a bogus value, so
	 it is necessary to make sure that ADDR_MAX does not exceed
	 0x3fffffff.  */
      if (addr_max > GRUB_LINUX_INITRD_MAX_ADDRESS)
	addr_max = GRUB_LINUX_INITRD_MAX_ADDRESS;
    }
  else
    addr_max = GRUB_LINUX_INITRD_MAX_ADDRESS;

  if (linux_mem_size != 0 && linux_mem_size < addr_max)
    addr_max = linux_mem_size;

  /* Linux 2.3.xx has a bug in the memory range check, so avoid
     the last page.
     Linux 2.2.xx has a bug in the memory range check, which is
     worse than that of Linux 2.3.xx, so avoid the last 64kb.  */
  addr_max -= 0x10000;

  addr_min = (grub_addr_t) prot_mode_target + prot_init_space
             + page_align (size);
(gdb) p /x prot_mode_target 
$36 = 0x1000000
(gdb) p /x prot_init_space 
$37 = 0x8df000

  /* Put the initrd as high as possible, 4KiB aligned.  */
  addr = (addr_max - size) & ~0xFFF;
(gdb) p /x addr
$39 = 0x37d5f000
(gdb) p/x  addr_min
$40 = 0x1b70000

  if (addr < addr_min)
    {
      grub_error (GRUB_ERR_OUT_OF_RANGE, "the initrd is too big");
      goto fail;
    }

  {
    grub_relocator_chunk_t ch;
    err = grub_relocator_alloc_chunk_align (relocator, &ch,
					    addr_min, addr, size, 0x1000,
					    GRUB_RELOCATOR_PREFERENCE_HIGH,
					    1);
(gdb) print_chunks relocator 
chunk: 0x7fda880, src = 0x782d000, target = 0x782d000, srcv = 0x782d000, size = 0x2906cc
chunk: 0x7fdb0d0, src = 0x100000, target = 0x1000000, srcv = 0x100000, size = 0x8df000
    if (err)
      return err;
    initrd_mem = get_virtual_current_address (ch);
    initrd_mem_target = get_physical_target_address (ch);
(gdb) p initrd_mem
$41 = (void *) 0x782d000
(gdb) p /x initrd_mem_target 
$43 = 0x782d000
  }

  ptr = initrd_mem;
  for (i = 0; i < nfiles; i++)
    {
      grub_ssize_t cursize = grub_file_size (files[i]);
(gdb) p /x cursize 
$46 = 0x2906c9
      if (grub_file_read (files[i], ptr, cursize) != cursize)
	{
	  if (!grub_errno)
	    grub_error (GRUB_ERR_FILE_READ_ERROR, N_("premature end of file %s"),
			argv[i]);
	  goto fail;
	}
      ptr += cursize;
      grub_memset (ptr, 0, ALIGN_UP_OVERHEAD (cursize, 4));
      ptr += ALIGN_UP_OVERHEAD (cursize, 4);
    }

  grub_dprintf ("linux", "Initrd, addr=0x%x, size=0x%x\n",
		(unsigned) addr, (unsigned) size);
(gdb) p /x addr
$48 = 0x37d5f000
(gdb) p /x size
$49 = 0x2906cc

  linux_params.ramdisk_image = initrd_mem_target;
  linux_params.ramdisk_size = size;
  linux_params.root_dev = 0x0100; /* XXX */

 fail:
  for (i = 0; i < nfiles; i++)
    grub_file_close (files[i]);
  grub_free (files);
(gdb) p linux_params 
$51 = {video_cursor_x = 0 '\000', video_cursor_y = 0 '\000', ext_mem = 32768, 
  video_page = 0, video_mode = 0 '\000', video_width = 0 '\000', 
  padding1 = "\000", video_ega_bx = 0, padding2 = "\000", 
  video_height = 0 '\000', have_vga = 0 '\000', font_size = 16, 
  lfb_width = 0, lfb_height = 0, lfb_depth = 0, lfb_base = 0, lfb_size = 0, 
  cl_magic = 41791, cl_offset = 4096, lfb_line_len = 0, 
  red_mask_size = 0 '\000', red_field_pos = 0 '\000', 
  green_mask_size = 0 '\000', green_field_pos = 0 '\000', 
  blue_mask_size = 0 '\000', blue_field_pos = 0 '\000', 
  reserved_mask_size = 0 '\000', reserved_field_pos = 0 '\000', 
  vesapm_segment = 0, vesapm_offset = 0, lfb_pages = 0, vesa_attrib = 0, 
  capabilities = 0, padding3 = "\000\000\000\000\000", apm_version = 0, 
  apm_code_segment = 0, apm_entry = 0, apm_16bit_code_segment = 0, 
  apm_data_segment = 0, apm_flags = 0, apm_code_len = 0, apm_data_len = 0, 
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
      efi_mmap_hi = 0}}, alt_mem = 32768, padding8 = "\000\000\000", 
  mmap_size = 0 '\000', padding9 = "\000\000\000\000\000\000\000", 
  setup_sects = 29 '\035', root_flags = 1, syssize = 47475, swap_dev = 3, 
  ram_size = 0, vid_mode = 65535, root_dev = 256, padding10 = 0 '\000', 
  ps_mouse = 0 '\000', jump = 25323, header = 1400005704, version = 522, 
  realmode_swtch = 0, start_sys = 4096, kernel_version = 12256, 
  type_of_loader = 114 'r', loadflags = 129 '\201', setup_move_size = 32768, 
  code32_start = 16777216, ramdisk_image = 126013440, ramdisk_size = 2688716, 
  bootsect_kludge = 0, heap_end_ptr = 36352, ext_loader_ver = 0 '\000', 
  ext_loader_type = 0 '\000', cmd_line_ptr = 0, initrd_addr_max = 2147483647, 
  kernel_alignment = 16777216, relocatable_kernel = 1 '\001', 
  pad1 = "\r\000", cmdline_size = 2047, hardware_subarch = 0, 
  hardware_subarch_data = 0, payload_offset = 108, payload_length = 3888088, 
  setup_data = 0, 
  pad2 = "\000\000\000\001\000\000\000\000\000\360\215\000\214؎\300\374\214\322\071\302\211\342t\026\272@M\366\006\021\002\200t\004\213\026$\002\201\302\000\
002s\002\061҃\342\374u\003\272\374\377\216\320f\017\267\342\373\036h\233\002\313f\201>|:U\252ZZu\027\277\200:\271CMf1\300)\371\301\351\002\363f\253f\350\201*\000\000f\270@\003\000\000f\350[\000\000\000\364\351\374\377fSf", e820_map = {
    {addr = 9468470261485202563, size = 3861923132410616, type = 3898998784}, 
    {addr = 7413076062126473189, size = 4956043164643610856, type = 462884}, {
      addr = 7421933389105907559, size = 6667692558638138566, 
      type = 828775460}, {addr = 4706351654201033, size = 1782015459328, 
      type = 751076198}, {addr = 9900692531650648934, 
      size = 13872216867045009347, type = 1130760820}, {
      addr = 17144077881485551718, size = 2337172526878579558, 
      type = 1970562419}, {addr = 8386105371570413680, 
      size = 7959390400868086389, type = 774778468}, {addr = 429504151814154, 
      size = 5360972406433298790, type = 2162564980}, {
      addr = 5288919717233059044, size = 9288118376811991924, 
      type = 3874185444}, {addr = 7403972105352839848, 
      size = 7404875461093975747, type = 3526452819}, {
      addr = 16901727696821084814, size = 9900603030087886436, 
      type = 2372298698}, {addr = 7356195521970962716, 
      size = 16573809675471316034, type = 2707842432}, {
      addr = 16998683756064014864, size = 1047480971247576579, 
      type = 1533411840}, {addr = 9468346965594391398, size = 1102655663340, 
      type = 2215594854}, {addr = 140532646082, size = 9612651929639184486, 
      type = 1711699136}}}

  return grub_errno;
}
```

# LINKS
  * [initrd](https://en.wikipedia.org/wiki/Initrd)
  * [file system initrd](http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/initrd.html)