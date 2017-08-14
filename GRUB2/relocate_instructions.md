# Relocate instructions

Why relocation needed?

grub_relocator32_start and grub_relocator32_end are defined in grub-core/lib/i386/relocator32.S

grub_relocator_forward_start, grub_relocator_forward_end, grub_relocator_backward_start, grub_relocator_backward_end are defined in grub-core/lib/i386/relocator_asm.S

GRUB initializes variables in above code blocks and copy the code to specified destionantion address, GRUB will execute instructions later before transfering to linux code.

```grub_relocator_code_blocks
grub_relocator32_start: 0x7f9e080
grub_relocator32_end: 0x7f9e150
grub_relocator_forward_start: 0x7f9dee0
grub_relocator_forward_end: 0x7f9def6
grub_relocator_backward_start: 0x7f9dec0
grub_relocator_backward_end: 0x7f9dee0
```


```grub_relocator32_boot
grub_relocator32_boot (rel=0x7fdb210, state=..., avoid_efi_bootservices=0)
    at lib/i386/relocator.c:167

grub-core/lib/i386/relocator.c:156

grub_err_t
grub_relocator32_boot (struct grub_relocator *rel,
                       struct grub_relocator32_state state,
                       int avoid_efi_bootservices)
{
  grub_err_t err;
  void *relst;
  grub_relocator_chunk_t ch;

  err = grub_relocator_alloc_chunk_align (rel, &ch, 0,
                                          (0xffffffff - RELOCATOR_SIZEOF (32))
                                          + 1, RELOCATOR_SIZEOF (32), 16,
                                          GRUB_RELOCATOR_PREFERENCE_NONE,
                                          avoid_efi_bootservices);
  if (err)
    return err;

  grub_relocator32_eax = state.eax;
  grub_relocator32_ebx = state.ebx;
  grub_relocator32_ecx = state.ecx;
  grub_relocator32_edx = state.edx;
  grub_relocator32_eip = state.eip;
  grub_relocator32_esp = state.esp;
  grub_relocator32_ebp = state.ebp;
  grub_relocator32_esi = state.esi;
  grub_relocator32_edi = state.edi;

  grub_memmove (get_virtual_current_address (ch), &grub_relocator32_start,
                RELOCATOR_SIZEOF (32));
(gdb) p ch.srcv 
$21 = (void *) 0x9df000
(gdb) p &grub_relocator32_start 
$22 = (grub_uint8_t *) 0x7f96600 <grub_relocator32_start> "\211\306\005\t"

  err = grub_relocator_prepare_relocs (rel, get_physical_target_address (ch),
                                       &relst, NULL);
  if (err)
    return err;

(gdb) x/10i $eip
   0x7f9e3a5 <grub_relocator32_boot+183>:    cli    
   0x7f9e3a6 <grub_relocator32_boot+184>:    call *-0x10(%ebp)
   0x7f9e3a9 <grub_relocator32_boot+187>:    
    jmp    0x7f9e3ad <grub_relocator32_boot+191>
   0x7f9e3ab <grub_relocator32_boot+189>:    mov    %eax,%ebx
   0x7f9e3ad <grub_relocator32_boot+191>:    lea    -0x8(%ebp),%esp
   0x7f9e3b0 <grub_relocator32_boot+194>:    mov    %ebx,%eax
   0x7f9e3b2 <grub_relocator32_boot+196>:    pop    %ebx
   0x7f9e3b3 <grub_relocator32_boot+197>:    pop    %esi
   0x7f9e3b4 <grub_relocator32_boot+198>:    pop    %ebp
   0x7f9e3b5 <grub_relocator32_boot+199>:    ret    $0x28
(gdb) info registers ebp
ebp            0x7fbbc    0x7fbbc
(gdb) x/w 0x7fbbc-0x10
0x7fbac:    0x009df0d0
  asm volatile ("cli");
  ((void (*) (void)) relst) ();

  /* Not reached.  */
  return GRUB_ERR_NONE;
}

-------------------------------------------------------------------------------------------------------------

grub_relocator_alloc_chunk_align (rel=rel@entry=0x7fdb210, 
    out=out@entry=0x7fbb0, avoid_efi_boot_services=0) at lib/relocator.c:1317

grub-core/lib/relocator.c:1316

grub_err_t
grub_relocator_alloc_chunk_align (struct grub_relocator *rel,
                  grub_relocator_chunk_t *out,
                  grub_phys_addr_t min_addr,
                  grub_phys_addr_t max_addr,
                  grub_size_t size, grub_size_t align,
                  int preference,
                  int avoid_efi_boot_services)
{
  grub_addr_t min_addr2 = 0, max_addr2;
  struct grub_relocator_chunk *chunk;

  if (max_addr > ~size)
(gdb) p /x min_addr
$1 = 0x0
(gdb) p /x max_addr
$2 = 0xffffff30
(gdb) p /x size
$3 = 0xd0
(gdb) p /x align 
$4 = 0x10
(gdb) p preference 
$5 = 0
    max_addr = ~size;

#ifdef GRUB_MACHINE_PCBIOS
  if (min_addr < 0x1000)
    min_addr = 0x1000;
#endif

  grub_dprintf ("relocator", "chunks = %p\n", rel->chunks);
(gdb) p /x max_addr
$6 = 0xffffff2f
(gdb) p /x min_addr
$7 = 0x1000

  chunk = grub_malloc (sizeof (struct grub_relocator_chunk));
  if (!chunk)
    return grub_errno;

  if (malloc_in_range (rel, min_addr, max_addr, align,
               size, chunk,
               preference != GRUB_RELOCATOR_PREFERENCE_HIGH, 1))
    {
      grub_dprintf ("relocator", "allocated 0x%llx/0x%llx\n",
            (unsigned long long) chunk->src,
            (unsigned long long) chunk->src);
      grub_dprintf ("relocator", "chunks = %p\n", rel->chunks);
(gdb) p /x chunk->src
$9 = 0x9df000
(gdb) p /x rel->chunks 
$10 = 0x7fe1970
      chunk->target = chunk->src;
      chunk->size = size;
      chunk->next = rel->chunks;
      rel->chunks = chunk;
      chunk->srcv = grub_map_memory (chunk->src, chunk->size);
      *out = chunk;
      return GRUB_ERR_NONE;
(gdb) p /x chunk->target 
$11 = 0x9df000
(gdb) p /x chunk->size
size      size_t    sizetype  
(gdb) p /x chunk->size
$12 = 0xd0
(gdb) p /x chunk->next 
$13 = 0x7fe1970
(gdb) p /x rel->chunks 
$14 = 0x7fa46f0
(gdb) p /x chunk
$15 = 0x7fa46f0
(gdb) p /x chunk->srcv 
$16 = 0x9df000
    }

  adjust_limits (rel, &min_addr2, &max_addr2, min_addr, max_addr);
  grub_dprintf ("relocator", "Adjusted limits from %lx-%lx to %lx-%lx\n",
        (unsigned long) min_addr, (unsigned long) max_addr,
        (unsigned long) min_addr2, (unsigned long) max_addr2);

  do
    {
      if (malloc_in_range (rel, min_addr2, max_addr2, align,
               size, chunk, 1, 1))
    break;

      if (malloc_in_range (rel, rel->highestnonpostaddr, ~(grub_addr_t)0, 1,
               size, chunk, 0, 1))
    {
      if (rel->postchunks > chunk->src)
        rel->postchunks = chunk->src;
      break;
    }

      return grub_error (GRUB_ERR_OUT_OF_MEMORY, N_("out of memory"));
    }
  while (0);

  {
    int found = 0;
    auto int NESTED_FUNC_ATTR hook (grub_uint64_t, grub_uint64_t,
                    grub_memory_type_t);
    int NESTED_FUNC_ATTR hook (grub_uint64_t addr, grub_uint64_t sz,
                   grub_memory_type_t type)
    {
      grub_uint64_t candidate;
      if (type != GRUB_MEMORY_AVAILABLE)
    return 0;
      candidate = ALIGN_UP (addr, align);
      if (candidate < min_addr)
    candidate = ALIGN_UP (min_addr, align);
      if (candidate + size > addr + sz
      || candidate > ALIGN_DOWN (max_addr, align))
    return 0;
      if (preference == GRUB_RELOCATOR_PREFERENCE_HIGH)
    candidate = ALIGN_DOWN (min (addr + sz - size, max_addr), align);
      if (!found || (preference == GRUB_RELOCATOR_PREFERENCE_HIGH
             && candidate > chunk->target))
    chunk->target = candidate;
      if (!found || (preference == GRUB_RELOCATOR_PREFERENCE_LOW
             && candidate < chunk->target))
    chunk->target = candidate;
      found = 1;
      return 0;
    }

#ifdef GRUB_MACHINE_EFI
    grub_efi_mmap_iterate (hook, avoid_efi_boot_services);
#elif defined (__powerpc__)
    (void) avoid_efi_boot_services;
    grub_machine_mmap_iterate (hook);
#else
    (void) avoid_efi_boot_services;
    grub_mmap_iterate (hook);
#endif
    if (!found)
      return grub_error (GRUB_ERR_BAD_OS, "couldn't find suitable memory target");
  }
  while (1)
    {
      struct grub_relocator_chunk *chunk2;
      for (chunk2 = rel->chunks; chunk2; chunk2 = chunk2->next)
    if ((chunk2->target <= chunk->target
         && chunk->target < chunk2->target + chunk2->size)
        || (chunk->target <= chunk2->target && chunk2->target
        < chunk->target + size))
      {
        if (preference == GRUB_RELOCATOR_PREFERENCE_HIGH)
          chunk->target = ALIGN_DOWN (chunk2->target, align);
        else
          chunk->target = ALIGN_UP (chunk2->target + chunk2->size, align);
        break;
      }
      if (!chunk2)
    break;
    }

  grub_dprintf ("relocator", "relocators_size=%ld\n",
        (unsigned long) rel->relocators_size);

  if (chunk->src < chunk->target)
    rel->relocators_size += grub_relocator_backward_size;
  if (chunk->src > chunk->target)
    rel->relocators_size += grub_relocator_forward_size;

  grub_dprintf ("relocator", "relocators_size=%ld\n",
        (unsigned long) rel->relocators_size);

  chunk->size = size;
  chunk->next = rel->chunks;
  rel->chunks = chunk;
  grub_dprintf ("relocator", "cur = %p, next = %p\n", rel->chunks,
        rel->chunks->next);
  chunk->srcv = grub_map_memory (chunk->src, chunk->size);
  *out = chunk;
#ifdef DEBUG_RELOCATOR
  grub_memset (chunk->srcv, 0xfa, chunk->size);
  grub_mm_check ();
#endif
  return GRUB_ERR_NONE;
}
```

Copy instructions started from grub_relocator32_start to adderss 0x9df000, length 208, 

```grub_memmove
grub_memmove (dest=0x9df000, src=0x7f96600 <grub_relocator32_start>, 
    n=n@entry=208) at kern/misc.c:52

grub-core/kern/misc.c:46

void *
grub_memmove (void *dest, const void *src, grub_size_t n)
{
  char *d = (char *) dest;
  const char *s = (const char *) src;

  if (d < s)
    while (n--)
      *d++ = *s++;
  else
    {
      d += n;
      s += n;

      while (n--)
        *--d = *--s;
    }

  return dest;
}
```
Result of grub\_memmove in destination, address start from 0x9df000 size 0xd0

![grub_memmove_in_grub_relocator32_boot](/GRUB2/resource/grub_memmove_in_grub_relocator32_boot.png)

Instructions starting from 0x9df000 as follow:

```assembly

   0x9df000:	mov    %eax,%esi
   0x9df002:	add    $0x9,%eax
   0x9df007:	jmp    *%eax
   0x9df009:	lea    0x48(%esi),%eax
   0x9df00f:	mov    %eax,0x40(%esi)
   0x9df015:	lea    0xb0(%esi),%eax
   0x9df01b:	mov    %eax,0x32(%esi)
   0x9df021:	lgdtl  0x30(%esi)
   0x9df028:	ljmp   *0x40(%esi)
(gdb) info registers esi
esi            0x9df000	 10350592
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
   0x9df0a5:	lea    0x0(%esi,%eiz,1),%esi
   0x9df0a9:	lea    0x0(%edi,%eiz,1),%edi
   0x9df0b0:	add    %al,(%eax)
```

Original assembly code of above instructions in memory as follow, where PREAMBLE and RELOAD_GDT defined in grub-core/lib/i386/relocator_common.S


```relocator32_assembly
grub-core/lib/i386/relocator32.S:19

/* The code segment of the protected mode.  */
#define CODE_SEGMENT	0x10

/* The data segment of the protected mode.  */
#define DATA_SEGMENT	0x18

#include "relocator_common.S"

	.p2align	4	/* force 16-byte alignment */

VARIABLE(grub_relocator32_start)
	PREAMBLE

	RELOAD_GDT
	.code32
	/* Update other registers. */
	movl	$DATA_SEGMENT, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %fs
	movl	%eax, %gs
	movl	%eax, %ss

	DISABLE_PAGING

#ifdef __x86_64__
	/* Disable amd64. */
	movl	$GRUB_MEMORY_CPU_AMD64_MSR, %ecx
	rdmsr
	andl	$(~GRUB_MEMORY_CPU_AMD64_MSR_ON), %eax
	wrmsr
#endif

	/* Turn off PAE. */
	movl	%cr4, %eax
	andl	$(~GRUB_MEMORY_CPU_CR4_PAE_ON), %eax
	movl	%eax, %cr4

	jmp	LOCAL(cont2)
LOCAL(cont2):
	.code32

	/* mov imm32, %eax */
	.byte	0xb8
VARIABLE(grub_relocator32_esp)
	.long	0

	movl	%eax, %esp

	/* mov imm32, %eax */
	.byte	0xb8
VARIABLE(grub_relocator32_ebp)
	.long	0

	movl	%eax, %ebp

	/* mov imm32, %eax */
	.byte	0xb8
VARIABLE(grub_relocator32_esi)
	.long	0

	movl	%eax, %esi

	/* mov imm32, %eax */
	.byte	0xb8
VARIABLE(grub_relocator32_edi)
	.long	0

	movl	%eax, %edi

	/* mov imm32, %eax */
	.byte	0xb8
VARIABLE(grub_relocator32_eax)
	.long	0

	/* mov imm32, %ebx */
	.byte	0xbb
VARIABLE(grub_relocator32_ebx)
	.long	0

	/* mov imm32, %ecx */
	.byte	0xb9
VARIABLE(grub_relocator32_ecx)
	.long	0

	/* mov imm32, %edx */
	.byte	0xba
VARIABLE(grub_relocator32_edx)
	.long	0

	/* Cleared direction flag is of no problem with any current
	   payload and makes this implementation easier.  */
	cld

	.byte	0xea
VARIABLE(grub_relocator32_eip)
	.long	0
	.word	CODE_SEGMENT

	/* GDT. Copied from loader/i386/linux.c. */
	.p2align	4
LOCAL(gdt):
	/* NULL.  */
	.byte 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00

	/* Reserved.  */
	.byte 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00

	/* Code segment.  */
	.byte 0xFF, 0xFF, 0x00, 0x00, 0x00, 0x9A, 0xCF, 0x00

	/* Data segment.  */
	.byte 0xFF, 0xFF, 0x00, 0x00, 0x00, 0x92, 0xCF, 0x00
LOCAL(gdt_end):

VARIABLE(grub_relocator32_end)
```

Continue routines in grub\_relocator32\_boot

```grub_relocator_prepare_relocs
grub_relocator_prepare_relocs (rel=rel@entry=0x7fdb210, addr=10350592, 
    relstart=relstart@entry=0x7fbac, relsize=0x0) at lib/relocator.c:1488

grub-core/lib/relocator.c:1485

grub_err_t
grub_relocator_prepare_relocs (struct grub_relocator *rel, grub_addr_t addr,
                   void **relstart, grub_size_t *relsize)
{
  grub_uint8_t *rels;
  grub_uint8_t *rels0;
  struct grub_relocator_chunk *sorted;
  grub_size_t nchunks = 0;
  unsigned j;
  struct grub_relocator_chunk movers_chunk;

  grub_dprintf ("relocator", "Preparing relocs (size=%ld)\n",
        (unsigned long) rel->relocators_size);

  if (!malloc_in_range (rel, 0, ~(grub_addr_t)0 - rel->relocators_size + 1,
            grub_relocator_align,
            rel->relocators_size, &movers_chunk, 1, 1))
    return grub_error (GRUB_ERR_OUT_OF_MEMORY, N_("out of memory"));
  movers_chunk.srcv = rels = rels0
    = grub_map_memory (movers_chunk.src, movers_chunk.size);
(gdb) p movers_chunk 
$1 = {next = 0x5600, src = 10350800, srcv = 0x9df0d0, target = 0, size = 61, 
  subchunks = 0x7f80df0, nsubchunks = 1}

  if (relsize)
    *relsize = rel->relocators_size;

  grub_dprintf ("relocator", "Relocs allocated at %p\n", movers_chunk.srcv);

  {
    unsigned i;
    grub_size_t count[257];
    struct grub_relocator_chunk *from, *to, *tmp;

    grub_memset (count, 0, sizeof (count));

(gdb) print_chunks 
chunk: 0x7fa46f0, src = 0x9df000, target = 0x9df000, srcv = 0x9df000, size = 0xd0
chunk: 0x7fe1970, src = 0x7feae60, target = 0x8b000, srcv = 0x7feae60, size = 0x5000
chunk: 0x7fda990, src = 0x782d000, target = 0x782d000, srcv = 0x782d000, size = 0x2906cc
chunk: 0x7fdb1e0, src = 0x100000, target = 0x1000000, srcv = 0x100000, size = 0x8df000

    {
        struct grub_relocator_chunk *chunk;
    for (chunk = rel->chunks; chunk; chunk = chunk->next)
      {
        grub_dprintf ("relocator", "chunk %p->%p, 0x%lx\n", 
              (void *) chunk->src, (void *) chunk->target,
              (unsigned long) chunk->size);
        nchunks++;
        count[(chunk->src & 0xff) + 1]++;
      }
    }
(gdb) p nchunks
$2 = 4
(gdb) p count
$3 = {0, 3, 0 <repeats 95 times>, 1, 0 <repeats 159 times>}
    from = grub_malloc (nchunks * sizeof (sorted[0]));
    to = grub_malloc (nchunks * sizeof (sorted[0]));
    if (!from || !to)
      {
    grub_free (from);
    grub_free (to);
    return grub_errno;
      }

    for (j = 0; j < 256; j++)
      count[j+1] += count[j];
(gdb) p count
$4 = {0, 3 <repeats 96 times>, 4 <repeats 160 times>}

    {
      struct grub_relocator_chunk *chunk;
      for (chunk = rel->chunks; chunk; chunk = chunk->next)
    from[count[chunk->src & 0xff]++] = *chunk;
    }
(gdb) p from[0]
$5 = {next = 0x7fe1970, src = 10350592, srcv = 0x9df000, target = 10350592, 
  size = 208, subchunks = 0x7f930c0, nsubchunks = 1}
(gdb) p from[1]
$6 = {next = 0x7fdb1e0, src = 126013440, srcv = 0x782d000, 
  target = 126013440, size = 2688716, subchunks = 0x7fb5c40, nsubchunks = 1}
(gdb) p from[2]
$7 = {next = 0x0, src = 1048576, srcv = 0x100000, target = 16777216, 
  size = 9302016, subchunks = 0x7fd99d0, nsubchunks = 1}
(gdb) p from[3]
$8 = {next = 0x7fda990, src = 134131296, srcv = 0x7feae60, target = 569344, 
  size = 20480, subchunks = 0x7fa2d60, nsubchunks = 1}

    for (i = 1; i < GRUB_CPU_SIZEOF_VOID_P; i++)
      {
    grub_memset (count, 0, sizeof (count));
    for (j = 0; j < nchunks; j++)
      count[((from[j].src >> (8 * i)) & 0xff) + 1]++;
    for (j = 0; j < 256; j++)
      count[j+1] += count[j];
    for (j = 0; j < nchunks; j++)
      to[count[(from[j].src >> (8 * i)) & 0xff]++] = from[j];
    tmp = to;
    to = from;
    from = tmp;
      }
(gdb) p from[0]
$9 = {next = 0x0, src = 1048576, srcv = 0x100000, target = 16777216, 
  size = 9302016, subchunks = 0x7fd99d0, nsubchunks = 1}
(gdb) p from[1]
$10 = {next = 0x7fe1970, src = 10350592, srcv = 0x9df000, target = 10350592, 
  size = 208, subchunks = 0x7f930c0, nsubchunks = 1}
(gdb) p from[2]
$11 = {next = 0x7fdb1e0, src = 126013440, srcv = 0x782d000, 
  target = 126013440, size = 2688716, subchunks = 0x7fb5c40, nsubchunks = 1}
(gdb) p from[3]
$12 = {next = 0x7fda990, src = 134131296, srcv = 0x7feae60, target = 569344, 
  size = 20480, subchunks = 0x7fa2d60, nsubchunks = 1}

    sorted = from;
    grub_free (to);
  }

  for (j = 0; j < nchunks; j++)
    {
      grub_dprintf ("relocator", "sorted chunk %p->%p, 0x%lx\n", 
            (void *) sorted[j].src, (void *) sorted[j].target,
            (unsigned long) sorted[j].size);
      if (sorted[j].src < sorted[j].target)
    {
      grub_cpu_relocator_backward ((void *) rels,
                       sorted[j].srcv,
                       grub_map_memory (sorted[j].target,
                            sorted[j].size),
                       sorted[j].size);
      rels += grub_relocator_backward_size;
    }
      if (sorted[j].src > sorted[j].target)
    {
      grub_cpu_relocator_forward ((void *) rels,
                      sorted[j].srcv,
                      grub_map_memory (sorted[j].target,
                               sorted[j].size),
                      sorted[j].size);
      rels += grub_relocator_forward_size;
    }
      if (sorted[j].src == sorted[j].target)
    grub_arch_sync_caches (sorted[j].srcv, sorted[j].size);
    }
  grub_cpu_relocator_jumper ((void *) rels, (grub_addr_t) addr);
  *relstart = rels0;
(gdb) p rels
$13 = (grub_uint8_t *) 0x9df106 "\270"
(gdb) p *relstart 
$14 = (void *) 0x9df0d0
  grub_free (sorted);
  return GRUB_ERR_NONE;
}
```

grub\_cpu\_relocator\_backward and grub\_cpu\_relocator\_forward are similar, set values defined in grub-core/lib/i386/relocator\_asm.S and move the code in relocator_asm.S to destitnation. We will further check the code again lator.

```grub_cpu_relocator_backward_grub_cpu_relocator_forward
grub-core/lib/i386/relocator.c:130

void
grub_cpu_relocator_backward (void *ptr, void *src, void *dest,
                 grub_size_t size)
{
  grub_relocator_backward_dest = dest;
  grub_relocator_backward_src = src;
  grub_relocator_backward_chunk_size = size;

  grub_memmove (ptr,
        &grub_relocator_backward_start,
        RELOCATOR_SIZEOF (_backward));
}

void
grub_cpu_relocator_forward (void *ptr, void *src, void *dest,
                 grub_size_t size)
{
  grub_relocator_forward_dest = dest;
  grub_relocator_forward_src = src;
  grub_relocator_forward_chunk_size = size;

  grub_memmove (ptr,
        &grub_relocator_forward_start,
        RELOCATOR_SIZEOF (_forward));
}
```

Set the assembly instructions from address 0x9df106.

```grub_cpu_relocator_jumper
Breakpoint 6, grub_cpu_relocator_jumper (rels=rels@entry=0x9df106, 
    addr=addr@entry=10350592) at lib/i386/relocator.c:105

grub-core/lib/i386/relocator.c:103

void
grub_cpu_relocator_jumper (void *rels, grub_addr_t addr)
{
  grub_uint8_t *ptr;
  ptr = rels;
#ifdef __x86_64__
  /* movq imm64, %rax (for relocator) */
  *(grub_uint8_t *) ptr = 0x48;
  ptr++;
  *(grub_uint8_t *) ptr = 0xb8;
  ptr++;
  *(grub_uint64_t *) ptr = addr;
  ptr += sizeof (grub_uint64_t);
#else
  /* movl imm32, %eax (for relocator) */
  *(grub_uint8_t *) ptr = 0xb8;
  ptr++;
  *(grub_uint32_t *) ptr = addr;
  ptr += sizeof (grub_uint32_t);
#endif
  /* jmp $eax/$rax */
  *(grub_uint8_t *) ptr = 0xff;
  ptr++;
  *(grub_uint8_t *) ptr = 0xe0;
  ptr++;
(gdb) x/b 0x9df106
0x9df106:    0xb8
(gdb) x/w 0x9df107
0x9df107:    0x009df000
(gdb) x/b 0x9df107+4
0x9df10b:    0xff
(gdb) x/b 0x9df107+5
0x9df10c:    0xe0
(gdb) x/10i 0x9df106
   0x9df106:    mov    $0x9df000,%eax
   0x9df10b:    jmp    *%eax
   0x9df10d:    add    %al,(%eax)
   0x9df10f:    add    %dl,0x7eb4a(%eax)
   0x9df115:    add    %al,(%eax)
   0x9df117:    add    %al,(%ebx)
   0x9df119:    add    %al,(%eax)
   0x9df11b:    add    %ah,%al
   0x9df11d:    out    %al,(%dx)
   0x9df11e:    popa   
}
```

In grub_relocator32_boot, it involves function located at address 0x9df0d0, let's check what grub does before jumping to linux code. 

1. Grub copies two blocks of code:

   a. source address: 0x100000+0x8df000-0x1, destination address: 0x1000000+0x8df000-0x1, length: 0x8df000
   
   b. source address: 0x7e9fa70, destination address: 0x8b000, length: 0x5000
 
2. Jump to address 0x9df000

Where is following instructions come from?

```assembly
#Instuctions between 0x9df0d0 and 0x9df0ee generated by grub_cpu_relocator_backward
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
--------------------------------------------------------------------------------------------------------------------------------
#Instructions between 0x9df0f0 and 0x9df104 generated by grub_cpu_relocator_forward
   0x9df0f0:	mov    $0x8b000,%eax
   0x9df0f5:	mov    %eax,%edi
   0x9df0f7:	mov    $0x7e9fa70,%eax
   0x9df0fc:	mov    %eax,%esi
   0x9df0fe:	mov    $0x5000,%ecx
   0x9df103:	cld    
   0x9df104:	rep movsb %ds:(%esi),%es:(%edi)
--------------------------------------------------------------------------------------------------------------------------------
#Following instructions generated by grub_cpu_relocator_jumper
   0x9df106:	mov    $0x9df000,%eax
   0x9df10b:	jmp    *%eax
```

The input parameters of grub_cpu_relocator_backward when it involved first time as follow, source address is 0x100000, destination address is 0x1000000, length is 9302016(0x0x8df000).

```input_parameters_of_grub_cpu_relocator_backward
grub_cpu_relocator_backward (ptr=ptr@entry=0x9df0d0, src=0x100000, 
    dest=0x1000000, size=9302016) at lib/i386/relocator.c:135
```

Inside grub_cpu_relocator_backward, it assigns the addresses and length to specified variables, after that it copies instructions starting from grub_relocator_backward_start to address starting at 0x9df0d0, assembly code from grub_relocator_backward_start to grub_relocator_backward_end shown in following code block.

```assembly
grub-core/lib/i386/relocator_asm.S:24

VARIABLE(grub_relocator_backward_start)
        /* mov imm32, %eax */
        .byte   0xb8
VARIABLE(grub_relocator_backward_dest)
        .long   0
        movl    %eax, %edi

        /* mov imm32, %eax */
        .byte   0xb8
VARIABLE(grub_relocator_backward_src)
        .long   0
        movl    %eax, %esi

        /* mov imm32, %ecx */
        .byte   0xb9
VARIABLE(grub_relocator_backward_chunk_size)
        .long   0

        add     %ecx, %esi
        add     %ecx, %edi


        /* Backward movsb is implicitly off-by-one.  compensate that.  */
        sub     $1,     %esi
        sub     $1,     %edi

        /* Backward copy.  */
        std

        rep
        movsb
VARIABLE(grub_relocator_backward_end)
```

The source and target address of array sorted is same, let's check the fourth item of the array, this time grub involves grub_cpu_relocator_forward, the input parameters as follow:

```input_parameters_of_grub_cpu_relocator_forward
grub_cpu_relocator_forward (ptr=ptr@entry=0x9df0f0, src=0x7feae60, 
    dest=0x8b000, size=20480) at lib/i386/relocator.c:148
```

Yes, it's similar behavior with grub_cpu_relocator_backward. Instructions between grub_relocator_forward_start and grub_relocator_forward_end shown in following block.

```assembly
VARIABLE(grub_relocator_forward_start)
        /* mov imm32, %eax */
        .byte   0xb8
VARIABLE(grub_relocator_forward_dest)
        .long   0
        movl    %eax, %edi

        /* mov imm32, %rax */
        .byte   0xb8
VARIABLE(grub_relocator_forward_src)
        .long   0
        movl    %eax, %esi

        /* mov imm32, %ecx */
        .byte   0xb9
VARIABLE(grub_relocator_forward_chunk_size)
        .long   0

        /* Forward copy.  */
        cld
        rep
        movsb
VARIABLE(grub_relocator_forward_end)

```
