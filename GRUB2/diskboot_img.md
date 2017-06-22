# disk boot image

Disk boot image is the first sector of grub core image when boot from a hard disk. It's use to read rest of grub core image into memory and starts the kernel, size of disk boot image is 512 bytes.

Parameters used to read rest grub core image start at address 0x81f4.

```
0x8000          +-------------------------+
                | Disk image main body    |
                |                         |
                |                         |
                |                         |
0x81f4          +-------------------------+
                | sector start parameter  |
                |                         |
0x81f4 + 0x8    +-------------------------+
                |number of sector to read |
0x81f4 + 0xa    +-------------------------+
                |rest kernel dst addresss |
                +-------------------------+
```

Continue the boot process, in previous chapter MBR copied disk boot image to address 0x8000 and jumped to this address.

```
    0x7d65:     pop   %ds
    0x7d66: popa
(gdb) x/h 0x7c5a
0x7c5a: 0x8000
    0x7d67:     jmp   *0x7c5a

----------------------------------------------------------------------

grub-core/boot/i386/pc/boot.S:332

popw %ds
popa

/* boot kernel */
jmp *(kernel_address)

/* END OF MAIN LOOP */
```

Save drive type and DAP, print notification message and enter bootloop.

```assembly
   0x8000:    push   %dx
   0x8001:    push   %si
(gdb) info registers dx si
dx             0x80    128
si             0x7c05    31749
   0x8002:    mov    $0x811b,%si
(gdb) x/s 0x811b
0x811b:    "loading"
   0x8005:    call   0x8141
   0x8008:    pop    %si
   0x8009:    mov    $0x81f4,%di
   0x800c:    mov    (%di),%ebp
   0x800f:    cmpw   $0x0,0x8(%di)
   0x8013:    je     0x80f9
   0x8017:    cmpb   $0x0,-0x1(%si)

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

Check the number of sector to read, in my environment its value is 60, Enter setup-sectors. Setup DAP and read rest of kernel with BIOS interrupt, jump to copy buffer\(0x80c9\).

```assembly
   0x800f:    cmpw   $0x0,0x8(%di)
(gdb) info registers di
di             0x81f4    -32268
(gdb) x/h 0x81f4+0x8
0x81fc:    0x003c
(gdb) p 0x3c
$2 = 60
   0x8013:    je     0x80f9
   0x8017:    cmpb   $0x0,-0x1(%si)
(gdb) info registers si
si             0x7c05    31749
(gdb) x/b 0x7c05-0x1
0x7c04:    0x01
   0x801b:    je     0x8063
   0x801d:    mov    (%di),%ebx
   0x8020:    mov    0x4(%di),%ecx
   0x8024:    xor    %eax,%eax
   0x8027:    mov    $0x7f,%al
   0x8029:    cmp    %ax,0x8(%di)
   0x802c:    jg     0x8031
   0x802e:    mov    0x8(%di),%ax
   0x8031:    sub    %ax,0x8(%di)
(gdb) x/h 0x81f4+0x8
0x81fc:    0x0000
   0x8034:    add    %eax,(%di)
   0x8037:    adcl   $0x0,0x4(%di)
   0x803c:    movw   $0x10,(%si)
   0x8040:    mov    %ax,0x2(%si)
   0x8043:    mov    %ebx,0x8(%si)
   0x8047:    mov    %ecx,0xc(%si)
   0x804b:    movw   $0x7000,0x6(%si)
   0x8050:    push   %ax
   0x8051:    movw   $0x0,0x4(%si)
   0x8056:    mov    $0x42,%ah
   0x8058:    int    $0x13
(gdb) info registers 
eax            0x3c    60
ecx            0x0    0
edx            0x80    128
ebx            0x2    2
esp            0x1ffa    0x1ffa
ebp            0x2    0x2
esi            0x7c05    31749
edi            0x81f4    33268
eip            0x805a    0x805a
eflags         0x246    [ PF ZF IF ]
cs             0x0    0
ss             0x0    0
ds             0x0    0
es             0x0    0
fs             0x0    0
gs             0x0    0
   0x805a:    jb     0x810d
   0x805e:    mov    $0x7000,%bx
   0x8061:    jmp    0x80c9
   0x8063:    mov    0x4(%di),%eax
   0x8067:    or     %eax,%eax
   0x806a:    jne    0x8105
   0x806e:    mov    (%di),%eax

-----------------------------------------------------------------------

grub-core/boot/i386/pc/diskboot.S:64

       /* this is the loop for reading the rest of the kernel in */
LOCAL(bootloop):

        /* check the number of sectors to read */
        cmpw    $0, 8(%di)

        /* if zero, go to the start function */
        je      LOCAL(bootit)

LOCAL(setup_sectors):
        /* check if we use LBA or CHS */
        cmpb    $0, -1(%si)

        /* use CHS if zero, LBA otherwise */
        je      LOCAL(chs_mode)

        /* load logical sector start */
        movl    (%di), %ebx
        movl    4(%di), %ecx

        /* the maximum is limited to 0x7f because of Phoenix EDD */
        xorl    %eax, %eax
        movb    $0x7f, %al

        /* how many do we really want to read? */
        cmpw    %ax, 8(%di)     /* compare against total number of sectors */

        /* which is greater? */
        jg      1f

        /* if less than, set to total */
        movw    8(%di), %ax

1:
        /* subtract from total */
        subw    %ax, 8(%di)

        /* add into logical sector start */
        addl    %eax, (%di)
        adcl    $0, 4(%di)

        /* set up disk address packet */

        /* the size and the reserved byte */
        movw    $0x0010, (%si)

        /* the number of sectors */
        movw    %ax, 2(%si)

        /* the absolute address */
        movl    %ebx, 8(%si)
        movl    %ecx, 12(%si)

        /* the segment of buffer address */
        movw    $GRUB_BOOT_MACHINE_BUFFER_SEG, 6(%si)

        /* save %ax from destruction! */
        pushw   %ax

        /* the offset of buffer address */
        movw    $0, 4(%si)

/*
 * BIOS call "INT 0x13 Function 0x42" to read sectors from disk into memory
 *      Call with       %ah = 0x42
 *                      %dl = drive number
 *                      %ds:%si = segment:offset of disk address packet
 *      Return:
 *                      %al = 0x0 on success; err code on failure
 */

        movb    $0x42, %ah
        int     $0x13

        jc      LOCAL(read_error)

        movw    $GRUB_BOOT_MACHINE_BUFFER_SEG, %bx
        jmp     LOCAL(copy_buffer)
```

Copy data\(60\*512 bytes\) in buffer to address start at 0x8200, jump to bootloop again. In bootloop, now all grub kernel already read from disk image and copy completed, jump to bootit\(0x80f9\)

```assembly
   0x80c9:    mov    0xa(%di),%es
   0x80cc:    pop    %ax
   0x80cd:    shl    $0x5,%ax
   0x80d0:    add    %ax,0xa(%di)
   0x80d3:    pusha  
   0x80d4:    push   %ds
   0x80d5:    shl    $0x3,%ax
   0x80d8:    mov    %ax,%cx
   0x80da:    xor    %di,%di
   0x80dc:    xor    %si,%si
   0x80de:    mov    %bx,%ds
   0x80e0:    cld    
   0x80e1:    rep movsw %ds:(%si),%es:(%di)
   0x80e3:    pop    %ds
   0x80e4:    mov    $0x8123,%si
(gdb) x/s 0x8123
0x8123:    "."
   0x80e7:    call   0x8141
   0x80ea:    popa   
   0x80eb:    cmpw   $0x0,0x8(%di)
   0x80ef:    jne    0x8017
   0x80f3:    sub    $0xc,%di
   0x80f6:    jmp    0x800f
   0x80f9:    mov    $0x8125,%si
   0x80fc:    call   0x8141
   0x80ff:    pop    %dx
   0x8100:    ljmp   $0x0,$0x8200
   0x8105:    mov    $0x8128,%si
   0x8108:    call   0x8141
   0x810b:    jmp    0x8113
   0x810d:    mov    $0x812d,%si
   0x8110:    call   0x8141

-----------------------------------------------------------------------

grub-core/boot/i386/pc/diskboot.S:247

LOCAL(copy_buffer):

        /* load addresses for copy from disk buffer to destination */
        movw    10(%di), %es    /* load destination segment */

        /* restore %ax */
        popw    %ax

        /* determine the next possible destination address (presuming
                512 byte sectors!) */
        shlw    $5, %ax         /* shift %ax five bits to the left */
        addw    %ax, 10(%di)    /* add the corrected value to the destination
                                   address for next time */

        /* save addressing regs */
        pusha
        pushw   %ds

        /* get the copy length */
        shlw    $3, %ax
        movw    %ax, %cx

        xorw    %di, %di        /* zero offset of destination addresses */
        xorw    %si, %si        /* zero offset of source addresses */
        movw    %bx, %ds        /* restore the source segment */

        cld             /* sets the copy direction to forward */

        /* perform copy */
        rep             /* sets a repeat */
        movsw           /* this runs the actual copy */

        /* restore addressing regs and print a dot with correct DS
           (MSG modifies SI, which is saved, and unused AX and BX) */
        popw    %ds
        MSG(notification_step)
        popa

        /* check if finished with this dataset */
        cmpw    $0, 8(%di)
        jne     LOCAL(setup_sectors)

        /* update position to load from */
        subw    $GRUB_BOOT_MACHINE_LIST_SIZE, %di

        /* jump to bootloop */
        jmp     LOCAL(bootloop)
```

In bootit, print new line and jump to the start of rest kernel image code\(0x8200\).

```assembly
   0x80f9:    mov    $0x8125,%si
(gdb) x/s 0x8125
0x8125:    "\r\n"
   0x80fc:    call   0x8141
   0x80ff:    pop    %dx
(gdb) info registers 
eax            0xe00    3584
ecx            0x0    0
edx            0x80    128
ebx            0x1    1
esp            0x1ffe    0x1ffe
ebp            0x2    0x2
esi            0x8127    33063
edi            0x81e8    33256
eip            0x8100    0x8100
eflags         0x246    [ PF ZF IF ]
cs             0x0    0
ss             0x0    0
ds             0x0    0
es             0x820    2080
fs             0x0    0
gs             0x0    0
   0x8100:    ljmp   $0x0,$0x8200
   0x8105:    mov    $0x8128,%si
   0x8108:    call   0x8141
   0x810b:    jmp    0x8113
   0x810d:    mov    $0x812d,%si
   0x8110:    call   0x8141
   0x8113:    mov    $0x8132,%si

-----------------------------------------------------------------------

grub-core/boot/i386/pc/diskboot.S:297

LOCAL(bootit):
        /* print a newline */
        MSG(notification_done)
        popw    %dx     /* this makes sure %dl is our "boot" drive */
        ljmp    $0, $(GRUB_BOOT_MACHINE_KERNEL_ADDR + 0x200)
```

# Links

* [GRUB image files](https://www.gnu.org/software/grub/manual/html_node/Images.html#Images)



