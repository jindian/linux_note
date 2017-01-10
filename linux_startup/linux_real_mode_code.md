# linux real mode code

linux kernel real mode code is arch/x86/boot/header.S.

In the early days of linux it have to be a bootable image, so in arch/x86/boot/header.S first part of the code is same as MBR.

```linux_mbr

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