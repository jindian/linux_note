# linux real mode code

linux kernel real mode code is arch/x86/boot/header.S.

In the early days of linux it have to be a bootable image, so in arch/x86/boot/header.S first part of the code is same as MBR, linux boot loader is not used any more, but its data section presents part of linux boot protocol, it will be used later in linux scratch code.

```linux_boot_code

arch/x86/boot/header.S:43

        .code16
        .section ".bstext", "ax"

        .global bootsect_start
bootsect_start:

        # Normalize the start address
        ljmp    $BOOTSEG, $start2

start2:
        movw    %cs, %ax
        movw    %ax, %ds
        movw    %ax, %es
        movw    %ax, %ss
        xorw    %sp, %sp
        sti
        cld

        movw    $bugger_off_msg, %si

msg_loop:
        lodsb
        andb    %al, %al
        jz      bs_die
        movb    $0xe, %ah
        movw    $7, %bx
        int     $0x10
        jmp     msg_loop

bs_die:
        # Allow the user to press a key, then reboot
        xorw    %ax, %ax
        int     $0x16
        int     $0x19

        # int 0x19 should never return.  In case it does anyway,
        # invoke the BIOS reset code...
        ljmp    $0xf000,$0xfff0

        .section ".bsdata", "a"
bugger_off_msg:
        .ascii  "Direct booting from floppy is no longer supported.\r\n"
        .ascii  "Please use a boot loader program instead.\r\n"
        .ascii  "\n"
        .ascii  "Remove disk and press any key to reboot . . .\r\n"
        .byte   0


        # Kernel attributes; used by setup.  This is part 1 of the
        # header, from the old boot sector.

        .section ".header", "a"
        .globl  hdr
hdr:
setup_sects:    .byte 0                 /* Filled in by build.c */
root_flags:     .word ROOT_RDONLY
syssize:        .long 0                 /* Filled in by build.c */
ram_size:       .word 0                 /* Obsolete */
vid_mode:       .word SVGA_MODE
root_dev:       .word 0                 /* Filled in by build.c */
boot_flag:      .word 0xAA55
```

linux real mode code memory deployment could be found in file arch/x86/boot/setup.ld, offset of boot_flag is 510, it presents the signature of master boot recorder.

```setup

/*
 * setup.ld
 *
 * Linker script for the i386 setup code
 */
OUTPUT_FORMAT("elf32-i386", "elf32-i386", "elf32-i386")
OUTPUT_ARCH(i386)
ENTRY(_start)

SECTIONS
{
        . = 0;
        .bstext         : { *(.bstext) }
        .bsdata         : { *(.bsdata) }

        . = 497;
        .header         : { *(.header) }
        .entrytext      : { *(.entrytext) }
        .inittext       : { *(.inittext) }
        .initdata       : { *(.initdata) }
        __end_init = .;

        .text           : { *(.text) }
        .text32         : { *(.text32) }

        . = ALIGN(16);
        .rodata         : { *(.rodata*) }

        .videocards     : {
                video_cards = .;
                *(.videocards)
                video_cards_end = .;
        }

        . = ALIGN(16);
        .data           : { *(.data*) }

        .signature      : {
                setup_sig = .;
                LONG(0x5a5aaa55)
        }


        . = ALIGN(16);
        .bss            :
        {
                __bss_start = .;
                *(.bss)
                __bss_end = .;
        }
        . = ALIGN(16);
        _end = .;

        /DISCARD/ : { *(.note*) }

        /*
         * The ASSERT() sink to . is intentional, for binutils 2.14 compatibility:
         */
        . = ASSERT(_end <= 0x8000, "Setup too big!");
        . = ASSERT(hdr == 0x1f1, "The setup header has the wrong offset!");
        /* Necessary for the very-old-loader check to work... */
        . = ASSERT(__end_init <= 5*512, "init sections too big!");

}
```

Rest part of linux boot protocol follows master boot recorder code, for detail of header field could be found in [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)

```linux_boot_protocol_part_2

arch/x86/boot/header.S:105

	# offset 512, entry point

	.globl	_start
_start:
		# Explicitly enter this as bytes, or the assembler
		# tries to generate a 3-byte jump here, which causes
		# everything else to push off to the wrong offset.
		.byte	0xeb		# short (2-byte) jump
		.byte	start_of_setup-1f
1:

	# Part 2 of the header, from the old setup.S

		.ascii	"HdrS"		# header signature
		.word	0x020a		# header version number (>= 0x0105)
					# or else old loadlin-1.5 will fail)
		.globl realmode_swtch
realmode_swtch:	.word	0, 0		# default_switch, SETUPSEG
start_sys_seg:	.word	SYSSEG		# obsolete and meaningless, but just
					# in case something decided to "use" it
		.word	kernel_version-512 # pointing to kernel version string
					# above section of header is compatible
					# with loadlin-1.5 (header v1.5). Don't
					# change it.

type_of_loader:	.byte	0		# 0 means ancient bootloader, newer
					# bootloaders know to change this.
					# See Documentation/i386/boot.txt for
					# assigned ids

# flags, unused bits must be zero (RFU) bit within loadflags
loadflags:
LOADED_HIGH	= 1			# If set, the kernel is loaded high
CAN_USE_HEAP	= 0x80			# If set, the loader also has set
					# heap_end_ptr to tell how much
					# space behind setup.S can be used for
					# heap purposes.
					# Only the loader knows what is free
		.byte	LOADED_HIGH

setup_move_size: .word  0x8000		# size to move, when setup is not
					# loaded at 0x90000. We will move setup
					# to 0x90000 then just before jumping
					# into the kernel. However, only the
					# loader knows how much data behind
					# us also needs to be loaded.

code32_start:				# here loaders can put a different
					# start address for 32-bit code.
		.long	0x100000	# 0x100000 = default for big kernel

ramdisk_image:	.long	0		# address of loaded ramdisk image
					# Here the loader puts the 32-bit
					# address where it loaded the image.
					# This only will be read by the kernel.

ramdisk_size:	.long	0		# its size in bytes

bootsect_kludge:
		.long	0		# obsolete

heap_end_ptr:	.word	_end+STACK_SIZE-512
					# (Header version 0x0201 or later)
					# space from here (exclusive) down to
					# end of setup code can be used by setup
					# for local heap purposes.

ext_loader_ver:
		.byte	0		# Extended boot loader version
ext_loader_type:
		.byte	0		# Extended boot loader type

cmd_line_ptr:	.long	0		# (Header version 0x0202 or later)
					# If nonzero, a 32-bit pointer
					# to the kernel command line.
					# The command line should be
					# located between the start of
					# setup and the end of low
					# memory (0xa0000), or it may
					# get overwritten before it
					# gets read.  If this field is
					# used, there is no longer
					# anything magical about the
					# 0x90000 segment; the setup
					# can be located anywhere in
					# low memory 0x10000 or higher.

ramdisk_max:	.long 0x7fffffff
					# (Header version 0x0203 or later)
					# The highest safe address for
					# the contents of an initrd
					# The current kernel allows up to 4 GB,
					# but leave it at 2 GB to avoid
					# possible bootloader bugs.

kernel_alignment:  .long CONFIG_PHYSICAL_ALIGN	#physical addr alignment
						#required for protected mode
						#kernel
#ifdef CONFIG_RELOCATABLE
relocatable_kernel:    .byte 1
#else
relocatable_kernel:    .byte 0
#endif
min_alignment:		.byte MIN_KERNEL_ALIGN_LG2	# minimum alignment
pad3:			.word 0

cmdline_size:   .long   COMMAND_LINE_SIZE-1     #length of the command line,
                                                #added with boot protocol
                                                #version 2.06

hardware_subarch:	.long 0			# subarchitecture, added with 2.07
						# default to 0 for normal x86 PC

hardware_subarch_data:	.quad 0

payload_offset:		.long ZO_input_data
payload_length:		.long ZO_z_input_len

setup_data:		.quad 0			# 64-bit physical pointer to
						# single linked list of
						# struct setup_data

pref_address:		.quad LOAD_PHYSICAL_ADDR	# preferred load addr

#define ZO_INIT_SIZE	(ZO__end - ZO_startup_32 + ZO_z_extract_offset)
#define VO_INIT_SIZE	(VO__end - VO__text)
#if ZO_INIT_SIZE > VO_INIT_SIZE
#define INIT_SIZE ZO_INIT_SIZE
#else
#define INIT_SIZE VO_INIT_SIZE
#endif
init_size:		.long INIT_SIZE		# kernel initialization size

# End of setup header #####################################################
```

Following code not used any more in our boot procedure, let's ignore it here.

```final_part_of_linux_real_mode_code

arch/x86/boot/header.S:240

	.section ".entrytext", "ax"
start_of_setup:
#ifdef SAFE_RESET_DISK_CONTROLLER
# Reset the disk controller.
	movw	$0x0000, %ax		# Reset disk controller
	movb	$0x80, %dl		# All disks
	int	$0x13
#endif

# Force %es = %ds
	movw	%ds, %ax
	movw	%ax, %es
	cld

# Apparently some ancient versions of LILO invoked the kernel with %ss != %ds,
# which happened to work by accident for the old code.  Recalculate the stack
# pointer if %ss is invalid.  Otherwise leave it alone, LOADLIN sets up the
# stack behind its own code, so we can't blindly put it directly past the heap.

	movw	%ss, %dx
	cmpw	%ax, %dx	# %ds == %ss?
	movw	%sp, %dx
	je	2f		# -> assume %sp is reasonably set

	# Invalid %ss, make up a new stack
	movw	$_end, %dx
	testb	$CAN_USE_HEAP, loadflags
	jz	1f
	movw	heap_end_ptr, %dx
1:	addw	$STACK_SIZE, %dx
	jnc	2f
	xorw	%dx, %dx	# Prevent wraparound

2:	# Now %dx should point to the end of our stack space
	andw	$~3, %dx	# dword align (might as well...)
	jnz	3f
	movw	$0xfffc, %dx	# Make sure we're not zero
3:	movw	%ax, %ss
	movzwl	%dx, %esp	# Clear upper half of %esp
	sti			# Now we should have a working stack

# We will have entered with %cs = %ds+0x20, normalize %cs so
# it is on par with the other segments.
	pushw	%ds
	pushw	$6f
	lretw
6:

# Check signature at end of setup
	cmpl	$0x5a5aaa55, setup_sig
	jne	setup_bad

# Zero the bss
	movw	$__bss_start, %di
	movw	$_end+3, %cx
	xorl	%eax, %eax
	subw	%di, %cx
	shrw	$2, %cx
	rep; stosl

# Jump to C code (should not return)
	calll	main

# Setup corrupt somehow...
setup_bad:
	movl	$setup_corrupt, %eax
	calll	puts
	# Fall through...

	.globl	die
	.type	die, @function
die:
	hlt
	jmp	die

	.size	die, .-die

	.section ".initdata", "a"
setup_corrupt:
	.byte	7
	.string	"No setup signature found...\n"
```

# LINKS
  * [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)