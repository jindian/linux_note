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


# Links

  * [Trampoline](https://en.wikipedia.org/wiki/Trampoline_(computing)
