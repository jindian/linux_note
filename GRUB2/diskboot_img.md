Disk boot image
================================
Disk boot image is used as the first sector of the core image when booting from a hard disk. It reads the rest of the core image into memory and starts the kernel.

Continue the boot process after MBR copied boot disk image to address 0x8000 and jumped to the address.

Save drive type and DAP, print notification message and enter bootloop.
```assembly
   0x8000:	push   %dx
   0x8001:	push   %si
(gdb) info registers dx si
dx             0x80	128
si             0x7c05	31749
   0x8002:	mov    $0x811b,%si
(gdb) x/s 0x811b
0x811b:	"loading"
   0x8005:	call   0x8141
   0x8008:	pop    %si
   0x8009:	mov    $0x81f4,%di
   0x800c:	mov    (%di),%ebp
   0x800f:	cmpw   $0x0,0x8(%di)
   0x8013:	je     0x80f9
   0x8017:	cmpb   $0x0,-0x1(%si)

------------------------------------------------------------------------

grub-core/boot/i386/pc/diskboot.S:37

start:
_start:
        /*
         * _start is loaded at 0x2000 and is jumped to with
         * CS:IP 0:0x2000 in kernel.
         */

        /*
         * we continue to use the stack for boot.img and assume that
         * some registers are set to correct values. See boot.S
         * for more information.
         */

        /* save drive reference first thing! */
        pushw   %dx

        /* print a notification message on the screen */
        pushw   %si
        MSG(notification_string)
        popw    %si

        /* this sets up for the first run through "bootloop" */
        movw    $LOCAL(firstlist), %di

        /* save the sector number of the second sector in %ebp */
        movl    (%di), %ebp

```

```assembly
   0x800f:	cmpw   $0x0,0x8(%di)
   0x8013:	je     0x80f9
   0x8017:	cmpb   $0x0,-0x1(%si)
   0x801b:	je     0x8063
   0x801d:	mov    (%di),%ebx
   0x8020:	mov    0x4(%di),%ecx
   0x8024:	xor    %eax,%eax
   0x8027:	mov    $0x7f,%al
   0x8029:	cmp    %ax,0x8(%di)
   0x802c:	jg     0x8031

```