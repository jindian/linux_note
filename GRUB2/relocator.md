Relocator, prepare before transfer to linux
========================================================================================================================================

In grub_relocator allocate memory for the destination of relocated code and move relocate code to destionation from grub_relocator32_start which defined in relocator32.S.

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

Result of grub_memmove in destination, address start from 0x9df000 size 0xd0

![grub_memmove_in_grub_relocator32_boot](/GRUB2/grub_memmove_in_grub_relocator32_boot.png)

Continue routines in grub_relocator32_boot

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
(gdb) p /x addr
$2 = 0x9df000
(gdb) p /x relstart 
$3 = 0x7fbac
(gdb) p /x relsize 
$4 = 0x0
(gdb) p rel->relocators_size 
$5 = 61
(gdb) p grub_relocator_align 
$6 = 1

  if (!malloc_in_range (rel, 0, ~(grub_addr_t)0 - rel->relocators_size + 1,
                        grub_relocator_align,
                        rel->relocators_size, &movers_chunk, 1, 1))
    return grub_error (GRUB_ERR_OUT_OF_MEMORY, N_("out of memory"));
  movers_chunk.srcv = rels = rels0
    = grub_map_memory (movers_chunk.src, movers_chunk.size);
$7 = {next = 0x5600, src = 10350800, srcv = 0x42c0, target = 0, size = 61, 
  subchunks = 0x7f80df0, nsubchunks = 1}

  if (relsize)
    *relsize = rel->relocators_size;

  grub_dprintf ("relocator", "Relocs allocated at %p\n", movers_chunk.srcv);

  {
    unsigned i;
    grub_size_t count[257];
    struct grub_relocator_chunk *from, *to, *tmp;

    grub_memset (count, 0, sizeof (count));

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
    from = grub_malloc (nchunks * sizeof (sorted[0]));
(gdb) p nchunks 
$11 = 4
(gdb) p count
$12 = {0, 3, 0 <repeats 95 times>, 1, 0 <repeats 159 times>}
    to = grub_malloc (nchunks * sizeof (sorted[0]));
    if (!from || !to)
      {
        grub_free (from);
        grub_free (to);
        return grub_errno;
      }

    for (j = 0; j < 256; j++)
      count[j+1] += count[j];

    {
      struct grub_relocator_chunk *chunk;
      for (chunk = rel->chunks; chunk; chunk = chunk->next)
(gdb) p count
$14 = {0, 3 <repeats 96 times>, 4 <repeats 160 times>}
(gdb) p from[0]
$3 = {next = 0x40, src = 2817, srcv = 0x44, target = 3585, size = 72, 
  subchunks = 0x1201, nsubchunks = 76}
(gdb) p from[1]
$4 = {next = 0x1401, src = 80, srcv = 0x3801, target = 84, size = 4097, 
  subchunks = 0x58, nsubchunks = 6145}
(gdb) p from[2]
$5 = {next = 0x5c, src = 15105, srcv = 0x60, target = 10753, size = 100, 
  subchunks = 0x2901, nsubchunks = 104}
(gdb) p from[3]
$6 = {next = 0x2f01, src = 108, srcv = 0x101, target = 112, size = 257, 
  subchunks = 0x74, nsubchunks = 257}
        from[count[chunk->src & 0xff]++] = *chunk;
    }

(gdb) p from[0]
$53 = {next = 0x40, src = 2817, srcv = 0x44, target = 3585, size = 72, 
  subchunks = 0x1201, nsubchunks = 76}
(gdb) p from[1]
$54 = {next = 0x1401, src = 80, srcv = 0x3801, target = 84, size = 4097, 
  subchunks = 0x58, nsubchunks = 6145}
(gdb) p from[2]
$55 = {next = 0x7fe1970, src = 10350592, srcv = 0x9df000, target = 10350592, 
  size = 208, subchunks = 0x7f930c0, nsubchunks = 1}
(gdb) p from[3]
$56 = {next = 0x2f01, src = 108, srcv = 0x101, target = 112, size = 257, 
  subchunks = 0x74, nsubchunks = 257}
(gdb) p count
$57 = {6, 3 <repeats 95 times>, 5, 4 <repeats 160 times>}

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
(gdb) p /x addr
$64 = 0x9df000
  *relstart = rels0;
  grub_free (sorted);
(gdb) p *relstart 
$67 = (void *) 0x9df0d0
  return GRUB_ERR_NONE;
}

```
