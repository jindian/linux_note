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

The first instruction in grub kernel is jump to codestart(0x821c), we  go to codestart directly, it set up data sections, real mode stack, save boot drive and reset disk system(INT13H_AH0H), operating mode changed from real mode to protect mode(call 0x82d2).
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

Before step into real_to_prot function, here are some special variables we need pay attention. In which gdt is a simple structure describing real mode and protected mode memory area.
```assembly
grub-core/kern/i386/realmode.S:45
/*
 *  This is the area for all of the special variables.
 */

protstack:
        .long   GRUB_MEMORY_MACHINE_PROT_STACK

        .macro PROT_TO_REAL
        call    prot_to_real
        .endm

        .macro REAL_TO_PROT
        DATA32  call    real_to_prot
        .endm

/*
 * This is the Global Descriptor Table
 *
 *  An entry, a "Segment Descriptor", looks like this:
 *
 * 31          24         19   16                 7           0
 * ------------------------------------------------------------
 * |             | |B| |A|       | |   |1|0|E|W|A|            |
 * | BASE 31..24 |G|/|L|V| LIMIT |P|DPL|  TYPE   | BASE 23:16 |  4
 * |             | |D| |L| 19..16| |   |1|1|C|R|A|            |
 * ------------------------------------------------------------
 * |                             |                            |
 * |        BASE 15..0           |       LIMIT 15..0          |  0
 * |                             |                            |
 * ------------------------------------------------------------
 *
 *  Note the ordering of the data items is reversed from the above
 *  description.
 */

        .p2align        5       /* force 4-byte alignment */
gdt:
        .word   0, 0
        .byte   0, 0, 0, 0

        /* -- code segment --
         * base = 0x00000000, limit = 0xFFFFF (4 KiB Granularity), present
         * type = 32bit code execute/read, DPL = 0
         */
        .word   0xFFFF, 0
        .byte   0, 0x9A, 0xCF, 0

        /* -- data segment --
         * base = 0x00000000, limit 0xFFFFF (4 KiB Granularity), present
         * type = 32 bit data read/write, DPL = 0
         */
        .word   0xFFFF, 0
        .byte   0, 0x92, 0xCF, 0

        /* -- 16 bit real mode CS --
         * base = 0x00000000, limit 0x0FFFF (1 B Granularity), present
         * type = 16 bit code execute/read only/conforming, DPL = 0
         */
        .word   0xFFFF, 0
        .byte   0, 0x9E, 0, 0

        /* -- 16 bit real mode DS --
         * base = 0x00000000, limit 0x0FFFF (1 B Granularity), present
         * type = 16 bit data read/write, DPL = 0
         */
        .word   0xFFFF, 0
        .byte   0, 0x92, 0, 0


        .p2align 5
/* this is the GDT descriptor */
gdtdesc:
        .word   0x27                    /* limit */
        .long   gdt                     /* addr */
LOCAL(realidt):
        .word 0x400
        .long 0
protidt:
        .word 0
        .long 0
```

In real_to_prot, it loads gdt descriptor, enables protected mode, saves real mode stack, set stack registers with protected mode stack, stores real mode idt(interrupt descriptor table) and load protected mode idt instead.
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
   0x82fa:	mov    %ax,%fs
   0x82fc:	mov    %ax,%gs
   0x82fe:	mov    %ax,%ss
   0x8300:	mov    (%si),%ax
   0x8302:	and    $0xa3,%al
   0x8304:	lock pop %ds
   0x8306:	add    %al,(%bx,%si)
   0x8308:	mov    0x8268,%ax
   0x830b:	add    %al,(%bx,%si)
   0x830d:	mov    %ax,%sp
   0x830f:	mov    %ax,%bp
   0x8311:	mov    0x1ff0,%ax
   0x8314:	add    %al,(%bx,%si)
   0x8316:	mov    %ax,(%si)
   0x8319:	xor    %ax,%ax
   0x831b:	sidtw  (%di)
   0x8322:	lidtw  (%di)
(gdb) info registers 
eax            0x0	0
ecx            0x0	0
edx            0x80	128
ebx            0x1	1
esp            0x7fff0	0x7fff0
ebp            0x7fff0	0x7fff0
esi            0x8127	33063
edi            0x81e8	33256
eip            0x8329	0x8329
eflags         0x46	[ PF ZF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x10	16
gs             0x10	16
(gdb) x/w 0x7fff0
0x7fff0:	0x0000823e
(gdb) 
   0x8329:	ret

-----------------------------------------------------------------------

grub-core/kern/i386/realmode.S:133

real_to_prot:
        .code16
        cli

        /* load the GDT register */
        xorw    %ax, %ax
        movw    %ax, %ds
        DATA32  ADDR32  lgdt    gdtdesc

        /* turn on protected mode */
        movl    %cr0, %eax
        orl     $GRUB_MEMORY_CPU_CR0_PE_ON, %eax
        movl    %eax, %cr0

        /* jump to relocation, flush prefetch queue, and reload %cs */
        DATA32  ljmp    $GRUB_MEMORY_MACHINE_PROT_MODE_CSEG, $protcseg

        .code32
protcseg:
        /* reload other segment registers */
        movw    $GRUB_MEMORY_MACHINE_PROT_MODE_DSEG, %ax
        movw    %ax, %ds
        movw    %ax, %es
        movw    %ax, %fs
        movw    %ax, %gs
        movw    %ax, %ss

        /* put the return address in a known safe location */
        movl    (%esp), %eax
        movl    %eax, GRUB_MEMORY_MACHINE_REAL_STACK

        /* get protected mode stack */
        movl    protstack, %eax
        movl    %eax, %esp
        movl    %eax, %ebp

        /* get return address onto the right stack */
        movl    GRUB_MEMORY_MACHINE_REAL_STACK, %eax
        movl    %eax, (%esp)

        /* zero %eax */
        xorl    %eax, %eax

        sidt LOCAL(realidt)
        lidt protidt

        /* return on the old (or initialized) stack! */
        ret
```

Continue previous codestart, call routine grub_gate_a20 at address 0x88a5 to enable address line 20 for high memory, the input parameter stored in ax(After step in grub_gate_a20, actual address of the first instruction is 0x88a7).
```assembly
   0x823e:	inc    %ax
   0x823f:	cld    
(gdb) info registers 
eax            0x1	1
ecx            0x0	0
edx            0x80	128
ebx            0x1	1
esp            0x7fff4	0x7fff4
ebp            0x7fff0	0x7fff0
esi            0x8127	33063
edi            0x81e8	33256
eip            0x8240	0x8240
eflags         0x2	[ ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x10	16
gs             0x10	16
(gdb) x/w 0x7fff4
0x7fff4:	0x00000000
   0x8240:	call   0x88a5
   0x8243:	add    %al,(%bx,%si)
   0x8245:	mov    (%di),%dx
   0x8247:	or     %al,0x0(%bp,%si)
   0x824b:	add    $0x3bd,%dx
   0x824f:	add    %al,(%bx,%si)
   0x8251:	mov    (%di),%cx
   0x8253:	adc    %al,0x0(%bp,%si)

-----------------------------------------------------------------------

grub-core/boot/i386/pc/startup_raw.S:100

        /* The ".code32" directive takes GAS out of 16-bit mode. */
        .code32

        incl    %eax
        cld
        call    grub_gate_a20

```

Links:
------------------------------------
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Protected mode](https://en.wikipedia.org/wiki/Protected_mode)
  * [Instruction Prefixes: addr32](http://www.delorie.com/gnu/docs/binutils/as_265.html)
  * [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)