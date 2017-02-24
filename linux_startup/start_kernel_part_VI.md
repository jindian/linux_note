# start_kernel part VI

## setup trampoline page table

  In computer programming, the word trampoline has a number of meanings, and is generally associated with jumps (i.e., moving to different code paths).

Low-level programming

  Trampolines (sometimes referred to as indirect jump vectors) are memory locations holding addresses pointing to interrupt service routines, I/O routines, etc. Execution jumps into the trampoline and then immediately jumps out, or bounces, hence the term trampoline. They have many uses:

CPUs

  Trampoline can be used to overcome the limitations imposed by a CPU architecture that expects to always find vectors in fixed locations.
  When an operating system is booted on a symmetric multiprocessing (SMP) machine, only one processor, the boot-strap processor, will be active. After the operating system has configured itself, it will instruct the other processors to jump to a piece of trampoline code that will initialize the processors and wait for the operating system to start scheduling threads on them.

High-level programming

  As used in some Lisp implementations, a trampoline is a loop that iteratively invokes thunk-returning functions (continuation-passing style). A single trampoline suffices to express all control transfers of a program; a program so expressed is trampolined, or in trampolined style; converting a program to trampolined style is trampolining. Programmers can use trampolined functions to implement tail-recursive function calls in stack-oriented programming languages.
  In Java, trampoline refers to using reflection to avoid using inner classes, for example in event listeners. The time overhead of a reflection call is traded for the space overhead of an inner class. Trampolines in Java usually involve the creation of a GenericListener to pass events to an outer class.

  `setup_trampoline_page_table` copy kernel address and initialize low mappings, inside `setup_trampoline_page_table` it involve `clone_pgd_range` to copy and initialize operation.

```clone_pgd_range

clone_pgd_range (dst=0xc18db018, count=1, src=<optimized out>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/pgtable.h:621
    
clone_pgd_range (dst=0xc18db000, count=1, src=<optimized out>)
    at /home/start-kernel/work_space/github/linux_startup/linux-2.6.32.69/arch/x86/include/asm/pgtable.h:621
```

## setup APIC response hook

  In an MP-compliant system, interrupts are controlled through the APIC.
  
  The Intel Advanced Programmable Interrupt Controller (APIC) is based on a distributed architecture. Interrupt control functions are distributed between two basic functional units: the local unit and the I/O unit. The local and I/O units communicate through a bus called the ICC bus.
  
  The I/O unit senses an interrupt input, addresses it to a local unit, and sends it over the ICC bus. The local unit that is addressed accepts the message sent by the I/O unit.
  
  In an MP-compliant system, one local APIC per CPU is required. Depending on the total number of interrupt lines in an MP system, one or more I/O APICs may be used. The bus interrupt line assignments can be implementation-specific and can be defined by the MP configuration table.

  The Intel 82489DX APIC is a “discrete APIC” implementation. The programming interface of the 82489DX APIC units serves as the base of the MP specification. Each APIC has a version register that contains the version number of a specific APIC implementation. The version register of the 82489DX family has a version number of “0x,” where x is a four-bit hexadecimal number. Version number “1x” refers to Pentium processors with integrated APICs, such as the Pentium 735\90 and 815\100 processors, and x is a four-bit hexadecimal number. The integrated APIC maintains the same programming interface as the 82489DX APIC.
  
  To encourage future extendibility and innovation, the Intel APIC architecture definition is limited to the programming interface of the APIC units. The ICC bus protocol and electrical specifications are considered implementation-specific. That is, while different versions of APIC implementations may execute the same binary software, different versions of APIC components may be implemented with different bus protocols or electrical specifications. Care must be taken when using different versions of the APIC in a system. 
  
  The APIC architecture is designed to be scaleable. The 82489DX APIC has an 8-bit ID register that can address from one to 255 APIC devices. Furthermore, the Logical Destination register for the 82489DX APIC supports 32 bits, which can address up to 32 devices. For small system implementations, the APIC ID register can be reduced to the least significant 4 bits and the Logical   Destination register can be reduced to the most significant 8 bits. To ensure software compatibility with all versions of APIC implementations, software developers must follow the following programming guidelines:
  
  1. Assign an 8-bit APIC ID starting from zero.
  2. Assign logical destinations starting from the most significant byte of the 32-bit register.
  3. Program the APIC spurious vector to hexadecimal “xF,” where x is a 4-bit hexadecimal number.
  
  The following features are only available in the integrated APIC:
  
  1. The I/O APIC interrupt input signal polarity can be programmable.
  2. A new interprocessor interrupt, STARTUP IPI is defined. In general, the operating system must use the STARTUP IPI to wake up application processors in systems with integrated APICs, but must use INIT IPI in systems with the 82489DX APIC.

# Links

  * [Trampoline](https://en.wikipedia.org/wiki/Trampoline_(computing)
  * [Intel MultiProcessor Specification](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf) 
