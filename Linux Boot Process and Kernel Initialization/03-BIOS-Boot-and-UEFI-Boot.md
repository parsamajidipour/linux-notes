# BIOS Boot and UEFI Boot

BIOS and UEFI can both lead to the same Linux kernel, but they do not take the same road.

Legacy BIOS boot is built around old PC conventions: firmware selects a device, reads a small block from a conventional location, places it at a conventional address, and jumps into tiny loader code. UEFI defines a richer pre-OS execution environment: firmware discovers handles and protocols, reads a standardized filesystem, loads a typed executable image, passes formal parameters, and manages persistent boot options.

Calling UEFI "a modern BIOS" is convenient in casual speech and harmful in technical reasoning. The setup screen is not the architecture. A graphical BIOS-like menu tells you almost nothing about whether the machine will start code through an MBR sector, a Compatibility Support Module, a native UEFI Boot#### entry, a removable-media fallback path, or a direct EFI-stub kernel.

This chapter traces both models, compares their contracts, and shows how to prove which path a running Linux system actually used.

---

## 1. One goal, two execution models

Both firmware models must answer a high-level question: which program should receive control next? Their answers differ.

Legacy BIOS generally treats the selected boot device as a sequence of sectors. The firmware need not understand the installed operating system or its filesystem. It loads conventional boot code and lets that code discover everything else.

UEFI treats boot targets as executable images reachable through device paths and firmware protocols. The firmware Boot Manager can load a PE/COFF-format EFI application from a filesystem it understands, normally a FAT filesystem on an EFI System Partition. Persistent variables describe boot options and their order.

The consequences are structural:

| Question | Legacy BIOS model | Native UEFI model |
|---|---|---|
| Initial object | boot sector or firmware extension | EFI executable image |
| Typical location | first sector plus loader-specific embedding | file on the EFI System Partition |
| Selection | device boot order and boot signature | BootOrder, BootNext, Boot####, fallback rules |
| Firmware filesystem knowledge | not required for the initial disk handoff | required through Simple File System and related protocols |
| Executable format | raw initial machine code | PE/COFF EFI application |
| Persistent option identity | usually device-oriented | structured variable containing device path and file path |
| Handoff services | BIOS interrupts and conventions | UEFI protocols, boot services, system table |
| End of firmware boot ownership | informal transition as loader/kernel takes over | explicit ExitBootServices() boundary |

This table describes native paths. A UEFI platform can include a Compatibility Support Module that emulates legacy BIOS behavior. If it does, the machine may be physically modern while the operating system was booted through the legacy model.

---

## 2. The legacy BIOS path

### 2.1 Firmware chooses a device

After platform initialization, BIOS follows configured policy to choose a candidate device: a disk, optical medium, network adapter, removable drive, or another source. For a traditional disk path, it reads the first 512-byte sector, checks the conventional signature bytes 0x55 and 0xAA at the end, loads the sector at a conventional memory address, and transfers control.

The signature says only that the sector is marked as bootable by convention. It does not authenticate the code, prove a valid partition table, or guarantee that later loader stages exist.

The CPU enters the boot sector's code in a constrained 16-bit environment. Memory is limited, firmware services are accessed through software interrupts, and the sector has very little room after any partition table fields. A full filesystem-aware loader cannot normally fit there.

### 2.2 Why loaders are staged

The size limit creates a chain:

1. Tiny first-stage code runs from the boot sector.
2. It locates additional loader code in known sectors, an embedding area, a BIOS Boot Partition, or filesystem blocks.
3. The larger loader understands configuration and filesystems.
4. It loads the kernel and initramfs and enters Linux through the x86 boot protocol.

GRUB on BIOS systems is a good example. The exact layout depends on partitioning and installation choices. Historically, space after the MBR and before the first partition was often used for embedded loader code. GPT does not promise an arbitrary post-MBR gap, so GRUB conventionally uses a small partition with the BIOS Boot Partition GUID to embed core code. This partition does not contain a mounted filesystem and is not the EFI System Partition.

Confusing the BIOS Boot Partition with the ESP is a common mistake:

- a BIOS Boot Partition holds loader embedding data for BIOS-mode GRUB on GPT;
- an EFI System Partition holds files used by UEFI applications;
- a dual-mode disk may contain both;
- neither is the Linux root filesystem merely because it participates in startup.

### 2.3 BIOS interrupt services

Legacy firmware exposes callable services through software interrupts. Historically important examples include disk access, video output, keyboard input, and memory information. Early loaders use these interfaces while running in the expected CPU mode.

These calls are constrained by historical addressing and device models. Extensions such as Enhanced Disk Drive services were added to handle larger disks and linear block addresses, but the interface remains rooted in compatibility.

Once a loader or kernel switches processor modes and takes ownership, normal Linux drivers replace BIOS services. Linux does not perform ordinary storage I/O by calling the BIOS after startup.

### 2.4 The active partition convention

With an MBR partition table, firmware may simply execute MBR code, and that code may search for a partition marked active, load that partition's volume boot record, and chain to it. This is a loader convention rather than an operating-system requirement. GRUB can use different strategies, and vendor firmware behavior varies.

The active flag has no equivalent role in native UEFI boot. Marking a GPT partition "bootable" through a tool whose terminology comes from MBR may not create a UEFI boot entry or install an EFI executable.

---

## 3. The native UEFI path

UEFI provides a defined environment for firmware applications. It is not a small sector jump with a collection of old interrupt calls.

### 3.1 EFI images

An EFI application is a PE/COFF-format executable built for the platform architecture and UEFI environment. On x86-64 the filename often ends in .efi, but the format and machine type matter more than the suffix.

The firmware image service loads the executable, performs required relocation and setup, creates an image handle, and calls its entry point with:

- an image handle identifying the loaded image;
- a pointer to the EFI System Table.

The system table leads to boot services, runtime services, and console handles. The application can locate additional protocols through the handle database.

GRUB's EFI build, systemd-boot, shim, firmware update utilities, UEFI shell programs, and a Linux kernel with the EFI stub can all be EFI images.

### 3.2 Handles and protocols

UEFI models devices and services through handles carrying protocols. A handle can support a device path protocol, block I/O, a simple filesystem, loaded-image information, graphics output, network service bindings, or many other interfaces.

A boot application does not need to know every controller register. It can ask firmware for handles supporting a protocol and use the defined interface. Firmware drivers produce those protocols during platform initialization.

This abstraction is why a small EFI loader can read a file from a supported filesystem without embedding a native Linux storage driver. It also explains the ownership transition: those firmware boot services are not the operating system's permanent device model.

### 3.3 Device paths

A UEFI device path is a structured sequence of nodes identifying how to reach an object through the platform topology. A boot option may describe a PCI path, storage device, GPT partition identity, and file path. It is more precise than a label such as "first disk," but hardware changes can still make a stored path stale.

The human-readable output of efibootmgr -v often shows a condensed device path. For example, a boot option might reference a GPT partition by its unique GUID and then a file such as EFI/vendor/grubx64.efi.

### 3.4 Boot variables

The UEFI Boot Manager uses variables such as:

- **BootOrder:** ordered list of option numbers;
- **BootNext:** one-time option attempted before normal order;
- **BootCurrent:** option used for the current boot, when available;
- **Boot####:** a structured load option containing attributes, description, device path, and optional data.

These variables normally live in firmware-managed nonvolatile storage and can be exposed by Linux through efivarfs. They are data structures, not shell scripts.

On a UEFI-booted Linux system:

~~~sh
sudo efibootmgr -v
~~~

The command is read-only without modification options. It can show stale entries, duplicates, inactive entries, and current order. Do not assume the first friendly description uniquely identifies a file; read the device and file path.

### 3.5 Boot Manager selection

A simplified UEFI flow is:

1. If a valid BootNext is present, attempt it once.
2. Iterate active entries named by BootOrder according to policy.
3. Load the referenced image through its device path.
4. Authenticate it when Secure Boot policy requires verification.
5. Start the image.
6. If it returns with failure or cannot be loaded, continue recovery or fallback policy.

Actual firmware may provide vendor menus, boot discovery, network policy, or recovery behavior around the standardized mechanisms.

---

## 4. The EFI System Partition

The EFI System Partition, or ESP, is a GPT partition identified by the EFI System Partition type GUID. UEFI expects a filesystem conforming to its FAT filesystem specification, and operating-system tools commonly format it as FAT32 for normal disk sizes.

The ESP is a shared platform boot filesystem, not necessarily dedicated to one operating system. A typical layout might contain:

~~~text
EFI/
├── BOOT/
│   └── BOOTX64.EFI
├── Microsoft/
│   └── Boot/
├── Linux/
│   └── linux-uki.efi
└── vendor/
    ├── grubx64.efi
    └── shimx64.efi
~~~

Names and paths vary. The architecture-specific removable-media fallback filename is standardized; for x86-64 it is EFI/BOOT/BOOTX64.EFI. Firmware can try this path when no usable persistent boot entry exists or for removable media.

The fallback path is valuable for recovery but can surprise administrators. A machine may boot successfully despite missing NVRAM entries because the fallback file exists. Another may boot only when a vendor entry is present because no fallback file was installed.

### Mounting is an operating-system choice

Linux distributions mount the ESP at paths such as /boot/efi, /efi, or /boot. The mount point is not stored in the partition and means nothing to firmware. Firmware identifies the partition and reads its filesystem through UEFI protocols before Linux mount namespaces exist.

Inspect safely:

~~~sh
lsblk -o NAME,PARTTYPE,PARTTYPENAME,FSTYPE,PARTUUID,MOUNTPOINTS
findmnt -t vfat,fat
~~~

Do not identify the ESP only by its current mount point. It may be unmounted.

---

## 5. ExitBootServices(): the explicit boundary

UEFI boot services include memory allocation, protocol discovery, event handling, image loading, and device access intended for the pre-OS environment. The operating system must call ExitBootServices() before assuming full ownership.

The call takes a map key from the current firmware memory map. This creates a deliberate race check:

1. The loader obtains the memory map and its key.
2. It prepares to exit without making allocations that change the map.
3. It calls ExitBootServices() with the key.
4. If the key is stale, the call fails; the loader obtains a fresh map and retries.

After success, boot services are no longer available. Firmware must not continue asynchronous boot-time device ownership as though it were still the platform manager. The operating system uses the final map to distinguish reclaimable boot-service memory from runtime regions and other reserved areas.

Runtime services are different. UEFI can preserve selected functions such as variable access, time, and reset. Their code and data remain described by runtime memory attributes, and Linux establishes mappings and calling arrangements for them.

This formal boundary is one of UEFI's most important conceptual improvements over the loose collection of legacy conventions.

---

## 6. Compatibility Support Module

A Compatibility Support Module, or CSM, lets some UEFI firmware follow legacy BIOS boot conventions. The presence of UEFI firmware does not therefore prove the installed system uses native UEFI boot.

CSM can execute legacy option ROMs, provide BIOS-style interrupt services, and load sector boot code. It exists for compatibility with operating systems and devices that do not provide native EFI images.

CSM introduces ambiguity:

- the same physical disk may contain both an MBR-compatible loader and an ESP;
- firmware menus may show the same device twice, once as UEFI and once as legacy;
- installing while booted in one mode may configure only that mode;
- Secure Boot normally requires native UEFI and is incompatible with legacy option-ROM execution;
- switching firmware mode after installation can make a healthy disk appear unbootable.

Modern platforms increasingly omit CSM. New deployments should normally use native UEFI unless a specific compatibility requirement says otherwise.

---

## 7. How to prove the current boot mode

On Linux, a widely useful check is:

~~~sh
test -d /sys/firmware/efi && echo UEFI || echo legacy-or-no-EFI-runtime
~~~

When the kernel boots through EFI and includes the relevant support, it exposes EFI information under /sys/firmware/efi. A populated efivars directory further indicates runtime variable access:

~~~sh
findmnt /sys/firmware/efi/efivars
cat /sys/firmware/efi/fw_platform_size 2>/dev/null
~~~

The platform-size file can distinguish 32-bit and 64-bit EFI firmware where exposed. That value describes firmware, not necessarily the userspace bitness.

Additional evidence:

~~~sh
journalctl -b -k | grep -Ei 'efi|uefi|BIOS-e820|EFI v'
sudo efibootmgr -v
~~~

If efibootmgr reports that EFI variables are not supported, possibilities include legacy boot, a kernel without EFI variable support, efivarfs not mounted, container restrictions, or firmware/runtime quirks. Use multiple signals.

The partition table does not prove the mode. GPT works with BIOS loaders; MBR-partitioned media can contain UEFI-readable arrangements in limited cases. The existence of an ESP suggests UEFI preparation but does not prove this particular boot traversed it.

---

## 8. Linux as an EFI application

When built with EFI stub support, the Linux kernel image can present itself as a PE/COFF EFI executable. Firmware or an EFI boot manager can load and start it directly. The stub gathers UEFI information, handles the command line and initramfs through defined mechanisms, exits boot services, and enters the normal architecture boot path.

This avoids a conventional multi-feature loader, though practical systems still need a way to manage configuration, versions, initramfs, and recovery. systemd-stub and Unified Kernel Images build on the EFI executable model by packaging kernel-related components into one authenticated object.

A direct entry might conceptually point firmware to an EFI-executable kernel file on the ESP. Command-line delivery through firmware load-option data has quoting and portability considerations, so distributions often use a small manager or UKI design rather than manually maintaining long NVRAM option strings.

The important point is that "boot loader" is a role, not a mandatory product. UEFI can load Linux when the kernel image itself satisfies the EFI application contract.

---

## 9. Chainloading in both models

Chainloading means one boot component loads another boot component instead of directly loading the final kernel.

Under legacy BIOS, a loader can load and execute another partition's boot sector. This preserves an operating system's expected sector-based entry.

Under UEFI, an EFI application can load another EFI image through boot services. GRUB may chainload a Windows boot manager; shim loads a second-stage loader after applying its trust logic; firmware can load a vendor manager which selects a UKI.

Chainloading is not inherently insecure or inefficient. The important questions are:

- which stage selects the next image;
- how the path is resolved;
- whether the image is authenticated;
- which command line or optional data is mutable;
- what happens when the child returns;
- which component owns recovery policy.

A boot menu entry labeled with an operating-system name may actually traverse several signed and unsigned stages before reaching a kernel.

---

## 10. Network boot comparison

Legacy PXE traditionally combines DHCP-based discovery with a firmware network stack and TFTP download of a network bootstrap program. That program then loads a richer environment or operating-system artifacts.

UEFI defines network protocols and can load EFI network applications, supporting IPv4, IPv6, HTTP boot on capable platforms, and architecture-aware files. The selected application may be GRUB, iPXE, shim, or a vendor deployment agent.

The same boundary method applies:

1. Did firmware initialize the network device?
2. Did address and boot-server discovery succeed?
3. Was the boot program downloaded?
4. Did signature policy accept it?
5. Did that program obtain its configuration and kernel?
6. Did the kernel acquire the network configuration needed for a remote root?

"PXE failed" is too broad. A DHCP timeout and a kernel NFS-root failure occur in different stages despite sharing the network.

---

## 11. Secure Boot belongs to the UEFI image path

UEFI Secure Boot can verify EFI images before execution according to keys and signature databases managed by platform policy. It does not scan an arbitrary legacy boot sector and make the BIOS path secure. Enabling CSM generally prevents the native verified-image chain Secure Boot is designed to enforce.

The initial verification may cover shim or another first-stage EFI image. That image can extend trust to GRUB, a kernel, or a UKI. Every later mutable component must be considered. A signed loader that accepts an unsigned configuration capable of redirecting execution needs additional restrictions if the threat model includes configuration tampering.

Chapter 6 treats the full chain. For now, remember that native UEFI gives firmware a typed executable and validation hook that the raw legacy sector convention lacks.

---

## 12. Worked comparison: one disk, two installation modes

Imagine a GPT disk containing:

- a one-megabyte BIOS Boot Partition;
- a 512-megabyte FAT ESP;
- an ext4 root partition;
- GRUB installed for both BIOS and x86-64 EFI.

In BIOS mode:

1. Firmware reads the protective MBR's boot code.
2. GRUB's small code finds core data embedded in the BIOS Boot Partition.
3. GRUB modules locate configuration and files.
4. GRUB loads Linux and initramfs from the chosen filesystem.
5. GRUB follows the Linux/x86 boot protocol.

In UEFI mode:

1. Firmware evaluates a Boot#### entry.
2. Its device path identifies the ESP and an EFI executable.
3. Firmware reads and loads grubx64.efi, or shim followed by GRUB.
4. The EFI build of GRUB obtains configuration and files.
5. GRUB uses the appropriate Linux entry protocol and exits firmware boot services.

The Linux root can be identical. The loader configuration can look similar. Yet changing firmware from legacy-only to UEFI-only selects a different installed loader path. If only one path was installed correctly, the other mode reports no bootable target.

This is why repair environments must be booted in the intended mode. Running a distribution installer in legacy mode may install or repair BIOS GRUB while the administrator expects a UEFI NVRAM entry, and vice versa.

---

## 13. Inspecting a dual-mode machine

Use read-only commands first:

~~~sh
test -d /sys/firmware/efi && echo 'current boot: UEFI' || echo 'current boot: non-UEFI path'
lsblk -o NAME,SIZE,PTTYPE,PARTTYPE,PARTTYPENAME,FSTYPE,MOUNTPOINTS
sudo efibootmgr -v 2>/dev/null || true
findmnt /boot /boot/efi /efi 2>/dev/null || true
sudo fdisk -l
~~~

Questions to answer:

- Is the disk MBR or GPT?
- Is there an ESP, a BIOS Boot Partition, both, or neither?
- Which mode started the current kernel?
- Which EFI file does the current Boot#### option reference?
- Does a standardized fallback file exist?
- Where does the distribution mount the ESP?
- Is the active kernel file stored on the ESP or on a separate /boot filesystem?

Do not modify BootOrder or reinstall a loader until this map is clear.

---

## 14. Common misconceptions

### "GPT means UEFI"

GPT and UEFI are designed to work well together, but they describe different layers. GRUB can boot from GPT in BIOS mode using a BIOS Boot Partition. Conversely, special media can support UEFI boot alongside other layouts.

### "The ESP must be mounted at /boot/efi"

That is a distribution convention. Firmware has no concept of the Linux mount point. The ESP may be mounted at /efi, /boot, /boot/efi, or only during updates.

### "The active flag makes a UEFI partition bootable"

Native UEFI selection uses partition type, filesystem protocols, device paths, boot variables, and fallback rules. The MBR active flag belongs to legacy conventions.

### "The UEFI boot menu reads grub.cfg"

Firmware normally selects EFI applications through boot variables. GRUB reads grub.cfg after firmware loads GRUB. The firmware and GRUB menus are different policy layers.

### "Deleting an NVRAM entry deletes the operating system"

The variable points toward an EFI image. Deleting it normally leaves partitions and files intact, though the machine may lose its normal selection path. A fallback file or recreated entry can restore access.

### "Secure Boot encrypts the boot files"

Secure Boot validates signatures. ESP contents are normally readable FAT files. Confidentiality requires different mechanisms.

---

## 15. Failure classification

### Firmware says no bootable device

Check selection mode, device presence, boot variables, the referenced ESP, fallback path, and signature policy. Do not begin with root filesystem repair; Linux has not run.

### UEFI entry exists but immediately returns

The device path may be stale, the file missing, the architecture wrong, the image malformed, or verification rejected. Firmware logs and another EFI shell or boot entry can help isolate the file from the variable.

### Legacy loader prints only partial text

First-stage code likely ran. Embedded core data, disk geometry assumptions, moved sectors, or later loader files may be missing. The last visible loader message matters.

### Disk boots in one firmware mode but not another

Inventory which loader path is installed. Do not toggle random compatibility options. Decide the intended mode, boot rescue media in that mode, and repair that path.

### Boot works only through the firmware's "boot from file" browser

The EFI image and filesystem are usable, but persistent Boot#### selection may be absent or stale. Compare the manually selected path with efibootmgr output and the fallback location.

---

## 16. Safe UEFI shell thinking

An EFI shell can reveal mappings such as fs0:, list files, inspect devices, and start an EFI application directly. The mapping numbers are assigned for the current environment and should not be treated as permanent disk identities.

A diagnostic sequence can be conceptually simple:

~~~text
map -r
fs0:
ls EFI
EFI\vendor\grubx64.efi
~~~

Backslashes follow UEFI file-path conventions. Starting the file proves that firmware can access and execute it; it does not prove the persistent Boot#### path is correct. If the image starts manually, compare the device path and variable configuration rather than reinstalling unrelated kernel packages.

Use shell commands appropriate to the shell implementation and avoid write operations on important systems until a backup exists.

---

## 17. Migration from BIOS to UEFI

Migration is not achieved by changing one firmware toggle. A reliable plan addresses:

1. firmware native UEFI support;
2. partitioning space for an ESP;
3. a FAT filesystem with the correct partition type;
4. an architecture-correct EFI loader;
5. loader configuration and kernel/initramfs paths;
6. a persistent boot entry or fallback file;
7. Secure Boot requirements if enabled;
8. rescue media and a rollback path.

Converting an MBR disk to GPT is a separate storage operation with data-loss risk. Some systems already use GPT in BIOS mode and need only an ESP and loader installation; others require layout changes. Encryption, software RAID, and unusual /boot arrangements add constraints.

Perform migration from a tested backup or clone. Confirm native UEFI boot before removing the legacy path. The ability to boot both temporarily is useful evidence, not untidiness.

---

## 18. Mechanism summary

Legacy BIOS boot begins with a deliberately tiny, weakly typed sector convention. Because firmware need not understand the operating-system filesystem, loader code grows through stages. UEFI instead offers a structured executable environment with handles, protocols, filesystem access, persistent variables, device paths, image loading, and an explicit end to boot services.

Neither model directly explains what Linux later does with root storage. Both can prepare the same kernel and initramfs. The important engineering difference is the contract by which the first boot program is located and executed.

You should now be able to distinguish:

- physical firmware from the boot mode used on this run;
- a firmware Boot Manager from an operating-system boot loader;
- a BIOS Boot Partition from an EFI System Partition;
- an NVRAM boot entry from the EFI file it references;
- a native UEFI application from legacy sector code under CSM;
- boot services from runtime services;
- successful device discovery from successful boot-image selection.

The next chapter moves one level down into persistent storage. It explains how MBR and GPT describe partitions, how the ESP is identified, how redundancy and unique identifiers work, and how boot metadata can be inspected or repaired without confusing partition tables with filesystems.

---

## Primary references

- UEFI Specification, Boot Manager chapter: https://uefi.org/specs/UEFI/2.11/03_Boot_Manager.html
- UEFI specifications index: https://uefi.org/specifications
- Linux EFI boot stub: https://docs.kernel.org/admin-guide/efi-stub.html
- Linux/x86 boot protocol: https://docs.kernel.org/arch/x86/boot.html
- GNU GRUB manual: https://www.gnu.org/software/grub/manual/grub/
- GNU GRUB BIOS installation details: https://www.gnu.org/software/grub/manual/grub/html_node/BIOS-installation.html
- systemd Boot Loader Specification: https://uapi-group.org/specifications/specs/boot_loader_specification/
- efibootmgr project: https://github.com/rhboot/efibootmgr
