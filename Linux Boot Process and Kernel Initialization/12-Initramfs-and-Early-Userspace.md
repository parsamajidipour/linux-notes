# Initramfs and Early Userspace

Initramfs solves a sequencing problem with ordinary userspace tools.

The real root filesystem may require a driver module, encrypted-volume unlock, software RAID assembly, LVM activation, integrity verification, network configuration, or a remote-storage client. Those facilities cannot be read from the real root before that root exists. Rather than compile every policy and driver into the kernel, the loader supplies a small archive. Linux unpacks it into an in-memory root and executes its /init program.

Early userspace is not a magical kernel mode. It is a normal process environment with an unusual root filesystem and a narrow mission: construct the storage and policy path needed to enter the real system.

---

## 1. initrd and initramfs are related, not identical

The historical **initrd** is a compressed filesystem image loaded into memory and exposed as a RAM block device. The kernel mounts its filesystem, runs early setup, and later changes root.

Modern **initramfs** is normally one or more cpio archives. The kernel extracts files directly into rootfs, the initial in-memory filesystem. There is no required intermediate block device or filesystem image.

Distributions still name files initrd.img even when they contain initramfs cpio streams. Commands and configuration use the words loosely. Determine format by inspection, not filename.

The kernel retains support and documentation for historical initrd behavior, but contemporary general-purpose Linux boot normally means initramfs.

---

## 2. rootfs exists before the archive

Linux creates rootfs as the initial root of the VFS. It is based on ramfs or tmpfs behavior depending on configuration. It cannot be unmounted because it is the kernel's root foundation, but its contents can be removed after switching to another root.

The initramfs archive is unpacked into rootfs. Directories such as /dev, /proc, /sys, /run, /usr, and /newroot then exist according to the archive's contents and init script.

This corrects a common picture: the kernel does not mount the cpio file itself. It reads archive records and creates files, symlinks, nodes, and directories in rootfs.

---

## 3. The cpio newc format

Kernel initramfs tooling commonly uses the cpio newc format. It describes a sequence of path records with metadata and file data, ending with a trailer. The stream may be compressed.

The kernel unpacker can process concatenated archives and padding. This enables patterns such as an uncompressed early microcode archive followed by a compressed main archive.

An archive should use relative paths such as init, bin/sh, and etc/lvm/lvm.conf. Unsafe traversal paths must not escape the extraction root. Build tools normalize ownership and modes.

Inspect with distribution tools:

~~~sh
lsinitrd /boot/initramfs-$(uname -r).img 2>/dev/null | less
lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | less
~~~

Use whichever tool matches the image generator. Manual decompression guesses can fail on concatenated or mixed streams.

---

## 4. Built-in versus external initramfs

The kernel build can embed an initramfs from CONFIG_INITRAMFS_SOURCE. A loader can also supply an external archive. Build and boot paths determine how content is combined.

Embedded archives are useful for appliances and rescue kernels because kernel and early userspace travel together. External archives are easy for distributions to regenerate after driver, storage, or policy changes without rebuilding the kernel.

A UKI packages an external-style initramfs section inside a signed EFI executable, combining update flexibility with a single authenticated artifact.

When debugging, ask which sources are active. Rebuilding /boot/initramfs does not change a built-in archive, and replacing an external file does not help if the loader entry points to a UKI containing another copy.

---

## 5. Unpacking during kernel initialization

The loader supplied the archive address and size. The kernel reserved it during early memory setup. During rootfs population, the unpacker detects compression, expands the stream, and creates files.

Failures can include unsupported compression, truncated data, corrupt cpio records, duplicate path semantics, out-of-memory conditions, and invalid file types.

After unpacking, the original compressed buffer can be released when safe. The extracted files consume page-cache or tmpfs/ramfs memory. Peak use includes both forms.

Kernel logs may report unpacking errors. A silent or immediate fallback into direct-root logic can indicate no executable /init exists.

---

## 6. /init becomes the first userspace program

If rootfs contains executable /init, the kernel executes it. This process becomes PID 1 for early userspace. Later exec of the real init preserves PID 1; there are not two simultaneous first processes.

/init can be:

- a shell script using BusyBox;
- a compiled program;
- systemd running in initrd mode;
- dracut's generated logic;
- initramfs-tools scripts;
- mkinitcpio hooks;
- a custom appliance initializer.

It must have a valid interpreter or dynamic linker inside the archive. A script beginning with #!/bin/sh fails if /bin/sh is missing. A dynamically linked binary fails with ENOENT-like behavior when its ELF interpreter is absent, even though the executable file exists.

Inspect dependencies before packaging:

~~~sh
file path/to/init
readelf -l path/to/program | grep -i interpreter
ldd path/to/program
~~~

For hostile files, ldd can be unsafe on historical implementations; prefer readelf and known build artifacts.

---

## 7. Mounting virtual filesystems

Early userspace mounts interfaces needed for discovery:

~~~sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev
mount -t tmpfs tmpfs /run
~~~

Exact setup and pre-mounted state vary. procfs exposes process and kernel data, sysfs exposes the device model, devtmpfs provides device nodes, and /run holds transient userspace state.

The mounts are not cosmetic. udev needs sysfs and netlink events. Storage tools need /dev nodes. Reading /proc/cmdline requires procfs. Later root switching must preserve or recreate these mounts appropriately.

---

## 8. Device discovery loop

Early /init does not assume all hardware appeared synchronously. It starts a device manager or listens for uevents, loads modules, and waits for identifiers required by policy.

A robust loop:

1. parse command-line requirements;
2. load likely controller and bus modules;
3. process existing devices and new uevents;
4. create stable links from filesystem and partition metadata;
5. trigger storage assembly when dependencies appear;
6. enforce a meaningful timeout or rootwait policy;
7. report exactly which identifier remains missing.

A fixed sleep is simpler and less reliable. Fast machines waste time; slow devices race.

---

## 9. Storage driver modules

Initramfs must contain modules needed to reach root and their dependencies. This may include:

- PCI storage controller or virtio transport;
- NVMe, AHCI, SCSI, USB storage;
- filesystem driver;
- device-mapper targets;
- encryption algorithms;
- RAID personalities;
- network driver and protocol for remote root;
- firmware files required by those drivers.

Host-only generators inspect the current machine and include a small set. Generic images include broad hardware support for portability. Host-only images are smaller and can fail after moving the disk to different hardware. Generic images consume more space and boot work.

After changing storage hardware or virtualization bus, rebuild a generic image before migration or ensure the new driver is present.

---

## 10. LUKS and device mapper

For encrypted root, early userspace locates a LUKS container, obtains a key, and asks device mapper to create a decrypted block mapping.

Key sources include:

- interactive passphrase;
- key file on protected media;
- TPM-sealed secret;
- network-bound decryption;
- hardware token;
- remote administrative delivery.

The unlock prompt must be visible on the intended console. A machine can appear hung while prompting on another terminal.

After successful activation, a /dev/mapper name appears. That mapping may contain a filesystem directly or an LVM physical volume.

Read-only inspection in emergency shell:

~~~sh
cat /proc/cmdline
blkid
ls -l /dev/mapper
dmsetup ls --tree
~~~

Avoid cryptsetup repair or LUKS header writes without a header backup and verified target.

---

## 11. LVM

LVM stores physical-volume and volume-group metadata on block devices. Early userspace scans devices, identifies PVs, assembles VGs, and activates required LVs.

The layered path can be:

~~~text
NVMe partition
  -> LUKS mapping
    -> LVM physical volume
      -> volume group
        -> root logical volume
~~~

Commands:

~~~sh
pvs
vgs
lvs -a -o +devices
dmsetup ls --tree
~~~

An LVM scan before encryption unlock finds no PV by design. Ordering is policy derived from identifiers and metadata.

Duplicate PV identifiers from cloned disks can cause safety refusal or unexpected selection. Disconnect the clone or use explicit filters before activation.

---

## 12. MD RAID

Linux MD assembles component devices into arrays using superblock metadata or external policy. Initramfs includes md modules and mdadm configuration/tools when root depends on an array.

Assembly must handle missing members according to RAID level and policy. Automatically starting a degraded array improves availability and can risk incorrect assumptions during recovery. Early userspace should report members and state clearly.

Inspect:

~~~sh
cat /proc/mdstat
mdadm --detail --scan 2>/dev/null
~~~

Do not use --create to "reassemble" an existing array. Creation writes metadata and can destroy recoverability. Use assemble operations only after recording member superblocks.

---

## 13. Network root

NFS root, iSCSI, NBD, and other remote-root designs require networking in initramfs.

Early userspace may need:

- network-interface driver and firmware;
- link setup;
- static or DHCP configuration;
- VLAN or bonding modules;
- DNS or direct server address;
- authentication and initiator identity;
- route to the server;
- remote filesystem or block client;
- retry and timeout policy.

Firmware PXE success does not prove Linux networking works. Firmware used its own network driver and configuration. Linux must initialize the interface again.

Keep a local rescue path. A remote-root outage otherwise turns network maintenance into a boot outage with limited tools.

---

## 14. Root integrity

Early userspace can assemble dm-verity, fs-verity policy inputs, or integrity-protected device-mapper stacks. dm-verity verifies blocks against a Merkle tree and trusted root hash.

The root hash must come from the authenticated boot chain—commonly an embedded UKI command line or signed metadata. If both storage and root hash are attacker-mutable, the attacker builds a new tree.

Integrity failure policy can panic, restart, log, or expose an error according to deployment requirements. Appliances often pair read-only verified root with a separate writable data partition.

---

## 15. Resume from hibernation

Hibernation saves memory to swap or another device. On boot, initramfs may locate the resume target before mounting root and ask the kernel to restore the image.

The resume device can be inside encryption or LVM, so those layers must be activated first. A resume offset may be needed for swap files.

Restoring an image produced by an incompatible kernel or changed storage state is unsafe; kernels validate signatures and metadata. Encryption keys must match the suspended image's storage context.

A stale resume parameter can delay boot while early userspace waits for a nonexistent identifier. Treat resume as another explicitly modeled dependency.

---

## 16. Distribution generators

### dracut

dracut builds initramfs from modules selected by detected requirements and configuration. It emphasizes event-driven discovery and command-line rd.* policy. lsinitrd inspects images.

### initramfs-tools

Common on Debian-derived systems, it organizes hooks and boot scripts into phases and provides update-initramfs. lsinitramfs lists content.

### mkinitcpio

Common on Arch-derived systems, it uses configured HOOKS and modules. Ordering matters because hooks implement discovery and storage layers.

### custom tools

Appliances may use BusyBox scripts, buildroot, booster, or distribution-specific generators. Read the generated /init and manifest. The generator's name does not fully describe one image's contents.

Always use the tool matching the distribution and image format. Regenerating with a foreign tool can omit integration and overwrite the known-good file.

---

## 17. Microcode archives

On x86, early microcode data can be placed in an uncompressed cpio archive at the beginning of initramfs. The kernel finds and applies it before normal driver initialization, then processes the remaining archive.

This is one reason naive "decompress the whole file with gzip" commands can fail: the file may be concatenated streams with different treatment.

Distribution inspection tools understand the layout. Keep CPU-vendor packages current and ensure rebuilding initramfs preserves early microcode.

---

## 18. Emergency shells

When root preparation fails, initramfs may open a shell. This is a diagnostic success: the kernel initialized, unpacked early userspace, executed /init, and provided a console.

Capture:

~~~sh
cat /proc/cmdline
cat /proc/modules
cat /proc/partitions
ls -l /dev/disk/by-uuid /dev/mapper 2>/dev/null
blkid
dmesg | tail -n 200
~~~

Then follow the storage graph. Do not run fsck on a mounted filesystem, activate duplicate LVM clones, or recreate RAID metadata impulsively.

Shell utilities may be limited BusyBox implementations. Check command help rather than assuming full GNU options.

---

## 19. Building a tiny lab initramfs

Use a disposable VM and a statically linked BusyBox.

Conceptual tree:

~~~text
lab/
├── init
├── bin/busybox
├── dev/
├── proc/
├── sys/
├── run/
└── newroot/
~~~

/init can mount proc, sysfs, and devtmpfs, print /proc/cmdline, and launch a shell. It must be executable and use an interpreter present in the archive.

Package from inside the tree with reproducible ownership into newc cpio, compress it, create a temporary loader entry, and retain the distribution image. Do not replace the only initramfs.

Once the shell works, add one mechanism at a time: load a virtio module, wait for a filesystem UUID, mount a read-only test root, and use switch_root. The incremental path makes each boundary observable.

---

## 20. switch_root versus pivot_root

pivot_root is a system call that rearranges mounts so the new root becomes / and the old root is mounted beneath it. It requires mount relationships and leaves old root accessible until explicitly unmounted.

switch_root is a userspace utility designed for initramfs transition. It moves or prepares virtual mounts, changes root, deletes files from the old rootfs so memory can be reclaimed, and execs the new init. Implementation details vary.

chroot alone is not enough. It changes pathname resolution for a process but does not establish a system-wide root transition, clean old root, or move essential mounts.

The transition must use exec so the PID 1 process becomes the real init rather than leaving an unnecessary parent early-init process.

---

## 21. Failure patterns

### Archive unpack error

Check file truncation, compression support, concatenated layout, storage integrity, and peak memory.

### /init not found

Inspect archive path, executable bit, shebang interpreter, ELF architecture, and dynamic linker. ENOENT can mean a missing interpreter.

### Root UUID absent

Determine whether the raw device driver loaded, whether encryption/RAID/LVM layers activated, and whether the identifier changed.

### Works after rebuilding initramfs

Compare manifests to learn what was missing. The repair is not understood until the changed module, firmware, script, or configuration is identified.

### Prompt appears on wrong console

Align loader, kernel console= ordering, initramfs prompt tool, serial settings, and graphical splash. Remote systems need tested console policy.

---

## 22. Mechanism summary

Initramfs is a cpio-based early userspace unpacked into rootfs. Its /init process mounts kernel interfaces, processes uevents, loads modules and firmware, constructs RAID/encryption/LVM/network/integrity layers, mounts the candidate real root, and executes the final init through a root-switch operation.

Its purpose is not limited to drivers or encryption. It moves changeable storage policy out of the kernel while preserving the chicken-and-egg solution: everything required to reach root is available before root.

The next chapter follows the final transition in detail: root= resolution, filesystem mounting, switch_root, init search paths, and the special PID 1 contract.

---

## Primary references

## 23. End-to-end encrypted-LVM example

Assume root lives in an LVM logical volume inside a LUKS2 container on an NVMe GPT partition. The loader provides a generic distribution kernel and initramfs plus parameters naming the LUKS UUID and root LV.

The kernel unpacks the archive and executes /init. Early userspace mounts devtmpfs, sysfs, procfs, and tmpfs for /run. It starts udev, which processes devices already present and future netlink uevents. The PCI core has already enumerated the controller; if nvme is modular, a modalias event causes modprobe to load nvme and dependencies from the archive.

The NVMe driver registers a namespace block device. Partition scanning creates the target partition. blkid or udev reads its LUKS signature and creates stable links. Early policy matches the requested LUKS UUID rather than assuming a device name.

cryptsetup reads the LUKS2 header, selects a keyslot, derives or obtains a volume key, and creates a dm-crypt device-mapper target. Device mapper emits a new device event. LVM scans the decrypted mapping, recognizes a PV, assembles the VG, and activates the root LV, producing another device-mapper node.

The filesystem driver is loaded if modular. Early userspace mounts the LV read-only at /sysroot, checks that the expected machine files and init executable exist, moves or prepares /dev, /proc, /sys, and /run, then executes switch_root with /sysroot and /sbin/init.

Every arrow can be observed:

~~~text
PCI function -> nvme controller -> namespace -> GPT partition
-> LUKS mapping -> LVM PV/VG/LV -> filesystem -> /sysroot -> PID 1
~~~

If the LUKS partition exists but the LV does not, do not return to NVMe probe. If the LV exists but mount fails, do not regenerate cryptographic keys. Work at the first missing arrow.

## 24. Reproducible image construction

Initramfs is executable supply-chain material. A repeatable build improves auditing and incident comparison.

Control:

- input kernel release and module tree;
- hook/module configuration;
- firmware packages;
- file ownership and modes;
- locale and environment;
- compression algorithm and options;
- timestamps where tooling supports normalization;
- embedded secrets, which ideally are references rather than literal values;
- output hash and signature or UKI packaging.

After building, create a manifest containing paths, modes, sizes, hashes for critical binaries, module versions, firmware names, and generator version. Compare with the previous image. An unexplained shell, network client, or private key deserves investigation; a missing storage module predicts recovery failure.

Compression can obscure reproducibility through timestamps or nondeterministic parallel output. Use generator-supported reproducible modes rather than manually rewriting an image that contains concatenated microcode and archives.

## 25. Size analysis

A large initramfs can slow storage reads, decompression, measurement, and memory use. A tiny host-only image can fail after hardware migration.

Classify contents:

~~~text
kernel modules
device firmware
executables and shared libraries
localization and terminfo
configuration and certificates
microcode
debug utilities
duplicate files
~~~

Do not delete libraries by filename guesses. Use dependency analysis and generator configuration. Firmware may be requested dynamically based on discovered hardware. Recovery tools add deliberate size.

Measure compressed size, expanded cpio size, unpack time, and peak boot memory. An algorithm with a smaller file can use more CPU and scratch memory. Test on target-class hardware.

## 26. Debugging the generator, not only the result

When a required item is missing, find why selection excluded it. Host-only generators inspect loaded modules, current storage graph, configuration, and command-line hints. If a driver was built in on the build host but modular on the target kernel, detection can be misleading. Cross-building requires explicit modules.

Run the generator in verbose mode and save logs. Identify which hook owns the requirement. Add a targeted configuration rule, rebuild, and confirm the manifest. Copying a module manually into a compressed archive can omit dependencies, signatures, firmware, indexes, and reproducible update behavior.

Test the generated image using a one-shot boot entry. Then test the recovery scenario it is meant to support: alternate controller, degraded RAID, TPM unavailable with passphrase fallback, or network root with one server down. An image is not validated merely because the normal machine booted once.

## 23. End-to-end encrypted-LVM example

Assume root is an LVM LV inside LUKS2 on an NVMe GPT partition. The loader supplies kernel, initramfs, LUKS UUID, and root LV.

The kernel executes /init. Early userspace mounts procfs, sysfs, devtmpfs, and /run, then starts device event processing. The NVMe module loads from the archive, exposes a namespace, and partition scanning creates the LUKS partition. Policy matches UUID rather than guessing a device name.

cryptsetup obtains a key and creates a dm-crypt mapping. Its uevent triggers LVM scanning, which recognizes the PV, activates the VG and root LV, and emits another device. The filesystem driver mounts the LV at /sysroot. Early userspace validates /sysroot/sbin/init, moves virtual mounts, cleans rootfs, and execs switch_root.

~~~text
PCI -> NVMe -> namespace -> partition -> LUKS -> PV/VG/LV -> filesystem -> PID 1
~~~

If LUKS exists but the LV does not, the NVMe layer is already proven. If the LV exists but mount fails, recreating encryption metadata attacks the wrong boundary.

## 24. Reproducible construction

Control kernel release, module tree, hooks, firmware packages, ownership, modes, environment, compression, and generator version. Produce a manifest containing paths, critical hashes, module versions, firmware names, and output digest.

Compare manifests after updates. An unexpected shell or key deserves review; a missing storage module predicts recovery failure. Use generator-supported reproducible modes instead of manually rewriting concatenated microcode and cpio streams.

## 25. Size and peak-memory audit

Classify content into modules, firmware, executables/libraries, localization, configuration, microcode, and recovery utilities. Do not delete libraries by filename guesses. Host-only images trade portability for size.

Measure compressed bytes, expanded archive bytes, decompression time, and peak memory. A smaller algorithm can require more CPU or scratch space. During unpack, compressed input and expanded files coexist. On a small machine, peak memory rather than final rootfs size causes failure.

## 26. Test recovery properties

A normal successful boot does not validate an initramfs. Test the cases it promises to recover: alternate storage controller, degraded RAID, TPM unavailable with passphrase fallback, missing network server, and older kernel fallback.

Use versioned one-shot entries in a VM. Capture the exact emergency message and verify the shell contains diagnostic tools without embedding sensitive secrets. An archive that can boot only the machine that generated it is not a generic recovery image, even if its filename says generic.

## 27. Generator debugging

When something is absent, find why selection omitted it. Host-only generators inspect loaded modules and current topology. Cross-building or migrating hardware requires explicit requirements.

Run verbose generation, identify the owning hook, add a targeted rule, rebuild, inspect manifest, and test the exact loader path. Manually copying one .ko can omit dependencies, aliases, signature, firmware, and indexes. Fix the production rule rather than patching one opaque archive.

## 28. Network-root failure case

Firmware successfully PXE-loads a kernel and initramfs, but /init cannot mount NFS root. This does not contradict firmware success. Firmware used its own NIC driver, DHCP client, and download protocol. Linux must load its NIC driver and firmware, name the interface, configure an address and route, then run an NFS client.

Inspect whether the interface exists, carrier is up, DHCP data reached early userspace, routes include the server, and root options match server exports. Capture packets from another host when possible. If no Linux packet leaves, server-side NFS repair is premature.

Include DNS only when the root parameter uses a name; using an address can isolate resolver failure. Include bonding, VLAN, or iSCSI authentication modules when the production path needs them. Test loss of one network path and ensure emergency access does not itself depend on the failed root.

## 29. Secret-handling review

Initramfs often contains enough code to unlock the most sensitive storage. Inspect whether it embeds key files, network credentials, private certificates, or recovery tokens. An unencrypted ESP or /boot makes external archives readable.

Prefer TPM-sealed references, interactive acquisition, hardware tokens, or network protocols with scoped credentials. If a key must be embedded, authenticate and protect the container that carries it and document the physical-access assumption.

Command-line secrets are especially poor: they can appear in loader configuration, /proc/cmdline, logs, crash dumps, and attestation measurements. Pass identifiers, not raw keys.

The early image should also have a retirement plan. Remove obsolete kernels and initramfs files only after selection metadata no longer references them and a newer recovery path is tested. An ESP or /boot filled by abandoned artifacts can make the next atomic update fail halfway, turning routine maintenance into a boot incident.

Monitor free space before upgrades and verify the new archive checksum after writing. A truncated but correctly named file can survive until the next reboot.

- ramfs, rootfs, and initramfs: https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html
- Historical initrd: https://docs.kernel.org/admin-guide/initrd.html
- Kernel initramfs buffer format: https://docs.kernel.org/driver-api/early-userspace/buffer-format.html
- dracut documentation: https://dracut-ng.github.io/dracut/
- initramfs-tools manual: https://manpages.debian.org/initramfs-tools
- mkinitcpio documentation: https://man.archlinux.org/man/mkinitcpio.8
- device mapper documentation: https://docs.kernel.org/admin-guide/device-mapper/
- LUKS specification and cryptsetup: https://gitlab.com/cryptsetup/cryptsetup
- switch_root manual: https://man7.org/linux/man-pages/man8/switch_root.8.html
