# Partition Tables, Boot Sectors, and the ESP

A disk does not become bootable because one flag says so.

Several independent structures participate: a partition table describes regions; a filesystem gives structure to bytes inside a region; firmware or a loader applies selection policy; executable files or embedded code provide the next program; and configuration identifies the kernel to load. Tools often display all of these in one screen, which makes it easy to confuse a partition type with a filesystem type, a boot flag with executable code, or an EFI System Partition with the Linux root.

This chapter separates those layers. It explains MBR and GPT as on-disk data structures, shows where legacy boot code and UEFI files live, and develops a conservative inspection and recovery workflow. The important rule is simple: never repair a structure until you can name that structure and show that it is the failed boundary.

---

## 1. Five layers on one storage device

Consider an NVMe device containing a normal Linux installation. At least five layers may be relevant:

1. **Device transport:** PCIe and the NVMe protocol expose namespaces as block devices.
2. **Partition table:** GPT divides the namespace into typed address ranges.
3. **Partition contents:** one range may contain FAT, another LUKS, another swap.
4. **Boot metadata:** UEFI variables and device paths point to a file in the ESP.
5. **Operating-system policy:** loader entries and kernel arguments identify kernel, initramfs, and future root.

The same sector can be meaningful to more than one layer, but the layers are not interchangeable. Formatting an ESP recreates a filesystem and destroys its files; it does not necessarily recreate the GPT entry. Recreating a UEFI Boot#### variable does not rewrite the filesystem. Reinstalling GRUB files does not repair a corrupt GPT header. Changing root= does not affect firmware's ability to find the EFI application.

When a tool prints "boot," ask what it means in that tool's model.

---

## 2. Sectors, logical blocks, and addresses

Partition tables describe ranges in units of logical blocks. Traditional language says sectors, often assuming 512 bytes, but modern devices may expose 4096-byte logical blocks or 512-byte logical blocks backed by 4096-byte physical sectors.

Linux reports useful properties through sysfs and block tools:

~~~sh
lsblk -o NAME,SIZE,LOG-SEC,PHY-SEC,MIN-IO,OPT-IO,ALIGNMENT
sudo blockdev --getss /dev/nvme0n1
sudo blockdev --getpbsz /dev/nvme0n1
~~~

Replace the device only after identifying it. These commands read properties; partition-writing commands are a separate and risky step.

A **logical block address**, or LBA, is a numbered block in the device's exposed address space. Modern partition structures normally store starting and ending LBAs. Alignment matters because a partition beginning at an awkward logical location can cause one filesystem write to span multiple physical blocks or storage stripes. Contemporary tools generally align starts to one-megabyte boundaries, which works well across common devices.

Boot behavior should not be explained using obsolete cylinder/head/sector geometry unless investigating an actual legacy constraint. Modern interfaces use linear addresses; historical geometry may still appear in compatibility fields.

---

## 3. MBR structure

The Master Boot Record occupies the first logical sector in the traditional layout. In a 512-byte sector, the familiar structure is:

- bytes 0 through 445: bootstrap code area;
- bytes 446 through 509: four 16-byte partition entries;
- bytes 510 and 511: 0x55AA signature.

The partition entries encode a type byte, starting and ending information, an active marker, and a 32-bit starting LBA and size. The 32-bit sector count creates an address-size limit near two tebibytes with 512-byte sectors. The four-entry table creates a primary-partition limit.

### Extended partitions

MBR works around the four-entry limit with an extended partition. One primary entry identifies a container. Linked Extended Boot Records inside it describe logical partitions. This is a chain, not a fully redundant modern table. Damage to links can make later logical partitions inaccessible even when their data remains.

### Bootstrap code is not the partition table

The first 446 bytes and the partition entries occupy the same sector but are conceptually distinct. A loader installer can update bootstrap code while preserving the table. A partitioning tool can update entries while preserving code. A careless whole-sector write can destroy both.

This distinction matters for backup. Saving only partition entry text is enough to recreate boundaries but not original boot code. Saving the whole sector captures both, but restoring it to a disk whose layout changed can overwrite correct current metadata.

### The active flag

One byte in an MBR entry can mark a partition active. Traditional MBR bootstrap code may select that entry and chainload its volume boot record. Firmware does not necessarily interpret the flag itself, and sophisticated loaders need not use it.

Native UEFI boot does not select an ESP by this active marker. Some tools label unrelated GPT attributes as "boot" for compatibility, which is why the raw type and firmware mode are more reliable evidence.

---

## 4. MBR volume boot records

A partition can begin with its own boot sector, often called a volume boot record. Filesystem metadata and bootstrap code may share this early region according to the filesystem's format.

Traditional chainloading works like this:

1. BIOS loads the disk's MBR code.
2. MBR code chooses an active partition.
3. It loads that partition's volume boot record.
4. Volume code locates a larger loader or operating-system-specific file.

This design lets each operating system own its partition entry path while a generic MBR chooses among partitions. Modern GRUB installations often use different embedding and configuration mechanisms, but chainloading remains relevant for multi-boot and legacy systems.

Never assume the first sector of a filesystem is disposable free space. Filesystem boot regions can contain both format metadata and executable code.

---

## 5. GPT structure

The GUID Partition Table replaces the fixed four-entry MBR design with a header and an array of partition entries identified by globally unique identifiers.

A normal GPT disk contains:

1. a protective MBR at LBA 0;
2. a primary GPT header, normally at LBA 1;
3. the primary partition-entry array;
4. usable space containing partitions;
5. a backup partition-entry array near the end;
6. a backup GPT header in the final logical block.

The primary header records:

- the GPT signature;
- revision and header size;
- a CRC for the header;
- its own and alternate header LBAs;
- first and last usable LBAs;
- a disk GUID;
- location, count, and size of partition entries;
- a CRC for the entry array.

The backup is not a live mirror updated by the operating system at arbitrary times; partitioning tools update both structures as part of a table change. Redundancy helps recover from damage to one end of the disk, but recovery must choose the correct surviving state.

### CRCs provide corruption detection, not authenticity

GPT checksums help a tool detect accidental corruption. They do not prove who created the table or prevent malicious changes. A modified table with recomputed CRCs is structurally valid.

### Partition entries

Each entry includes:

- a partition type GUID;
- a unique partition GUID;
- first and last LBA;
- attribute bits;
- a UTF-16 name.

The type GUID describes intended use. The unique GUID identifies this specific partition. The human-readable name is a label for administrators. Filesystem type and filesystem UUID live inside the partition contents and are different again.

For example, an ESP has the well-known EFI System Partition type GUID. Its unique partition GUID might appear in PARTUUID=. The FAT filesystem inside has its own volume identifier, which blkid may report as UUID=. Reformatting the filesystem changes the filesystem identity without necessarily changing the GPT partition GUID.

---

## 6. The protective MBR

GPT reserves LBA 0 for a protective MBR. Its partition table normally contains one type 0xEE entry covering the GPT disk as far as the MBR size field permits.

The purpose is defensive compatibility. An old MBR-only tool sees a disk that appears occupied rather than an empty disk it may safely repartition. The protective MBR does not convert GPT into MBR and does not make the disk BIOS-bootable by itself.

It may contain boot code, so a GPT disk can participate in a BIOS boot arrangement. The GPT specification and boot-loader installation policy determine where larger loader data goes. For GRUB, a BIOS Boot Partition is the robust GPT embedding location.

---

## 7. Hybrid MBRs

A hybrid MBR exposes selected GPT partitions as ordinary MBR entries in addition to the protective entry. It was used for compatibility with operating systems or firmware that understood only MBR while another system used GPT.

This creates two partition descriptions for the same data. If they diverge, different software can write using different boundaries, causing corruption. Hybrid layouts should be treated as specialized legacy mechanisms, not as a clever general multi-boot solution.

When inspecting an unfamiliar disk, a tool warning about a hybrid MBR deserves attention. Do not "fix" one table without understanding which systems rely on the hybrid representation and without a complete backup.

---

## 8. The EFI System Partition in detail

An ESP is identified by its partition type. It normally contains a FAT filesystem because UEFI defines filesystem access around its FAT-derived format. Firmware drivers may support other filesystems, but portable boot design uses the specified path.

The ESP contains ordinary directories and files. Common objects include:

- first-stage loaders such as shim;
- GRUB EFI binaries and support files;
- systemd-boot;
- Windows Boot Manager;
- firmware update capsules or vendor utilities;
- Unified Kernel Images;
- the standardized EFI/BOOT fallback executable.

The ESP is not automatically secret or immutable. It is often unencrypted so firmware can read it. Security comes from verifying executable content and protecting update paths, not from assuming FAT permissions enforce Linux ownership.

### Size and placement

UEFI does not require one magical size for every system. Distribution recommendations account for FAT implementation compatibility, multiple kernels or UKIs, firmware quirks, and update behavior. An ESP that stores only small first-stage loaders needs less space than one storing several UKIs and initramfs payloads.

The ESP need not be the first partition, though old firmware may contain placement quirks. Multiple ESPs can exist, but update tools and firmware policy must know which one is authoritative. Redundancy is useful only if the system can actually select and maintain the alternate.

### Mount policy

Firmware reads the ESP without Linux mounting it. Linux mounts it only to install or update files. Keeping it permanently mounted simplifies updates; mounting it only during controlled operations reduces accidental modification. Both policies can work if tooling is consistent.

Check the current association:

~~~sh
findmnt -t vfat,fat
lsblk -o NAME,PARTTYPE,PARTTYPENAME,FSTYPE,LABEL,PARTLABEL,PARTUUID,UUID,MOUNTPOINTS
~~~

---

## 9. Boot variables point into storage

A UEFI Boot#### variable normally includes a device path that identifies a disk path, GPT partition, and EFI file. That relationship connects firmware NVRAM to GPT and FAT contents.

Suppose Boot0003 references a partition's unique GUID and the file EFI/Linux/linux.efi. Several independent failures are possible:

- BootOrder omits Boot0003;
- Boot0003 is inactive;
- the referenced disk is absent;
- the GPT partition GUID changed during cloning;
- the FAT filesystem is corrupt;
- the file was renamed or deleted;
- the executable architecture is wrong;
- Secure Boot rejects its signature;
- the file starts but its own configuration is broken.

All may look like "UEFI cannot boot Linux," but their repair layers differ.

Inspect the relationship with:

~~~sh
sudo efibootmgr -v
lsblk -o PATH,PARTUUID,FSTYPE,MOUNTPOINTS
sudo find /boot/efi/EFI -maxdepth 3 -type f 2>/dev/null
~~~

Adapt the mount point. Comparing the partition GUID and file path is more reliable than comparing friendly boot labels.

---

## 10. Identifiers and what changes them

Boot configurations use several identifiers because device names such as /dev/sda can change.

| Identifier | Owned by | Usually changes when |
|---|---|---|
| GPT disk GUID | partition table | table is recreated or cloned and randomized |
| GPT partition GUID / PARTUUID | partition entry | entry is recreated or explicitly changed |
| partition type GUID | partition entry | intended type is changed |
| partition name / PARTLABEL | partition entry | administrator renames it |
| filesystem UUID | filesystem | filesystem is created or UUID is changed |
| filesystem label | filesystem | administrator renames it |
| LUKS UUID | LUKS header | container is formatted or identity changed |
| LVM PV/VG/LV IDs | LVM metadata | LVM objects are created or modified |

Cloning a disk can duplicate identifiers. That may be desirable for an exact offline replacement and dangerous when both copies are attached simultaneously. Firmware entries, initramfs scripts, and fstab can select an unexpected duplicate.

Always state which UUID an error mentions. "UUID" by itself is ambiguous.

---

## 11. Read-only partition inspection

Start with several independent views:

~~~sh
lsblk -e7 -o PATH,SIZE,TYPE,PTTYPE,PARTTYPE,PARTTYPENAME,PARTLABEL,PARTUUID,FSTYPE,LABEL,UUID,MOUNTPOINTS
sudo fdisk -l
sudo blkid
findmnt --real
~~~

For GPT-specific validation, gdisk's verify operation or sgdisk's verification mode can check header and entry-array consistency. Read documentation for the installed version before running any repair action.

Capture exact device size and sector sizes. A common GPT warning after cloning to a larger disk says the backup header is not at the physical end. The data partitions may still work, but the backup metadata should be relocated using a tool that understands GPT. After cloning to a smaller disk, partitions may genuinely extend beyond the target, which is not a cosmetic warning.

### Raw inspection without interpretation

A hex dump of the first sector can reveal the MBR signature and bytes, but it is not a substitute for a parser:

~~~sh
sudo dd if=/dev/sdX bs=512 count=1 status=none | hexdump -C
~~~

This command reads when if= is correct, but device-name mistakes and copying examples into modified commands are dangerous. Prefer a disposable image file for learning:

~~~sh
truncate -s 128M disk-lab.img
fdisk -l disk-lab.img
~~~

Creating partitions in an ordinary image is much safer than practicing on a real disk.

---

## 12. Backing up metadata safely

A partition table is small compared with the data it indexes, but losing it can hide every filesystem at once. Back up metadata before planned layout work.

Useful approaches include:

- text output from fdisk and lsblk for human review;
- an sfdisk dump that can recreate supported layouts;
- a GPT-specific backup created by sgdisk;
- copies of EFI files and a listing of UEFI boot variables;
- filesystem-specific metadata backups where appropriate;
- a full tested data backup, because metadata backup does not preserve file contents.

Store the backup somewhere other than the disk being changed. Record sector size and exact device size. A binary metadata image restored to a different-size disk can place backup GPT structures incorrectly or overwrite new data.

Example read-only capture commands include:

~~~sh
sudo sfdisk --dump /dev/sdX
sudo sgdisk --print /dev/sdX
sudo efibootmgr -v
lsblk -f
~~~

To save output, use your normal secured administrative process. Review it for serial numbers and identifiers before publishing.

---

## 13. Recovery reasoning

### Primary GPT damaged, backup valid

A GPT-aware tool can often reconstruct the primary header and array from the backup. First prove the backup passes structural checks and partition boundaries match expected filesystems. Do not accept the first recovery prompt blindly.

### Backup GPT misplaced after disk expansion

If the source disk was cloned to a larger device, the backup structures may remain at the old end. A GPT tool can relocate them to the new final LBA, after which free space can be partitioned. This operation should not require recreating existing filesystems, but a backup is still mandatory.

### ESP filesystem damaged

The partition entry can be intact while FAT is corrupt. Repair or recreate the filesystem affects files and should follow a backup or recovery copy. After recreation, reinstall EFI applications and ensure Boot#### paths or fallback files match. Reformatting changes the FAT identifier and deletes loaders; changing the GPT type alone does neither.

### NVRAM entry missing, ESP intact

Boot the file manually through firmware if possible, use the fallback path, or recreate the entry with a tool such as efibootmgr from a UEFI-booted rescue environment. The partition and loader do not need reformatting merely because a variable was lost.

### Wrong partition type

If an existing FAT partition contains correct EFI files but has the wrong GPT type, correcting the type may make it discoverable. Verify contents and boundaries first. Never format as an automatic consequence of changing a type code.

---

## 14. Worked example: stale UEFI device path

A motherboard replacement preserves the disk, but firmware NVRAM is new. The ESP and Linux root are healthy. The new firmware reports no installed Linux entry.

Boundary analysis:

- the disk is visible, so basic hardware discovery works;
- GPT tools show the ESP, so partition metadata is readable;
- mounting the ESP shows shim and GRUB, so the filesystem and files exist;
- NVRAM contains no vendor entry, so the missing structure is selection metadata.

A minimal recovery is to create the correct Boot#### entry or place a suitable loader at the standardized fallback path. Reinstalling the root filesystem would not repair the missing variable.

After recreating the entry, verify:

1. its device path names the intended disk and partition GUID;
2. its file path matches case-insensitive FAT lookup expectations and actual directories;
3. Secure Boot accepts the first executable;
4. BootOrder includes the entry;
5. a cold reboot, not only a one-time firmware file browser, succeeds.

---

## 15. Worked example: correct ESP, wrong root

Firmware loads GRUB from the ESP. GRUB loads the kernel and initramfs. Early userspace cannot find the root UUID.

The ESP has already fulfilled its role. Formatting it is unrelated to the observed failure. Compare:

~~~sh
cat /proc/cmdline
blkid
ls -l /dev/disk/by-uuid
~~~

If the root filesystem was recreated, update the loader-generated command line and fstab as required, then rebuild the loader configuration using the distribution's supported method. If the identifier exists only after unlocking encryption or activating LVM, diagnose that layer in initramfs.

One physical disk contains both the successful boot filesystem and the failing root path, but they are different partitions, filesystems, identifiers, and stages.

---

## 16. Alignment, RAID, encryption, and complex layouts

Partition boundaries are only the outer map. Inside a partition may be:

- a software RAID member;
- a LUKS container;
- an LVM physical volume;
- a filesystem directly;
- swap;
- a vendor recovery format;
- another disk image or container.

Tools such as blkid identify signatures, but multiple stale signatures can remain after repurposing a partition. Wiping signatures is destructive and should occur only after the intended ownership is proven.

Firmware normally cannot interpret Linux software RAID, LUKS, or LVM directly. That is why the first boot files often live in a simple ESP, and why initramfs exists to assemble complex root storage. Some loaders can read encrypted or RAID layouts through their own modules, but that adds another independent implementation and key-handling boundary.

Keep the early path as simple as the threat model and operational requirements allow.

---

## 17. Common mistakes

### Formatting before identifying

"The partition is not booting, so format it FAT32" destroys evidence and may erase correct EFI files. Confirm partition type, filesystem health, and file paths separately.

### Treating flags as content

Changing a boot or esp flag in a partition tool changes metadata. It does not install an executable, create a filesystem, or add firmware variables.

### Restoring metadata to the wrong device

Device names can reorder. Resolve by stable hardware identity and capacity, then confirm. A partition-table restore to the wrong disk is immediately destructive.

### Repairing both GPT copies independently

The headers cross-reference each other and protect entry arrays with CRCs. Use a GPT-aware tool to reconstruct a consistent pair from the known-good state.

### Mounting the wrong ESP during updates

Systems with multiple disks can update an ESP firmware never selects. Compare the mounted partition GUID with the active boot entry and document redundancy policy.

---

## 18. Mechanism summary

MBR combines bootstrap space and a small partition table in the first sector, with extended partitions working around its limits. GPT uses typed GUID entries, checksummed primary and backup structures, and a protective MBR. Neither table contains a Linux filesystem simply by naming a range.

Native UEFI boot connects three structures:

1. a Boot#### variable and device path;
2. a GPT partition identified as an ESP;
3. a FAT file containing an EFI executable.

The executable may then apply another configuration layer to choose a kernel. Keeping these layers distinct makes both design and recovery safer.

You should now be able to identify disk GUIDs, partition GUIDs, filesystem UUIDs, and human labels without collapsing them into "the UUID." You should be able to explain why an intact ESP does not guarantee a valid NVRAM entry, why a valid entry does not guarantee a signed file, and why repairing a loader does not repair a corrupt partition table.

The next chapter follows the file selected from this storage and asks what the boot loader must do before it can hand the machine to Linux.

## 19. Disposable image lab

You can study partition metadata without touching a physical disk. Create a sparse ordinary file in a scratch directory, attach it to no important workflow, and use tools that understand disk images:

~~~sh
truncate -s 512M boot-layout-lab.img
parted -s boot-layout-lab.img mklabel gpt
parted -s boot-layout-lab.img mkpart ESP fat32 1MiB 129MiB
parted -s boot-layout-lab.img set 1 esp on
parted -s boot-layout-lab.img mkpart root ext4 129MiB 100%
fdisk -l boot-layout-lab.img
~~~

The file now contains a GPT and partition boundaries, but it does not yet contain filesystems. The words fat32 and ext4 supplied to this parted operation guide alignment or set type hints; they do not format the regions. This is an excellent demonstration of the difference between partition metadata and partition contents.

Map the image only on a disposable machine and only after reading the loop-device documentation for your system. A typical lab uses losetup with partition scanning, records the returned loop path, verifies that the path belongs to the image, formats the first mapped partition as FAT, mounts it, and creates an EFI directory tree. The critical safety property is that every later command uses the explicitly returned loop device rather than a guessed physical name.

After unmounting and detaching the image, inspect the first and final blocks. Corrupt a copy, never the only image:

~~~sh
cp --reflink=auto boot-layout-lab.img boot-layout-damaged.img
~~~

On the copy, overwrite only a carefully calculated header area, then ask GPT-aware tools what survived. Repeat with the final header instead. Observe that checksums detect damage and that the remaining header describes the location of its partner. The lesson is not a memorized repair command; it is how redundancy lets a parser distinguish one damaged structure from a coherent alternate.

Next, copy the image to a larger sparse file and observe the warning that the backup GPT is no longer at the physical end. The partitions have not moved, but one metadata invariant is stale because device size changed. Let a GPT-aware tool relocate the backup, verify both CRCs, and only then consider growing the final partition. This separates three operations that graphical tools may present as one gesture: repairing table placement, changing a partition boundary, and growing a filesystem.

Finally, compare identifiers before and after formatting only the FAT partition. The GPT partition GUID should remain stable while the filesystem UUID changes. Delete and recreate the partition entry with the same boundaries and observe the opposite possibility: filesystem bytes can remain while the partition GUID changes. That experiment makes the identifier table in this chapter concrete and explains many failures after cloning or manual reconstruction.

---

## Primary references

- UEFI specification and GPT chapter: https://uefi.org/specifications
- UEFI Boot Manager: https://uefi.org/specs/UEFI/2.11/03_Boot_Manager.html
- GNU Parted manual: https://www.gnu.org/software/parted/manual/parted.html
- util-linux fdisk and sfdisk manual pages: https://www.kernel.org/pub/linux/utils/util-linux/
- GPT fdisk documentation: https://www.rodsbooks.com/gdisk/
- Boot Loader Specification: https://uapi-group.org/specifications/specs/boot_loader_specification/
- efibootmgr source: https://github.com/rhboot/efibootmgr
- GNU GRUB BIOS installation: https://www.gnu.org/software/grub/manual/grub/html_node/BIOS-installation.html
