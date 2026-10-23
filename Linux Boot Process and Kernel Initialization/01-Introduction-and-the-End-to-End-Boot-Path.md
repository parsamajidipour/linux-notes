# Introduction and the End-to-End Boot Path

Boot is the process of creating assumptions.

At power-on, the machine has hardware but almost none of the abstractions an operating system user recognizes. There is no process, file, socket, mount point, driver model, page allocator, or system log. A processor begins in an architecture-defined reset state and executes code chosen by the platform. Several stages then cooperate to construct a usable system. Each stage receives a limited environment, adds capabilities, and hands control to a successor.

People often compress the whole path into four words: firmware, boot loader, kernel, init. That summary is useful as a map, but it is too coarse for engineering. A useful explanation must answer four questions at every boundary:

1. Which component currently controls the CPU?
2. Which memory, device, and execution services are available?
3. What information must be passed to the next component?
4. What evidence would prove that the handoff succeeded?

This chapter builds that boundary model before later chapters open each stage in detail. The goal is not to memorize a longer list. It is to learn how to place an observation in the correct part of the timeline. Once you can do that, a blank screen, a GRUB prompt, an early kernel panic, an initramfs emergency shell, and a failed userspace service stop looking like versions of the same vague problem called "the machine does not boot."

---

## 1. What counts as boot?

There is no single universally correct finishing line for boot.

A firmware engineer may say boot completed when control left firmware. A kernel developer may measure until the kernel executes init. A distribution may report boot complete when the service manager reaches a target. A desktop benchmark may wait until a graphical session is responsive. A server operator may care about the first successful health check. These measurements describe different intervals and should not be compared without naming their boundaries.

For this series, the main path begins at processor reset and ends when the kernel successfully executes the first real userspace program as PID 1. That scope contains several nested phases:

- **Platform initialization:** firmware makes the hardware usable enough to load another executable.
- **Boot selection:** firmware policy chooses a boot option and starts a boot application.
- **Kernel loading:** a loader places the kernel and supporting data in memory and enters Linux through an architecture-defined contract.
- **Early kernel execution:** small entry code establishes the environment required to run the main kernel.
- **Kernel initialization:** Linux creates its core subsystems and discovers enough hardware to run userspace.
- **Early userspace:** an optional initramfs environment prepares the storage and policy needed for the real root.
- **Root and init handoff:** Linux or early userspace activates the final root filesystem and executes PID 1.

The rest of system startup happens after this series. PID 1 may mount additional filesystems, start logging, configure networking, launch databases, present login prompts, or start a desktop. Those tasks are important, but they are userspace policy. Separating them from kernel initialization prevents a common diagnostic mistake: treating a slow web service as if the firmware or kernel were still booting.

### A practical definition

When someone reports a boot problem, replace "Does it boot?" with observable milestones:

- Does firmware display its setup interface or logo?
- Does firmware list the intended boot device?
- Does a boot menu or boot application appear?
- Does the kernel emit any message?
- Does an initramfs prompt appear?
- Is the real root filesystem found and mounted?
- Does the kernel execute PID 1?
- Does PID 1 reach the intended operating state?

The first missing milestone gives a much smaller search area than the original report.

---

## 2. Boot begins in more than one way

"The machine starts" hides several entry conditions. They share later stages but do not establish identical initial state.

### 2.1 Cold boot

A cold boot begins after power was absent or the platform entered a mechanically off state. Power rails stabilize, reset logic holds components in known states, and processors begin at their reset vectors. Firmware may perform full memory training and extensive device initialization because it cannot rely on state from a previous execution.

Cold boot timing is affected by hardware operations that Linux does not control: DRAM training, controller firmware, storage-device self-initialization, network boot negotiation, and platform security checks. If a machine spends thirty seconds before any vendor logo or console output, changing systemd unit ordering cannot recover those seconds.

### 2.2 Warm reset

A warm reset restarts execution without necessarily removing power from every component. Firmware may choose faster paths because some hardware state is retained or already characterized. The exact behavior is platform-specific. A bug that appears only after reboot but not after full power removal can involve stale device, firmware, or controller state. "Turn it off and remove power" is not superstition in that case; it changes the starting condition.

### 2.3 Suspend and resume

Resume from suspend-to-RAM is not a normal boot. Memory contents remain present, and execution resumes inside a previously running kernel after platform and device state is restored. Firmware may participate, but no boot loader needs to load a new kernel image. Logs may contain resume-related initialization that looks like boot, yet the process tree and kernel instance are the old ones.

Resume from hibernation does begin with an ordinary boot path far enough to load a kernel and discover the hibernation image. That kernel then restores a saved memory image and transfers into the suspended kernel state. The visible path is therefore a hybrid: firmware and loader run normally, an early kernel starts, but the final running system comes from the restored image.

### 2.4 kexec

The kexec system call allows a running Linux kernel to load another kernel into memory and jump to it without returning through normal firmware initialization. Crash kernels use this mechanism so that a small reserved kernel can start after a production kernel panics and capture memory for analysis. Administrators may also use kexec for faster planned reboots.

Because firmware is bypassed, the next kernel depends on the first kernel to prepare a valid handoff. Hardware may not be reset as completely as during a power cycle. A kernel that starts reliably through firmware but fails through kexec, or the reverse, is evidence about the handoff path rather than generic proof that the image is corrupt.

### 2.5 Virtual-machine start

A virtual machine still has a reset model, firmware, CPUs, memory maps, buses, and boot devices, but the hypervisor constructs much of that platform in software. OVMF can provide UEFI. SeaBIOS can provide legacy BIOS. Virtio devices replace or supplement emulated physical devices. A cloud provider may combine firmware policy with metadata-driven disk and network configuration.

Virtualization makes experimentation safer and observations clearer, but it does not make the boot stages imaginary. The guest kernel sees a platform contract. The fact that the PCI function or interrupt controller is synthesized by a hypervisor changes implementation and performance, not the need for discovery and initialization.

---

## 3. The actors in the boot chain

It helps to treat every stage as an independent program with a particular trust and execution environment.

### 3.1 Platform firmware

Firmware is the first substantial software layer. It performs platform initialization, builds description tables, exposes pre-OS services, applies boot policy, and selects a boot target. On conventional PCs this means legacy BIOS or UEFI firmware, though real implementations contain many internal phases and auxiliary controllers.

Firmware is not a Linux component. It may start Windows, Linux, a network boot program, a diagnostic utility, or another firmware application. It can contain vendor bugs that the operating system later works around. It can also enforce platform security policy before any Linux code exists.

### 3.2 Boot manager and boot application

Under UEFI, the firmware Boot Manager evaluates configured options and starts an EFI application. That application might be GRUB, systemd-boot, shim, a vendor recovery tool, a network loader, or a Linux kernel containing the EFI stub. The UEFI specification gives "Boot Manager" a precise firmware meaning. Desktop conversations often use "boot manager" and "boot loader" interchangeably, but keeping them separate makes UEFI paths easier to reason about.

Under legacy BIOS, firmware traditionally selects a device, reads a conventional boot sector into memory, and jumps to it. Because a sector is tiny, loader code is commonly staged: the first code locates and loads a larger second environment that understands filesystems and configuration.

### 3.3 Boot loader

A boot loader's essential job is to prepare the next executable and transfer control according to its entry contract. A sophisticated loader may read many filesystems, offer a menu and command shell, chainload other programs, verify signatures, retrieve images from a network, edit command lines, and load modules. Those features support the job but do not change the core handoff.

For Linux, the loader normally provides:

- the kernel image at an allowed memory location;
- an optional initramfs or sequence of archives;
- a kernel command line;
- a platform memory map and architecture-specific boot parameters;
- references to firmware tables or a Device Tree;
- video, console, random-seed, and other optional information;
- processor state required by the chosen boot protocol.

It then jumps to the kernel entry point. It does not return later to manage processes or drivers.

### 3.4 Kernel entry and decompressor

The on-disk image is designed to be loadable before Linux exists. On x86, it includes setup and entry code around a compressed kernel payload. The early code cannot assume the normal kernel allocator, final page tables, exception handling, or console infrastructure. It validates the entry environment, creates temporary essentials, chooses placement, decompresses the main image, and jumps onward.

This stage explains why the file commonly named vmlinuz is not equivalent to the uncompressed, symbol-rich vmlinux used for debugging. It also explains why an error can occur after a loader says it loaded the file but before familiar kernel messages appear.

### 3.5 The initialized kernel

The main kernel begins with architecture-specific assembly and C code, then reaches start_kernel(), the conventional center of architecture-independent initialization. "Initialized kernel" is not a binary state. Linux gradually establishes facilities in a strict dependency order.

For example, normal dynamic allocation cannot be used until memory topology is known and allocators exist. Interrupt-driven device operation cannot be relied on before interrupt controllers and handlers are ready. Secondary CPUs cannot safely run general work before per-CPU areas, scheduling structures, synchronization, and architecture state are prepared. Driver discovery depends on buses and core device infrastructure. Filesystem mounting depends on memory, block devices, filesystem drivers, and often userspace policy.

### 3.6 Early userspace

Initramfs lets the kernel delegate complicated preparation to userspace. The kernel unpacks an archive into an in-memory root and, if an executable /init exists, runs it. That program can use device manager logic, load modules, assemble RAID, activate LVM, unlock encrypted storage, verify integrity, configure a network, or locate a remote root.

Early userspace is not the boot loader and is not a half-kernel. It consists of ordinary processes making system calls to the already running kernel. Its root happens to be temporary, and its purpose is to make the final root possible.

### 3.7 PID 1

The first process has special kernel-visible responsibilities. Orphaned processes are reparented toward an appropriate reaper, and PID 1 has unusual signal semantics. More importantly for boot, successful execution of the final init proves that the kernel found a usable root and a runnable userspace image.

If the specified init cannot be executed and no fallback path works, the kernel cannot continue into normal userspace and panics. If init executes but later reports a failed mount or service, the boundary has moved: the kernel-to-init handoff succeeded, and diagnosis belongs to userspace startup.

---

## 4. The end-to-end timeline

The exact call graph varies, but this high-level timeline is robust.

### Stage A: reset and platform establishment

The bootstrap processor begins at a defined reset location. Firmware code establishes a stack or temporary execution environment, performs processor and chipset setup, initializes memory, and discovers enough platform resources to continue. Firmware may authenticate components, initialize embedded controller relationships, and run option ROM or UEFI driver code.

Evidence that this stage progressed includes firmware diagnostics, POST codes, serial output, a vendor logo, or access to setup. Absence of display output does not prove absence of execution; display initialization is itself a firmware task. Server boards often expose a baseboard management controller or debug code that can reveal earlier progress.

### Stage B: boot target selection

Firmware evaluates its configured policy. Legacy BIOS selects a boot device according to its order and looks for a conventional boot signature. UEFI processes BootNext, BootOrder, active Boot#### entries, removable-media fallback paths, and possibly network options. Secure Boot policy may reject an otherwise readable EFI image.

At this boundary, a disk can be electrically present and visible in setup yet still not provide a usable boot target. "Disk detected" and "boot application located and accepted" are different claims.

### Stage C: loader execution

The selected loader initializes its own modules and user interface, locates configuration, chooses an entry, and reads the kernel-related files. With GRUB, the path may involve an embedded core image, modules, grub.cfg, filesystem drivers, and commands such as linux and initrd. With systemd-boot, firmware already supplies UEFI filesystem access and the loader reads simple entry descriptions from the ESP or extended boot loader partition. With a Unified Kernel Image, one EFI executable can contain the stub, kernel, initrd, command line, and metadata.

Evidence includes a menu, loader console, successful file-load messages, or a loader-specific error. A loader rescue prompt strongly suggests firmware already loaded and ran at least part of the loader.

### Stage D: kernel handoff

The loader assembles boot parameters and enters the kernel. This is a one-way ownership transition in normal boot. Under UEFI, memory descriptors and the ExitBootServices() boundary are particularly important because firmware boot-time allocations and services stop being generally available after the operating system takes control.

The first kernel code runs under constraints described by the architecture protocol. Registers have required meanings. Interrupt state, CPU mode, page-table state, alignment, and parameter locations are contractual. A loader that violates the contract may fail before any high-level diagnostic is possible.

### Stage E: decompression and architecture setup

Early code creates the minimal safe environment, decompresses the kernel, handles relocation and address randomization where configured, builds initial mappings, and enters the main architecture startup. The kernel imports boot data rather than rediscovering everything from scratch. Firmware tables and memory maps are inputs, though Linux validates and sometimes works around them.

An early console or architecture debug port can make this phase observable. Without one, a reset or blank screen here can be difficult to distinguish from a loader failure.

### Stage F: core kernel initialization

The kernel establishes memory management, traps and interrupts, timekeeping, scheduling, RCU, VFS foundations, caches, work queues, device infrastructure, and built-in subsystems. The order is not merely aesthetic. Each initialization step can use only facilities already established.

Kernel log lines gradually become richer. Early printk output may use a primitive console. Later output can include timestamps, CPU identities, device names, and asynchronous probe activity because those facilities now exist.

### Stage G: device discovery and root preparation

Built-in driver initialization and bus enumeration discover hardware. Modules needed before the real root are loaded from initramfs. Storage stacks may be layered: a PCI function exposes an NVMe controller; the controller exposes a namespace block device; a partition parser exposes a partition; device mapper exposes decrypted storage; LVM exposes a logical volume; a filesystem driver interprets the final bytes.

The question "Is the root disk visible?" must therefore name a layer. Seeing the PCI controller does not guarantee the namespace exists. Seeing the raw partition does not guarantee the encrypted mapping is open. Seeing the logical volume does not guarantee the filesystem can mount.

### Stage H: early userspace

The kernel unpacks initramfs and runs /init. Distribution logic parses relevant command-line parameters, creates device nodes or waits for uevents, loads modules, assembles storage, and mounts the intended root at a staging path. Failures often drop to an emergency shell because userspace now has enough tools for diagnosis.

An initramfs shell is good news in one sense: firmware selection, loader execution, kernel handoff, decompression, major kernel initialization, and execution of early userspace have all worked. The remaining problem is narrower.

### Stage I: root switch and init

Early userspace prepares virtual filesystems and state, changes the root, and executes the final init. The exact tool may be switch_root or a carefully managed pivot_root sequence. The goal is not just to change a path; the old initramfs root should become unreachable so its memory can be reclaimed.

The service manager now applies userspace policy. The kernel remains the running kernel; it does not hand the hardware away. "Handoff" means it begins scheduling the first userspace control process and responding to its system calls.

---

## 5. A boundary table for diagnosis

| Last thing you can prove | Likely next boundary | Useful evidence |
|---|---|---|
| Power is present | reset to firmware execution | board diagnostics, BMC, POST codes |
| Firmware UI works | firmware to selected boot application | boot entries, device paths, Secure Boot status |
| GRUB or another loader appears | loader to kernel | selected entry, file paths, loader console, signatures |
| "Loading Linux" appears | kernel early entry | early console, architecture parameters, image integrity |
| Kernel banner appears | core initialization | full dmesg, panic text, command line |
| Storage controller appears | layered root discovery | sysfs, /dev, uevents, module list |
| Initramfs shell appears | early userspace to real root | /proc/cmdline, blkid, cryptsetup, lvm, mount errors |
| Real-root emergency mode appears | PID 1 userspace graph | journal, failed units, fstab, generators |
| Login appears but application fails | service/application startup | service logs and application health checks |

The table is deliberately evidence-first. A firmware logo can remain displayed while later software runs, so the current pixels do not always identify the current owner. A reliable milestone is something only a later stage could have produced: a kernel banner proves the kernel ran; an initramfs shell proves a process was executed; a systemd unit name proves PID 1 started far enough to parse units.

---

## 6. Worked example: wrong root identifier

Suppose a machine reaches the loader, prints normal kernel messages, and then drops into an initramfs shell saying that a UUID does not exist.

A command-first response might reinstall GRUB. The boundary model argues against that. GRUB successfully loaded the kernel and initramfs. The kernel decompressed, initialized enough memory and devices to run a shell, and executed /init. The failing contract is between configuration presented to early userspace and the block-device graph it can discover.

Start with evidence:

~~~sh
cat /proc/cmdline
ls -l /dev/disk/by-uuid 2>/dev/null
blkid
cat /proc/partitions
~~~

Three distinct causes can produce the same headline:

1. The command line contains an old filesystem UUID after the root filesystem was recreated.
2. The UUID is correct, but the controller driver or module is absent, so the device never appears.
3. The UUID identifies a filesystem inside an encrypted or LVM layer that early userspace failed to activate.

Comparing identifiers answers the first. Looking for the controller and module answers the second. Inspecting device-mapper, cryptsetup, and LVM state answers the third. Reinstalling the boot loader would change an earlier stage that evidence already shows working.

---

## 7. Worked example: black screen after a kernel update

A black screen is not a stage. It is an observation about one output device.

If the machine responds to network connections, the system booted and display startup failed. If Caps Lock state changes but no network appears, the kernel may be alive with a console or graphics problem. If the machine immediately resets, early kernel failure or watchdog policy may be involved. If the firmware logo remains forever, later software may still have reused or failed to replace the framebuffer.

Improve observability before changing components:

- remove quiet and graphical splash parameters;
- increase kernel log level;
- select a text console explicitly;
- enable a serial console in a virtual machine or server;
- keep the previous kernel entry available;
- inspect the previous boot journal after recovering.

Typical temporary command-line additions include:

~~~text
loglevel=7 ignore_loglevel systemd.show_status=1
~~~

For architecture-specific early failures, earlycon may help when supported and correctly configured. It is not a magic generic value; the early console must match a device the kernel can access before normal console drivers initialize.

Booting the previous kernel is a controlled comparison. If the same loader, command line, root storage, and userspace work with the old image, the changed kernel or its matching initramfs becomes a strong suspect. That still does not prove a kernel-code regression: the new initramfs may omit a module, or the loader entry may reference a missing file.

---

## 8. Time is stage-local

Boot performance discussions often subtract timestamps that do not share a clock.

Firmware may report time spent before ExitBootServices(). A boot loader may maintain its own clock. Kernel printk timestamps are normally relative to a kernel time base established during startup. systemd records userspace and kernel milestones using monotonic clocks available after Linux initializes them. Wall-clock time may jump after an RTC is read or network time synchronizes.

Commands such as systemd-analyze time can present firmware, loader, kernel, initrd, and userspace figures when the platform and boot path provide the necessary data. Treat those figures as measured intervals with defined boundaries, not as one universal stopwatch.

~~~sh
systemd-analyze time
systemd-analyze critical-chain
journalctl -b -o short-monotonic
dmesg --time-format=delta
~~~

The critical chain is not a list of every slow unit. It shows a dependency path that constrained a target. A service can take a long time in parallel without delaying the requested milestone. The same principle applies earlier: slow firmware network discovery matters only if boot policy waits for it.

For reproducible comparisons, keep the entry condition consistent. Cold boot, warm reboot, cached storage, virtual-machine snapshot restore, and hibernation resume are different experiments.

---

## 9. Safe inventory experiment

The following lab records the path without changing it.

### Step 1: identify architecture and kernel

~~~sh
uname -a
uname -m
cat /etc/os-release
~~~

Do not assume an x86-64 laptop and an arm64 board have the same entry protocol merely because both run the same distribution.

### Step 2: identify firmware mode

~~~sh
if test -d /sys/firmware/efi; then
    echo "Kernel reports an EFI boot environment"
else
    echo "No EFI runtime directory is exposed"
fi
~~~

The absence of the directory on an x86 machine commonly means legacy boot, but it can also reflect kernel configuration or restricted runtime exposure. Phrase the conclusion at the strength of the evidence.

### Step 3: record the kernel command line

~~~sh
cat /proc/cmdline
~~~

Separate parameters interpreted by the kernel from those interpreted by initramfs or PID 1. The same line is a transport shared by multiple consumers.

### Step 4: map storage layers

~~~sh
lsblk -o NAME,TYPE,FSTYPE,PKNAME,PARTUUID,UUID,MOUNTPOINTS
findmnt -no SOURCE,FSTYPE,OPTIONS /
~~~

On an encrypted LVM system, the source of / may be a logical volume, whose parent is a device-mapper mapping, whose parent is a partition, whose parent is a physical disk. That graph predicts what initramfs must construct.

### Step 5: inspect boot artifacts

~~~sh
ls -lh /boot
find /boot -maxdepth 3 -type f -printf '%p\n' 2>/dev/null | sort
~~~

Do not infer active files only from names. Loader configuration, EFI variables, and the current command line identify what was actually selected.

### Step 6: capture logs by boundary

~~~sh
journalctl -b -k
journalctl -b --no-pager
systemd-analyze time 2>/dev/null
~~~

Kernel-only output and the complete userspace journal answer different questions. Save both before rebooting away the evidence.

---

## 10. Boot in a virtual machine as a learning tool

A virtual machine is the best place to make boundaries visible.

Create a small guest with UEFI firmware and a serial console. Take a snapshot. Then perform one change at a time:

1. Change BootOrder so the disk is not selected.
2. Restore and rename the EFI boot application.
3. Restore and edit the loader entry to reference a missing kernel.
4. Restore and pass an invalid root identifier.
5. Restore and remove a required storage module from a copied initramfs.
6. Restore and set init= to a nonexistent path.

The point is not to collect failures. It is to correlate each failure with the last successful milestone and the message source. Firmware reports one class, the loader another, the kernel another, and early userspace another.

When the guest uses a serial console, save a complete transcript. Add annotations after the experiment, not during it:

~~~text
[firmware] UEFI selects the disk entry
[loader]   menu and selected kernel path
[kernel]   Linux version banner
[kernel]   storage controller probe
[initrd]   root lookup failure
[shell]    emergency prompt
~~~

This labeled transcript becomes a reusable diagnostic model for physical machines where visibility is poorer.

---

## 11. Boundaries are also trust transitions

Each handoff transfers not only execution but trust.

Firmware chooses what pre-OS code may run. Secure Boot can require cryptographic authentication of EFI images. A loader may verify the kernel or a Unified Kernel Image. The kernel can enforce module signatures and lockdown policy. Initramfs may verify root integrity or obtain decryption keys. PID 1 and later userspace establish service identities and access policy.

No single mechanism automatically protects the whole chain. Signed firmware-loaded code does not guarantee an unsigned configuration file was not modified unless the design authenticates that file. Disk encryption does not prove code integrity after an authorized unlock. Measured boot records values but does not necessarily block execution. Secure Boot blocks images according to policy but does not by itself keep filesystem contents secret.

Thinking in boundaries exposes the question: what exactly is authenticated before control crosses this edge, by which key, and what mutable inputs remain outside the authenticated object?

---

## 12. Boundaries are also ownership transitions

Before ExitBootServices(), a UEFI application can use firmware boot services. After the transition, the operating system owns general resource management and must not treat those services as available. Before the kernel scheduler runs, early code controls execution directly. After scheduling and interrupt infrastructure exist, work can become concurrent. Before root switch, initramfs owns the current userspace root. After a successful switch and exec, the real init owns userspace policy.

Many subtle bugs are ownership violations:

- a loader leaves data in memory it did not correctly reserve;
- the kernel reuses firmware memory before the final map transition is valid;
- a driver assumes interrupts or allocation contexts that are not ready;
- early userspace removes a device mapping before the new root no longer depends on it;
- two stages both try to manage the same console or storage state without a defined transfer.

The question "Who owns this resource now?" is often as powerful as "What failed?"

---

## 13. A disciplined troubleshooting loop

Use this loop for every boot incident.

### Observe

Capture exact text, timestamps, screen photographs, serial logs, boot variables, command line, and device inventory. "It freezes" is not evidence. Record the last new output and whether keyboard, network, watchdog, or disk activity continues.

### Locate

Place the observation in the earliest stage that could have produced it. A line prefixed by dracut is early userspace, even if it mentions the kernel. A GRUB error is loader-time, even if it mentions vmlinuz. A systemd mount-unit failure is post-init userspace, even if the underlying disk was discovered earlier.

### State the contract

Name what the next stage required: a valid EFI image, a readable kernel file, correct boot parameters, executable decompressed image, a discovered root device, a filesystem driver, an init executable.

### Compare

Change one variable or compare with a known-good path: previous boot entry, previous kernel, rescue image, alternate console, copied virtual disk, or cold boot instead of warm reset.

### Repair minimally

Repair the narrowest violated contract. Recreate one initramfs, correct one identifier, restore one EFI file, or change one parameter. Avoid reinstalling every boot component at once; doing so destroys evidence and can create additional failure modes.

### Verify the full chain

After repair, confirm not only that the immediate error disappeared but that the intended security and redundancy properties remain. A machine that boots only because verification was disabled is a diagnostic clue, not necessarily a finished repair.

---

## 14. Source map for the rest of the series

The following sources define the major contracts:

- The UEFI specification describes the Boot Manager, image loading, protocols, variables, memory services, and Secure Boot behavior.
- The Linux architecture boot documents describe required processor and parameter state at kernel entry.
- The EFI stub documentation explains how a Linux image can be entered as a PE/COFF application.
- Kernel source under arch/*/boot and arch/*/kernel implements early entry.
- init/main.c contains the architecture-independent initialization path centered on start_kernel().
- Kernel memory-management documentation and source explain memblock, page allocation, and boot-time topology.
- drivers/base and initcall definitions expose driver-core initialization and staged callbacks.
- usr/ and init/ contain initramfs creation, unpacking, and init execution paths.
- The GNU GRUB manual defines loader behavior and commands.
- systemd's bootup documentation explains the userspace graph after PID 1 begins.

Always read source that matches the relevant version. Internal sequences evolve even when the external handoff contract remains stable.

---

## 15. What you should now be able to say

After this chapter, "Linux boot" should no longer mean an undifferentiated progress bar.

You should be able to describe a chain in which firmware establishes the platform and chooses a boot application; a loader prepares a documented kernel entry environment; early kernel code turns a compressed image into a running kernel; the main kernel constructs its foundational subsystems in dependency order; built-in code and early userspace discover and assemble the storage graph; and the kernel finally executes PID 1 on the real root.

You should also be able to resist premature repair. A visible loader proves earlier firmware work. A kernel banner proves the entry and decompression path. An initramfs shell proves the kernel can schedule userspace. A PID 1 error proves the init executable ran. Each proof removes entire classes of speculation.

The next chapter returns to the first boundary and slows it down: what actually happens between reset and the moment firmware is ready to select a boot target?

---

## Primary references

- Linux/x86 boot protocol: https://docs.kernel.org/arch/x86/boot.html
- EFI boot stub: https://docs.kernel.org/admin-guide/efi-stub.html
- Kernel parameters: https://docs.kernel.org/admin-guide/kernel-parameters.html
- UEFI specifications: https://uefi.org/specifications
- GNU GRUB manual: https://www.gnu.org/software/grub/manual/grub/
- initramfs overview: https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html
- systemd boot-up sequence: https://www.freedesktop.org/software/systemd/man/latest/bootup.html
- Linux kernel source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/
