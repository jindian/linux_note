Preparation before get to C code grub_main
================================
Save routines' address, copy back decompressed part to address start from 0x9000, then jump to LOCAL (cont) located at address 0x9025, the address of LOCAL (cont) changed after copied back. After jumped to LOCAL (cont), initialize BSS and call C code entrance grub_main.
```assembly
   0x100000:	mov    %ecx,0x41(%esi)
   0x100006:	mov    %edi,0x45(%esi)
   0x10000c:	mov    %eax,0x160(%esi)
   0x100012:	mov    $0x6e54,%ecx
   0x100017:	mov    $0x9000,%edi
   0x10001c:	rep movsb %ds:(%esi),%es:(%edi)
   0x10001e:	mov    $0x9025,%esi
(gdb) info registers esi
esi            0x9025	36901
   0x100023:	jmp    *%esi
   0x100025:	mov    $0xfe54,%edi
   0x10002a:	mov    $0x17508,%ecx
   0x10002f:	sub    %edi,%ecx
   0x100031:	xor    %eax,%eax
   0x100033:	cld    
   0x100034:	rep stos %al,%es:(%edi)
   0x100036:	mov    %edx,0xfe58
   0x10003c:	call   0x1039fb

-----------------------------------------------------------------------

grub-core/kern/i386/pc/startup.S:55

        .globl  start, _start, __start
start:
_start:
__start:
#ifdef __APPLE__
LOCAL(start):
#endif
        .code32

        movl    %ecx, (LOCAL(real_to_prot_addr) - _start) (%esi)
        movl    %edi, (LOCAL(prot_to_real_addr) - _start) (%esi)
        movl    %eax, (EXT_C(grub_realidt) - _start) (%esi)

        /* copy back the decompressed part (except the modules) */
#ifdef __APPLE__
        movl    $EXT_C(_edata), %ecx
        subl    $LOCAL(start), %ecx
#else
        movl    $(_edata - _start), %ecx
#endif
        movl    $(_start), %edi
        rep
        movsb

        movl    $LOCAL (cont), %esi
        jmp     *%esi
LOCAL(cont):

#if 0
        /* copy modules before cleaning out the bss */
        movl    EXT_C(grub_total_module_size), %ecx
        movl    EXT_C(grub_kernel_image_size), %esi
        addl    %ecx, %esi
        addl    $_start, %esi
        decl    %esi
        movl    $END_SYMBOL, %edi
        addl    %ecx, %edi
        decl    %edi
        std
        rep
        movsb
#endif

#ifdef __APPLE__
        /* clean out the bss */
        movl    $EXT_C(_edata), %edi

        /* compute the bss length */
        movl    $GRUB_MEMORY_MACHINE_SCRATCH_ADDR, %ecx
#else
        /* clean out the bss */
        movl    $BSS_START_SYMBOL, %edi

        /* compute the bss length */
        movl    $END_SYMBOL, %ecx
#endif
        subl    %edi, %ecx

        /* clean out */
        xorl    %eax, %eax
        cld
        rep
        stosb

        movl    %edx, EXT_C(grub_boot_device)

        /*
         *  Call the start of main body of C code.
         */
        call EXT_C(grub_main)
```