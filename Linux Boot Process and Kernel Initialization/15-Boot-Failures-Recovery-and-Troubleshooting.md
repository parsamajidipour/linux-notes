# Boot Failures, Recovery, and Troubleshooting

The safest boot repair begins by refusing to repair the whole boot process.

Firmware, loader, kernel entry, kernel initialization, device discovery, initramfs, root mount, and PID 1 are separate boundaries. Evidence normally proves that several early boundaries already succeeded. Changing them anyway destroys evidence and can turn one recoverable configuration error into several independent failures.

This chapter provides a stage-first troubleshooting method, concrete failure patterns, and a conservative recovery workflow. Commands are examples of questions to ask, not permission to write to the first disk matching a familiar name.

---

## 1. Recovery principles

### Preserve data before configuration elegance

If storage health or filesystem integrity is uncertain, prioritize imaging, read-only access, and metadata capture. A system that boots after a destructive repair but loses files is not a successful recovery.

### Change one boundary

If firmware starts GRUB and GRUB loads the kernel, do not update firmware and reinstall GRUB before investigating the kernel message. Minimal changes preserve causal information and rollback.

### Keep a known-good path

Retain an older kernel, signed recovery image, alternate boot entry, rescue medium, and encryption recovery key. Test them before an incident.

### Record before reboot

Save exact errors, command line, block topology, boot variables, logs, and recent changes. Reboot can clear volatile evidence or cause a degraded array to change state.

### Understand every write

Partition-table repair, filesystem repair, RAID creation, LUKS formatting, and boot-loader installation write different structures. Never run them as generic "make bootable" actions.

---

## 2. Build a last-successful-milestone statement

Replace "it won't boot" with a sentence such as:

> Native UEFI firmware loads shim, shim starts GRUB, GRUB reads its configuration and loads the selected kernel and initramfs, the kernel initializes the NVMe controller, and dracut drops to an emergency shell because LUKS UUID X is absent.

That statement excludes power, firmware selection, ESP readability, first-stage signature, GRUB configuration access, kernel image loading, decompression, major kernel initialization, console, and execution of initramfs /init.

The remaining search is storage identity, driver/module presence, and early userspace policy.

Write the statement even when knowledge is incomplete. Mark uncertain edges. Troubleshooting is the process of replacing uncertainty with evidence.

---

## 3. Prepare rescue access before failure

A serious system should have:

- rescue media matching architecture and firmware mode;
- tested UEFI and legacy entry as applicable;
- serial, BMC, hypervisor, or physical console access;
- offline encryption recovery credentials;
- partition-table and LUKS-header backups;
- a known-good kernel/initramfs or UKI;
- firmware setup credentials and key documentation;
- external storage for logs and images;
- a record of normal lsblk, efibootmgr, findmnt, and boot timing;
- backups verified by restoration, not only creation.

Remote servers need out-of-band access. An SSH-only recovery plan fails exactly when networking or root startup fails.

---

## 4. Stage 0: no power or no platform activity

Symptoms:

- no LEDs, fans, BMC, or POST code;
- power cycling;
- firmware never produces display or serial output;
- memory training loops.

Focus on power, board, reset, memory installation, firmware recovery, and platform event logs. Disconnect recently added hardware methodically. Restore supported memory settings. Consult vendor recovery procedures.

Do not rewrite the ESP; no evidence says the CPU reached boot selection.

If BMC is alive, collect sensor, power, and machine-check/event data before clearing logs. A BMC is a separate computer and may operate while the host is dead.

---

## 5. Stage 1: firmware runs but finds no boot target

Symptoms:

- "No bootable device";
- return to firmware setup;
- expected entry missing;
- network boot attempted unexpectedly.

Questions:

1. Is the storage device detected?
2. Is current mode native UEFI or legacy/CSM?
3. Does the disk contain the corresponding loader path?
4. Does GPT identify an ESP?
5. Are Boot#### and BootOrder present?
6. Does the referenced partition GUID still exist?
7. Does the EFI file exist and match architecture?
8. Does Secure Boot reject it?
9. Does the fallback path work?

Boot rescue media in the intended firmware mode. Inspect read-only:

~~~sh
lsblk -o PATH,SIZE,PTTYPE,PARTTYPE,PARTUUID,FSTYPE,MOUNTPOINTS
sudo efibootmgr -v
sudo fsck.fat -n /dev/ESP_DEVICE
~~~

The -n option requests no write for fsck.fat, but verify tool behavior and target. Mount read-only to inspect files.

If manual firmware "boot from file" starts the loader, the image and ESP are readable; persistent selection metadata is the lead.

---

## 6. Stage 2: Secure Boot rejection

Symptoms:

- security violation;
- invalid signature;
- loader works only with Secure Boot disabled;
- recently updated dbx or shim.

Inventory the chain:

~~~sh
mokutil --sb-state
sbverify --list /path/to/image.efi
sudo efibootmgr -v
~~~

Confirm first-stage signer, revocation state, shim/grub compatibility, UKI/kernel signature, and MOK enrollment. A valid signature by an untrusted key is still rejected. A once-trusted vulnerable binary can be rejected after dbx update.

Do not permanently disable Secure Boot as the first repair if the system depends on it for disk-unlock trust or compliance. Use a signed recovery path and update components in the correct order.

---

## 7. Stage 3: loader starts but configuration or files fail

Symptoms:

- GRUB rescue prompt;
- unknown filesystem;
- missing grub.cfg;
- kernel file not found;
- systemd-boot shows no entries.

At a GRUB prompt, inspect variables and devices without writing:

~~~text
set
ls
ls ($root)/
ls ($prefix)/
~~~

Questions:

- Did partition GUID or filesystem UUID change?
- Is $prefix on the expected filesystem?
- Are required GRUB modules available?
- Is /boot separate and currently the loader's root?
- Did an update fill or unmount the ESP?
- Does configuration reference versioned files that were removed?

Booting a kernel manually once can restore service, but repair the generated configuration and update pipeline afterward.

---

## 8. Stage 4: kernel never reaches normal messages

Symptoms:

- immediate reset after selection;
- decompression error;
- unsupported CPU;
- black screen before Linux banner.

Compare a known-good older kernel using the same loader and command-line visibility. Remove quiet/splash, add early console appropriate to platform, and capture serial.

Verify artifact integrity and architecture:

~~~sh
file /boot/vmlinuz-VERSION
sha256sum /boot/vmlinuz-VERSION
~~~

Compare hash to the installed package or trusted source. Do not use an arbitrary internet hash for a distribution-patched build.

If nokaslr changes behavior, collect memory-map and placement evidence. If cold boot differs from warm reset, include firmware and hardware state. If every kernel fails after a firmware setting change, restore one setting at a time.

---

## 9. Stage 5: early kernel panic

Capture the first error and call trace. The final "Kernel panic" line only states that continuation is impossible.

Classify:

- unable to handle page fault;
- attempted to kill init;
- no working init;
- VFS unable to mount root;
- machine check;
- BUG or assertion;
- soft/hard lockup followed by watchdog;
- stack corruption or protection fault.

Use full trace, exact kernel build, taint state, hardware/VM definition, and command line. A matching vmlinux with symbols is essential for address resolution.

pstore or kdump can preserve evidence. Repeated reboot without capture is not progress.

---

## 10. Stage 6: root block device missing

Common messages include unknown-block, waiting for root device, or an initramfs UUID timeout.

Follow layers:

~~~sh
cat /proc/cmdline
cat /proc/partitions
ls /sys/bus/pci/devices
cat /proc/modules
ls -l /dev/disk/by-uuid /dev/mapper 2>/dev/null
blkid
dmsetup ls --tree 2>/dev/null
cat /proc/mdstat
pvs 2>/dev/null
~~~

Questions:

1. Is controller enumerated?
2. Is the expected driver built in or loaded?
3. Did firmware blob load?
4. Does raw block device exist?
5. Is GPT readable?
6. Does expected partition identifier exist?
7. Must RAID, encryption, or LVM activate first?
8. Is root= referring to the correct layer?

Rebuilding initramfs is justified when it lacks a required module or policy. Compare old and new manifests to understand the repair.

---

## 11. Stage 7: encrypted root will not unlock

Possible causes:

- wrong LUKS UUID in command line or crypttab copy;
- prompt on another console;
- missing keyboard, TPM, network, or token support;
- damaged LUKS header;
- keyslot changed;
- TPM PCR policy changed after update;
- required cryptographic module absent;
- cloned duplicate containers.

Inspect header read-only:

~~~sh
cryptsetup luksDump /dev/DEVICE
~~~

Confirm the target twice. luksDump does not unlock data but reveals identifiers and slots. Do not run luksFormat; that creates a new header and destroys access to old key material.

For TPM unlock failure, use recovery passphrase, then inspect PCR/event changes and enrollment policy. Do not erase and recreate encryption merely because automatic unlock failed.

---

## 12. Stage 8: filesystem mount failure

The device exists, but mount fails.

Collect:

- filesystem type and UUID;
- exact kernel mount error;
- clean/dirty state;
- hardware I/O errors;
- mount options;
- whether another layer is read-only;
- whether filesystem driver is available.

Mount read-only first from rescue:

~~~sh
mount -o ro,norecovery /dev/DEVICE /mnt
~~~

The norecovery option is filesystem-specific and not valid universally. For ext filesystems, noload has different consequences; for XFS, norecovery requires read-only; Btrfs options differ. Consult the exact filesystem documentation.

If I/O errors exist, image failing hardware before repair. Filesystem repair writes metadata and stresses the device.

---

## 13. Stage 9: init missing or not executable

Messages can say requested init failed or no working init found.

Mount candidate root and inspect within it:

~~~sh
ls -l /mnt/sbin/init
readlink -f /mnt/sbin/init
file /mnt/sbin/init
readelf -l /mnt/sbin/init | grep -i interpreter
~~~

Then verify the interpreter path under /mnt. Check mount noexec, file permissions, architecture, library corruption, and security labels.

Boot with init=/bin/sh as a controlled comparison only if /bin/sh and its interpreter exist. If shell works, normal init or its dependencies are narrowed.

---

## 14. Stage 10: PID 1 emergency mode

If systemd displays emergency mode, PID 1 executed. Read the journal:

~~~sh
systemctl --failed
journalctl -b -p warning
systemctl list-jobs
findmnt --verify
~~~

Common causes:

- stale fstab UUID;
- required mount timeout;
- broken generator input;
- dependency cycle;
- failed fsck;
- missing credentials;
- service explicitly configured as boot-critical.

Use systemctl status for the first failed prerequisite, not only the final target. The systemd dependency graph can propagate one mount failure into many skipped units.

---

## 15. Stage 11: boot completes but is slow

Define the delayed endpoint. Then separate firmware, loader, kernel, initrd, and userspace:

~~~sh
systemd-analyze time
systemd-analyze critical-chain
systemd-analyze blame
journalctl -b -o short-monotonic
~~~

Look for timeouts rather than CPU work: absent devices, DNS, DHCP, entropy, network mounts, password prompts, controller reset, and fsck.

Optimize dependencies and activation. Do not disable security checks or data recovery merely to lower a benchmark. Measure multiple equivalent boots.

---

## 16. A conservative chroot repair workflow

Use a trusted rescue environment booted in the same firmware mode when repairing loaders.

1. Inventory disks and identifiers.
2. Activate encryption, RAID, and LVM deliberately.
3. Mount real root at /mnt, initially read-only if integrity is uncertain.
4. Mount a separate /boot and the correct ESP at their intended paths under /mnt.
5. Verify /mnt/etc/fstab and actual device IDs.
6. Bind or recursively bind /dev, /proc, /sys, and /run only as required by tools.
7. Enter chroot and confirm package database, kernel versions, and target paths.
8. Rebuild the affected initramfs or loader artifact using distribution tools.
9. Verify output files and free space.
10. Recreate UEFI entry only if selection metadata is missing.
11. Exit, unmount cleanly, and test with a one-shot entry while retaining fallback.

The exact commands vary. Chroot does not automatically make rescue /boot and ESP mounts correct. Many failed repairs updated the rescue medium or an unselected ESP.

---

## 17. Rebuilding initramfs safely

Before rebuilding:

- identify exact kernel release;
- ensure /lib/modules/release is complete;
- check /boot space and filesystem health;
- preserve the old image under a versioned name;
- inspect current image manifest;
- confirm required storage modules, firmware, and policy.

Generate a new temporary/versioned image if tooling permits, inspect it, then update the loader entry. Do not leave kernel and initramfs mismatched.

Test the new entry once. Keep old artifacts until a cold boot and application health check succeed.

---

## 18. Reinstalling a UEFI loader safely

First prove the ESP:

- correct GPT type;
- readable FAT;
- intended disk and partition GUID;
- sufficient free space;
- mounted at the target root's expected path;
- backed up.

Then use distribution-supported loader installation. Inspect resulting EFI files, signatures, and Boot#### path. If Secure Boot is enabled, verify the installed first stage is trusted.

Do not format the ESP unless filesystem recovery requires recreation and all boot files are backed up. Multiple operating systems may share it.

---

## 19. GRUB rescue versus GRUB normal shell

GRUB rescue has a limited command set because normal modules or prefix were not found. GRUB normal shell has richer modules and commands.

In rescue, identifying the filesystem containing /boot/grub and setting root/prefix may let you load normal mode temporarily. This proves core GRUB is running and locates missing configuration.

A permanent repair reinstalls the correct platform build and modules into the intended locations and regenerates configuration. Copying random core images can create version mismatches.

Record the commands that booted successfully; they define the correct paths for repair.

---

## 20. Partition-table recovery

If GPT verification reports one valid and one invalid header, stop writes and save current metadata. Compare partition boundaries with known filesystems and backups. Use GPT-aware recovery to reconstruct from the valid copy.

If both are invalid, partition starts may be recoverable from filesystem signatures and backups, but guessing boundaries is high risk. Image the disk first.

Never create new filesystems to "make partitions visible." Formatting overwrites the evidence used to recover old boundaries and content.

A missing partition table and a missing filesystem are separate failures.

---

## 21. Storage hardware failure

Boot errors can be the first visible symptom of failing storage.

Warning signs:

- I/O errors, resets, link CRC errors;
- SMART/NVMe critical warnings;
- increasing media errors;
- filesystem corruption recurring after repair;
- device disappearing under load;
- very slow reads and timeouts.

Prioritize a block-level image with a recovery-oriented tool that handles bad areas and maintains a map. Avoid repeated fsck and package reinstall on the failing original. Every write and scan consumes remaining life.

Controller cables, power, and enclosure bridges can mimic media failure. Preserve logs and test the least destructive path.

---

## 22. Kernel regression workflow

When an older kernel boots and a new one fails:

1. Keep loader entry, command line, and hardware constant.
2. Ensure each kernel uses its matching initramfs and modules.
3. Capture complete logs from both.
4. Identify the first divergence.
5. Test whether the issue is kernel image, initramfs content, module ABI, or command-line generation.
6. Search upstream and distribution reports using exact error and hardware IDs.
7. If reproducible, bisect in a VM or spare machine when practical.

Report kernel versions, configuration difference, firmware, CPU, device IDs, logs, and reproduction. "Latest kernel doesn't boot" is too broad for maintainers.

---

## 23. Firmware regression workflow

After firmware update, compare:

- default settings reset;
- UEFI/CSM mode;
- Secure Boot keys and state;
- storage controller mode, such as AHCI versus RAID;
- TPM state and PCR measurements;
- memory training and capacity;
- ACPI tables and device topology;
- boot entries and order;
- IOMMU settings.

Do not downgrade firmware casually; rollback protection and security fixes may apply. Consult vendor guidance and preserve recovery power. Often the update reset one policy option rather than breaking Linux code.

---

## 24. Virtual-machine-specific failures

Check VM definition as part of boot artifacts:

- BIOS versus OVMF;
- persistent NVRAM file;
- chipset/machine version;
- CPU model;
- disk bus and controller;
- Secure Boot keys;
- TPM state;
- serial console;
- device ordering.

Changing a disk from SATA to virtio can make root disappear if initramfs lacks the driver. Replacing the OVMF variables file can erase boot entries and enrolled keys while the virtual disk remains healthy.

Snapshots can restore disk without matching TPM/NVRAM state, breaking sealed keys or Secure Boot policy. Snapshot the security state coherently.

---

## 25. Common destructive mistakes

- Running mkfs on a partition because firmware cannot find a loader.
- Running mdadm --create when intending to assemble.
- Running cryptsetup luksFormat when automatic unlock fails.
- Restoring an MBR/GPT backup to the wrong disk.
- Reinstalling loaders to every disk without mapping the selected path.
- Running fsck on a mounted or failing filesystem.
- Deleting the only known-good kernel for space before testing the new one.
- Clearing TPM or Secure Boot keys without recovery secrets.
- Using /dev/sdX from an old guide without resolving current identity.
- Disabling every security mechanism and calling the resulting boot a finished repair.

Pause whenever a command contains create, format, wipe, repair, write, install, clear, or force. State the exact structure it will modify and the rollback.

---

## 26. Incident capture template

Record:

~~~text
Machine / VM:
Architecture:
Firmware version and mode:
Secure Boot state:
Last known-good boot:
Recent change:
Last successful milestone:
First exact error:
Kernel and initramfs pair:
Command line:
Storage graph:
Root identifier requested / observed:
Console and capture method:
Comparison result with fallback:
Writes already performed:
Backup and recovery-key status:
~~~

This prevents repeated guessing and makes escalation useful.

---

## 27. Recovery completion criteria

A machine displaying a login prompt is not the only success criterion.

Verify:

- intended firmware mode and Secure Boot state;
- expected boot entry and fallback;
- signed/verified artifacts where required;
- kernel/initramfs/module version consistency;
- correct root storage graph and identifiers;
- filesystem health and mount options;
- PID 1 and critical services;
- network and application health;
- logs free of repeated storage or hardware errors;
- next cold boot;
- rollback entry still works;
- monitoring and backups resumed.

Document root cause and remove diagnostic weakening options.

---

## 28. Final mechanism summary

Boot troubleshooting is boundary testing.

Power and reset precede firmware. Firmware selection precedes loader execution. Loader file access precedes kernel entry. Decompression precedes start_kernel(). Driver probe precedes block devices. Raw devices precede encryption, RAID, and LVM. Those layers precede filesystem mount. Root mount precedes init exec. PID 1 precedes service policy.

At every failure, identify the last proof, state the next contract, collect evidence, compare one variable, and repair the narrowest violated structure. Preserve data and a rollback path.

Once this method becomes habit, "unbootable" stops being one frightening category. It becomes a finite chain of testable handoffs—and each successful handoff is useful evidence.

---

## Primary references

## 29. Compact decision tree

Use this when pressure makes it tempting to skip evidence.

**No firmware output:** move outward to power, reset, board diagnostics, memory training, and BMC. Do not touch disk metadata.

**Firmware output, no loader:** verify boot mode, device, GPT/ESP or legacy code, Boot#### path, file presence, architecture, and signature.

**Loader output, no Linux banner:** verify selected kernel file, image format, signature, entry parameters, early console, CPU compatibility, and decompressor evidence.

**Linux banner, no initramfs shell:** capture the first panic or hang; focus on memory, interrupts, CPU startup, initcalls, archive unpacking, and /init execution.

**Initramfs shell:** map controller to raw device to partition to encryption/RAID/LVM to filesystem. Compare requested identifiers with observed ones.

**Real root mounted, no init:** inspect executable, symlink, shebang/ELF interpreter, architecture, permissions, mount flags, and security policy.

**PID 1 emergency:** inspect generated mounts, fstab, unit dependencies, filesystem checks, and first failed prerequisite.

**System reachable, service unhealthy:** boot chain succeeded to the application's dependency layer. Use service and application diagnostics.

At each branch, a fallback comparison should alter one object: previous kernel, alternate initramfs, manual EFI file, recovery root, single CPU, or alternate console. Record the result before the next experiment.

## 30. Example postmortem

Incident: after a kernel update, a virtual server stopped in initramfs waiting for root UUID. The previous kernel entry booted.

Initial temptation was to reinstall GRUB. Evidence showed GRUB loaded both entries and each kernel printed a banner. The new initramfs contained the filesystem driver but not the virtio-scsi transport module because it had been generated in a chroot where the module was not loaded and host-only detection was enabled. The old archive contained it.

Recovery booted the old entry, generated a new versioned archive with the transport explicitly included, inspected its manifest, created a temporary loader entry, and rebooted through that entry. The new kernel found the controller, root UUID appeared, and the application health check passed. The old entry remained until a cold reboot was tested.

Contributing factors were a build process that inferred target hardware from the build environment and a deployment check that verified file existence but not archive contents. Corrective actions added an explicit module manifest, a VM boot test using the production controller model, artifact pairing checks, and automatic rollback.

The postmortem names the failed boundary and improves the system that produced it. "Rebuilt initramfs" alone would describe an action, not a cause.

## 31. When to stop

Stop active repair and escalate to imaging or specialist recovery when:

- storage produces new I/O errors during reads;
- partition boundaries are unknown and both GPT copies are invalid;
- a LUKS header is damaged without backup;
- RAID member roles are uncertain;
- commands already wrote conflicting metadata;
- the only backup is unverified and the data is irreplaceable;
- firmware update or key enrollment can lose platform access;
- safety-critical or regulated systems require controlled procedure.

Knowing when not to type is part of boot engineering. Read-only evidence and a clone preserve future options; speculative writes remove them.

## 29. Compact decision tree

No firmware output: move outward to power, reset, memory training, and BMC. Firmware but no loader: inspect boot mode, GPT/ESP or legacy code, Boot####, image, and signature. Loader but no Linux banner: inspect selected kernel, format, entry state, and early console. Linux banner but no initramfs shell: capture panic around memory, CPUs, initcalls, archive, and /init. Initramfs shell: map controller through root filesystem. Root mounted but no init: inspect executable and interpreter. PID 1 emergency: inspect fstab, units, and first failed prerequisite. Reachable host with unhealthy app: the boot chain succeeded to the application's layer.

At each branch, compare one fallback and record the result.

## 30. Example postmortem

After a kernel update, a VM waited for root UUID. The previous entry worked. GRUB loaded both kernels, so reinstalling it would target a proven layer. The new initramfs lacked virtio-scsi because host-only detection ran in a build environment using another controller.

Recovery used the old entry, rebuilt a versioned archive with an explicit module, inspected its manifest, and tested a one-shot entry. Corrective action added a required-module manifest, production-controller VM boot test, kernel/initramfs pairing check, and rollback.

"Rebuilt initramfs" describes the action. "Hardware inference from the build host omitted the target transport" describes the cause.

## 31. When to stop

Stop writing and move to imaging or specialist recovery when storage develops new I/O errors, both GPT copies are invalid, a LUKS header lacks backup, RAID roles are uncertain, prior commands wrote conflicting metadata, or data is irreplaceable.

Also stop when firmware key changes can remove platform access or regulated procedure requires controlled handling. Read-only evidence and a clone preserve options; speculative repair removes them.

## 32. Communicating the handoff

When escalating, send the milestone statement, exact first error, full unedited surrounding log, versions, command line, storage graph, fallback comparison, and every write already performed. Separate observations from hypotheses. "The disk is dead" is a hypothesis; repeated controller resets and unreadable sectors at named LBAs are observations.

A concise evidence package lets another engineer continue instead of repeating risky discovery. Good recovery is collaborative memory as much as command knowledge.

- Linux kernel administration guide: https://docs.kernel.org/admin-guide/
- Kernel parameters: https://docs.kernel.org/admin-guide/kernel-parameters.html
- initramfs and rootfs: https://docs.kernel.org/filesystems/ramfs-rootfs-initramfs.html
- kdump: https://docs.kernel.org/admin-guide/kdump/kdump.html
- pstore: https://docs.kernel.org/admin-guide/pstore-blk.html
- GNU GRUB manual: https://www.gnu.org/software/grub/manual/grub/
- UEFI specifications: https://uefi.org/specifications
- systemd debugging: https://systemd.io/DEBUGGING/
- util-linux manuals: https://www.kernel.org/pub/linux/utils/util-linux/
- cryptsetup project: https://gitlab.com/cryptsetup/cryptsetup
- LVM documentation: https://docs.kernel.org/admin-guide/device-mapper/
