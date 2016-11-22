MBR\(Master Boot Recorder\)
===========================

MBR is part of grub2, it installed in the first sector of disk image. After BIOS initialization, MBR loaded to memory at address 0x7c00


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
GRUB_BOOT_MACHINE_PART_START 0x1be
GRUB_BOOT_MACHINE_PART_END   0x1fe
```

