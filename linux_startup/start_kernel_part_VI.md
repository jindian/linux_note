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
  
  `generic_apic_probe` check if command line APIC exist, if not, loop the pre-configureation in apic_probe, find the first fit APIC configuration and set it to global APIC variable.

```final_apic_configure

(gdb) printf "APIC configure name = %s\n", apic->name
APIC configure name = default
```

  The configruation of default APIC

```apic_default

struct apic apic_default = {

        .name                           = "default",
        .probe                          = probe_default,
        .acpi_madt_oem_check            = NULL,
        .apic_id_registered             = default_apic_id_registered,

        .irq_delivery_mode              = dest_LowestPrio,
        /* logical delivery broadcast to all CPUs: */
        .irq_dest_mode                  = 1,

        .target_cpus                    = default_target_cpus,
        .disable_esr                    = 0,
        .dest_logical                   = APIC_DEST_LOGICAL,
        .check_apicid_used              = default_check_apicid_used,
        .check_apicid_present           = default_check_apicid_present,

        .vector_allocation_domain       = default_vector_allocation_domain,
        .init_apic_ldr                  = default_init_apic_ldr,

        .ioapic_phys_id_map             = default_ioapic_phys_id_map,
        .setup_apic_routing             = setup_apic_flat_routing,
        .multi_timer_check              = NULL,
        .apicid_to_node                 = default_apicid_to_node,
        .cpu_to_logical_apicid          = default_cpu_to_logical_apicid,
        .cpu_present_to_apicid          = default_cpu_present_to_apicid,
        .apicid_to_cpu_present          = default_apicid_to_cpu_present,
        .setup_portio_remap             = NULL,
        .check_phys_apicid_present      = default_check_phys_apicid_present,
        .enable_apic_mode               = NULL,
        .phys_pkg_id                    = default_phys_pkg_id,
        .mps_oem_check                  = NULL,

        .get_apic_id                    = default_get_apic_id,
        .set_apic_id                    = NULL,
        .apic_id_mask                   = 0x0F << 24,

        .cpu_mask_to_apicid             = default_cpu_mask_to_apicid,
        .cpu_mask_to_apicid_and         = default_cpu_mask_to_apicid_and,

        .send_IPI_mask                  = default_send_IPI_mask_logical,
        .send_IPI_mask_allbutself       = default_send_IPI_mask_allbutself_logical,
        .send_IPI_allbutself            = default_send_IPI_allbutself,
        .send_IPI_all                   = default_send_IPI_all,
        .send_IPI_self                  = default_send_IPI_self,

        .trampoline_phys_low            = DEFAULT_TRAMPOLINE_PHYS_LOW,
        .trampoline_phys_high           = DEFAULT_TRAMPOLINE_PHYS_HIGH,

        .wait_for_init_deassert         = default_wait_for_init_deassert,

        .smp_callin_clear_local_apic    = NULL,
        .inquire_remote_apic            = default_inquire_remote_apic,

        .read                           = native_apic_mem_read,
        .write                          = native_apic_mem_write,
        .icr_read                       = native_apic_icr_read,
        .icr_write                      = native_apic_icr_write,
        .wait_icr_idle                  = native_apic_wait_icr_idle,
        .safe_wait_icr_idle             = native_safe_apic_wait_icr_idle,
};
```

## set quirks for PIC devices

  Sometimes there are bugs in the hardware itself. Quirks are software ways to work around these bugs.
  
```early_quirks

void __init early_quirks(void)
{
	int slot, func;

	if (!early_pci_allowed())
		return;

	/* Poor man's PCI discovery */
	/* Only scan the root bus */
	for (slot = 0; slot < 32; slot++)
		for (func = 0; func < 8; func++) {
			/* Only probe function 0 on single fn devices */
			if (check_dev_quirk(0, slot, func))
				break;
		}
}

```
  
  In `early_quirks` checks if early PCI is allowed with `early_pci_allowed`.

```early_pci_allowed

int early_pci_allowed(void)
{
	return (pci_probe & (PCI_PROBE_CONF1|PCI_PROBE_NOEARLY)) ==
			PCI_PROBE_CONF1;
}
```

  Where `pci_probe` defined as follow:

```pci_probe

unsigned int pci_probe = PCI_PROBE_BIOS | PCI_PROBE_CONF1 | PCI_PROBE_CONF2 |
				PCI_PROBE_MMCONF;
```

  Apply early quirk to a given PCI device with `check_dev_quirk`, it read PCI configuration, if the result matches with information in pre-configured `early_qrk`, apply it. `early_qrk` defination as follow:
  
```early_qrk

/*
 * Only works for devices on the root bus. If you add any devices
 * not on bus 0 readd another loop level in early_quirks(). But
 * be careful because at least the Nvidia quirk here relies on
 * only matching on bus 0.
 */
static struct chipset early_qrk[] __initdata = {
	{ PCI_VENDOR_ID_NVIDIA, PCI_ANY_ID,
	  PCI_CLASS_BRIDGE_PCI, PCI_ANY_ID, QFLAG_APPLY_ONCE, nvidia_bugs },
	{ PCI_VENDOR_ID_VIA, PCI_ANY_ID,
	  PCI_CLASS_BRIDGE_PCI, PCI_ANY_ID, QFLAG_APPLY_ONCE, via_bugs },
	{ PCI_VENDOR_ID_AMD, PCI_DEVICE_ID_AMD_K8_NB,
	  PCI_CLASS_BRIDGE_HOST, PCI_ANY_ID, 0, fix_hypertransport_config },
	{ PCI_VENDOR_ID_ATI, PCI_DEVICE_ID_ATI_IXP400_SMBUS,
	  PCI_CLASS_SERIAL_SMBUS, PCI_ANY_ID, 0, ati_bugs },
	{ PCI_VENDOR_ID_ATI, PCI_DEVICE_ID_ATI_SBX00_SMBUS,
	  PCI_CLASS_SERIAL_SMBUS, PCI_ANY_ID, 0, ati_bugs_contd },
	{}
};
```

```check_dev_quirk

static int __init check_dev_quirk(int num, int slot, int func)
{
	u16 class;
	u16 vendor;
	u16 device;
	u8 type;
	int i;

	class = read_pci_config_16(num, slot, func, PCI_CLASS_DEVICE);

	if (class == 0xffff)
		return -1; /* no class, treat as single function */

	vendor = read_pci_config_16(num, slot, func, PCI_VENDOR_ID);

	device = read_pci_config_16(num, slot, func, PCI_DEVICE_ID);

	for (i = 0; early_qrk[i].f != NULL; i++) {
		if (((early_qrk[i].vendor == PCI_ANY_ID) ||
			(early_qrk[i].vendor == vendor)) &&
			((early_qrk[i].device == PCI_ANY_ID) ||
			(early_qrk[i].device == device)) &&
			(!((early_qrk[i].class ^ class) &
			    early_qrk[i].class_mask))) {
				if ((early_qrk[i].flags &
				     QFLAG_DONE) != QFLAG_DONE)
					early_qrk[i].f(num, slot, func);
				early_qrk[i].flags |= QFLAG_APPLIED;
			}
	}

	type = read_pci_config_byte(num, slot, func,
				    PCI_HEADER_TYPE);
	if (!(type & 0x80))
		return -1;

	return 0;
}
```

  `read_pci_config_16` and `read_pci_config_byte` used to read [PCI configuration space](https://en.wikipedia.org/wiki/PCI_configuration_space), where `0xcf8` is  Configuration Space Address I/O port and `0xcfc` is Configuration Space Data I/O port, about `outl` and `inw` please check [x86 instruction listings](https://en.wikipedia.org/wiki/X86_instruction_listings)

```read_pci_config_16

u16 read_pci_config_16(u8 bus, u8 slot, u8 func, u8 offset)
{
	u16 v;
	outl(0x80000000 | (bus<<16) | (slot<<11) | (func<<8) | offset, 0xcf8);
	v = inw(0xcfc + (offset&2));
	pr_debug("%x reading 2 from %x: %x\n", slot, offset, v);
	return v;
}
```

## 

# Links

  * [Trampoline](https://en.wikipedia.org/wiki/Trampoline_(computing)
  * [Intel MultiProcessor Specification](http://download.intel.com/design/archives/processors/pro/docs/24201606.pdf)
  * [What are PCI quirks?](http://unix.stackexchange.com/questions/83390/what-are-pci-quirks) 
  * [Quirks](https://wiki.ubuntu.com/X/Quirks)
  * [PCI configuration space](https://en.wikipedia.org/wiki/PCI_configuration_space)
  * [x86 instruction listings](https://en.wikipedia.org/wiki/X86_instruction_listings)
  * [Advanced Configuration and Power Interface](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
  * [Desktop Management Interface](https://en.wikipedia.org/wiki/Desktop_Management_Interface)
  * [System Management BIOS](https://en.wikipedia.org/wiki/System_Management_BIOS)
  
