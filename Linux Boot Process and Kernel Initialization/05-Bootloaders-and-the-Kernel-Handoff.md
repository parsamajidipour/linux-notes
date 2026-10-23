# Bootloaders and the Kernel Handoff

A boot loader does not merely copy a file and hope the CPU understands it.

Linux defines architecture-specific entry contracts. The loader must locate the right artifacts, place them in acceptable memory, build parameter structures, establish required processor state, and jump to the documented entry point. The kernel can then rely on a small but precise set of facts even though its normal allocators, drivers, filesystems, and process model do not exist yet.

This chapter compares common loader designs and then follows the handoff itself. The details use x86-64 frequently because its boot protocol is extensively documented and historically layered. The method applies elsewhere: read the architecture booting contract, distinguish required state from loader convenience, and never assume an x86 register or image layout is universal.

---

## 1. The loader's real responsibilities

A Linux loader usually performs five jobs:

1. **Selection:** decide which kernel entry to use and expose recovery choices.
2. **Acquisition:** read the kernel, initramfs, and possibly Device Tree or auxiliary blobs from storage or a network.
3. **Validation:** check file structure and, when policy requires it, cryptographic signatures or hashes.
4. **Construction:** create the command line and architecture-specific boot parameters and reserve memory for every artifact.
5. **Transfer:** establish the required CPU environment and jump to the kernel entry point.

Menus, themes, scripting, filesystem modules, and network support are features around these responsibilities. A minimal EFI loader may delegate storage access to firmware and support only simple configuration. GRUB can implement a rich pre-OS environment. A direct EFI-stub boot can collapse loader and kernel image into one executable path. All must satisfy the final kernel contract.

### What the loader does not do

The loader does not create Linux processes, mount the final VFS root, initialize the Linux scheduler, bind Linux drivers, or remain as a resident service manager. It may read the same disk and parse a similar filesystem before handoff, but its implementation and ownership are independent.

---

## 2. GRUB as a modular loader

GRUB is designed to load multiple operating systems across multiple firmware models and storage layouts. It includes a command interpreter, modules, filesystem readers, partition parsers, video and terminal support, cryptographic facilities, network components, and platform-specific installation code.

### 2.1 GRUB's own stages

In BIOS mode, tiny initial code must locate a larger core image. The core can load modules and reach configuration and filesystems. On GPT, a BIOS Boot Partition commonly provides embedding space.

In UEFI mode, firmware loads GRUB's EFI executable from the ESP. The executable can use UEFI services and include or load GRUB modules. There is no need for MBR first-stage code in the native path.

In both cases, GRUB eventually presents the same conceptual environment: devices use GRUB naming, modules expose filesystems, configuration executes commands, and a selected menu entry runs loader commands.

### 2.2 A simple Linux entry

A conceptual GRUB entry is:

~~~text
menuentry 'Linux example' {
    search --fs-uuid --set=root 1111-2222
    linux /boot/vmlinuz-example root=UUID=aaaa-bbbb ro
    initrd /boot/initramfs-example.img
}
~~~

There are two roots here:

- GRUB's root identifies where GRUB should read kernel files.
- the kernel root= argument identifies the future operating-system root or gives early userspace enough policy to find it.

They can refer to different filesystems. For example, GRUB may read an unencrypted /boot partition while initramfs unlocks an encrypted root.

### 2.3 Configuration is program logic

grub.cfg is not merely a static list. It can contain conditions, searches, variables, submenus, and generated commands. Distributions normally generate it from supported configuration files and scripts. Manual edits may be overwritten and can create security problems if configuration remains mutable outside the verified chain.

To understand the active entry, inspect the generated commands rather than only the source defaults. Do not regenerate configuration as a ritual before identifying which file the active GRUB instance actually reads.

---

## 3. systemd-boot as a small UEFI manager

systemd-boot deliberately depends on UEFI. Firmware provides the image-loading environment and filesystem protocol, so systemd-boot can focus on selecting entries stored according to the Boot Loader Specification.

Type #1 entries use text files describing a title, kernel or EFI path, initrd paths, and options. Type #2 entries are Unified Kernel Images: EFI executables containing a kernel and associated payloads.

A conceptual Type #1 entry looks like:

~~~text
title   Linux example
linux   /vmlinuz-example
initrd  /initramfs-example.img
options root=UUID=aaaa-bbbb ro
~~~

The simplicity reduces loader-side complexity but imposes a layout: relevant files must be on a filesystem accessible in the defined boot hierarchy. systemd-boot is not a BIOS loader and does not attempt GRUB's broad filesystem and platform coverage.

The comparison is not "powerful versus primitive." It is two architectural choices:

- GRUB brings a broad modular environment to many platforms and layouts.
- systemd-boot relies on native UEFI and a constrained file organization to minimize policy and code.

Choose based on storage, security, multi-boot, recovery, and operational requirements.

---

## 4. The EFI stub and direct boot

Linux built with EFI stub support can be loaded as a normal EFI PE/COFF image. Architecture-specific wrapper data lets UEFI recognize the image; stub code gathers firmware information, arranges the initramfs and command line, exits boot services, and enters the kernel's normal boot flow.

The stub makes a conventional loader optional, not configuration optional. Something must still choose the kernel version, supply parameters, locate initramfs, support recovery, and update boot selection. Firmware variables can point directly to a kernel, but maintaining long optional-data command lines in NVRAM is awkward and firmware behavior varies.

Unified Kernel Images package the EFI stub, Linux kernel, initrd, command line, OS metadata, and potentially other sections into one PE file. This creates a useful signed unit: verification can cover the exact kernel, early userspace, and embedded command line together.

The tradeoff is update granularity. Changing an embedded initramfs or command line means building and signing a new UKI. That is often desirable in measured, declarative deployments and less convenient for ad-hoc command-line experimentation. Loaders may permit a temporary interactive override, but allowing unsigned overrides changes the security argument.

---

## 5. The Linux/x86 image

The x86 bootable image convention evolved under strong backward-compatibility pressure. The file commonly named bzImage contains more than one conceptual region:

- a setup header and early setup code understood by loaders;
- architecture-specific compressed-kernel entry code;
- a compressed payload containing the main kernel;
- metadata describing protocol features, sizes, alignment, preferred addresses, relocatability, and entry capabilities;
- when EFI stub support is enabled, a PE/COFF view that lets UEFI treat the image as an application.

The name bzImage historically means a "big zImage" layout, not that bzip2 compression is used. The compression algorithm is selected at kernel build time and may be gzip, xz, LZ4, Zstandard, or another supported choice.

The uncompressed vmlinux is usually an ELF image with symbols useful for debugging. It is not ordinarily the artifact that a PC firmware or standard loader uses directly.

Inspecting file identity helps:

~~~sh
file /boot/vmlinuz-$(uname -r)
readelf -h /usr/lib/debug/boot/vmlinux-$(uname -r) 2>/dev/null || true
~~~

Paths vary, and stripped production systems may not install vmlinux.

---

## 6. The x86 boot protocol

The official Linux/x86 boot protocol defines a setup header and a boot-parameter structure often called the zero page. Protocol revisions add fields while preserving older loaders where possible.

The loader examines header fields to learn what the image supports. Relevant concepts include:

- protocol version;
- load flags and extended load flags;
- relocatable-kernel support;
- required alignment;
- preferred load address;
- total initialization size;
- command-line size;
- initrd placement limits for older protocols;
- payload offsets and lengths;
- entry points for 32-bit, 64-bit, and EFI paths.

Do not memorize numerical offsets as a general boot model. They are valuable when implementing a loader or diagnosing a protocol violation. For normal reasoning, the important fact is that loader and kernel communicate through a versioned binary contract, not an undocumented jump.

### 32-bit entry

For the 32-bit boot protocol, the loader prepares protected-mode processor state, segment descriptors, disabled interrupts, parameter memory, and registers according to the specification. It jumps to the defined 32-bit entry with a register pointing to boot parameters.

### 64-bit entry

The 64-bit protocol requires long mode and paging with identity mappings for defined ranges, appropriate segment descriptors, disabled interrupts, and the boot-parameter pointer in the required register. The 64-bit entry address is calculated according to the image layout.

### EFI PE entry

With the EFI stub, firmware's ordinary PE/COFF entry reaches stub code, which creates the Linux boot state. An older EFI handover protocol existed for loaders to jump into the stub with loader-built parameters; current documentation marks it deprecated in favor of the normal PE entry point.

These are alternative entry contracts, not sequential steps all taken during one boot.

---

## 7. Boot parameters as a data handoff

The boot-parameter block carries data that early Linux cannot cheaply rediscover:

- memory map information;
- command-line location;
- initramfs location and size;
- screen and video state;
- loader identity;
- ACPI root pointer in applicable paths;
- EFI system information;
- setup_data extensions;
- hardware subarchitecture details;
- architecture-specific fields.

The loader must keep pointed-to buffers alive and avoid overlapping them with the kernel's decompression destination or other reserved regions. The kernel copies, reserves, or transforms data as initialization progresses.

A classic loader bug is not "the file failed to copy" but "two valid objects were placed in incompatible memory ranges" or "a pointer was correct before relocation and stale afterward." The protocol's size and alignment fields exist to prevent such failures.

---

## 8. Memory placement

Before the kernel's allocator exists, the loader uses firmware services or its own memory model to place artifacts. It must account for:

- the compressed image and its setup portion;
- space required for decompression and runtime initialization;
- initramfs, which may be large;
- command-line storage;
- boot-parameter structures;
- firmware tables and runtime regions;
- reserved low-memory areas on legacy x86;
- address limitations in older protocol versions;
- alignment and relocatability;
- randomized placement when supported.

The kernel image may be relocatable, but relocatable does not mean "place every object anywhere." The header advertises constraints. The decompressor also needs scratch space and must avoid overwriting its input before data is consumed.

UEFI loaders allocate pages with firmware boot services and obtain a final memory map. Legacy loaders use other discovery interfaces. Either way, Linux receives both objects and a description of physical memory ownership.

---

## 9. The kernel command line

The command line is a byte string transported by the loader but consumed by several later components.

Examples:

~~~text
root=UUID=aaaa-bbbb ro console=ttyS0,115200n8 loglevel=7
~~~

The kernel interprets root= in simple direct-root cases, console=, loglevel=, memory and CPU controls, and thousands of subsystem parameters. Initramfs frameworks interpret parameters such as rd.luks.*, rd.lvm.*, or distro-specific network-root options. PID 1 may interpret systemd.* options. Unrecognized arguments without a dot can be passed toward init depending on parsing rules.

This shared transport means a parameter can be present in /proc/cmdline yet ignored by the component the administrator expected. Always check the documentation of the intended consumer.

### Quoting and length

Loader configuration syntax determines how spaces and quotes become the final byte string. Firmware optional data, GRUB script parsing, shell-like configuration, and Boot Loader Specification files do not all have identical quoting rules.

The image header advertises supported command-line size in modern x86 protocols. Truncation can remove parameters at the end and produce misleading partial behavior. Keep production command lines auditable rather than treating them as an unlimited configuration database.

After boot:

~~~sh
cat /proc/cmdline
~~~

This is the kernel's recorded command line and better evidence than an inactive configuration file.

---

## 10. Loading initramfs

The loader places the initramfs image in memory and communicates its address and size. The kernel later identifies compression and cpio streams and unpacks them into rootfs.

GRUB's initrd command can load one or more components in order. Concatenation is meaningful because the kernel's unpacker and early-boot conventions can process sequences such as an uncompressed early microcode archive followed by a compressed main initramfs.

The loader does not execute /init. It only transports bytes and metadata. The kernel's unpacking and process creation happen much later.

A file named initrd may contain initramfs cpio data. Filenames reflect distribution convention more than the historical block-device distinction. Inspect content with the distribution's supported tools instead of relying on the suffix.

---

## 11. Device Tree and architecture handoff

On systems using Device Tree, the loader passes a DTB address according to the architecture protocol. It may select among board-specific blobs, apply overlays, or receive a tree from firmware. The kernel parses the tree to discover memory, CPUs, buses, interrupt routing, clocks, and devices not safely enumerable.

A wrong DTB can boot surprisingly far and then fail when a driver accesses incorrect registers, clocks, or GPIO relationships. "The kernel image is correct" does not prove the platform description is correct.

ARM64 boot documentation defines register state, memory placement, cache requirements, and the DTB pointer. RISC-V defines its own contract. Never copy x86 assumptions about a zero page or bzImage into another architecture.

---

## 12. Exiting UEFI boot services

An EFI loader or stub must obtain the current memory map and call ExitBootServices() with its map key. Any intervening allocation can invalidate the key. Robust implementations retry when firmware reports a stale map.

After success:

- boot services must not be called;
- memory classified as boot-service code or data can eventually be reclaimed under OS rules;
- runtime-service regions remain special;
- the kernel owns interrupts and devices as it initializes native support;
- the final memory map becomes evidence for early allocation.

The loader must not print through ordinary UEFI console boot services after exit. This is one reason output can go dark briefly between the loader and kernel console initialization.

---

## 13. Security and validation

Validation can occur at several layers:

- UEFI verifies the first EFI image under Secure Boot.
- shim verifies a second-stage loader or UKI using built-in and owner-managed trust.
- GRUB can enforce signature checks on modules, configuration, kernels, and initramfs depending on build and policy.
- a UKI signature can cover its embedded components.
- the kernel can enforce signed modules and lockdown behavior.

A hash check without a trusted expected value detects accidental corruption only if the expected hash itself is protected. A signature is meaningful only if the verifier's keys and policy are protected. Interactive command-line editing can bypass an otherwise signed configuration if untrusted users have console access and the signed component accepts arbitrary unsigned inputs.

Security design must cover the entire path actually used, including fallback files and rescue entries.

---

## 14. Worked example: separate /boot and encrypted root

Suppose the ESP contains shim and GRUB. A separate ext4 /boot contains kernels and initramfs images. The root filesystem lives inside LUKS and LVM.

The path is:

1. UEFI loads shim from the ESP.
2. shim validates and loads GRUB.
3. GRUB reads its configuration and ext4 module.
4. GRUB reads vmlinuz and initramfs from unencrypted /boot.
5. GRUB constructs a command line identifying the encrypted storage policy and future root.
6. GRUB enters the kernel.
7. The kernel initializes and runs /init from initramfs.
8. Early userspace unlocks LUKS, activates LVM, and mounts real root.

Three separate filesystem readers may be involved: UEFI reads FAT, GRUB reads ext4, and Linux reads the final root filesystem. A failure in one does not imply the others are broken.

If GRUB finds its menu but says the kernel file is missing, investigate /boot and configuration. If the kernel runs but early userspace cannot unlock root, investigate initramfs policy and encryption. Recreating the UEFI Boot#### entry addresses neither.

---

## 15. Worked example: mismatched kernel and initramfs

A loader entry points to a new kernel but an old initramfs. The kernel boots, but root storage requires a module built for the new kernel. Module loading reports an invalid format or version mismatch, and the root device never appears.

The loader did its job: both named files were loaded. The failure is semantic consistency between artifacts.

Inspect from rescue:

~~~sh
uname -r
ls -l /boot
lsinitramfs /boot/initramfs-example.img 2>/dev/null | head
~~~

Distribution tools differ; dracut images may be inspected with lsinitrd. Confirm the entry pairs the intended versions, rebuild only the affected initramfs using supported tooling, and retain a known-good entry.

Versioned filenames and atomic update strategies reduce this risk. Overwriting one generic kernel path before its initramfs is complete creates a window where the menu points to an inconsistent pair.

---

## 16. Loader diagnostics

### GRUB shell

GRUB's command line can list devices, inspect variables, search by filesystem UUID, load modules, and test file existence. Commands operate in GRUB's device model, not Linux /dev naming.

Useful non-destructive questions include:

~~~text
set
ls
ls ($root)/
search --fs-uuid 1111-2222
~~~

Syntax depends on version and loaded modules. Record $prefix and $root because they explain where GRUB expects modules and configuration.

### systemd-boot status

From a running system, bootctl can report firmware mode, loader information, selected entry, ESP paths, and discovered entries:

~~~sh
bootctl status
bootctl list
~~~

Treat output as a view of current configuration. Verify files on the exact partition shown.

### Kernel-side evidence

After a successful handoff:

~~~sh
cat /proc/cmdline
journalctl -b -k | head -n 100
cat /sys/kernel/boot_params/data 2>/dev/null | hexdump -C | head
~~~

The raw boot-parameter exposure is architecture and configuration dependent. Kernel logs normally provide the useful interpretation.

---

## 17. Failure patterns

### Loader cannot read the kernel file

Check loader root, partition and filesystem modules, configuration path, and actual filename. The root= kernel argument is irrelevant because the kernel has not run.

### "Invalid magic" or unsupported image

The selected file may be corrupt, wrong architecture, not a bootable kernel format for this command, or processed by the wrong loader path. Compare file type and checksum with the installed package.

### Immediate reboot after selection

Possibilities include signature rejection with poor firmware messaging, invalid entry state, decompressor failure, early triple fault, watchdog reset, or a platform-specific problem. Enable loader debug, early console, and a known-good kernel comparison.

### Kernel starts with unexpected arguments

Inspect /proc/cmdline and the exact selected entry. Multiple configuration files, saved-entry logic, firmware optional data, and UKI-embedded command lines can differ from the file the administrator edited.

### Initramfs apparently ignored

Verify that the loader actually loaded it after the kernel, that the address and size reached the kernel, and that the archive is structurally valid. Kernel logs report unpacking failures. A missing /init can lead the kernel toward direct-root logic instead of the expected early userspace.

---

## 18. Designing reliable loader updates

Boot artifacts should be updated as a coherent set.

A robust strategy:

1. Write new kernel and initramfs under new versioned names.
2. Flush and verify the files.
3. Create a new entry referencing only complete artifacts.
4. Keep at least one known-good entry.
5. Update default selection atomically where possible.
6. Reboot once with a one-shot entry for testing.
7. Remove old artifacts only after the new path is proven.

UKI workflows similarly write a new signed image before changing selection. Updating a file in place can leave a torn artifact after power loss. FAT filesystems and firmware behavior make careful ordering particularly important on the ESP.

Remote systems need out-of-band console access or tested automatic rollback. A successful package command does not prove firmware will select the new path after a cold boot.

---

## 19. Mechanism summary

The loader is a protocol participant. It selects and reads artifacts, validates them according to policy, arranges memory, constructs command-line and platform data, establishes architecture-defined processor state, and jumps to Linux.

GRUB supplies a broad modular environment across BIOS and UEFI. systemd-boot supplies a small UEFI-native selector. The EFI stub lets Linux itself satisfy the EFI application contract, and a UKI packages several boot artifacts into one signed executable. These designs differ in policy placement, not in the kernel's need for a valid entry state.

You should now be able to separate:

- loader filesystem root from Linux root;
- kernel image from uncompressed vmlinux;
- acquisition from authentication;
- command-line transport from parameter consumption;
- initramfs loading from initramfs execution;
- 32-bit, 64-bit, and EFI entry alternatives;
- successful file reads from a correct memory and register handoff.

The next chapter pauses the functional path and studies trust: which stage verifies which object, how keys are enrolled, where shim and owner keys fit, and why Secure Boot, measured boot, and disk encryption answer different questions.

## 20. Artifact-to-entry audit lab

A useful audit begins at the running kernel and walks backward to the loader entry instead of guessing forward from files in /boot.

First record what actually ran:

~~~sh
uname -r
cat /proc/cmdline
cat /etc/os-release
journalctl -b -k | head -n 80
~~~

Then identify the active loader path. On a UEFI system, BootCurrent from efibootmgr can name the firmware option, while bootctl status can identify a systemd-boot or compatible entry. GRUB environments may record a selected entry, but saved defaults, submenu paths, generated configuration, and one-shot selection mean the default configuration is not automatically proof of the current choice.

Inventory candidate artifacts and calculate hashes without changing them:

~~~sh
find /boot /efi /boot/efi -maxdepth 3 -type f \( -name 'vmlinuz*' -o -name 'initr*' -o -name '*.efi' \) -print 2>/dev/null
sha256sum /boot/vmlinuz-* /boot/initr* 2>/dev/null
~~~

Hashes provide stable comparison labels for this audit; by themselves they are not authenticity guarantees. Record file size, modification time, package ownership, and signature state using distribution tools. A package database may show which kernel version installed a file, while a PE signature-inspection tool can show certificates on an EFI image.

For a UKI, use a PE-aware inspection tool or objdump/readelf support available on the system to list sections. Expected section names can include an operating-system release section, command line, kernel payload, initrd, and metadata. Do not extract and execute content from an untrusted image merely to inspect it. Compare the embedded command line with /proc/cmdline, remembering that loader policy may allow additions or overrides depending on Secure Boot state and stub design.

Now draw the chain as concrete objects:

~~~text
Boot0007
  -> GPT PARTUUID 1234...
  -> EFI/Linux/linux-6.x.efi, SHA-256 ...
  -> embedded kernel release 6.x
  -> embedded or external initramfs, SHA-256 ...
  -> observed /proc/cmdline ...
  -> observed uname -r 6.x
~~~

For GRUB with separate artifacts, include the first EFI image, GRUB configuration actually loaded, selected vmlinuz path, initramfs path, and final command line. The exercise often exposes stale files that are harmless, stale entries that are dangerous, or a maintenance script updating an ESP different from the one firmware uses.

Repeat after a kernel update. A reliable update changes a deliberate subset of nodes and keeps an intact fallback chain. If the package manager reports success but the graph points to old hashes, the update and boot-selection paths are disconnected. If the graph points to new files but the current boot still reports the old release, selection or fallback behavior—not compilation—deserves attention.

---

## Primary references

### Cross-architecture handoff checklist

When approaching an unfamiliar architecture, do not translate x86 nouns mechanically. Write a fresh contract table: executable image format, required privilege level, MMU and cache state, interrupt state, register arguments, platform-description pointer, permitted image and initramfs locations, alignment, memory reservations, and the exact action that ends firmware ownership. Then identify which component supplies each value. A loader can be completely correct for x86 and unusable for arm64 because correctness belongs to the target protocol, not to the abstract label "boot loader." This checklist is also useful when comparing firmware boot, kexec, and a hypervisor direct-kernel boot: all can enter the same kernel through different contracts and therefore leave different evidence.

- Linux/x86 boot protocol: https://docs.kernel.org/arch/x86/boot.html
- Linux EFI stub: https://docs.kernel.org/admin-guide/efi-stub.html
- GNU GRUB manual: https://www.gnu.org/software/grub/manual/grub/
- GRUB Linux loading commands: https://www.gnu.org/software/grub/manual/grub/html_node/GNU_002fLinux.html
- GRUB initrd command: https://www.gnu.org/software/grub/manual/grub/html_node/initrd.html
- Boot Loader Specification: https://uapi-group.org/specifications/specs/boot_loader_specification/
- Unified Kernel Image specification: https://uapi-group.org/specifications/specs/unified_kernel_image/
- systemd-stub manual: https://www.freedesktop.org/software/systemd/man/latest/systemd-stub.html
- arm64 booting documentation: https://docs.kernel.org/arch/arm64/booting.html
- RISC-V boot requirements: https://docs.kernel.org/arch/riscv/boot.html
