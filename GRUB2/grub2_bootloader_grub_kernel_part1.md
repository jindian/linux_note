Decompress grub kernel image
================================

Before continuing real mode code, let's check following parameters first.
```assembly

grub-core/boot/i386/pc/startup_raw.S:49

	/*
	 *  This is a special data area.
	 */

	. = _start + GRUB_DECOMPRESSOR_MACHINE_COMPRESSED_SIZE
LOCAL(compressed_size):
	.long 0
	. = _start + GRUB_DECOMPRESSOR_MACHINE_UNCOMPRESSED_SIZE
LOCAL(uncompressed_size):
	.long 0

	. = _start + GRUB_KERNEL_I386_PC_REED_SOLOMON_REDUNDANCY
reed_solomon_redundancy:
	.long	0
	. = _start + GRUB_KERNEL_I386_PC_NO_REED_SOLOMON_LENGTH
	.short	(LOCAL(reed_solomon_part) - _start)

/*
 *  This is the area for all of the special variables.
 */
	. = _start + GRUB_DECOMPRESSOR_I386_PC_BOOT_DEVICE
LOCAL(boot_dev):
	.byte	0xFF, 0xFF, 0xFF
LOCAL(boot_drive):
	.byte	0x00

-----------------------------------------------------------------------

include/grub/offset.h

/* The offset of GRUB_COMPRESSED_SIZE.  */
#define GRUB_DECOMPRESSOR_I386_PC_COMPRESSED_SIZE       0x08

/* The offset of GRUB_COMPRESSED_SIZE.  */
#define GRUB_DECOMPRESSOR_I386_PC_UNCOMPRESSED_SIZE     0x0c

/* Offset of reed_solomon_redundancy.  */
#define GRUB_KERNEL_I386_PC_REED_SOLOMON_REDUNDANCY     0x10

/* Offset of field holding no reed solomon length.  */
#define GRUB_KERNEL_I386_PC_NO_REED_SOLOMON_LENGTH      0x14

#define GRUB_DECOMPRESSOR_I386_PC_BOOT_DEVICE           0x18

-----------------------------------------------------------------------

# compressed_size
(gdb) x/w 0x8200+0x8
0x8208:	0x00005db2
# uncompressed_size
(gdb) x/w 0x8200+0xc
0x820c:	0x0000b7d0
# reed_solomon_redundancy
(gdb) x/w 0x8200+0x10
0x8210:	0x00000f1e
# no reed solomon length
(gdb) x/w 0x8200+0x14
0x8214:	0x00000773
# boot_dev
(gdb) x/b 0x8200+0x18
0x8218:	0xff
(gdb) x/b 0x8200+0x18+0x1
0x8219:	0xff
(gdb) x/b 0x8200+0x18+0x2
0x821a:	0xff
# boot_drive
(gdb) x/b 0x8200+0x18+0x3
0x821b:	0x00

```

The first instruction is jump to codestart(0x821c), let's start at codestart, In it after set up data sections, real mode stack, save boot drive and reset disk system(INT13H_AH0H), operating mode changed from real mode to protect mode(call 0x82d2).
```assembly
   0x821c:	cli    
   0x821d:	xor    %ax,%ax
   0x821f:	mov    %ax,%ds
   0x8221:	mov    %ax,%ss
   0x8223:	mov    %ax,%es
   0x8225:	mov    $0x1ff0,%ebp
   0x822b:	mov    %ebp,%esp
   0x822e:	sti    
   0x822f:	addr32 mov %dl,0x821b
   0x8236:	int    $0x13
   0x8238:	calll  0x82d2
   0x823e:	inc    %ax
   0x823f:	cld    
   0x8240:	call   0x88a5
   0x8243:	add    %al,(%bx,%si)
   0x8245:	mov    (%di),%dx
   0x8247:	or     %al,0x0(%bp,%si)
   0x824b:	add    $0x3bd,%dx
   0x824f:	add    %al,(%bx,%si)
   0x8251:	mov    (%di),%cx

-----------------------------------------------------------------------

grub-core/boot/i386/pc/startup_raw.S:75

/* the real mode code continues... */
LOCAL (codestart):
        cli             /* we're not safe here! */

        /* set up %ds, %ss, and %es */
        xorw    %ax, %ax
        movw    %ax, %ds
        movw    %ax, %ss
        movw    %ax, %es

        /* set up the real mode/BIOS stack */
        movl    $GRUB_MEMORY_MACHINE_REAL_STACK, %ebp
        movl    %ebp, %esp

        sti             /* we're safe again */

        /* save the boot drive */
        ADDR32  movb    %dl, LOCAL(boot_drive)

        /* reset disk system (%ah = 0) */
        int     $0x13

        /* transition to protected mode */
        DATA32  call real_to_prot
```

In real_to_prot
```assembly
   0x82d2:	cli    
   0x82d3:	xor    %ax,%ax
   0x82d5:	mov    %ax,%ds
   0x82d7:	addr32 lgdtl 0x82c0
   0x82e0:	mov    %cr0,%eax
   0x82e3:	or     $0x1,%eax
   0x82e7:	mov    %eax,%cr0
   0x82ea:	ljmpl  $0x8,$0x82f2
   0x82f2:	mov    $0xd88e0010,%eax
   0x82f8:	mov    %ax,%es

```

Links:
------------------------------------
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Protected mode](https://en.wikipedia.org/wiki/Protected_mode)
  * [Instruction Prefixes: addr32](http://www.delorie.com/gnu/docs/binutils/as_265.html)