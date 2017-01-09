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

Code from address 0x9df0d0, it copies 2 blocks of code to specified destination and jumps to 0x9df000.

```code_from_0x9df0d0

   0x9df0d0:	mov    $0x1000000,%eax
   0x9df0d5:	mov    %eax,%edi
   0x9df0d7:	mov    $0x100000,%eax
   0x9df0dc:	mov    %eax,%esi
   0x9df0de:	mov    $0x8df000,%ecx
   0x9df0e3:	add    %ecx,%esi
   0x9df0e5:	add    %ecx,%edi
   0x9df0e7:	sub    $0x1,%esi
   0x9df0ea:	sub    $0x1,%edi
   0x9df0ed:	std    
   0x9df0ee:	rep movsb %ds:(%esi),%es:(%edi)
   0x9df0f0:	mov    $0x8b000,%eax
   0x9df0f5:	mov    %eax,%edi
   0x9df0f7:	mov    $0x7feae60,%eax
   0x9df0fc:	mov    %eax,%esi
   0x9df0fe:	mov    $0x5000,%ecx
   0x9df103:	cld    
   0x9df104:	rep movsb %ds:(%esi),%es:(%edi)
   0x9df106:	mov    $0x9df000,%eax
   0x9df10b:	jmp    *%eax
```

Code from address 0x9df000, at address 0x9df09e grub jumps to linux code `ljmp   $0x10,$0x1000000`.

Register esi stores start address of source code grub-core/lib/i386/relocator32.S, the global descriptor table defined in it.

Instructions from 0x9df057 to 0x9df05f are used to disable paging, the macro of disabling paging defined in grub-core/lib/i386/relocator_common.S:38 

```assembly_disable_paging
     .macro DISABLE_PAGING
#ifdef GRUB_MACHINE_IEEE1275
#endif

        movl    %cr0, %eax
        andl    $(~GRUB_MEMORY_CPU_CR0_PAGING_ON), %eax
        movl    %eax, %cr0
        .endm
```

Instructions from 0x9df062 to 0x9df068 are used to turn off PAE

```

   0x9df000:	mov    %eax,%esi
   0x9df002:	add    $0x9,%eax
   0x9df007:	jmp    *%eax
(gdb) info registers eax
eax            0x9df009	   10350601
   0x9df009:	lea    0x48(%esi),%eax
(gdb) info registers esi
esi            0x9df000	   10350592
   0x9df00f:	mov    %eax,0x40(%esi)
(gdb) info registers eax
eax            0x9df048	   10350664
   0x9df015:	lea    0xb0(%esi),%eax
   0x9df01b:	mov    %eax,0x32(%esi)
   0x9df021:	lgdtl  0x30(%esi)
   0x9df028:	ljmp   *0x40(%esi)
(gdb) x/w 0x9df000+0x40
0x9df040:	0x009df048
   0x9df02e:	xchg   %ax,%ax
   0x9df030:	and    %al,(%eax)
   0x9df032:	add    %al,(%eax)
   0x9df034:	add    %al,(%eax)
   0x9df036:	lea    0x0(%esi),%esi
   0x9df039:	lea    0x0(%edi,%eiz,1),%edi
   0x9df040:	add    %al,(%eax)
   0x9df042:	add    %al,(%eax)
   0x9df044:	adc    %al,(%eax)
   0x9df046:	add    %al,(%eax)
   0x9df048:	mov    $0x18,%eax
   0x9df04d:	mov    %eax,%ds
   0x9df04f:	mov    %eax,%es
   0x9df051:	mov    %eax,%fs
   0x9df053:	mov    %eax,%gs
   0x9df055:	mov    %eax,%ss
   0x9df057:	mov    %cr0,%eax
   0x9df05a:	and    $0x7fffffff,%eax
(gdb) info registers eax
eax            0x11	17
   0x9df05f:	mov    %eax,%cr0
   0x9df062:	mov    %cr4,%eax
   0x9df065:	and    $0xffffffdf,%eax
   0x9df068:	mov    %eax,%cr4
   0x9df06b:	jmp    0x9df06d
   0x9df06d:	mov    $0x8b000,%eax
   0x9df072:	mov    %eax,%esp
   0x9df074:	mov    $0x0,%eax
   0x9df079:	mov    %eax,%ebp
   0x9df07b:	mov    $0x8b000,%eax
   0x9df080:	mov    %eax,%esi
   0x9df082:	mov    $0x0,%eax
   0x9df087:	mov    %eax,%edi
   0x9df089:	mov    $0x7fcbc,%eax
   0x9df08e:	mov    $0x0,%ebx
   0x9df093:	mov    $0x7fe1880,%ecx
   0x9df098:	mov    $0x400,%edx
   0x9df09d:	cld    
   0x9df09e:	ljmp   $0x10,$0x1000000
```

What preparations grub does before jumping to linux code?

# LINKS
  * [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
  * [PAE: Physical Address Extension](https://en.wikipedia.org/wiki/Physical_Address_Extension)