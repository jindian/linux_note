MBR\(Master Boot Recorder\)
===============================

MBR is part of grub2, it installed in the first sector of disk image. After BIOS initialization, MBR loaded to memory at address 0x7c00

Memory deloyment of MBR
-------------------------------

```shell 
0x7c00          +----------------------+
   446 bytes    | Boot Loader          |
                |                      |
                |                      |
                |                      |
                +----------------------+
   64 bytes     | Partition Table      |
                |                      |
                +----------------------+
   2 bytes      | Magic Signature      |
                +----------------------+
```

**grub-core/boot/i386/pc/boot.S**
```assembly
.globl _start, start;
_start:
start:
 /*
 * _start is loaded at 0x7c00 and is jumped to with CS:IP 0:0x7c00
 */

 /*
 * Beginning of the sector is compatible with the FAT/HPFS BIOS
 * parameter block.
 */

 jmp LOCAL(after_BPB)


....


 /*
 * This is where an MBR would go if on a hard disk. The code
 * here isn't even referenced unless we're on a floppy. Kinda
 * sneaky, huh?
 */
 . = _start + GRUB_BOOT_MACHINE_PART_START


...


 . = _start + GRUB_BOOT_MACHINE_PART_END

/* the last 2 bytes in the sector 0 contain the signature */
 .word GRUB_BOOT_MACHINE_SIGNATURE

```

**include/grub/i386/pc/boot.h**
```
#define GRUB_BOOT_MACHINE_PART_START 0x1be
#define GRUB_BOOT_MACHINE_PART_END   0x1fe
```

What MBR do?
-------------------------------
Copy part of grub2 kernel from the second sector

BIOS reads MBR into memory at address 0x7c00, the first instruction is jump to 0x7c65

```assembly
   0x7c00:	jmp    0x7c65
   0x7c02:	nop
   0x7c03:	adc    %cl,-0x4fff4330(%esi)
   0x7c09:	mov    $0xd88e0000,%eax
   0x7c0e:	mov    %eax,%es
   0x7c10:	sti    
   0x7c11:	mov    $0xbf7c00,%esi
   0x7c16:	push   %es
   0x7c17:	mov    $0xa4f30200,%ecx
   0x7c1c:	ljmp   $0xbebe,$0x621
----------------------------------------------------------------------

   /*
    * _start is loaded at 0x7c00 and is jumped to with CS:IP 0:0x7c00
    */

   /*
    * Beginning of the sector is compatible with the FAT/HPFS BIOS
    * parameter block.
    */

    jmp     LOCAL(after_BPB)
    nop     /* do I care about this ??? */

    /*
     * This space is for the BIOS parameter block!!!!  Don't change
     * the first jump, nor start the code anywhere but right after
     * this area.
     */

    . = _start + GRUB_BOOT_MACHINE_BPB_START
    . = _start + 4

```


```assembly

   0x7c65:	cli    
   0x7c66:	nop
   0x7c67:	nop
   0x7c68:	test   $0x80,%dl
(gdb) info registers dl
dl             0x80	-128
   0x7c6b:	je     0x7c72
   0x7c6d:	test   $0x70,%dl
   0x7c70:	je     0x7c74
   0x7c72:	mov    $0x80,%dl
   0x7c74:	ljmp   $0xc031,$0x7c79
   0x7c7b:	mov    %eax,%ds
```