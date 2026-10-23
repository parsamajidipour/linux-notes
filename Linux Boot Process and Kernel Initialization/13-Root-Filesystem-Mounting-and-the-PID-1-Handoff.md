# Root Filesystem Mounting and the PID 1 Handoff

The kernel can schedule processes and discover devices, but a normal Linux system is not present until one filesystem becomes / and one executable becomes PID 1.

Those two choices are coupled. The init executable is resolved inside the active root. If root points to the wrong filesystem, a perfectly healthy /sbin/init elsewhere is irrelevant. If root is correct but the executable or its dynamic linker is missing, the kernel cannot enter normal userspace.

This chapter follows root= from text to block device, explains direct-root and initramfs paths, separates mount from root switch, and examines why PID 1 is special.

---

## 1. What root means at each stage

The word root is overloaded:

- firmware has a boot device path;
- GRUB has a root used to find files;
- Linux VFS begins with rootfs;
- the command line names an intended final root;
- initramfs mounts a candidate at /sysroot or /newroot;
- a process has a root directory used for pathname resolution;
- the final mount namespace has / backed by the operating-system filesystem.

Always qualify the stage. "Root is wrong" is not actionable.

---

## 2. Direct root without initramfs

Linux can mount a real root directly when everything required is built into the kernel and the device can be identified without userspace policy.

Requirements include:

- controller, bus, block, partition, and filesystem drivers built in;
- root device available by the time mount is attempted or rootwait/rootdelay policy;
- no userspace-only encryption, RAID, LVM, or network setup;
- correct root= and rootfstype/rootflags where needed;
- an executable init and its interpreter on that filesystem.

This is practical for small appliances with fixed hardware. General distributions use initramfs because modular kernels and varied storage make direct root brittle.

If no /init exists in rootfs, kernel initialization proceeds toward prepare_namespace() and direct-root logic according to configuration and boot path.

---

## 3. root= forms

The kernel and early userspace support several naming forms depending on implementation:

~~~text
root=/dev/sda2
root=8:2
root=UUID=1111-2222
root=PARTUUID=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
root=LABEL=rootfs
root=PARTLABEL=Linux-root
root=/dev/nfs
~~~

Device names such as /dev/sda2 depend on enumeration and can change. Major:minor is similarly tied to a discovered device instance. Filesystem UUID and label identify filesystem metadata. PARTUUID and PARTLABEL identify partition-table metadata.

The in-kernel direct-root parser supports a defined set. Initramfs frameworks can implement richer syntax and translate stable identifiers to device nodes. Do not assume every form is interpreted by the kernel itself.

After boot:

~~~sh
cat /proc/cmdline
findmnt -no SOURCE,FSTYPE,OPTIONS /
lsblk -o NAME,TYPE,PARTUUID,PARTLABEL,UUID,LABEL,FSTYPE,MOUNTPOINTS
~~~

This connects requested identity to the mounted result.

---

## 4. rootwait and rootdelay

Device discovery can finish after mount logic first checks. rootwait tells the direct-root path to wait for the root device, useful for asynchronously probed storage. rootwait= with a timeout is supported in newer contexts where documented. rootdelay=N sleeps a fixed number of seconds before attempting.

An event-driven initramfs can wait for a specific stable identifier and show progress. Fixed delays hide why a device is late.

If rootdelay fixes boot, investigate driver load, firmware upload, USB enumeration, network storage, RAID assembly, or device-manager timing. Keep the delay only when the underlying device genuinely has a bounded unavoidable readiness time.

---

## 5. Filesystem driver availability

Finding the block device is not enough. Linux needs a registered filesystem type.

For direct root, the filesystem driver must be built in. A module stored on the root filesystem cannot load before that filesystem is mounted. With initramfs, the module can be loaded from the archive.

Check build state:

~~~sh
grep -E 'CONFIG_EXT4_FS=|CONFIG_XFS_FS=|CONFIG_BTRFS_FS=' /boot/config-$(uname -r) 2>/dev/null
cat /proc/filesystems
~~~

/proc/filesystems lists currently registered types. A module may register one after load.

rootfstype= can avoid probing and specify a type. rootflags= passes mount options. Incorrect flags can make a valid filesystem fail mount.

---

## 6. Initial read-only mounting

The ro command-line option requests an initial read-only root. Traditional boot flows mount read-only, run filesystem checks, then remount read-write after integrity is established. Modern filesystems and init systems vary, but read-only first remains a safe default in many configurations.

rw requests read-write immediately. This can be required for specialized systems and can be unsafe after an unclean shutdown when userspace policy expected to check first.

The kernel's ability to mount read-only does not prove the filesystem is healthy for writes. Error-handling options can force read-only mode later.

Inspect actual options:

~~~sh
findmnt -no OPTIONS /
grep ' / ' /proc/mounts
~~~

---

## 7. Filesystem checks

fsck is userspace policy, not a generic kernel mount phase. Tools are filesystem-specific. Ext filesystems use e2fsck; XFS normally replays its log during mount and xfs_repair is not a routine boot check; Btrfs has different check and scrub semantics.

systemd generators derive check units from fstab and metadata. Initramfs may check root before switching.

Never run a write-capable filesystem check on a mounted filesystem unless the filesystem's documented tool explicitly supports it. "Force fsck" copied across filesystem types can damage data.

Capture the exact mount error and filesystem type, then use that filesystem's recovery documentation.

---

## 8. Initramfs mounts a candidate root

Early userspace constructs lower layers and mounts the intended filesystem at a staging directory:

~~~sh
mount -o ro /dev/mapper/vg-root /sysroot
~~~

Before transition, it can verify:

- /sysroot contains expected release metadata;
- /sysroot/sbin/init or configured init exists;
- binaries have the correct architecture;
- essential mount points exist;
- integrity policy passed;
- machine identity or deployment slot is intended.

Mounting at /sysroot does not change PID 1's root directory. It only creates another mounted tree visible from the initramfs namespace.

---

## 9. Mount namespaces and the root view

Every process resolves absolute paths from a root directory within a mount namespace. Early userspace usually runs in the initial mount namespace. It must make the candidate mount become / for itself and descendants.

chroot changes a process's root directory but leaves the old mount tree and does not inherently clean or detach it. It is not sufficient for initramfs disposal.

pivot_root exchanges the root mount with a new mount and places the old root beneath it. Requirements about mount points and propagation apply. The caller then changes directory, unmounts old root, and removes it.

switch_root is a userspace helper tailored to initramfs. It arranges the transition, moves virtual filesystems as appropriate, deletes old rootfs contents, and execs the new init.

Exact implementations differ, but the goals are stable: new /, preserved kernel interface mounts, no references pinning old root, and exec rather than a lingering parent.

---

## 10. Why old initramfs contents must be deleted

rootfs cannot be unmounted like an ordinary temporary mount. If its extracted files remain referenced, their pages remain allocated. switch_root implementations remove files after moving to the new root so memory can be reclaimed.

The current working directory, open file descriptors, executable mappings, and mounts can pin the old tree. A robust transition closes unnecessary descriptors and changes directories.

A debug shell left running in initramfs can prevent complete cleanup. This is acceptable during diagnosis and a leak in normal boot.

---

## 11. Moving /dev, /proc, /sys, and /run

Early userspace mounts kernel interfaces. The final system also needs them. The transition can move mounts into the new tree or unmount and recreate them.

Moving preserves device-manager state and /run data created in initramfs. Many init systems expect /run handoff information, cryptsetup state, udev database, or command-line processing results.

Incorrect handling can produce a system that reaches PID 1 but lacks device nodes, sysfs, or early state. This is not a kernel device-discovery failure; it is mount handoff.

Inspect mount propagation and tree in emergency environments:

~~~sh
findmnt -R /
cat /proc/self/mountinfo
~~~

mountinfo contains IDs, parents, propagation, filesystem, source, and options needed for precise analysis.

---

## 12. Executing the real init

After root transition, early PID 1 calls execve() on the chosen init path. Successful exec replaces its process image while preserving PID 1.

The kernel's normal fallback sequence has historically attempted configured ramdisk/init paths and defaults such as /sbin/init, /etc/init, /bin/init, and /bin/sh, with details visible in init/main.c and logs. An init= parameter can specify another path. Early rdinit= can select initramfs init in supported paths.

Failure reasons include:

- file absent;
- not executable by mode or mount policy;
- wrong executable architecture;
- missing ELF interpreter/dynamic linker;
- missing script shebang interpreter;
- corrupted binary;
- security policy denial;
- I/O error.

The kernel often prints the attempted paths and error codes before panicking.

---

## 13. The misleading ENOENT

execve can return "No such file or directory" even when ls shows the executable. For scripts, the shebang interpreter may be missing. For dynamically linked ELF files, the PT_INTERP path may be missing.

Example:

~~~sh
readelf -l /sysroot/sbin/init | grep 'Requesting program interpreter'
ls -l /sysroot/lib64/ld-linux-x86-64.so.2
~~~

The path differs by architecture and distribution.

Also verify symlink targets inside the new root. A symlink that resolves on the rescue system may be broken after chroot because absolute targets are interpreted within the candidate root.

This mechanism explains many "init exists but kernel cannot execute it" incidents.

---

## 14. PID 1 semantics

PID 1 is special in several ways.

### Orphan reaping

When a process loses its parent, it is reparented to an appropriate subreaper or ultimately PID 1 in its PID namespace. Init must wait for dead children to prevent zombies.

### Signal handling

The kernel treats signals for PID 1 specially: signals with default terminating behavior are not necessarily delivered in the ordinary way unless PID 1 installs handlers, protecting the system's essential init accidentally. Explicitly handled signals work according to the program.

### System lifecycle

If PID 1 exits, the kernel normally panics because no process remains to manage the system. Containers have their own PID namespaces and namespace init; when that process exits, the container ends and remaining namespace processes are killed.

### Policy role

Real init mounts remaining filesystems, starts services, manages sessions, handles shutdown, and supervises processes. The kernel requires a viable PID 1 but does not require systemd specifically.

---

## 15. systemd's initrd-to-host transition

When systemd runs in initramfs, it can manage devices and mounts through initrd-specific targets and generators. It eventually transitions to the host root and reexecutes or switches into host mode while preserving PID 1.

State in /run can communicate which devices were activated and which command-line options were consumed. Initrd units differ from final host units even though the same systemd binary may execute.

The repository's systemd section covers the service graph after handoff. Here, the important fact is that PID 1 can participate in both early and final phases without the kernel creating a second PID 1.

---

## 16. fstab is later than root selection

/etc/fstab resides on the final root. It commonly describes additional mounts, swap, and mount options. It cannot tell firmware where the loader is or tell the kernel how to find the filesystem containing fstab unless a copy or generated policy exists in initramfs.

The root entry in fstab can influence remount options and fsck policy after the root is available. A mismatch between command-line root= and fstab can mount one filesystem initially and apply expectations intended for another.

Compare:

~~~sh
cat /proc/cmdline
findmnt /
grep -vE '^\s*(#|$)' /etc/fstab
~~~

---

## 17. Root on Btrfs subvolume

Btrfs can contain multiple subvolumes in one filesystem. root= identifies the block device or filesystem, while rootflags=subvol= or subvolid= selects the tree mounted as /.

A correct UUID with a missing or renamed subvolume still fails. Snapshot rollback may change the intended subvolume without changing filesystem UUID.

In rescue, mount the filesystem's top level explicitly and inspect subvolumes:

~~~sh
mount -o subvolid=5,ro /dev/DEVICE /mnt
btrfs subvolume list /mnt
~~~

Use the actual unmounted or read-only recovery target. Do not infer snapshot state from directory names alone.

---

## 18. Read-only and verified roots

Appliances may mount a dm-verity-protected filesystem read-only and use overlayfs or separate data partitions for writes. Root selection then includes:

- base data device;
- hash tree device or region;
- trusted root hash;
- verity mapping;
- filesystem mount;
- overlay upper/work directories if used.

PID 1 may see a writable merged / even though the verified lower root is immutable. findmnt -R and dmsetup reveal layers.

Recovery should never "repair" the verified lower filesystem in place. Deploy a new authenticated image and root hash.

---

## 19. Containers and pivot_root

Container runtimes use mount namespaces and pivot_root or related operations to give a process a different filesystem root. This resembles initramfs transition but occurs after the host has booted and inside an isolated namespace.

The container's PID 1 has special namespace reaping and signal responsibilities. It is not host PID 1 and did not mount the host's root.

The analogy is useful: both transitions construct a mount tree and execute a first userspace process for an environment. The security boundaries differ; chroot or pivot_root alone is not container isolation.

---

## 20. Rescue with init=/bin/sh

Passing init=/bin/sh asks the kernel to execute a shell as PID 1 on the real root. It can bypass a broken service manager, but limitations matter:

- root may be read-only;
- /proc, /sys, and /dev may need mounting;
- job control may be absent without a controlling terminal;
- the shell must reap children;
- networking and storage services are not started;
- typing exit can cause a kernel panic.

A safer repair flow may use a distribution rescue target or initramfs break point. If using the shell, remount only after verifying filesystem health and sync before an explicit reboot.

The option is a diagnostic proof: if /bin/sh executes, kernel, root mount, binary, and dynamic linker paths work; the normal init or its policy becomes the lead.

---

## 21. Failure classification

### Unknown-block panic

The kernel cannot map the requested root to a usable block device. Decode major/minor when shown, confirm driver and discovery, and inspect root= syntax.

### VFS unable to mount root

The device may exist while filesystem type, driver, flags, or metadata is wrong. Look at preceding messages.

### Initramfs mounted root but switch fails

Check that candidate is a mount point, virtual mounts and current directories are movable, old root references are closed, and switch_root syntax matches implementation.

### No working init found

Inspect paths and interpreters inside the active root, not rescue root. Verify architecture and mount noexec policy.

### PID 1 starts and enters emergency mode

Kernel handoff succeeded. Use PID 1 logs, fstab-generated units, dependencies, and service status. Do not reinstall the boot loader solely because the word boot appears.

---

## 22. End-to-end verification

From a healthy system, map the complete chain:

~~~sh
cat /proc/cmdline
findmnt -no SOURCE,FSTYPE,OPTIONS /
lsblk -o NAME,TYPE,PKNAME,FSTYPE,UUID,MOUNTPOINTS
dmsetup ls --tree 2>/dev/null
readlink -f /sbin/init
ps -p 1 -o pid,comm,args
cat /proc/1/mountinfo | head
~~~

Then answer:

- Which component interpreted root=?
- Which device node was mounted first?
- Which mappings lie underneath?
- Was root first read-only?
- Did initramfs or kernel perform the mount?
- Which executable is PID 1 now?
- Is it the same binary that ran in initramfs?

---

## 23. Mechanism summary

The final handoff requires a resolved root device, a registered filesystem driver, valid mount options, a real root mount, a clean transition from rootfs, and an executable init with every interpreter dependency present.

Direct root embeds all required mechanisms in the kernel. Initramfs builds complex storage in userspace, mounts a candidate, moves kernel interface mounts, cleans old root, and execs the real init while preserving PID 1.

PID 1 then owns userspace policy, child reaping, lifecycle, and service startup. A failure before exec is a kernel/early-userspace boundary problem; a reported service failure afterward belongs to the init system.

The next chapter turns this entire path into measurements and logs.

---

## Primary references

## 24. System-call view of the handoff

A simplified early-init transition clarifies which operations change kernel state:

1. mount() attaches the real filesystem at /newroot.
2. stat/open/access checks confirm required paths.
3. mount with MS_MOVE relocates /dev, /proc, /sys, and /run beneath the new tree, or equivalent setup recreates them.
4. chdir() enters the new mount.
5. pivot_root() or switch_root's underlying sequence changes the process's root and mount arrangement.
6. umount2() detaches the old mount when applicable.
7. unlinkat/rmdir cleanup removes initramfs files so pages can be reclaimed.
8. execve() replaces early /init with the final init executable.

PID does not change across execve. Open file descriptors without close-on-exec can cross accidentally. Signal dispositions, credentials, namespaces, controlling terminal, and environment follow normal exec rules. A clean implementation closes internal descriptors and passes only deliberate state.

strace can capture a custom init in a VM, but tracing PID 1 changes timing and requires the tracer and output destination inside early userspace. A safer learning lab runs the same mount-namespace sequence in a container-like unshared namespace after normal boot. The mount operations remain real inside that namespace, so use a disposable tree.

## 25. Error-code reasoning for init execution

When exec fails, the numeric errno narrows the contract:

- ENOENT: executable path, shebang interpreter, or ELF interpreter absent;
- EACCES: execute permission, directory search permission, noexec mount, or policy denial;
- ENOEXEC: unrecognized executable format;
- ELIBBAD or related loader error: invalid interpreter/library format;
- ETXTBSY: executable conflicts with a writable open reference in unusual cases;
- EIO: underlying storage read failed;
- ENOMEM: insufficient memory for exec structures or mappings.

Kernel messages may translate these into text. Reproduce by chrooting only after mounting the exact candidate root and its required filesystems. Running /mnt/sbin/init directly from rescue can use rescue's dynamic linker resolution context unless chroot or explicit loader paths are handled correctly.

Security modules can return permission errors even when Unix modes look correct. Inspect audit records and labels. Do not disable policy permanently before proving the denial and correcting expected labels.

## 26. A/B root deployments

Transactional systems often keep two root slots. Firmware or loader metadata selects a UKI and root slot; health confirmation marks the new slot successful; failure count triggers rollback.

The slots need unique identifiers or explicit pairing. If both cloned filesystems have the same UUID and are visible, root=UUID can select unpredictably. PARTUUID tied to a signed entry, distinct filesystem IDs, or verity metadata can remove ambiguity.

An update sequence should:

1. write inactive root completely;
2. verify filesystem or verity tree;
3. install and authenticate matching boot artifact;
4. create a one-shot selection with bounded attempts;
5. boot and run a health check beyond PID 1 startup;
6. commit default only after success;
7. preserve recovery if both slots fail.

The PID 1 handoff is necessary but not enough for health. A server can execute init while its application database is unusable. Rollback endpoints should match service requirements.

## 27. Root transition lab

In a disposable VM, build a tiny initramfs and a small ext4 root image containing a static shell as /sbin/init. Pass the root image as a second virtual disk and identify it by filesystem UUID.

First, let the kernel mount it directly with the filesystem driver built in. Capture messages. Second, boot through initramfs: wait for UUID, mount it at /newroot, list the candidate, and switch_root. Compare /proc/1/exe, mountinfo, and memory use.

Then break one dependency per copied image: remove /sbin/init, change its executable bit, replace it with a script whose interpreter is absent, and specify the wrong UUID. Record how far each case reaches. The four failures can all produce an unbootable guest but have distinct evidence.

Restore snapshots after each run. The value is the labeled boundary, not learning to repair by trial and error.

## 24. System-call view

A simplified transition uses mount() for the candidate root, stat/open checks for required paths, MS_MOVE for virtual mounts, chdir(), pivot_root() or switch_root's equivalent sequence, umount2() for old root where applicable, cleanup of rootfs files, and finally execve().

PID does not change across exec. Open descriptors lacking close-on-exec can leak. Signal dispositions, credentials, namespaces, controlling terminal, and environment follow normal exec rules. A clean early init passes only deliberate state.

## 25. Error-code reasoning

ENOENT can mean missing executable, shebang interpreter, or ELF interpreter. EACCES can mean mode, directory search permission, noexec, or security policy. ENOEXEC means unrecognized format; I/O errors point lower; ENOMEM points to exec allocation.

Inspect inside the candidate root. Running /mnt/sbin/init from rescue can use the wrong root context. Security modules can deny despite ordinary modes; read audit evidence before disabling policy.

## 26. A/B root design

Transactional systems keep two root slots. The update writes the inactive slot, verifies it, installs a matching authenticated boot artifact, selects it for bounded attempts, boots, runs a service-level health check, and commits only after success.

Cloned slots need distinct or explicitly paired identifiers. If both expose the same filesystem UUID, root=UUID can choose unpredictably. Signed entry metadata, PARTUUID pairing, or verity roots can remove ambiguity.

PID 1 startup is not sufficient health. A database appliance may execute init while its data migration failed. Rollback criteria must match the service.

## 27. Root-transition lab

In a VM, create a small ext4 root with a static /sbin/init. Boot it directly with the driver built in, then through initramfs and switch_root. Compare /proc/1/exe and mountinfo.

On copied images, remove init, remove execute permission, replace it with a script whose interpreter is absent, and specify a wrong UUID. Record the different errors. All four are "unbootable" to a user; the kernel evidence distinguishes identity, mount, permission, and interpreter contracts.

Restore snapshots between runs. The learning objective is a reliable boundary map, not ad-hoc repair commands.

## 28. Open-descriptor trap

If early PID 1 opens a file on rootfs and keeps the descriptor across exec, the old file pages and mount relationships can remain referenced. Similarly, current working directories or helper processes left behind can prevent cleanup.

Before handoff, terminate helpers, close descriptors, change into the new root, and verify no process remains in the old tree. This is why a debug shell must not survive normal transition. Resource lifetime is part of root correctness, not only neatness.

## 29. Duplicate-identity case study

An administrator clones the root disk and leaves both original and clone attached. Both filesystems and LVM PVs have identical identifiers. After a reboot, enumeration order changes and early userspace refuses duplicate PVs or selects an unexpected filesystem.

Device names cannot solve the ambiguity reliably. Disconnect one copy for recovery, then deliberately randomize GPT GUIDs, filesystem UUIDs, LUKS UUID where appropriate, and LVM identities on the clone using format-specific tools. Update loader entries and fstab only after deciding which disk is authoritative.

Do not randomize identifiers blindly on an encrypted or RAID member. Layered metadata references other identities. A clone intended as an offline exact spare should normally remain disconnected so duplication never enters one discovery domain.

## 30. PID 1 in a container versus host

A container process with PID 1 in its namespace receives the same orphan-reaping and special-signal considerations within that namespace, but host kernel initialization and root mounting already happened. Container runtimes construct namespaces, rootfs, credentials, and cgroups before executing the container init.

This comparison clarifies the abstraction: PID 1 is relative to a PID namespace, while the initial namespace's PID 1 is the host init created by the boot path. Killing container PID 1 ends the container; losing host PID 1 normally panics or makes the whole system unrecoverable.

The root directory is likewise namespace-relative. Two processes can see different / trees while sharing one kernel. Boot establishes the initial mount namespace's root; later containers derive isolated views. This is why a path printed by rescue, initramfs, host PID 1, and a container can name four different objects.

- Kernel init source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/init/main.c
- Historical initrd transition: https://docs.kernel.org/admin-guide/initrd.html
- initramfs and rootfs: https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html
- Kernel root parameter documentation: https://docs.kernel.org/admin-guide/kernel-parameters.html
- pivot_root manual: https://man7.org/linux/man-pages/man2/pivot_root.2.html
- switch_root manual: https://man7.org/linux/man-pages/man8/switch_root.8.html
- execve manual: https://man7.org/linux/man-pages/man2/execve.2.html
- PID namespaces: https://man7.org/linux/man-pages/man7/pid_namespaces.7.html
- systemd initrd interface: https://systemd.io/INITRD_INTERFACE/
