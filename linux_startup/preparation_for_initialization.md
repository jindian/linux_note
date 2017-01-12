# preparation for initialization

After decompression completed and jumped to decompressed linux kernel code, now we are at address 0x1000000.

Code deployment in memory is defined in arch/x86/kernel/vmlinux.lds.S for decompressed linux kernel.

What kind of preparations here?

1.   load boot global descriptor table, the table defined at the end of code arch/x86/kernel/head_32.S
2.   clear BSS section
3.   copy boot parameters out from kernel real mode data area
4.   copy command line parameters
5.   construct page table and initialize the page table
6.   after construction completed, turn on PAE
7.   check supported functions with cpuid instruction
8.   enable paging
9.   load stack segment
10.  initialize eflags
11.  setup [interrupt descriptor table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table), set response function for interrupt 0, 6, 13, 14
12.  check cpu type, it's a magic procedure ...
13.  check floating point coprocessor
14.  load early global descriptor table and interrupt descriptor table
15.  setup stack segment and jump to linux initialization code

```assembly_preparation_for_initialization

arch/x86/kernel/head_32.S:75

/*
 * 32-bit kernel entrypoint; only used by the boot CPU.  On entry,
 * %esi points to the real-mode code as a 32-bit pointer.
 * CS and DS must be 4 GB flat segments, but we don't depend on
 * any particular GDT layout, because we load our own as soon as we
 * can.
 */
__HEAD
ENTRY(startup_32)
	/* test KEEP_SEGMENTS flag to see if the bootloader is asking
		us to not reload segments */
	testb $(1<<6), BP_loadflags(%esi)        -> 0x1000000:	testb  $0x40,0x211(%esi)
(gdb) info registers esi
esi            0x8b000	569344
(gdb) x/b 0x8b000+0x211
0x8b211:	0x81(gdb)
info registers eflags
eflags         0x46	[ PF ZF ]
	jnz 2f

/*
 * Set segments to known values.
 */
	lgdt pa(boot_gdt_descr)                                          -> 0x1000009:	lgdtl  0x163bc16
	movl $(__BOOT_DS),%eax                                           -> 0x1000010:	mov    $0x18,%eax
	movl %eax,%ds
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
2:

/*
 * Clear BSS first so that there are no surprises...
 */
	cld
	xorl %eax,%eax
	movl $pa(__bss_start),%edi                                       -> 0x1000020:	mov    $0x1732000,%edi
	movl $pa(__bss_stop),%ecx                                        -> 0x1000025:	mov    $0x17b9998,%ecx
	subl %edi,%ecx
(gdb) info registers ecx
ecx            0x87998	555416
	shrl $2,%ecx
(gdb) info registers ecx
ecx            0x21e66	138854
	rep ; stosl
/*
 * Copy bootup parameters out of the way.
 * Note: %esi still has the pointer to the real-mode data.
 * With the kexec as boot loader, parameter segment might be loaded beyond
 * kernel image and might not even be addressable by early boot page tables.
 * (kexec on panic case). Hence copy out the parameters before initializing
 * page tables.
 */
	movl $pa(boot_params),%edi                                       -> 0x1000031:	mov    $0x16ce1e0,%edi
	movl $(PARAM_SIZE/4),%ecx                                        -> 0x1000036:	mov    $0x400,%ecx
(gdb) info registers esi
esi            0x8b000	569344
	cld
	rep
	movsl
	movl pa(boot_params) + NEW_CL_POINTER,%esi                       -> 0x100003e:	mov    0x16ce408,%esi
(gdb) x/w 0x16ce408
0x16ce408:	0x0008f000
(gdb) info registers esi
esi            0x8f000	585728
	andl %esi,%esi
(gdb) x/w 0x8f000
0x8f000:	0x544f4f42
(gdb) info registers eflags
eflags         0x6	[ PF ]
	jz 1f			# No comand line
	movl $pa(boot_command_line),%edi                                 -> 0x1000048:	mov    $0x16cb240,%edi
	movl $(COMMAND_LINE_SIZE/4),%ecx                                 -> 0x100004d:	mov    $0x200,%ecx
	rep
	movsl
1:

#ifdef CONFIG_PARAVIRT
	/* This is can only trip for a broken bootloader... */
	cmpw $0x207, pa(boot_params + BP_version)
	jb default_entry

	/* Paravirt-compatible boot parameters.  Look to see what architecture
		we're booting under. */
	movl pa(boot_params + BP_hardware_subarch), %eax
	cmpl $num_subarch_entries, %eax
	jae bad_subarch

	movl pa(subarch_entries)(,%eax,4), %eax
	subl $__PAGE_OFFSET, %eax
	jmp *%eax

bad_subarch:
WEAK(lguest_entry)
WEAK(xen_entry)
	/* Unknown implementation; there's really
	   nothing we can do at this point. */
	ud2a

	__INITDATA

subarch_entries:
	.long default_entry		/* normal x86/PC */
	.long lguest_entry		/* lguest hypervisor */
	.long xen_entry			/* Xen hypervisor */
	.long default_entry		/* Moorestown MID */
num_subarch_entries = (. - subarch_entries) / 4
.previous
#endif /* CONFIG_PARAVIRT */

/*
 * Initialize page tables.  This creates a PDE and a set of page
 * tables, which are located immediately beyond __brk_base.  The variable
 * _brk_end is set up to point to the first "safe" location.
 * Mappings are created both at virtual address 0 (identity mapping)
 * and PAGE_OFFSET for up to _end.
 *
 * Note that the stack is not yet set up!
 */
default_entry:
#ifdef CONFIG_X86_PAE

	/*
	 * In PAE mode swapper_pg_dir is statically defined to contain enough
	 * entries to cover the VMSPLIT option (that is the top 1, 2 or 3
	 * entries). The identity mapping is handled by pointing two PGD
	 * entries to the first kernel PMD.
	 *
	 * Note the upper half of each PMD or PTE are always zero at
	 * this stage.
	 */

#define KPMDS (((-__PAGE_OFFSET) >> 30) & 3) /* Number of kernel PMDs */

	xorl %ebx,%ebx				/* %ebx is kept at zero */

	movl $pa(__brk_base), %edi                                       -> 0x1000056:	mov    $0x17ba000,%edi
	movl $pa(swapper_pg_pmd), %edx                                   -> 0x100005b:	mov    $0x1732000,%edx
	movl $PTE_IDENT_ATTR, %eax                                       -> 0x1000060:	mov    $0x3,%eax
10:
	leal PDE_IDENT_ATTR(%edi),%ecx		/* Create PMD entry */    -> 0x1000065:	lea    0x67(%edi),%ecx
(gdb) info registers ecx
ecx            0x17ba067	24879207
	movl %ecx,(%edx)			/* Store PMD entry */
						/* Upper half already zero */
(gdb) info registers edx
edx            0x1732000	24322048
0x0100006a in ?? ()
(gdb) x/w 0x1732000
0x1732000:	0x017ba067
	addl $8,%edx
(gdb) info registers edx
edx            0x1732008	24322056
	movl $512,%ecx
11:
	stosl
	xchgl %eax,%ebx
	stosl
	xchgl %eax,%ebx
	addl $0x1000,%eax
	loop 11b

	/*
	 * End condition: we must map up to the end + MAPPING_BEYOND_END.
	 */
	movl $pa(_end) + MAPPING_BEYOND_END + PTE_IDENT_ATTR, %ebp       -> 0x100007d:	mov    $0x1ae3003,%ebp
	cmpl %ebp,%eax
	jb 10b
1:
(gdb) info registers edi
edi            0x17c8000	24936448
	addl $__PAGE_OFFSET, %edi                                        -> 0x1000086:	add    $0xc0000000,%edi
(gdb) info registers edi
edi            0xc17c8000	-1048805376
	movl %edi, pa(_brk_end)                                          -> 0x1000095:	mov    %eax,0x17383e0
(gdb) info registers eax
eax            0x1c00003	29360131
	shrl $12, %eax
(gdb) info registers eax
eax            0x1c00	7168
	movl %eax, pa(max_pfn_mapped)                                    -> 0x1000095:	mov    %eax,0x17383e0

	/* Do early initialization of the fixmap area */
	movl $pa(swapper_pg_fixmap)+PDE_IDENT_ATTR,%eax                  -> 0x100009a:	mov    $0x1733067,%eax
	movl %eax,pa(swapper_pg_pmd+0x1000*KPMDS-8)                      -> 0x100009f:	mov    %eax,0x1732ff8
#else	/* Not PAE */

page_pde_offset = (__PAGE_OFFSET >> 20);

	movl $pa(__brk_base), %edi
	movl $pa(swapper_pg_dir), %edx
	movl $PTE_IDENT_ATTR, %eax
10:
	leal PDE_IDENT_ATTR(%edi),%ecx		/* Create PDE entry */
	movl %ecx,(%edx)			/* Store identity PDE entry */
	movl %ecx,page_pde_offset(%edx)		/* Store kernel PDE entry */
	addl $4,%edx
	movl $1024, %ecx
11:
	stosl
	addl $0x1000,%eax
	loop 11b
	/*
	 * End condition: we must map up to the end + MAPPING_BEYOND_END.
	 */
	movl $pa(_end) + MAPPING_BEYOND_END + PTE_IDENT_ATTR, %ebp
	cmpl %ebp,%eax
	jb 10b
	addl $__PAGE_OFFSET, %edi
	movl %edi, pa(_brk_end)
	shrl $12, %eax
	movl %eax, pa(max_pfn_mapped)

	/* Do early initialization of the fixmap area */
	movl $pa(swapper_pg_fixmap)+PDE_IDENT_ATTR,%eax
	movl %eax,pa(swapper_pg_dir+0xffc)
#endif
	jmp 3f                                                           -> 0x10000a4:	jmp    0x1434602
/*
 * Non-boot CPU entry point; entered from trampoline.S
 * We can't lgdt here, because lgdt itself uses a data segment, but
 * we know the trampoline has already loaded the boot_gdt for us.
 *
 * If cpu hotplug is not supported then this code can go in init section
 * which will be freed later
 */

__CPUINIT

#ifdef CONFIG_SMP
ENTRY(startup_32_smp)
	cld
	movl $(__BOOT_DS),%eax
	movl %eax,%ds
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
#endif /* CONFIG_SMP */
3:

/*
 *	New page tables may be in 4Mbyte page mode and may
 *	be using the global pages. 
 *
 *	NOTE! If we are on a 486 we may have no cr4 at all!
 *	So we do not try to touch it unless we really have
 *	some bits in it to set.  This won't work if the BSP
 *	implements cr4 but this AP does not -- very unlikely
 *	but be warned!  The same applies to the pse feature
 *	if not equally supported. --macro
 *
 *	NOTE! We have to correct for the fact that we're
 *	not yet offset PAGE_OFFSET..
 */
#define cr4_bits pa(mmu_cr4_features)
	movl cr4_bits,%edx                                               -> 0x1434602:	mov    0x163cde8,%edx
(gdb) x/w 0x163cde8
0x163cde8:	0x00000020
	andl %edx,%edx
(gdb) info registers eflags
eflags         0x2	[ ]
	jz 6f
	movl %cr4,%eax		# Turn on paging options (PSE,PAE,..)
(gdb) info registers eax
eax            0x0	0
	orl %edx,%eax
(gdb) info registers eax
eax            0x20	32
	movl %eax,%cr4

	btl $5, %eax		# check if PAE is enabled
	jnc 6f
(gdb) info registers eflags
eflags         0x47	[ CF PF ZF ]

	/* Check if extended functions are implemented */
	movl $0x80000000, %eax
	cpuid
(gdb) info registers eax
eax            0x80000004	-2147483644
	cmpl $0x80000000, %eax
(gdb) info registers eflags
eflags         0x2	[ ]
	jbe 6f
	mov $0x80000001, %eax
	cpuid
(gdb) info registers eax
eax            0x663	1635
	/* Execute Disable bit supported? */
(gdb) info registers edx
edx            0x0	0
	btl $20, %edx
(gdb) info registers eflags
eflags         0x46	[ PF ZF ]
	jnc 6f

	/* Setup EFER (Extended Feature Enable Register) */
	movl $0xc0000080, %ecx
	rdmsr

	btsl $11, %eax
	/* Make changes effective */
	wrmsr

6:

/*
 * Enable paging
 */
	movl pa(initial_page_table), %eax                                -> 0x1434642:	mov    0x16783ac,%eax
(gdb) x/w 0x16783ac
0x16783ac:	0x01637000
(gdb) info registers eax
eax            0x1637000	23293952
	movl %eax,%cr3		/* set the page table pointer.. */
	movl %cr0,%eax
	orl  $X86_CR0_PG,%eax                                            -> 0x143464d:	or     $0x80000000,%eax
	movl %eax,%cr0		/* ..and set paging (PG) bit */
	ljmp $__BOOT_CS,$1f	/* Clear prefetch and normalize %eip */   -> 0x1434655:	ljmp   $0x10,$0xc143465c
1:
	/* Set up the stack pointer */
	lss stack_start,%esp                                             -> 0xc143465c:	lss    0xc163bb00,%esp

/*
 * Initialize eflags.  Some BIOS's leave bits like NT set.  This would
 * confuse the debugger if this code is traced.
 * XXX - best to initialize before switching to protected mode.
 */
	pushl $0
	popfl

#ifdef CONFIG_SMP
	cmpb $0, ready                                                  -> 0xc1434666:	cmpb   $0x0,0xc163bb08
(gdb) x/w 0xc163bb08
0xc163bb08:	0x00000000
(gdb) info registers eflags
eflags         0x46	[ PF ZF ]
	jz  1f				/* Initial CPU cleans BSS */     -> 0xc143466d:	je     0xc1434671
	jmp checkCPUtype
1:
#endif /* CONFIG_SMP */

/*
 * start system 32-bit setup. We need to re-do some of the things done
 * in 16-bit mode for the "real" operations.
 */
	call setup_idt                                                  -> 0xc1434671:	call   0xc14347be

checkCPUtype:

	movl $-1,X86_CPUID		#  -1 for no CPUID initially     -> 0xc1434676:	movl   $0xffffffff,0xc167ab14

/* check if it is 486 or 386. */
/*
 * XXX - this does a lot of unnecessary setup.  Alignment checks don't
 * apply at our cpl of 0 and the stack ought to be aligned already, and
 * we don't need to preserve eflags.
 */

	movb $3,X86		# at least 386                           -> 0xc1434680:	movb   $0x3,0xc167ab00
(gdb) info registers eflags
eflags         0x46	[ PF ZF ]
	pushfl			# push EFLAGS
	popl %eax		# get EFLAGS
	movl %eax,%ecx		# save original EFLAGS
	xorl $0x240000,%eax	# flip AC and ID bits in EFLAGS
(gdb) info registers eax
eax            0x240046		2359366
	pushl %eax		# copy to EFLAGS
	popfl			# set EFLAGS
(gdb) info registers eflags
eflags         0x240046	[ PF ZF AC ID ]
	pushfl			# get new EFLAGS
	popl %eax		# put it in eax
(gdb) info registers eax
eax            0x240046		2359366
(gdb) info registers ecx
ecx            0x46	70
	xorl %ecx,%eax		# change in flags
(gdb) info registers eax ecx
eax            0x240000		2359296
ecx            0x46	70
	pushl %ecx		# restore original EFLAGS
	popfl
(gdb) info registers eflags
eflags         0x46	[ PF ZF ]
	testl $0x40000,%eax	# check if AC bit changed
(gdb) info registers eflags
eflags         0x6	[ PF ]
	je is386

	movb $4,X86		# at least 486
(gdb) info registers eax
eax            0x240000		2359296
	testl $0x200000,%eax	# check if ID bit changed
(gdb) info registers eflags
eflags         0x6	[ PF ]
	je is486

	/* get vendor info */
	xorl %eax,%eax			# call CPUID with 0 -> return vendor ID
	cpuid
(gdb) info registers eax ebx edx ecx
eax            0x4	4
ebx            0x756e6547	1970169159
edx            0x49656e69	1231384169
ecx            0x6c65746e	1818588270
	movl %eax,X86_CPUID		# save CPUID level
	movl %ebx,X86_VENDOR_ID		# lo 4 chars
	movl %edx,X86_VENDOR_ID+4	# next 4 chars
	movl %ecx,X86_VENDOR_ID+8	# last 4 chars

	orl %eax,%eax			# do we have processor info as well?
(gdb) info registers eflags
eflags         0x2	[ ]
	je is486

	movl $1,%eax		# Use the CPUID instruction to get CPU type
	cpuid
(gdb) info registers eax
eax            0x663	1635
	movb %al,%cl		# save reg for future use
	andb $0x0f,%ah		# mask processor family
	movb %ah,X86
	andb $0xf0,%al		# mask model
	shrb $4,%al
	movb %al,X86_MODEL
	andb $0x0f,%cl		# mask mask revision
	movb %cl,X86_MASK
(gdb) info registers edx
edx            0x781abf9	125938681
	movl %edx,X86_CAPABILITY

is486:	movl $0x50022,%ecx	# set AM, WP, NE and MP
	jmp 2f

is386:	movl $2,%ecx		# set MP
2:	movl %cr0,%eax
(gdb) info registers eax
eax            0x80000011	-2147483631
	andl $0x80000011,%eax	# Save PG,PE,ET
	orl %ecx,%eax
(gdb) info registers eax
eax            0x80050033	-2147155917
	movl %eax,%cr0

	call check_x87
	lgdt early_gdt_descr
	lidt idt_descr
	ljmp $(__KERNEL_CS),$1f
1:	movl $(__KERNEL_DS),%eax	# reload all the segment registers             -> 0xc143472a:	mov    $0x68,%eax
	movl %eax,%ss			# after changing gdt.

	movl $(__USER_DS),%eax		# DS/ES contains default USER segment         ->  0xc1434731:	mov    $0x7b,%eax
	movl %eax,%ds
	movl %eax,%es

	movl $(__KERNEL_PERCPU), %eax                                                 -> 0xc143473a:	mov    $0xd8,%eax
	movl %eax,%fs			# set this cpu's percpu

#ifdef CONFIG_CC_STACKPROTECTOR
	/*
	 * The linker can't handle this by relocation.  Manually set
	 * base address in stack canary segment descriptor.
	 */
	cmpb $0,ready                                                                 -> 0xc1434741:	cmpb   $0x0,0xc163bb08
(gdb) x/b 0xc163bb08
0xc163bb08:	0x00

	jne 1f
	movl $per_cpu__gdt_page,%eax
	movl $per_cpu__stack_canary,%ecx
(gdb) info registers eax ecx
eax            0xc1722000	-1049485312
ecx            0xc172a280	-1049451904
	movw %cx, 8 * GDT_ENTRY_STACK_CANARY + 2(%eax)      -> 0xc1434754:	mov    %cx,0xe2(%eax)
	shrl $16, %ecx
	movb %cl, 8 * GDT_ENTRY_STACK_CANARY + 4(%eax)      -> 0xc143475e:	mov    %cl,0xe4(%eax)
	movb %ch, 8 * GDT_ENTRY_STACK_CANARY + 7(%eax)      -> 0xc1434764:	mov    %ch,0xe7(%eax)
1:
#endif
	movl $(__KERNEL_STACK_CANARY),%eax                  -> 0xc143476a:	mov    $0xe0,%eax
	movl %eax,%gs

	xorl %eax,%eax			# Clear LDT
	lldt %ax

	cld			# gcc2 wants the direction flag cleared at all times
	pushl $0		# fake return address for unwinder
#ifdef CONFIG_SMP
	movb ready, %cl
(gdb) info registers cl
cl             0x0	0
	movb $1, ready
	cmpb $0,%cl		# the first CPU calls start_kernel
	je   1f
	movl (stack_start), %esp
1:
#endif /* CONFIG_SMP */
	jmp *(initial_code)

/*
 * We depend on ET to be correct. This checks for 287/387.
 */
check_x87:
	movb $0,X86_HARD_MATH
	clts
	fninit
	fstsw %ax
(gdb) info registers ax
ax             0x0	0
	cmpb $0,%al
	je 1f
	movl %cr0,%eax		/* no coprocessor: have to set bits */
	xorl $4,%eax		/* set EM */
	movl %eax,%cr0
	ret
	ALIGN
1:	movb $1,X86_HARD_MATH
	.byte 0xDB,0xE4		/* fsetpm for 287, ignored by 387 */                    -> 0xc14347bb:	fnsetpm(287 only)
	ret

/*
 *  setup_idt
 *
 *  sets up a idt with 256 entries pointing to
 *  ignore_int, interrupt gates. It doesn't actually load
 *  idt - that can be done only after paging has been enabled
 *  and the kernel moved to PAGE_OFFSET. Interrupts
 *  are enabled elsewhere, when we can be relatively
 *  sure everything is ok.
 *
 *  Warning: %esi is live across this function.
 */
setup_idt:
	lea ignore_int,%edx                                             -> 0xc14347be:	lea    0xc14348ac,%edx
(gdb) info registers edx
edx            0xc14348ac	-1052555092
	movl $(__KERNEL_CS << 16),%eax                                  -> 0xc14347c4:	mov    $0x600000,%eax
	movw %dx,%ax		/* selector = 0x0010 = cs */
(gdb) info registers eax
eax            0x6048ac		6310060
	movw $0x8E00,%dx	/* interrupt gate - dpl=0, present */
(gdb) info registers edx
edx            0xc1438e00	-1052537344
	lea idt_table,%edi                                              -> 0xc14347d0:	lea    0xc1638000,%edi
	mov $256,%ecx
(gdb) info registers edi
edi            0xc1638000	-1050443776
rp_sidt:
	movl %eax,(%edi)
	movl %edx,4(%edi)
	addl $8,%edi
	dec %ecx
	jne rp_sidt

.macro set_early_handler handler,trapno
	lea \handler,%edx
	movl $(__KERNEL_CS << 16),%eax
	movw %dx,%ax
	movw $0x8E00,%dx	/* interrupt gate - dpl=0, present */
	lea idt_table,%edi
	movl %eax,8*\trapno(%edi)
	movl %edx,8*\trapno+4(%edi)
.endm

	set_early_handler handler=early_divide_err,trapno=0 ->
   0xc14347e6:	lea    0xc143485e,%edx
   0xc14347ec:	mov    $0x600000,%eax
   0xc14347f1:	mov    %dx,%ax
   0xc14347f4:	mov    $0x8e00,%dx
   0xc14347f8:	lea    0xc1638000,%edi
   0xc14347fe:	mov    %eax,(%edi)
   0xc1434800:	mov    %edx,0x4(%edi)
	set_early_handler handler=early_illegal_opcode,trapno=6 -> 
   0xc1434803:	lea    0xc1434864,%edx
   0xc1434809:	mov    $0x600000,%eax
   0xc143480e:	mov    %dx,%ax
   0xc1434811:	mov    $0x8e00,%dx
   0xc1434815:	lea    0xc1638000,%edi
   0xc143481b:	mov    %eax,0x30(%edi)
   0xc143481e:	mov    %edx,0x34(%edi)
	set_early_handler handler=early_protection_fault,trapno=13 -> 
   0xc1434821:	lea    0xc143486d,%edx
   0xc1434827:	mov    $0x600000,%eax
   0xc143482c:	mov    %dx,%ax
   0xc143482f:	mov    $0x8e00,%dx
   0xc1434833:	lea    0xc1638000,%edi
   0xc1434839:	mov    %eax,0x68(%edi)
   0xc143483c:	mov    %edx,0x6c(%edi)
	set_early_handler handler=early_page_fault,trapno=14 ->
   0xc143483f:	lea    0xc1434874,%edx
   0xc1434845:	mov    $0x600000,%eax
   0xc143484a:	mov    %dx,%ax
   0xc143484d:	mov    $0x8e00,%dx
   0xc1434851:	lea    0xc1638000,%edi
   0xc1434857:	mov    %eax,0x70(%edi)
   0xc143485a:	mov    %edx,0x74(%edi)

	ret

early_divide_err:
	xor %edx,%edx
	pushl $0	/* fake errcode */
	jmp early_fault

early_illegal_opcode:
	movl $6,%edx
	pushl $0	/* fake errcode */
	jmp early_fault

early_protection_fault:
	movl $13,%edx
	jmp early_fault

early_page_fault:
	movl $14,%edx
	jmp early_fault

early_fault:
	cld
#ifdef CONFIG_PRINTK
	pusha
	movl $(__KERNEL_DS),%eax
	movl %eax,%ds
	movl %eax,%es
	cmpl $2,early_recursion_flag
	je hlt_loop
	incl early_recursion_flag
	movl %cr2,%eax
	pushl %eax
	pushl %edx		/* trapno */
	pushl $fault_msg
	call printk
#endif
	call dump_stack
hlt_loop:
	hlt
	jmp hlt_loop

/* This is the default interrupt "handler" :-) */
	ALIGN
ignore_int:
	cld
#ifdef CONFIG_PRINTK
	pushl %eax
	pushl %ecx
	pushl %edx
	pushl %es
	pushl %ds
	movl $(__KERNEL_DS),%eax
	movl %eax,%ds
	movl %eax,%es
	cmpl $2,early_recursion_flag
	je hlt_loop
	incl early_recursion_flag
	pushl 16(%esp)
	pushl 24(%esp)
	pushl 32(%esp)
	pushl 40(%esp)
	pushl $int_msg
	call printk

	call dump_stack

	addl $(5*4),%esp
	popl %ds
	popl %es
	popl %edx
	popl %ecx
	popl %eax
#endif
	iret

	__REFDATA
.align 4
ENTRY(initial_code)
	.long i386_start_kernel
ENTRY(initial_page_table)
	.long pa(swapper_pg_dir)

/*
 * BSS section
 */
__PAGE_ALIGNED_BSS
	.align PAGE_SIZE_asm
#ifdef CONFIG_X86_PAE
swapper_pg_pmd:
	.fill 1024*KPMDS,4,0
#else
ENTRY(swapper_pg_dir)
	.fill 1024,4,0
#endif
swapper_pg_fixmap:
	.fill 1024,4,0
#ifdef CONFIG_X86_TRAMPOLINE
ENTRY(trampoline_pg_dir)
	.fill 1024,4,0
#endif
ENTRY(empty_zero_page)
	.fill 4096,1,0

/*
 * This starts the data section.
 */
#ifdef CONFIG_X86_PAE
__PAGE_ALIGNED_DATA
	/* Page-aligned for the benefit of paravirt? */
	.align PAGE_SIZE_asm
ENTRY(swapper_pg_dir)
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR),0	/* low identity map */
# if KPMDS == 3
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR),0
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR+0x1000),0
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR+0x2000),0
# elif KPMDS == 2
	.long	0,0
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR),0
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR+0x1000),0
# elif KPMDS == 1
	.long	0,0
	.long	0,0
	.long	pa(swapper_pg_pmd+PGD_IDENT_ATTR),0
# else
#  error "Kernel PMDs should be 1, 2 or 3"
# endif
	.align PAGE_SIZE_asm		/* needs to be page-sized too */
#endif

.data
ENTRY(stack_start)
	.long init_thread_union+THREAD_SIZE
	.long __BOOT_DS

ready:	.byte 0

early_recursion_flag:
	.long 0

int_msg:
	.asciz "Unknown interrupt or fault at: %p %p %p\n"

fault_msg:
/* fault info: */
	.ascii "BUG: Int %d: CR2 %p\n"
/* pusha regs: */
	.ascii "     EDI %p  ESI %p  EBP %p  ESP %p\n"
	.ascii "     EBX %p  EDX %p  ECX %p  EAX %p\n"
/* fault frame: */
	.ascii "     err %p  EIP %p   CS %p  flg %p\n"
	.ascii "Stack: %p %p %p %p %p %p %p %p\n"
	.ascii "       %p %p %p %p %p %p %p %p\n"
	.asciz "       %p %p %p %p %p %p %p %p\n"

#include "../../x86/xen/xen-head.S"

/*
 * The IDT and GDT 'descriptors' are a strange 48-bit object
 * only used by the lidt and lgdt instructions. They are not
 * like usual segment descriptors - they consist of a 16-bit
 * segment size, and 32-bit linear address value:
 */

.globl boot_gdt_descr
.globl idt_descr

	ALIGN
# early boot GDT descriptor (must use 1:1 address mapping)
	.word 0				# 32 bit align gdt_desc.address
boot_gdt_descr:
	.word __BOOT_DS+7
	.long boot_gdt - __PAGE_OFFSET

	.word 0				# 32-bit align idt_desc.address
idt_descr:
	.word IDT_ENTRIES*8-1		# idt contains 256 entries
	.long idt_table

# boot GDT descriptor (later on used by CPU#0):
	.word 0				# 32 bit align gdt_desc.address
ENTRY(early_gdt_descr)
	.word GDT_ENTRIES*8-1
	.long per_cpu__gdt_page		/* Overwritten for secondary CPUs */

/*
 * The boot_gdt must mirror the equivalent in setup.S and is
 * used only for booting.
 */
	.align L1_CACHE_BYTES
ENTRY(boot_gdt)
	.fill GDT_ENTRY_BOOT_CS,8,0
	.quad 0x00cf9a000000ffff	/* kernel 4GB code at 0x00000000 */
	.quad 0x00cf92000000ffff	/* kernel 4GB data at 0x00000000 */

```

Linux initialization jump to c function i386_start_kernel at the end of execution of above assembly code, i386_start_kernel is succinct and easy to read. In i386_start_kernel it firstly reserves memory pages for different purposes, then reserve memory region for bios [Extended BIOS Data Area (EBDA)](http://wiki.osdev.org/Memory_Map_(x86)), finally involve start_kernel for linux startup.

```i386_start_kernel_context
i386_start_kernel
    |--reserve_trampoline_memory
        |--reserve_early
    |--reserve_early
    |--i386_default_early_setup
        |--reserve_ebda_region
            |--get_bios_ebda
            |--reserve_early_overlap_ok
    |--start_kernel
```

```i386_start_kernel

arch/x86/kernel/head32.c:30

void __init i386_start_kernel(void)
{
        reserve_trampoline_memory();

        reserve_early(__pa_symbol(&_text), __pa_symbol(&__bss_stop), "TEXT DATA BSS");

#ifdef CONFIG_BLK_DEV_INITRD
        /* Reserve INITRD */
        if (boot_params.hdr.type_of_loader && boot_params.hdr.ramdisk_image) {
(gdb) p boot_params.hdr.type_of_loader
$1 = 114 'r'
(gdb) p boot_params.hdr.ramdisk_image 
$2 = 125603840
                u64 ramdisk_image = boot_params.hdr.ramdisk_image;
                u64 ramdisk_size  = boot_params.hdr.ramdisk_size;
                u64 ramdisk_end   = ramdisk_image + ramdisk_size;
                reserve_early(ramdisk_image, ramdisk_end, "RAMDISK");
        }
#endif

        /* Call the subarch specific early setup function */
        switch (boot_params.hdr.hardware_subarch) {
(gdb) p boot_params.hdr.hardware_subarch
$3 = 0
        case X86_SUBARCH_MRST:
                x86_mrst_early_setup();
                break;
        default:
                i386_default_early_setup();
                break;
        }

        /*
         * At this point everything still needed from the boot loader
         * or BIOS or kernel text should be early reserved or marked not
         * RAM in e820. All other memory is free game.
         */

        start_kernel();
}
```

# Links
  * [Physical Address Extension OSDev](http://wiki.osdev.org/Physical_Address_Extension)
  * [Physical Address Extension wikipedia](https://en.wikipedia.org/wiki/Physical_Address_Extension)
  * [Setting Up Paging With PAE](http://wiki.osdev.org/Setting_Up_Paging_With_PAE)
  * [Paging](http://wiki.osdev.org/Paging)
  * [CPUID](https://en.wikipedia.org/wiki/CPUID)
  * [Intel® 64 and IA-32 Architectures Developer's Manual: Vol. 2A](http://www.intel.com/content/www/us/en/architecture-and-technology/64-ia-32-architectures-software-developer-vol-2a-manual.html)
  * [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) 
  * [AC bit](https://en.wikipedia.org/wiki/FLAGS_register)
  * [x87](https://en.wikipedia.org/wiki/X87)
  * [Extended BIOS Data Area (EBDA)](http://wiki.osdev.org/Memory_Map_(x86))
  
