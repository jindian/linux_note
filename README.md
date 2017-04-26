# linux\_note

Debug grub and linux source code with GDB

Knowledge of assembly, shell, C are needed

# work space

git@github.com:jindian/linux_startup.git

Both GRUB2 and linux source code included in above repository, file create_grub2_boot_disk_img.sh is used to create bootable disk image.

# guide of debug with gdb

About how to compile grub2 and linux source code, create bootable disk image etc, read the instructions in README of linux_startup repository

#### debug GRUB2
1. start qemu simulator: qemu-system-i386 -s -S disk.img
    disk.img is created with shell script create_grub2_boot_disk_img.sh
2. in directory grub-2.00/grub-core, start gdb: gdb -x gdb_grub
    gdb_grub is a gdb source file, in which we define useful routines used in gdb and configure command executed after gdb startup
3. set break point, for example at the entry of master boot record: break *0x7c00
4. press 'c' to continue
5. ...

#### debug linux
1. the first step is same with debug GRUB2
2. in directory linux start gdb: gdb -x gdb_linux
3. in gdb_linux the first break point at start_kernel already set, set other break points as you want
4. press 'c' to continue
5. ...

# Links

  * [x86 instruction](https://en.wikipedia.org/wiki/X86_instruction_listings)
  * [x86 Assembly](https://en.wikibooks.org/wiki/Category:X86_Assembly)
  * [x86 Registers](http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html)
  * [Extended Asm](http://www.ibiblio.org/gferg/ldp/GCC-Inline-Assembly-HOWTO.html#toc5)
  * [The GNU Assembler](http://tigcc.ticalc.org/doc/gnuasm.html)
  * [Debugging Assembly Code with gdb](http://web.cecs.pdx.edu/~apt/cs577_2008/gdb.pdf)
  * [The BIOS/MBR Boot Process](https://neosmart.net/wiki/mbr-boot-process/)
  * [GRUB bootloader - Full tutorial](http://www.dedoimedo.com/computers/grub.html)
  * [OS Creation](http://wiki.osdev.org/Main_Page)
  * [BIOS interrupt call](https://en.wikipedia.org/wiki/BIOS_interrupt_call)
  * [Intel 80386 Reference](https://pdos.csail.mit.edu/6.828/2008/readings/i386/toc.htm)
  * [GNU GRUB 2 Debugging with GDB](http://v3.sk/~lkundrak/grub2-gdb/howto.html)
  * [THE LINUX/x86 BOOT PROTOCOL](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [Debugging The Linux Kernel Using Gdb](http://www.elinux.org/Debugging_The_Linux_Kernel_Using_Gdb)
  * [X86 Opcode and Instruction Reference](http://ref.x86asm.net/coder32.html)
  * [LINUX SYSTEM CALL TABLE FOR X86 64](http://blog.rchapman.org/posts/Linux_System_Call_Table_for_x86_64/)
  * [i386 system calls](http://asm.sourceforge.net/syscall.html#p2)
  * [Demystifying the Kernel Bootstrap Process](https://www.ibm.com/support/knowledgecenter/linuxonibm/liaav/Demystifying-the-Kernel-Bootstrap-Process.pdf)
  * [Linux Kernel Debugging And Profiling Tools](https://events.linuxfoundation.org/sites/events/files/slides/kernel_profiling_debugging_tools_0.pdf)
  
  





