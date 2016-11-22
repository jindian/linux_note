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

```assembly
.globl _start, start;

_start:

start:

```