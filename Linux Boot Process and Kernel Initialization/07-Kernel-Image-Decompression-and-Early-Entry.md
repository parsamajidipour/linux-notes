# Kernel Image Decompression and Early Entry

The boot loader has jumped to Linux, but the normal kernel still does not exist in the form we usually imagine.

On x86, the bootable image normally contains small setup and decompression code wrapped around a compressed kernel payload. That early code runs with no process model, no general allocator, no normal driver framework, and only the entry state promised by the boot protocol. It must validate the machine, avoid the memory occupied by boot artifacts, construct temporary execution state, decompress a large image, apply relocation where necessary, and enter the architecture's main kernel startup.

This phase is short in wall-clock time and dense in mechanism. It is also difficult to observe: if it fails before a console is available, the visible symptom may be a reset or a silent screen. Understanding its constraints explains why loaders advertise alignment and size fields, why address randomization begins before ordinary memory management, and why vmlinuz is not just a compressed file that userspace could unarchive anywhere.

---

## 1. Build artifacts: vmlinux, Image, zImage, and bzImage

Kernel builds produce artifacts with different purposes.

**vmlinux** is the linked kernel in ELF form, commonly containing symbols before stripping. Debuggers, crash-analysis tools, and symbol utilities use it. Its sections have link-time virtual addresses and metadata useful to development tools.

**Image** names an architecture-specific uncompressed boot image on some platforms. The exact format is architecture-defined.

**zImage** and **bzImage** are historical x86 names for bootable compressed-image layouts. The "bz" in bzImage means the layout supported a big kernel, not that the payload uses bzip2. Modern x86 builds normally produce arch/x86/boot/bzImage and distributions install it under a name such as vmlinuz-version.

The bootable file includes entry structures and code; it is not merely compression applied to vmlinux as a whole. Build tooling extracts and transforms the payload, attaches a decompressor and protocol header, and may add an EFI-compatible PE view.

Different architectures use different formats. ARM64 commonly boots an Image file, optionally compressed by external loader conventions, and defines its own image header and entry state. Never infer format from a familiar filename alone.

### Inspecting installed artifacts

~~~sh
file /boot/vmlinuz-$(uname -r)
ls -lh /boot/vmlinuz-* 2>/dev/null
readelf -h /usr/lib/debug/lib/modules/$(uname -r)/vmlinux 2>/dev/null || true
~~~

Distribution paths vary. file output is heuristic and can identify the PE wrapper or Linux boot sector metadata without explaining every internal region.

---

## 2. Compression is a boot-time tradeoff

Kernel compression reduces storage reads and boot-medium footprint at the cost of decompression work and temporary memory. The fastest choice depends on CPU speed, storage speed, image size, and algorithm.

Kernel configuration supports algorithms such as gzip, bzip2, LZMA, xz, LZO, LZ4, and Zstandard depending on architecture and version. Their tradeoffs differ:

- high compression can reduce slow I/O but require more CPU or decompressor memory;
- fast decompression can win on modern storage even with a larger file;
- embedded systems may prioritize flash capacity;
- decompressor code must itself fit and run in the constrained early environment.

The kernel build includes the matching decompressor. The boot loader normally does not need to know the payload algorithm; it loads the bootable image and enters its code. This is different from initramfs compression, which the kernel detects and unpacks later.

---

## 3. Entry is a contract, not a clean slate

The first Linux instruction inherits state from the chosen entry path. Under the x86 64-bit boot protocol, the loader must provide long mode, paging, defined segment descriptors, disabled interrupts, identity mappings for required ranges, and a pointer to boot parameters in the specified register. The 32-bit path has a different contract. The EFI stub begins as a firmware-loaded PE application and builds a compatible kernel handoff.

Early code verifies what it can. It cannot make every arbitrary loader state safe. If a register, mapping, or buffer violates the protocol, failure can occur before structured logging.

The early code must also assume very little about firmware-initialized devices. A framebuffer description may exist, but no Linux graphics driver is active. Serial hardware may be usable through an early console only if platform information and code support it. Interrupts are typically disabled because Linux has not installed its handlers.

---

## 4. Real mode setup and historical compatibility

The x86 image retains a setup portion shaped by decades of compatibility. Traditional BIOS loaders entered 16-bit setup code, which queried legacy services, prepared parameter structures, enabled architectural features, and transitioned toward protected mode.

Modern 32-bit, 64-bit, and EFI entry protocols can bypass portions of the old path because the loader already establishes richer processor state. The image still carries header data at historically defined offsets so loaders can discover protocol capabilities.

This layered history explains apparently strange facts:

- a modern 64-bit kernel image contains structures rooted in a boot-sector convention;
- some fields are obsolete but remain for compatibility;
- protocol versions add fields rather than replacing the format wholesale;
- different entry points converge later in architecture startup.

Do not interpret retained compatibility code as evidence that every UEFI boot drops the CPU into old BIOS real mode first. The active path depends on the entry protocol.

---

## 5. The compressed-kernel environment

The compressed kernel is a small program with an unusually strict job. In the x86 source tree, relevant code lives under arch/x86/boot/compressed. It includes startup assembly, memory-layout logic, decompressor selection, relocation handling, page-table construction, EFI support used in this path, and output helpers.

Its environment needs:

- a stack;
- access to boot parameters;
- temporary page tables as appropriate;
- an output destination large enough for the decompressed kernel;
- scratch or heap space required by the algorithm;
- knowledge of reserved and unsafe ranges;
- an eventual entry address for the decompressed image.

Normal kmalloc and page allocation do not exist. The decompressor uses statically arranged, loader-provided, or specially selected memory. This is the first of many kernel bootstrapping problems: create a facility without using the facility being created.

---

## 6. Choosing a destination

The loader may place a relocatable compressed image at an allowed address, but the decompressed kernel needs a destination satisfying alignment, size, addressability, and overlap constraints.

The calculation considers:

- preferred physical address advertised by the image;
- kernel alignment requirements;
- initialization size, which can exceed compressed file size;
- the compressed input range;
- boot parameters and command line;
- initramfs range;
- firmware-reserved memory;
- architecture limits;
- KASLR choice;
- minimum and maximum supported addresses.

An output range cannot overwrite compressed input before it has been consumed unless the algorithm and copy strategy explicitly permit the overlap. It cannot overwrite initramfs or firmware data that later code needs.

The loader advertises and reserves objects, while early kernel code makes its own placement decisions within the contract. Both sides must agree about occupied memory.

---

## 7. Kernel Address Space Layout Randomization

KASLR makes kernel addresses less predictable. On x86, boot-time logic can randomize physical placement and the virtual base within supported constraints. Entropy may come from firmware, hardware instructions, boot-loader seeds, timing, or other architecture sources, mixed according to implementation.

Randomization occurs early because the destination and mappings must be chosen before the main image runs. Later moving a fully initialized kernel would be far more complicated.

KASLR has limits:

- alignment and address-space layout reduce possible positions;
- reserved ranges remove candidates;
- weak or repeated early entropy can reduce unpredictability;
- information leaks can reveal the selected layout;
- randomization is defense in depth, not memory safety.

The nokaslr command-line option disables it for diagnostics. That can make addresses reproducible for debugging but weakens a security mitigation and should not become a casual production default.

Kernel logs may report offsets or KASLR state depending on policy and log permissions. Modern systems restrict address exposure to unprivileged users.

---

## 8. Decompression itself

The decompressor reads the payload stream, writes the uncompressed image to the chosen output range, and checks format-specific integrity where provided. Failure conditions include:

- corrupt or truncated payload;
- insufficient output or scratch space;
- unsupported CPU capability;
- invalid image metadata;
- unsafe overlap;
- relocation outside supported ranges;
- decompression-library error.

The familiar message "Decompressing Linux" may be printed on some paths, but not every platform or console configuration shows it. Lack of the text does not prove decompression did not occur.

Compression integrity checks detect accidental corruption; they are not a trust decision. An attacker can create a correctly compressed malicious image. Authentication belongs to Secure Boot or another verified mechanism.

### Why the main image can be larger than expected

The decompressed kernel contains executable text, read-only data, writable data, symbol-related structures selected by configuration, init-only sections, and alignment gaps. BSS-like zero-initialized regions may contribute to memory footprint without occupying equivalent compressed bytes. The protocol's init_size helps loaders reserve enough space beyond the file length.

---

## 9. Relocation

A linked image assumes relationships among addresses. If the kernel executes at a different physical or virtual base, absolute references represented by relocation entries must be adjusted.

Position-independent techniques reduce required relocations, but the x86 kernel still has relocation machinery for configured randomized or nonpreferred placement. Build tools preserve the information early code needs and discard general ELF complexity not required in the bootable payload.

Relocation is not the same as decompression:

- decompression transforms encoded bytes into the original image;
- relocation adjusts address-dependent contents for the chosen location;
- page-table construction maps virtual addresses to physical locations;
- control transfer begins executing at the adjusted entry.

Collapsing all four into "the kernel loads" hides distinct failure and security mechanisms.

---

## 10. Temporary page tables

The main 64-bit kernel expects a virtual-memory environment, but final kernel page tables are not yet constructed. Early startup creates temporary mappings sufficient to execute, access boot data, reach the decompressed output, and transition into normal architecture code.

Important mapping ideas include:

- **identity mapping:** virtual address equals physical address for a range;
- **kernel mapping:** linked kernel virtual addresses map to selected physical pages;
- **direct map:** later, a broad kernel virtual region maps physical memory for efficient access;
- **fixmap:** controlled fixed virtual slots for special early or runtime access.

Processor features such as five-level paging, encryption address bits, and page-table format capabilities affect construction. The bootstrap page tables are replaced or expanded as Linux learns the full memory map and enables more protections.

Page-table permissions also evolve. Early broad mappings may be convenient for bootstrap; later code enforces read-only kernel text, non-executable data, and other hardening.

---

## 11. CPU capability checks

A distribution kernel may require a minimum processor feature set. Early code checks capabilities needed to execute safely: long mode for a 64-bit kernel, page-size and paging features, control-register behavior, and build-specific requirements.

Virtual machines can expose a CPU model missing features present on the host. Migrating a guest to a host with an incompatible CPU baseline can cause early failure. Conversely, selecting host-passthrough CPU features can improve capability while making migration harder.

Microcode may change feature behavior or mitigation availability. Firmware and early microcode loading therefore interact with later capability checks.

An "unsupported CPU" message is stronger evidence than a silent reset, but an early console must exist to display it. For headless systems, serial capture and hypervisor logs are invaluable.

---

## 12. Early exception handling

Decompression and page-table work can fault. Normal kernel exception handling is not ready, so early paths install minimal handlers or use tightly controlled code designed to avoid recoverable complexity.

On x86, a fault during mode transition or page-table setup can escalate if the required descriptor tables and handlers are invalid. A double fault without a working handler can lead to a triple fault and processor reset. To an observer, the machine may appear to reboot immediately after loader selection with no panic text.

This is why "instant reboot" does not necessarily mean firmware rejected the image. Disable automatic firmware retry where possible, use an emulator with instruction logging for development, enable early console, and compare a known-good image under the same loader state.

---

## 13. Early console

Normal console drivers are initialized later. Early-print mechanisms provide limited output before the driver model and interrupts are ready.

The earlycon command-line parameter can select a supported early console from firmware or explicit device information. On some architectures a device tree stdout-path or ACPI SPCR table describes the serial console. x86 debug ports, hypervisor consoles, and EFI framebuffer output offer other possibilities.

An early console must be safe for the stage. It commonly uses polling rather than interrupts and supports only basic output. Incorrect device address or clock information can produce garbage or no output.

Example forms are platform-specific:

~~~text
earlycon
earlycon=uart8250,mmio,<UART-MMIO-ADDRESS>,115200
console=ttyS0,115200n8
~~~

Do not copy a physical address from another machine. Discover it from platform documentation or firmware description.

The normal console= parameter and earlycon can refer to related devices but serve different initialization periods.

---

## 14. Transition to the main kernel

After successful decompression and relocation, early code prepares arguments and jumps to the decompressed kernel entry. Architecture assembly establishes the kernel's linked virtual context, processor-local foundations, descriptor tables, initial page tables, and other machine state.

On x86-64, startup paths eventually reach architecture C initialization and then start_kernel() in init/main.c. The exact function sequence evolves, and virtualization or EFI paths can join at different points.

This transition is not yet userspace. There is still one executing boot CPU, and many facilities are absent or incomplete. But the code now executing is the main linked kernel, not the small compressed wrapper.

### Freeing the decompressor

Memory occupied only by temporary boot code and data can later be reclaimed once no references remain. Init-only sections throughout the kernel are also freed after their initialization work completes. This is why functions and data marked __init must not be called or referenced later.

---

## 15. Comparing x86 with arm64

The broad problem is shared, but the mechanism differs.

An arm64 boot loader places the kernel Image at an appropriately aligned location, provides a Device Tree pointer in the specified register, configures caches and exception level according to the booting document, and enters the image header's code. Firmware interfaces such as UEFI can provide standardized entry there as well.

The arm64 kernel Image can be uncompressed; loaders or packaging may use external compression. The x86 bzImage self-decompressor and setup-header history should not be assumed.

Shared questions remain:

- What exact image format is loaded?
- Who decompresses it?
- Which CPU mode or exception level is required?
- Which register points to platform data?
- Where may image and DTB reside?
- Which memory is reserved?
- How are caches and MMU configured?

Architecture documentation answers them.

---

## 16. Source-reading map

For x86, start with:

- arch/x86/boot/header.S for image and protocol header relationships;
- arch/x86/boot/main.c for traditional setup flow;
- arch/x86/boot/compressed/head_64.S for compressed 64-bit entry;
- arch/x86/boot/compressed/misc.c for decompression orchestration;
- arch/x86/boot/compressed/kaslr.c for placement randomization;
- arch/x86/kernel/head_64.S for main-kernel architecture entry;
- arch/x86/kernel/setup.c for later architecture setup;
- init/main.c for start_kernel().

Names and exact paths can change. Use tags matching the target kernel. Read the linker scripts alongside code because they define section placement and symbols that assembly uses.

A productive technique is to choose one symbol and trace forward, rather than reading directories alphabetically. Use source cross-reference tools and objdump output for the built image.

---

## 17. Debugging with QEMU and GDB

A disposable debug build makes early entry visible.

Useful build choices include debug symbols, a known compression method, serial output, and disabled address randomization for the first session. QEMU can pause the virtual CPU at startup and expose a GDB server. The debugger uses vmlinux symbols while the guest boots the corresponding bzImage.

The difficult part is address translation before the final virtual base is active. Breakpoints at link-time symbols may need adjustment for physical placement and KASLR. Start with nokaslr and a documented QEMU machine. Once the fixed path is understood, repeat with randomization and derive the selected offset from logs or early state.

Do not debug an arbitrary distribution kernel using an unrelated vmlinux. Configuration, compiler, build ID, and exact binary must match.

Even without single-stepping, capture serial output:

~~~text
console=ttyS0 earlycon=uart8250,io,0x3f8,115200 ignore_loglevel
~~~

This example fits a conventional emulated PC UART, not every physical platform.

---

## 18. Failure patterns

### "Invalid or corrupt kernel image"

The loader may be using the wrong command for the image format, the file may be truncated, the architecture may not match, or header checks may fail. Verify package checksum and file identity before changing memory parameters.

### Decompression error

Compare the installed file with the package, check storage errors, and confirm the loader read the intended path. If multiple kernels fail identically, investigate memory stability and loader placement.

### Reboot with no kernel log

Use a known-good entry, early console, firmware watchdog settings, and a virtual reproduction. Consider protocol state, CPU requirements, page tables, and early faults—not only initramfs.

### Works with nokaslr

This comparison points toward placement, relocation, memory-map, or platform interactions, but it does not by itself identify the bug. Capture the failing map and kernel version. Keep KASLR enabled in normal operation after diagnosis.

### Works after cold power-off only

The decompressor may be an innocent victim of stale platform or device state. Compare firmware and microcode, memory training, kexec versus firmware boot, and warm-reset behavior.

---

## 19. Practical observations after a successful boot

Later Linux preserves evidence:

~~~sh
cat /proc/cmdline
journalctl -b -k | head -n 120
grep -Ei 'KASLR|Kernel Offset|Memory:|NX|SMEP|SMAP' /var/log/dmesg 2>/dev/null
cat /sys/kernel/notes 2>/dev/null | wc -c
~~~

Kernel logs may omit or restrict sensitive addresses. Build configuration is often available through /proc/config.gz or /boot/config-version:

~~~sh
zgrep -E 'CONFIG_(RELOCATABLE|RANDOMIZE_BASE|KALLSYMS|EFI_STUB|KERNEL_)' /proc/config.gz 2>/dev/null
grep -E 'CONFIG_(RELOCATABLE|RANDOMIZE_BASE|EFI_STUB)=' /boot/config-$(uname -r) 2>/dev/null
~~~

Configuration says which features were compiled; it does not prove a runtime branch was used. Pair it with logs and entry-path evidence.

---

## 20. Mechanism summary

The installed boot image is a purpose-built executable container, not merely compressed vmlinux. A loader satisfies an architecture contract and enters a small environment. That code validates CPU state, chooses nonconflicting memory, constructs temporary mappings, optionally randomizes placement, decompresses the payload, applies relocation, and transfers into the main kernel's architecture startup.

The constraints explain protocol metadata: alignment, preferred address, initialization size, relocatability, command-line limits, and initramfs placement are necessary because no normal allocator can fix a bad layout afterward.

You should now be able to distinguish file loading, payload decompression, relocation, virtual mapping, and control transfer. You should also understand why failures here can be nearly silent and why early consoles, matching symbols, virtual machines, and fixed-address diagnostic runs are so valuable.

The next chapter begins where this one ends. The main kernel is executing, but almost every subsystem is still absent. We will follow the command line through parsing and use start_kernel() as a map of dependency-ordered initialization.

## 21. Annotated memory-layout example

Assume a hypothetical x86-64 loader has placed four objects in usable RAM:

~~~text
0x0009_0000 .. 0x0009_FFFF   setup and low-memory data
0x0200_0000 .. 0x02BF_FFFF   compressed kernel image
0x0300_0000 .. 0x0300_FFFF   command line and boot parameters
0x0800_0000 .. 0x0BFF_FFFF   initramfs
~~~

These addresses are illustrative, not recommendations. The image header says the kernel is relocatable, requires a large alignment, and needs substantially more memory for initialization than the compressed file occupies. Firmware also reports reserved ranges for runtime code and devices.

Early placement code first creates an exclusion map. It cannot select an output range merely because the first page is marked conventional RAM. The complete output interval must be usable, aligned, addressable, and disjoint from all live input. If physical KASLR is enabled, candidate slots are generated within an allowed window and invalid slots are removed. Entropy selects among what remains.

Suppose the preferred output at 0x0100_0000 would extend into the compressed input. The kernel chooses an aligned slot at 0x1000_0000 instead. Temporary page tables identity-map code currently executing and map the output so the decompressor can write it. The compressed stream remains readable until its final bytes have been consumed. The initramfs stays untouched because its physical address and size are still needed after the main kernel starts.

After decompression, relocation entries adjust absolute locations for the selected base. Early assembly transfers to an address inside the new output. That code installs mappings appropriate for the kernel's linked virtual address and continues architecture setup. Much later, when boot parameters and initramfs have been copied, reserved, or consumed, their temporary ranges can be released or repurposed according to ownership.

Now change one fact at a time:

- If firmware falsely labels runtime memory as free, random placement can overwrite firmware state.
- If the loader reports the initramfs length incorrectly, decompression output may overlap bytes the kernel later tries to unpack.
- If init_size is ignored, a loader can place another object immediately after the compressed file even though the kernel needs that space while expanding.
- If the 64-bit entry lacks the promised identity mapping, the first memory reference can fault before output works.
- If entropy repeats, KASLR may choose the same valid slot across boots without causing functional failure but with weaker security.

This example shows why a boot failure can depend on RAM size, firmware version, or initramfs size even when the kernel file is unchanged. Those inputs reshape the early exclusion map.

## 22. Reproducible comparison experiment

Use a virtual machine with a serial console and retain a snapshot. Install two identical loader entries differing only in command line:

~~~text
entry A: console=ttyS0,115200 earlycon ignore_loglevel nokaslr
entry B: console=ttyS0,115200 earlycon ignore_loglevel
~~~

Boot each several times. Save complete serial output, /proc/cmdline, /proc/iomem, and the earliest kernel messages. With address exposure permitted in the lab, compare reported kernel offsets and memory reservations. Entry A should give a stable placement compatible with the build; entry B may vary.

Next increase initramfs size by adding an inert file to a copied archive and create a third entry. The added file should not change kernel semantics, but it changes one occupied physical range and can influence loader placement. The purpose is not performance benchmarking. It is to show that early memory layout is a negotiated result involving all loaded artifacts.

Finally, deliberately truncate a copy of the kernel image and confirm that the loader or decompressor rejects it. Restore the snapshot rather than repairing the broken copy in place. Label which component emitted the error. A loader header error and a decompressor payload error prove different amounts of the path executed.

In every run, keep firmware, virtual hardware, loader version, CPU model, and storage image constant. Early-boot experiments are meaningful only when the changed variable is known.

---

## Primary references

### A note on symbols and addresses

An early-boot address from a panic, emulator trace, or disassembly is meaningful only with the matching binary and placement. The symbolic vmlinux expresses link-time virtual addresses. The bootable image may contain transformed sections, and KASLR adds a runtime offset. Physical entry and decompression addresses are not necessarily the final virtual addresses printed by a later oops. Record the build ID, exact vmlinux, command line, chosen kernel offset where available, and whether the trace occurred before or after relocation. Tools such as addr2line and GDB produce confident but wrong answers when given a near-matching distribution kernel. Reproducible builds and archived debug packages turn otherwise opaque early traces into source locations.

Keep the raw address log as well as the symbolized version. If later analysis discovers a different KASLR offset or build artifact, the original can be reprocessed; a pasted function name cannot.

This rule also applies to emulator traces: save machine type and CPU model beside the addresses.

- Linux/x86 boot protocol: https://docs.kernel.org/arch/x86/boot.html
- x86 compressed boot source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/boot/compressed
- x86 boot source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/boot
- Kernel build documentation: https://docs.kernel.org/kbuild/
- Kernel parameters: https://docs.kernel.org/admin-guide/kernel-parameters.html
- arm64 booting requirements: https://docs.kernel.org/arch/arm64/booting.html
- RISC-V boot requirements: https://docs.kernel.org/arch/riscv/boot.html
- Kernel debugger documentation: https://docs.kernel.org/process/debugging/kgdb.html
- KASLR implementation notes and source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/boot/compressed/kaslr.c
