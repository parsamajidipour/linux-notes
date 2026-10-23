# Firmware and Hardware Initialization

Before Linux can discover a machine, something must make the machine discoverable.

That sentence captures the firmware problem. At reset, a processor can execute from a defined location, but normal RAM may not yet be usable, clocks may not be configured for their final rates, secondary CPUs may be held, buses may not be enumerated, and devices may contain their own uninitialized processors. Platform firmware creates the first coherent execution environment. It does not turn the hardware into a complete operating system; it establishes enough state and description for a boot application and later kernel to take over.

This chapter follows that work from reset through platform discovery. The details vary dramatically across x86 servers, laptops, arm64 systems, embedded boards, and virtual machines, so the emphasis is on contracts rather than vendor-specific sequences. We will also keep two different activities separate:

- **initializing hardware** means placing it in a state where subsequent software can use or reinitialize it;
- **describing hardware** means communicating what exists, where its resources live, and how components relate.

Firmware often does both. Linux does not blindly inherit every firmware decision, but it cannot begin with zero information either.

---

## 1. The reset environment

A processor reset is not equivalent to calling a normal function. Architecture defines an entry state: an instruction address, privilege level or mode, selected control-register values, interrupt state, and a set of guarantees about which execution resources exist. Platform wiring maps firmware code where that entry can fetch it.

On x86, historical compatibility shapes the reset model. The bootstrap processor begins from a reset vector associated with the top of the physical address space, in a mode compatible with early x86 execution. Modern firmware quickly escapes the severe historical environment, but it must begin from the architectural contract. On arm64, firmware and boot standards define exception levels and handoff conventions differently. A small microcontroller may begin directly from a vector table in on-chip flash.

The first instructions have a bootstrapping problem: ordinary DRAM is not yet trustworthy, but C code and nested calls want a stack and writable storage. Implementations may use processor cache as temporary RAM, on-chip SRAM, special chipset memory, or extremely small assembly routines until memory initialization succeeds. This is why "the firmware runs from ROM" is incomplete. Instructions may be fetched from flash or mapped firmware storage, while temporary data follows a carefully staged path before DRAM is ready.

### Bootstrap processor and secondary processors

Multiprocessor systems normally designate one logical processor as the bootstrap processor. It executes the initial firmware path while other processors remain stopped, held in reset, or wait in a firmware-controlled state. Firmware may inspect and prepare all CPUs, apply microcode, and build topology tables, but Linux later performs its own symmetric multiprocessing startup.

The distinction matters during failure analysis. A machine can display firmware output with one working CPU even if a secondary CPU later fails to start under Linux. Conversely, an early platform problem affecting the bootstrap processor prevents all higher-level diagnostics.

---

## 2. Power, reset, and platform controllers

The main CPU is not the only intelligence on a modern machine. Power-management controllers, embedded controllers, a platform security processor, management engines, baseboard management controllers, storage-controller firmware, and device-local processors can execute before or alongside the host CPU.

A simplified power-on sequence looks like this:

1. Power rails reach acceptable ranges.
2. Clock and reset logic stabilizes essential domains.
3. A controller releases the bootstrap processor according to platform policy.
4. The processor fetches from its reset entry.
5. Early firmware configures enough silicon and interconnect state to reach later firmware volumes.
6. Memory is trained and made available.
7. The firmware expands into a richer environment and discovers the platform.

This is intentionally generic. Real systems may authenticate firmware regions before executing them, initialize a management controller independently, recover from a redundant firmware bank, or wait for power sequencing across multiple sockets.

When a machine is completely silent, the failure can precede host CPU execution. Server boards expose POST codes, BMC event logs, serial-over-LAN, or vendor diagnostics precisely because the normal display and operating system are not available at this stage.

---

## 3. Memory training: why RAM is not immediately RAM

DRAM is not a passive byte array that becomes usable the moment power appears. A memory controller must be configured for installed modules, signaling parameters, timing, topology, and frequencies. Training procedures find working timing windows and calibrate communication. Platforms may read module metadata, test channels, account for ranks and interleaving, and retry at conservative settings when aggressive values fail.

This work explains several real observations:

- The first cold boot after changing memory can take much longer than later boots.
- A platform can loop or restart several times while training.
- Memory overclocking can fail before any boot loader or kernel output exists.
- A warm reboot may succeed because training data is reused, while a cold boot fails.
- Firmware updates can change memory compatibility even when Linux is unchanged.

Firmware also decides which physical address ranges are usable RAM and which are reserved for devices, firmware runtime data, chipset windows, security processors, or persistent regions. It communicates a memory map to the operating system. Linux validates and transforms that map, but the initial description is a critical input.

### Error correction and degraded memory

Servers with ECC can detect and sometimes correct memory errors. Firmware may perform scrubbing, disable a failing rank, or report corrected and uncorrected conditions through platform records. Linux later owns normal memory-error handling, but early firmware may already have changed the available topology.

If installed capacity and Linux-visible capacity differ, do not jump directly to a kernel bug. Compare firmware setup, hardware event logs, the firmware memory map, reserved ranges, and kernel messages. Some loss is ordinary address reservation; large or unstable differences deserve investigation.

---

## 4. A useful UEFI PI mental model

Many PC UEFI implementations are organized around concepts from the UEFI Platform Initialization specifications. Exact internal architecture is vendor-specific, but the common phase model is useful:

- **SEC:** establish a trusted and minimal execution environment after reset.
- **PEI:** initialize permanent memory and produce small records describing discovered state.
- **DXE:** run a richer driver environment, initialize buses and services, and build UEFI protocols.
- **BDS:** apply boot-device selection policy and enter the UEFI Boot Manager flow.
- **Runtime:** preserve a limited set of firmware services after the operating system takes control.

These names are not Linux stages. They describe firmware construction. The key transition is from extremely constrained early execution to a driver-oriented environment after DRAM exists. By the time an ordinary EFI application such as GRUB runs, the firmware has built a much richer world: memory allocation services, handles, protocols, filesystem access for supported volumes, console services, variables, device paths, and image loading.

Do not assume every UEFI system exposes or internally implements all details identically. Treat the PI model as a map of common responsibilities, then use platform documentation or firmware source such as EDK II for implementation-level study.

---

## 5. Processor initialization and microcode

Modern CPUs implement complex behavior in microcode. Vendors release microcode updates to correct errata or modify security-relevant behavior. A platform may apply a bundled update during firmware initialization. Linux can apply a newer update very early during kernel boot, commonly with microcode data prepended to or carried alongside initramfs.

The two delivery paths complement each other:

- Firmware microcode protects the pre-kernel environment and does not depend on the operating system.
- Operating-system delivery can update more quickly than a platform vendor releases firmware.

Linux reports microcode information in the kernel log and exposes processor details through procfs and sysfs. On x86, an observational check may include:

~~~sh
grep -i microcode /proc/cpuinfo | sort -u
journalctl -b -k | grep -i microcode
~~~

The displayed revision alone does not tell you whether firmware or early Linux supplied it. Read the ordered boot messages and distribution packaging. A microcode update is not an ordinary late driver; applying it early reduces the interval in which CPUs execute with known errata.

Firmware also establishes basic processor configuration, discovers sockets and logical processors, and communicates topology. Linux later constructs its own per-CPU structures, chooses mitigation policy, and starts secondary CPUs. Firmware discovery and kernel ownership are separate phases.

---

## 6. Buses and devices: discovery is layered

A platform contains buses nested under other buses. PCI Express is a central example: host bridges expose buses; bridges create subordinate buses; functions carry vendor and device identifiers; base address registers request memory or I/O windows; interrupt capabilities describe signaling options.

Firmware commonly enumerates enough of this topology to allocate non-conflicting resources and make boot devices usable. It may execute device option ROMs or UEFI drivers. Linux later enumerates the bus again through its own PCI subsystem, consumes firmware resource descriptions, and binds native drivers.

Consider an NVMe drive:

1. Firmware configures the PCIe fabric and observes an NVMe controller function.
2. Firmware may use its own NVMe driver to read an EFI System Partition.
3. Firmware loads a boot application from that filesystem.
4. The loader uses firmware protocols or its own code to read the kernel.
5. After handoff, Linux enumerates the PCI function.
6. The Linux NVMe driver binds, configures queues, and discovers namespaces.
7. The block layer exposes a namespace device.
8. Partition parsing exposes partitions.
9. Higher storage layers may create encryption, RAID, or LVM mappings.

The same physical drive appears through different software ownership at different times. The fact that firmware read it does not guarantee Linux has a working driver. The fact that Linux sees the PCI function does not prove the namespace or filesystem is usable.

### Read-only observation

After boot, compare layers:

~~~sh
lspci -nn
lspci -k
lsblk -o NAME,TYPE,TRAN,MODEL,FSTYPE,MOUNTPOINTS
journalctl -b -k | grep -Ei 'pci|nvme|ahci|scsi'
~~~

The first lists functions and identifiers. The second adds kernel driver binding. The third shows block-level results. Kernel messages show ordering and failures.

---

## 7. Option ROMs and firmware drivers

Devices sometimes provide code intended to run in the pre-OS environment. Legacy PCI option ROMs can extend BIOS behavior, historically for video initialization, storage controllers, and network boot. UEFI option ROMs can contain EFI drivers that produce protocols used by other firmware components and applications.

This design makes otherwise unknown hardware bootable without permanently embedding every driver in motherboard firmware. It also expands the pre-boot trust surface: firmware may execute code sourced from an add-in device before the operating system's security controls exist.

Secure Boot and platform policy can authenticate EFI drivers, while modern systems may disable legacy option ROM execution when configured for native UEFI. Network-interface PXE support and storage-controller boot utilities are visible examples.

Do not confuse firmware's driver with the Linux driver. They may be written by different projects, support different features, and have different bugs. The firmware driver exists to support pre-OS tasks. Linux normally resets or reconfigures the device and uses its own driver after handoff.

---

## 8. ACPI: description plus control methods

On many x86 and arm64 servers, the Advanced Configuration and Power Interface is the main standardized channel through which firmware describes platform topology and power-management relationships to the operating system.

Firmware places an ACPI Root System Description Pointer where the boot contract lets the operating system find it. The pointer leads to tables with signatures identifying their purpose. Examples include:

- **FADT:** fixed platform and power-management information;
- **MADT:** interrupt-controller and processor topology information;
- **MCFG:** PCI Express configuration-space access description;
- **SRAT and SLIT:** NUMA affinity and distance information;
- **HPET:** high precision event timer description on applicable systems;
- **DSDT and SSDTs:** definition blocks containing ACPI Machine Language objects and control methods.

ACPI is more than static inventory. AML methods allow firmware-defined logic to participate in operations such as device power, thermal control, battery reporting, hotplug, and platform-specific sequencing. The operating system includes an AML interpreter and evaluates methods under defined circumstances.

This power comes with complexity. Firmware bugs can present invalid tables, incorrect resource descriptions, or methods that assume behavior of one operating system. Linux carries quirks and workarounds because the specification alone cannot repair deployed firmware.

### Safe ACPI inspection

Linux exposes raw tables under sysfs on many systems:

~~~sh
ls /sys/firmware/acpi/tables 2>/dev/null
for table in /sys/firmware/acpi/tables/*; do
    printf '%s %s bytes\n' "$table" "$(stat -c %s "$table" 2>/dev/null)"
done
~~~

The acpidump and iasl tools can capture and disassemble tables. Disassembled AML is firmware-provided program logic, not a simple configuration file. Inspect copies; do not override production tables casually. A malformed override can prevent boot or damage power-management behavior.

Useful kernel messages include:

~~~sh
journalctl -b -k | grep -Ei 'ACPI|APIC|NUMA'
~~~

Warnings should be interpreted in context. Some firmware errors affect an unused method and are harmless in practice; others explain missing devices or broken suspend.

---

## 9. Device Tree: explicit hardware topology

Many embedded and arm systems use a Flattened Device Tree rather than relying primarily on ACPI. A Device Tree describes hardware nodes, addresses, interrupts, clocks, reset lines, buses, compatible strings, and relationships that software cannot reliably probe.

The source form, normally DTS and included fragments, is compiled into a binary DTB. Firmware or a boot loader passes the DTB address to the kernel according to the architecture boot protocol. The kernel matches compatible strings to drivers and uses properties to configure resources.

A simplified node might look like:

~~~dts
serial@10000000 {
    compatible = "vendor,soc-uart";
    reg = <0x0 0x10000000 0x0 0x1000>;
    interrupts = <0 33 4>;
    status = "okay";
};
~~~

This does not make the UART operate. It tells Linux which driver family may match, where registers are mapped, and how interrupt routing is described. The driver still performs initialization.

On a running system, the kernel may expose the live tree:

~~~sh
find /sys/firmware/devicetree/base -maxdepth 2 -type d 2>/dev/null | head
~~~

Properties are binary and often contain NUL-terminated strings or big-endian cells, so indiscriminate text tools can mislead. The dtc tool can convert a captured tree into readable source form.

Device Tree and ACPI solve overlapping description problems through different ecosystems and abstractions. Neither is simply a list of device names.

---

## 10. SMBIOS and human-facing inventory

The System Management BIOS specification defines structures for inventory-like information: system vendor and product, board identity, firmware version, processor sockets, memory devices, chassis data, and related attributes. Linux tools such as dmidecode read these structures.

~~~sh
sudo dmidecode -t system
sudo dmidecode -t bios
sudo dmidecode -t memory
~~~

Treat the values as firmware claims. Vendors sometimes leave placeholder strings, report module data inconsistently, or virtualize the information. SMBIOS is useful for fleet inventory and matching firmware packages, but it is not a cryptographic identity and does not replace kernel discovery.

Compare three sources when investigating hardware:

- SMBIOS says what the platform claims was installed or configured.
- ACPI or Device Tree describes resources and relationships for the OS.
- Linux bus and driver subsystems show what the kernel actually discovered and bound.

Disagreement is evidence, not a reason to arbitrarily choose one source.

---

## 11. Firmware memory maps

Before Linux can safely allocate a page, it needs to know whether that address is usable RAM, firmware code, runtime data, device memory, an ACPI region, persistent memory, or reserved for another purpose.

Legacy x86 environments developed interfaces such as the BIOS E820 memory map. UEFI provides a typed memory map through GetMemoryMap(). The loader passes appropriate information to Linux. The map is dynamic while firmware boot services allocate and free memory; this is why UEFI ExitBootServices() requires a current map key. If allocations change between retrieving the map and exiting, the call can fail and the loader must retry correctly.

Linux transforms firmware descriptors into its own resource and memory-management representation. It reserves the kernel image, initramfs, firmware runtime regions, ACPI data, crash-kernel ranges, and other special areas before releasing ordinary pages to general allocation.

After boot, useful views include:

~~~sh
cat /proc/iomem
dmesg | grep -Ei 'e820|efi.*mem|BIOS.*map|reserved'
~~~

Reading /proc/iomem may require privilege for full addresses on hardened systems. A "reserved" range is not wasted by definition. It may protect device windows or data needed later. Reclassifying it without understanding ownership risks corruption.

---

## 12. Randomness and entropy handoff

Address randomization, cryptographic operations, and security policy benefit from unpredictable seeds very early. Firmware can expose a hardware random-number protocol or pass seed material. Boot loaders may mix firmware randomness and transfer it to the kernel. Linux combines inputs into its random subsystem rather than treating one vendor source as unquestionable truth.

Early randomness illustrates a general rule: a stage can pass useful state without remaining the owner. The kernel consumes and transforms a seed; normal randomness thereafter belongs to the kernel subsystem.

Poor virtual firmware or cloned machine images can create surprising repetition if entropy is mishandled. Contemporary boot designs attempt to carry fresh seed material across boundaries, but claims about cryptographic readiness should be tied to the actual platform, kernel, and boot-loader implementation.

---

## 13. Firmware runtime services

UEFI distinguishes boot services from runtime services. Boot services support the pre-OS environment and end at ExitBootServices(). Selected runtime services and memory regions remain callable by the operating system, for example variable access, time services, and reset requests.

Linux maps and manages runtime regions carefully because calling firmware after the kernel owns the machine crosses an unusual boundary. Firmware code executes in the context prepared by the kernel and may contain assumptions or bugs. Some systems disable or restrict runtime services through policy or kernel parameters when firmware is broken.

The presence of /sys/firmware/efi/efivars commonly indicates that the running kernel exposes EFI variable services:

~~~sh
mount | grep efivarfs
ls /sys/firmware/efi/efivars 2>/dev/null | head
~~~

EFI variables are not ordinary harmless files. Some control boot order, platform keys, or firmware behavior; storage capacity is limited; and buggy firmware has historically handled edge cases badly. Inspecting names is safe. Writing arbitrary data is not.

---

## 14. Firmware security boundaries

Firmware runs with exceptional authority before the operating system. Compromise can persist across disk replacement and can manipulate every later stage. Modern platforms therefore use several security mechanisms:

- cryptographic verification of firmware volumes and updates;
- hardware roots of trust and immutable initial code;
- rollback prevention for vulnerable firmware versions;
- authenticated UEFI variables;
- Secure Boot verification of EFI images;
- measured boot extending component measurements into a TPM;
- write protections and update capsules;
- isolation for management and security coprocessors.

These mechanisms have different guarantees. Verified firmware says code was accepted by a trust policy. Measured firmware records what ran for later evaluation. A signed update says who authorized a payload but still depends on correct update logic. Physical write protection can defend against a class of software attacks but may complicate service.

Linux cannot retroactively make untrusted early code trustworthy. It can measure, restrict, and validate later stages, and security architectures can use hardware roots to establish a chain. Chapter 6 follows that chain in detail.

---

## 15. Virtual firmware

In QEMU/KVM environments, OVMF commonly supplies UEFI while SeaBIOS supplies a legacy BIOS. The firmware still performs discovery, creates tables, exposes boot services, and selects a device, but the hardware model is configured by the virtual-machine definition.

This makes cause and effect unusually visible. Changing the virtual chipset changes PCI topology. Switching from emulated SATA to virtio-blk changes which driver must be available. Enabling UEFI replaces the legacy boot path. Adding a virtual TPM enables measured-boot experiments. A serial console can capture output from multiple stages.

A useful lab compares:

1. the same disk under legacy BIOS and UEFI;
2. the same kernel with ACPI tables from different machine types;
3. a boot disk attached through SATA, NVMe, and virtio;
4. cold VM creation versus snapshot resume.

Keep the guest filesystem constant while changing one platform property. The resulting differences reveal which assumptions belonged to firmware and device topology rather than distribution contents.

---

## 16. Failure patterns before the boot loader

### No visible activity

Possible causes include power sequencing, reset, bootstrap CPU, early firmware authentication, memory training, or display initialization. Use platform diagnostics rather than operating-system repair tools.

### Repeated power cycles

Memory training failure, watchdog reset, firmware recovery, unstable power, or incompatible configuration can cause loops. Record timing and POST codes. A loop before boot selection cannot be fixed by editing the kernel command line.

### Device visible in firmware setup but not bootable

The controller may be discovered while no acceptable boot application exists. The ESP may be missing, the boot variable may reference a stale device path, the image may fail signature policy, or legacy/native mode may not match disk preparation.

### Firmware boot is slow

Network PXE attempts, device enumeration, storage-controller initialization, memory tests, or unreachable removable devices can delay selection. Measure before the loader begins. Disabling random devices is not a scientific first move; change one option and preserve recoverability.

### Linux reports malformed ACPI or resource conflicts

Update firmware when appropriate, capture complete logs, and compare with vendor advisories. Kernel parameters that disable ACPI features can be useful as diagnostic experiments, but broad workarounds may remove power, interrupt, thermal, or device functionality and should not become permanent without understanding the consequence.

---

## 17. A read-only firmware inventory

Run this on a non-sensitive lab machine and save the output locally:

~~~sh
printf '%s\n' '== architecture =='
uname -a

printf '%s\n' '== firmware mode =='
test -d /sys/firmware/efi && echo UEFI || echo no-exposed-UEFI

printf '%s\n' '== DMI identity =='
for f in sys_vendor product_name product_version bios_vendor bios_version bios_date; do
    test -r "/sys/class/dmi/id/$f" && printf '%s: %s\n' "$f" "$(cat "/sys/class/dmi/id/$f")"
done

printf '%s\n' '== ACPI tables =='
ls /sys/firmware/acpi/tables 2>/dev/null || true

printf '%s\n' '== PCI drivers =='
lspci -k 2>/dev/null || true

printf '%s\n' '== firmware-related kernel messages =='
journalctl -b -k | grep -Ei 'efi|bios|acpi|dmi|smbios|firmware|microcode' || true
~~~

Review before sharing. Product serials, asset identifiers, network addresses, secure-boot variables, and platform-specific paths can be sensitive.

Now ask:

- Which claims came from firmware-provided tables?
- Which facts were independently discovered by a Linux bus driver?
- Which device has a firmware driver and a separate Linux driver?
- Which memory ranges are reserved, and does the log name their owner?
- Was microcode updated, and at what point in the log?

The exercise is about provenance, not collecting output.

---

## 18. Mechanism summary

Firmware solves a constrained bootstrapping problem. It begins with an architecture reset contract and gradually constructs permanent memory, processor state, bus resources, platform descriptions, boot services, and selection policy. It may initialize a device sufficiently to read a boot application, but that does not make it the long-term owner of the device. Linux later consumes descriptions, validates resources, starts secondary CPUs, binds native drivers, and takes control.

The most important distinctions are now explicit:

- power-on is not yet processor execution;
- processor execution is not yet usable DRAM;
- device discovery is not driver ownership;
- firmware's ability to read a disk does not prove Linux can read it;
- ACPI, Device Tree, and SMBIOS are different description mechanisms;
- boot services end, while a limited set of runtime services may remain;
- firmware initialization and kernel initialization overlap in subject but not in ownership.

The next chapter examines how the initialized firmware environment locates and starts boot code under two very different models: legacy BIOS and UEFI.

### One final rule for investigation

Never infer a whole stage from one familiar noun. A firmware screen that lists an NVMe model proves that firmware obtained identity data; it does not prove that its filesystem driver can read the ESP, that the selected EFI image is valid, or that Linux will bind the native driver. An ACPI table named after a device proves that firmware described an object; it does not prove the device responded. A PCI function in lspci proves configuration-space enumeration; it does not prove successful probe. A block device proves the storage stack created an object; it does not prove readable filesystem metadata.

State conclusions as a ladder of evidence. "The controller is enumerated, the Linux driver is bound, a namespace block device exists, its GPT is readable, and the expected filesystem UUID is present" is longer than "the disk is detected," but it points directly to the next unverified contract. Precise language is one of the best debugging tools available during boot, especially when several software layers reuse the same physical hardware.

---

## Primary references

- UEFI specifications: https://uefi.org/specifications
- UEFI Platform Initialization specifications: https://uefi.org/specifications
- ACPI specification: https://uefi.org/specifications
- Linux ACPI documentation: https://docs.kernel.org/firmware-guide/acpi/
- Linux Devicetree usage: https://docs.kernel.org/devicetree/usage-model.html
- Linux early boot documentation: https://docs.kernel.org/driver-api/early-userspace/
- Linux x86 boot protocol: https://docs.kernel.org/arch/x86/boot.html
- EDK II documentation and source: https://github.com/tianocore/edk2
- Device Tree specification: https://www.devicetree.org/specifications/
- DMTF SMBIOS specifications: https://www.dmtf.org/standards/smbios
