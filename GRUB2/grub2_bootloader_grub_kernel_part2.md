Decompress grub kernel
================================

Set destination address and size of grub kernel decompressed area, call _LzmaDecodeA to do the decompression.
```assembly
   0x89ce:	mov    $0x100000,%edi
   0x89d3:	mov    $0x8d30,%esi
   0x89d8:	push   %edi
   0x89d9:	mov    0x820c,%ecx
   0x89df:	lea    (%edi,%ecx,1),%ebx
   0x89e2:	push   %ecx
   0x89e3:	call   0x8ac7
   0x89e8:	pop    %ecx
   0x89e9:	pop    %esi
   0x89ea:	mov    0x8218,%edx

-----------------------------------------------------------------------

grub-core/boot/i386/pc/startup_raw.S:340

post_reed_solomon:

#ifdef ENABLE_LZMA
        movl    $GRUB_MEMORY_MACHINE_DECOMPRESSION_ADDR, %edi
#ifdef __APPLE__
        movl    $decompressor_end, %esi
#else
        movl    $LOCAL(decompressor_end), %esi
#endif
        pushl   %edi
        movl    LOCAL (uncompressed_size), %ecx
        leal    (%edi, %ecx), %ebx
        /* Don't remove this push: it's an argument.  */
        push    %ecx
        call    _LzmaDecodeA
```

Initialize uncompressed area and parameters used in decompress process. Call RangeDecoderBitDecode located at address 0x8a01.
```assembly
   0x8ac7:	push   %ebp
   0x8ac8:	mov    %esp,%ebp
   0x8aca:	sub    $0x24,%esp
   0x8acd:	push   %edi
   0x8ace:	cld    
   0x8acf:	mov    %ebx,%edi
   0x8ad1:	mov    $0x1f36,%ecx
   0x8ad6:	mov    $0x400,%eax
(gdb) info registers 
eax            0x400	1024
ecx            0x1f36	7990
edx            0xffffff90	-112
ebx            0x10b7d0	1095632
esp            0x7ffbc	0x7ffbc
ebp            0x7ffe4	0x7ffe4
esi            0x8d30	36144
edi            0x10b7d0	1095632
eip            0x8adb	0x8adb
eflags         0x6	[ PF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x10	16
gs             0x10	16
   0x8adb:	rep stos %eax,%es:(%edi)
(gdb) info registers 
eax            0x400	1024
ecx            0x0	0
edx            0xffffff90	-112
ebx            0x10b7d0	1095632
esp            0x7ffbc	0x7ffbc
ebp            0x7ffe4	0x7ffe4
esi            0x8d30	36144
edi            0x1134a8	1127592
eip            0x8add	0x8add
eflags         0x6	[ PF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x10	16
gs             0x10	16
(gdb) x/w 0x8d30
0x8d30:	0x83a34400
(gdb) x/w 0x4400+0x10b7d0
0x10fbd0:	0x00000400
(gdb) x/w 0x4400+0x10b7d0+4
0x10fbd4:	0x00000400
   0x8add:	pop    %edi
   0x8ade:	xor    %eax,%eax
   0x8ae0:	mov    %eax,-0x4(%ebp)
   0x8ae3:	mov    %eax,-0x8(%ebp)
   0x8ae6:	mov    %eax,-0x14(%ebp)
   0x8ae9:	inc    %eax
(gdb) info registers eax
eax            0x1	1
   0x8aea:	mov    %eax,-0x18(%ebp)
   0x8aed:	mov    %eax,-0x1c(%ebp)
   0x8af0:	mov    %eax,-0x20(%ebp)
   0x8af3:	mov    %eax,-0x24(%ebp)
   0x8af6:	neg    %eax
(gdb) info registers eax
eax            0xffffffff	-1
   0x8af8:	mov    %eax,-0xc(%ebp)
   0x8afb:	inc    %eax
   0x8afc:	mov    $0x5,%cl
   0x8afe:	shl    $0x8,%eax
   0x8b01:	lods   %ds:(%esi),%al
   0x8b02:	loop   0x8afe
   0x8b04:	mov    %eax,-0x10(%ebp)
(gdb) info registers eax
eax            0x44a383df	1151566815
   0x8b07:	mov    -0x4(%ebp),%eax
(gdb) info registers ebp
ebp            0x7ffe4	0x7ffe4
(gdb) x/w 0x7ffe4-0x4
0x7ffe0:	0x00000000
(gdb) x/w 0x7ffe4+0x8
0x7ffec:	0x0000b7d0
   0x8b0a:	cmp    0x8(%ebp),%eax
   0x8b0d:	jb     0x8b13
   0x8b0f:	mov    %ebp,%esp
   0x8b11:	pop    %ebp
   0x8b12:	ret    
   0x8b13:	and    $0x3,%eax
   0x8b16:	push   %eax
   0x8b17:	mov    -0x14(%ebp),%edx
   0x8b1a:	shl    $0x4,%edx
   0x8b1d:	add    %edx,%eax
   0x8b1f:	push   %eax
   0x8b20:	call   0x8a01
   0x8b25:	jb     0x8bc5
   0x8b2b:	mov    -0x4(%ebp),%eax
   0x8b2e:	and    $0x0,%eax

-----------------------------------------------------------------------

grub-core/boot/i386/pc/lzma_decode.S

/*
 * int LzmaDecode(CLzmaDecoderState *vs,
 *                const unsigned char *inStream,
 *                unsigned char *outStream,
 *                SizeT outSize);
 */

_LzmaDecodeA:

        pushl   %ebp
        movl    %esp, %ebp
        subl    $LOCAL_SIZE, %esp

#ifndef ASM_FILE
        pushl   %esi
        pushl   %edi
        pushl   %ebx

        movl    %eax, %ebx
        movl    %edx, %esi
        pushl   %ecx
#else
        pushl   %edi
#endif

        cld

#ifdef FIXED_PROPS
        movl    %ebx, %edi
        movl    $(Literal + (LZMA_LIT_SIZE << (FIXED_LC + FIXED_LP))), %ecx
#else
        movl    $LZMA_LIT_SIZE, %eax
        movb    lc, %cl
        addb    lp, %cl
        shll    %cl, %eax
        addl    $Literal, %eax
        movl    %eax, %ecx
        movl    probs, %edi
#endif

        movl    $(kBitModelTotal >> 1), %eax

        rep
        stosl

        popl    %edi

        xorl    %eax, %eax
        movl    %eax, now_pos
        movl    %eax, prev_byte
        movl    %eax, state

        incl    %eax
        movl    %eax, rep0
        movl    %eax, rep1
        movl    %eax, rep2
        movl    %eax, rep3

#ifndef FIXED_PROPS
        movl    %eax, %edx
        movb    pb, %cl
        shll    %cl, %edx
        decl    %edx
        movl    %edx, pos_state_mask

        movl    %eax, %edx
        movb    lp, %cl
        shll    %cl, %edx
        decl    %edx
        movl    %edx, lit_pos_mask;
#endif

        /* RangeDecoderInit */
        negl    %eax
        movl    %eax, range

        incl    %eax
        movb    $5, %cl

1:
        shll    $8, %eax
        lodsb
        loop    1b

        movl    %eax, code
lzma_decode_loop:
        movl    now_pos, %eax
        cmpl    out_size, %eax

        jb      1f

#ifndef ASM_FILE
        xorl    %eax, %eax

        popl    %ebx
        popl    %edi
        popl    %esi
#endif

        movl    %ebp, %esp
        popl    %ebp
        ret

1:
#ifdef FIXED_PROPS
        andl    $POS_STATE_MASK, %eax
#else
        andl    pos_state_mask, %eax
#endif
        pushl   %eax                            /* posState */
        movl    state, %edx
        shll    $kNumPosBitsMax, %edx
        addl    %edx, %eax
        pushl   %eax                            /* (state << kNumPosBitsMax) + posState */

        call    RangeDecoderBitDecode
        jc      1f
```

Let's step into RangeDecoderBitDecode routine to continue grub kernel decompress process.
```assembly
(gdb) info registers 
eax            0x0	0
ecx            0x0	0
edx            0x0	0
ebx            0x10b7d0	1095632
esp            0x7ffb4	0x7ffb4
ebp            0x7ffe4	0x7ffe4
esi            0x8d35	36149
edi            0x100000	1048576
eip            0x8a01	0x8a01
eflags         0x46	[ PF ZF ]
cs             0x8	8
ss             0x10	16
ds             0x10	16
es             0x10	16
fs             0x10	16
gs             0x10	16
   0x8a01:	lea    (%ebx,%eax,4),%eax
   0x8a04:	mov    %eax,%ecx
   0x8a06:	mov    (%ecx),%eax
   0x8a08:	mov    -0xc(%ebp),%edx
   0x8a0b:	shr    $0xb,%edx
(gdb) info registers edx
edx            0x1fffff	2097151
(gdb) info registers eax
eax            0x400	1024
   0x8a0e:	mul    %edx
(gdb) info registers eax
eax            0x7ffffc00	2147482624
   0x8a10:	cmp    -0x10(%ebp),%eax
(gdb) info registers eflags
eflags         0x16	[ PF AF ]
   0x8a13:	jbe    0x8a3d
   0x8a15:	mov    %eax,-0xc(%ebp)
   0x8a18:	mov    $0x800,%edx
   0x8a1d:	sub    (%ecx),%edx
(gdb) info registers edx
edx            0x400	1024
   0x8a1f:	shr    $0x5,%edx
(gdb) info registers edx
edx            0x20	32
   0x8a22:	add    %edx,(%ecx)
(gdb) info registers ecx
ecx            0x10b7d0	1095632
(gdb) x/w 0x10b7d0
0x10b7d0:	0x00000420
   0x8a24:	clc    
   0x8a25:	pushf  
   0x8a26:	cmpl   $0x1000000,-0xc(%ebp)
(gdb) info registers ebp
ebp            0x7ffe4	0x7ffe4
(gdb) x/w 0x7ffe4-0xc
0x7ffd8:	0x7ffffc00
   0x8a2d:	jae    0x8a3b
   0x8a2f:	shll   $0x8,-0x10(%ebp)
   0x8a33:	lods   %ds:(%esi),%al
   0x8a34:	mov    %al,-0x10(%ebp)
   0x8a37:	shll   $0x8,-0xc(%ebp)
   0x8a3b:	popf   
   0x8a3c:	ret    
```