# jump to linux

After relocator completed, the last function involoved by linux is `((void (*) (void)) relst) ();`, with assembly code, it look like following code block, grub call function located at 0x9df0d0.

```assembly_after_relocator

192	  ((void (*) (void)) relst) ();
(gdb) x/10i $eip
   0x7f96926 <grub_relocator32_boot+184>:	call   *-0x10(%ebp)
   0x7f96929 <grub_relocator32_boot+187>:	
    jmp    0x7f9692d <grub_relocator32_boot+191>
   0x7f9692b <grub_relocator32_boot+189>:	mov    %eax,%ebx
   0x7f9692d <grub_relocator32_boot+191>:	lea    -0x8(%ebp),%esp
   0x7f96930 <grub_relocator32_boot+194>:	mov    %ebx,%eax
   0x7f96932 <grub_relocator32_boot+196>:	pop    %ebx
   0x7f96933 <grub_relocator32_boot+197>:	pop    %esi
   0x7f96934 <grub_relocator32_boot+198>:	pop    %ebp
   0x7f96935 <grub_relocator32_boot+199>:	ret    $0x28
   0x7f96938 <grub_relocator16_boot>:	push   %ebp
(gdb) info registers $ebp
ebp            0x7fbbc	0x7fbbc
(gdb) x/w 0x7fbbc-0x10
0x7fbac:	0x009df0d0
(gdb) 
```

