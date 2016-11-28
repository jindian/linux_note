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

Continue previous codestart, call routine grub_gate_a20 at address 0x88a5 to enable address line 20 for high memory, the input parameter stored in ax.
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

Actually the first instruction after step in grub_gate_a20 locate at address 0x88a7(It's because I set architecture in gdb as 8086 for real mode code debug, set as i386 could correct it). I checked register information, address of next instruction after routine grub_gate_a20 complete stored at the top of stack as below. Let's step into grub_gate_a20, how grub kernel enable address line 20. First of all, it check the state of a20 line with routine gate_a20_check_state at address 0x8930.
```assembly
(gdb) info registers 
eax            0x1	1
ecx            0x0	0
edx            0x80	128
ebx            0x1	1
esp            0x7fff0	0x7fff0
ebp            0x7fff0	0x7fff0
esi            0x8127	33063
edi            0x81e8	33256
eip            0x88a7	0x88a7
eflags         0x2	[ ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x10	16
gs             0x10	16
(gdb) x/w 0x7fff0
0x7fff0:	0x00008245
   0x88a7:	mov    %ax,%dx
   0x88a9:	call   0x8930
   0x88ae:	cmp    %al,%dl
   0x88b0:	jne    0x88b3
   0x88b2:	ret    
   0x88b3:	push   %bp
   0x88b4:	call   0x832a
   0x88b7:	(bad)  
   0x88b8:	(bad)  

-----------------------------------------------------------------------

grub-core/boot/i386/pc/startup_raw.S:126

/*
 * grub_gate_a20(int on)
 *
 * Gate address-line 20 for high memory.
 *
 * This routine is probably overconservative in what it does, but so what?
 *
 * It also eats any keystrokes in the keyboard buffer.  :-(
 */

grub_gate_a20:
        movl    %eax, %edx

gate_a20_test_current_state:
        /* first of all, test if already in a good state */
        call    gate_a20_check_state
        cmpb    %al, %dl
        jnz     gate_a20_try_bios
        ret
```

Check if the a20 line was already enable by BIOS. Boot loader set less value at 0x8000, obtain the value at 0x108000 which is 1 MiB higher than 0x8000, values store in above two adresses is different, a20 line already enabled by BIOS. Return to codestart(0x8245) finally.
```assembly
   0x8930:	mov    $0x64,%cx
   0x8935:	call   0x8941
   0x893a:	cmp    %al,%dl
(gdb) info registers al dl
al             0x1	1
dl             0x1	1
   0x893c:	je     0x8940
   0x893e:	loop   0x8935
   0x8940:	ret    
(gdb) info registers esp
esp            0x7ffe8	0x7ffe8
(gdb) x/w 0x7ffe8
0x7ffe8:	0x0000893a
   0x8941:	push   %ebx
   0x8942:	push   %ecx
   0x8943:	xor    %eax,%eax
   0x8945:	mov    $0x8000,%ebx
   0x894a:	push   %ebx
   0x894b:	mov    (%ebx),%cl
   0x894d:	add    $0x100000,%ebx
   0x8953:	mov    (%ebx),%al
   0x8955:	pop    %ebx
(gdb) info registers cl al
cl             0x52	82
al             0x0	0
   0x8956:	mov    %al,%ch
(gdb) info registers ch
ch             0x0	0
   0x8958:	dec    %ch
info registers ch
ch             0xff	-1
   0x895a:	mov    %ch,(%ebx)
   0x895c:	out    %al,$0x80
   0x895e:	out    %al,$0x80
   0x8960:	push   %ebx
   0x8961:	add    $0x100000,%ebx
   0x8967:	mov    (%ebx),%ch
   0x8969:	sub    %ch,%al
   0x896b:	xor    $0x1,%al
   0x896d:	pop    %ebx
   0x896e:	mov    %cl,(%ebx)
   0x8970:	pop    %ecx
   0x8971:	pop    %ebx
   0x8972:	ret  

-----------------------------------------------------------------------

grub-core/boot/i386/pc/startup_raw.S:232

gate_a20_check_state:
        /* iterate the checking for a while */
        movl    $100, %ecx
1:
        call    3f
        cmpb    %al, %dl
        jz      2f
        loop    1b
2:
        ret
3:
        pushl   %ebx
        pushl   %ecx
        xorl    %eax, %eax
        /* compare the byte at 0x8000 with that at 0x108000 */
        movl    $GRUB_BOOT_MACHINE_KERNEL_ADDR, %ebx
        pushl   %ebx
        /* save the original byte in CL */
        movb    (%ebx), %cl
        /* store the value at 0x108000 in AL */
        addl    $0x100000, %ebx
        movb    (%ebx), %al
        /* try to set one less value at 0x8000 */
        popl    %ebx
        movb    %al, %ch
        decb    %ch
        movb    %ch, (%ebx)
        /* serialize */
        outb    %al, $0x80
        outb    %al, $0x80
        /* obtain the value at 0x108000 in CH */
        pushl   %ebx
        addl    $0x100000, %ebx
        movb    (%ebx), %ch
        /* this result is 1 if A20 is on or 0 if it is off */
        subb    %ch, %al
        xorb    $1, %al
        /* restore the original */
        popl    %ebx
        movb    %cl, (%ebx)
        popl    %ecx
        popl    %ebx
        ret
```

Prepare parameters for routine grub_reed_solomon_recover located at address 0x85a7. grub_reed_solomon_recover written in C(grub-core/lib/reed_solomon.c), compile result is rs_decoder.S which included by startup_raw.S. I will ignore this routine, jump to post_reed_solomon(located at 0x89ce).
```assembly
   0x8245:	mov    0x8208,%edx
   0x824b:	add    $0x3bd,%edx
   0x8251:	mov    0x8210,%ecx
   0x8257:	lea    0x8973,%eax
   0x825d:	cld    
   0x825e:	call   0x85a7
   0x8263:	jmp    0x89ce
   0x8268:	lock incl (%edi)
   0x826b:	add    %ch,%bl
   0x826d:	adc    -0x6f6f6f70(%eax),%dl

-----------------------------------------------------------------------

grub-core/boot/i386/pc/startup_raw.S:107

       movl    LOCAL(compressed_size), %edx
#ifdef __APPLE__
        addl    $decompressor_end, %edx
        subl    $(LOCAL(reed_solomon_part)), %edx
#else
        addl    $(LOCAL(decompressor_end) - LOCAL(reed_solomon_part)), %edx
#endif
        movl    reed_solomon_redundancy, %ecx
        leal    LOCAL(reed_solomon_part), %eax
        cld
        call    EXT_C (grub_reed_solomon_recover)
        jmp     post_reed_solomon
```

Links:
------------------------------------
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Protected mode](https://en.wikipedia.org/wiki/Protected_mode)
  * [Instruction Prefixes: addr32](http://www.delorie.com/gnu/docs/binutils/as_265.html)
  * [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
  * [A20 line](https://en.wikipedia.org/wiki/A20_line)
  * [Test A20 line](http://wiki.osdev.org/A20_Line)