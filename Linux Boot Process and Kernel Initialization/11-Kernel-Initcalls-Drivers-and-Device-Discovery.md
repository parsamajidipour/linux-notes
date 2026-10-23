# Kernel Initcalls, Drivers, and Device Discovery

Hardware does not appear in Linux merely because firmware saw it.

Linux builds a device model, registers buses and drivers, enumerates devices, matches identifiers, allocates resources, probes hardware, and publishes results to sysfs and userspace. Much of that work is launched through initcalls: functions placed into ordered linker sections and invoked after the foundational kernel environment exists.

The word "detected" is too vague for this chapter. A PCI function can be enumerated but unmatched. A driver can match but fail probe. A controller can probe while exposing no child devices. A block device can exist while its partitions or higher storage mappings do not. Precise stages turn "Linux cannot see my disk" into a tractable claim.

---

## 1. Why initcalls exist

The kernel can include thousands of built-in components. start_kernel() cannot maintain a hand-written call for every filesystem, bus, protocol, and driver. Instead, initialization functions are registered through macros that place function pointers into linker sections.

Conceptual levels include:

- early;
- core;
- postcore;
- arch;
- subsys;
- fs;
- rootfs;
- device;
- late.

During do_initcalls(), the kernel iterates the section ranges in order and invokes functions. A driver built into the kernel may register through a device-initcall macro. The same driver built as a module runs its module initialization when loaded rather than during the built-in boot sequence.

Levels provide coarse ordering, not a complete dependency solver. Within a level, link order can influence invocation. Correct drivers should rely on explicit subsystem readiness and deferred probe rather than accidental ordering between unrelated objects.

---

## 2. initcall return values and timing

An initcall returns zero on success or a negative error code. The kernel can warn about unexpected interrupt state or preemption imbalance around calls. With initcall_debug, it logs function names, return values, and durations.

Enable temporarily:

~~~text
initcall_debug
~~~

Then inspect:

~~~sh
journalctl -b -k | grep -E 'calling .* @|initcall .* returned'
~~~

A long initcall is not automatically a bug. Hardware reset, link training, firmware loading, or timeout detection can legitimately take time. The name identifies which component to study.

Built-in initcall failure does not always stop boot. A driver can fail and leave one device unavailable while the system continues. If that device contains root, the consequence appears later as a root-mount failure.

---

## 3. The Linux device model

The driver core represents devices, drivers, buses, classes, and their relationships through common structures.

- A **bus** defines matching and organization, such as PCI, USB, platform, I2C, or SPI.
- A **device** represents an instance with parentage, resources, properties, and lifecycle.
- A **driver** provides operations and supported identifiers for devices on a bus.
- A **class** groups devices by function for userspace, such as block, net, tty, or input.

sysfs exposes this model:

~~~sh
ls /sys/bus
ls /sys/devices
ls /sys/class
~~~

/sys/class/block/nvme0n1 is a class-oriented link into the same underlying device hierarchy shown under /sys/devices. It is not a duplicate device.

Parent-child relationships control probe, power management, and removal ordering. A USB storage disk is a child of a SCSI representation, which depends on a USB interface, which depends on a USB device, host controller, and PCI function.

---

## 4. Bus enumeration

Different buses discover devices differently.

### PCI and PCI Express

The kernel scans configuration space, reading vendor and device IDs, class codes, capabilities, and resource requests. Bridges reveal subordinate buses. Firmware may have allocated resources; Linux validates and can reassign according to policy.

PCIe links may require training. Hotplug and advanced error reporting add asynchronous behavior. A function present in lspci proves configuration-space enumeration, not driver success.

### USB

A host-controller driver initializes hardware. The USB core detects connection, assigns an address, reads descriptors, chooses configurations, and creates interface devices. Interface drivers match class or vendor identifiers.

### Platform devices

Devices that are not self-enumerating can be instantiated from Device Tree, ACPI, or board code. Their resources—MMIO, IRQs, clocks, resets—come from firmware description.

### I2C and SPI

These buses usually cannot enumerate arbitrary devices reliably. Firmware descriptions state which devices are present and their addresses. A driver probes the declared device; blindly scanning can disturb hardware.

Enumeration is a bus property, not a universal driver trick.

---

## 5. Driver matching

When a device and driver exist on the same bus, the bus match function compares appropriate identifiers.

PCI drivers carry tables of vendor/device or class matches. USB drivers match descriptor fields. Device Tree drivers provide compatible strings. ACPI matching uses hardware IDs. Platform matching can also use names or firmware nodes.

Linux records module aliases so userspace can map a device modalias to a module:

~~~sh
cat /sys/bus/pci/devices/0000:00:00.0/modalias 2>/dev/null
modprobe --show-depends "$(cat /sys/bus/pci/devices/0000:00:00.0/modalias)" 2>/dev/null
~~~

Use an actual path. udev normally receives a modalias event and asks modprobe to load a matching module based on depmod-generated databases.

A built-in driver needs no module load. Its match table is already in the kernel.

---

## 6. Probe

Matching says a driver claims compatibility; probe establishes operation.

A probe commonly:

1. enables the device and maps resources;
2. configures DMA masks;
3. allocates driver state;
4. resets or initializes hardware;
5. loads device firmware if required;
6. registers interrupts;
7. creates queues and worker contexts;
8. registers child devices or a functional class object;
9. exposes attributes.

Every step can fail. Good probe code unwinds completed steps in reverse order. Managed resource APIs simplify cleanup tied to device lifetime.

Probe may run asynchronously for performance. Logs from different devices can interleave. The timestamp of a class device appearing may be later than the controller's first probe line.

---

## 7. Deferred probe

A device may be valid but depend on a supplier not ready yet: a regulator, clock, GPIO controller, PHY, IOMMU, reset controller, or another device.

Returning EPROBE_DEFER asks the driver core to retry later rather than treating absence as permanent failure. Firmware graph links and device links help order consumers after suppliers.

Deferred probe solves legitimate dynamic dependencies but can hide the root cause. A storage controller waiting for a missing regulator may appear only as repeated deferral until late boot.

Kernel debug information can show devices still deferred and reasons where supported:

~~~sh
cat /sys/kernel/debug/devices_deferred 2>/dev/null
~~~

Mounting debugfs changes exposure and may reveal sensitive data; use it deliberately.

---

## 8. Built-in drivers versus modules

A built-in driver is linked into vmlinux and initialized through initcalls. A module is a separate ELF object loaded after module infrastructure and a filesystem source are available.

Root-storage requirements determine placement:

- If the driver is built in, it can probe without initramfs module loading.
- If it is a module and root depends on it, the module and dependencies must be in initramfs.
- Once the real root is mounted, later modules can come from /lib/modules for the running release.

Module files must match kernel ABI/version expectations and signature policy. A module for yesterday's kernel may exist in initramfs but fail to load under today's kernel.

Check:

~~~sh
zgrep CONFIG_NVME_CORE /proc/config.gz 2>/dev/null
modinfo nvme 2>/dev/null | head
lsmod | head
~~~

Y means built in, M means module in kernel configuration. lsmod lists loaded modules, not built-in drivers.

---

## 9. Module dependency resolution

depmod scans installed modules and builds indexes for symbols, aliases, soft dependencies, and filenames. modprobe uses those indexes to load a requested name or alias plus dependencies.

In initramfs, a matching module tree and indexes must be included. Early userspace's modprobe reads the archive's /lib/modules/release, not the future root.

Failure patterns:

- module not found: absent file or wrong release path;
- unknown symbol: dependency/ABI mismatch or missing provider;
- invalid module format: vermagic, architecture, or signature problem;
- key rejection: signature enforcement does not trust signer;
- device remains unbound: module loaded but no identifier matches or probe failed.

One line saying "modprobe succeeded" proves module initialization returned success, not that a specific device bound.

---

## 10. Device firmware blobs

Many devices contain processors requiring firmware uploaded by the host driver: Wi-Fi adapters, GPUs, DSPs, storage controllers, and others. The driver calls the firmware loader API with a filename.

Userspace normally supplies files from /lib/firmware. Built-in firmware can be compiled into the kernel. During early boot, required files must be available in initramfs or built in because the real root is not mounted.

Logs show missing names:

~~~sh
journalctl -b -k | grep -i firmware
~~~

A firmware request can be asynchronous. Some drivers degrade gracefully; others cannot operate. Licensing and distribution packaging often separate firmware from kernel modules, so copying only modules into a custom initramfs is insufficient.

Device firmware is distinct from UEFI platform firmware and CPU microcode, despite shared terminology.

---

## 11. devtmpfs and device nodes

The kernel identifies devices internally through major/minor numbers and objects. Userspace traditionally needed mknod to create special files under /dev.

devtmpfs lets the kernel automatically create a baseline device-node filesystem as devices register. Early userspace mounts it, giving immediate nodes without a full policy daemon. udev later manages names, permissions, ownership, symlinks, and userspace actions.

The node is an interface to a driver, not the device itself. Deleting /dev/nvme0n1 does not remove the kernel device; devtmpfs or udev can recreate it. A node can also exist while open operations fail because driver state changed.

Stable symlinks under /dev/disk/by-* are userspace policy based on identifiers. Early root discovery may wait for those links, but the underlying block device appears first.

---

## 12. uevents and udev

The kernel emits uevents when devices are added, removed, or changed. Events contain environment-like fields including ACTION, DEVPATH, SUBSYSTEM, and MODALIAS.

systemd-udevd or another manager listens, loads modules, applies rules, creates symlinks, changes permissions, and triggers associated units. In initramfs, a small udev environment can drive storage discovery.

Observe in a lab:

~~~sh
udevadm monitor --kernel --udev --property
udevadm info --query=all --name=/dev/nvme0n1
~~~

Do not trigger all devices indiscriminately on production. Replaying events can start helpers or alter state.

Kernel and userspace events are separate timestamps. A delay between them can indicate a busy userspace queue rather than slow hardware probe.

---

## 13. From PCI function to mounted filesystem

Follow an NVMe root:

1. PCI core enumerates the controller function.
2. The nvme driver matches identifiers/class and probes.
3. Driver maps registers, configures DMA and interrupts, creates admin/I/O queues.
4. It identifies controller and namespaces.
5. Block layer registers /dev/nvme0n1.
6. Partition parser reads GPT and registers /dev/nvme0n1p1, p2, and so on.
7. uevents create stable links.
8. cryptsetup may create a device-mapper target.
9. LVM may discover a physical volume and activate a logical volume.
10. Filesystem driver mounts the logical block device.

Commands by layer:

~~~sh
lspci -nnk | grep -A3 -i non-volatile
ls /sys/class/nvme
lsblk -o NAME,TYPE,FSTYPE,PKNAME,MOUNTPOINTS
dmsetup ls --tree 2>/dev/null
pvs 2>/dev/null
findmnt /
~~~

Each successful line proves a later boundary.

---

## 14. Root-device timing

Device discovery can be asynchronous. USB storage, network block devices, slow RAID assembly, and controller firmware may appear after the kernel first tries root.

rootwait waits indefinitely for a root block device; rootdelay=N sleeps a fixed number of seconds. A fixed delay can mask a race and slow every boot. Event-driven initramfs logic is usually more robust: wait for the specific identifier and report which dependency is missing.

If adding rootdelay makes boot work, do not stop there. Identify why the device appears late and whether the correct driver, udev rules, storage assembly, or firmware is present.

---

## 15. Probe ordering and asynchronous work

Boot performance encourages parallel device probing, but dependencies constrain it. async_probe lets selected drivers probe concurrently. Work queues and firmware requests also introduce parallelism.

Ordering should use:

- parent-child relationships;
- supplier-consumer device links;
- deferred probe;
- subsystem registration completion;
- explicit synchronization when required.

Relying on log order or link order is fragile. A race that appears only on fast storage or many CPUs often means a missing dependency previously hidden by timing.

initcall_debug can show function duration, while driver-core debug and tracepoints can show probe. Instrumentation changes timing, so reproduce more than once.

---

## 16. Driver binding controls

sysfs exposes driver binding relationships:

~~~sh
readlink /sys/bus/pci/devices/0000:00:00.0/driver 2>/dev/null
ls /sys/bus/pci/drivers
~~~

Manual unbind and bind files exist for development and recovery, but writing them changes live hardware ownership. Unbinding a root controller will crash or hang the system. Use a disposable device in a VM for experiments.

driver_override can force a different match for specialized frameworks such as VFIO. It does not make an incompatible driver safe.

The safe read-only question is: which bus object exists, which modalias does it advertise, which module provides a matching alias, and which driver symlink is present?

---

## 17. IOMMU and DMA setup

Devices perform DMA to memory. An IOMMU translates and restricts device-visible addresses, supporting isolation, virtualization, and devices with limited masks.

Firmware tables describe IOMMU units and device relationships. Linux initializes the IOMMU subsystem, creates domains, and attaches devices. Drivers use the DMA API so the architecture and IOMMU layer can return appropriate DMA addresses.

A device can enumerate and probe partly but fail when DMA mapping is wrong. Boot parameters that disable an IOMMU can be diagnostic but weaken isolation and change addressing behavior.

Inspect complete logs:

~~~sh
journalctl -b -k | grep -Ei 'IOMMU|DMAR|AMD-Vi|SMMU'
find /sys/kernel/iommu_groups -maxdepth 2 -type l 2>/dev/null
~~~

Group boundaries matter for safe device assignment to virtual machines.

---

## 18. Driver model and power management

Device lifecycle includes suspend, resume, runtime idle, shutdown, and removal. The driver core orders callbacks using dependency and parent relationships.

Probe success does not guarantee resume success. Firmware may leave hardware in a different state after warm reset or sleep. A device that works on cold boot but not after resume points toward lifecycle transitions.

Runtime power management can suspend an idle device while the system remains running. Drivers balance usage counts and dependencies. Debugging by disabling all power management can classify a problem but costs energy and hides the broken transition.

---

## 19. Failure workflow

For a missing device:

1. Check firmware description or bus enumeration.
2. Identify modalias or compatible string.
3. Find the expected driver and whether built in or modular.
4. Confirm the module is available for the running kernel and, if early, in initramfs.
5. Inspect signature and symbol errors.
6. Check driver binding symlink.
7. Read probe logs, including deferred-probe reasons and firmware requests.
8. Check whether the driver registered the expected child/class object.
9. Follow higher layers instead of repeating probe.

Example evidence:

~~~sh
lspci -nnk
lsusb -t
modinfo DRIVER
journalctl -b -k
ls -l /sys/bus/BUS/devices/DEVICE/driver
~~~

Do not load random drivers against hardware. Matching tables and documentation exist for a reason.

---

## 20. Mechanism summary

Initcalls let built-in components register initialization functions in coarse dependency levels. The driver core represents buses, devices, drivers, classes, and links. Buses enumerate or instantiate devices; matching selects candidate drivers; probe establishes operation; deferred probe waits for suppliers. Built-in drivers initialize from the kernel image, while modules must be loaded from initramfs or the real root with matching ABI and trust.

Firmware blobs, devtmpfs, uevents, and udev bridge kernel discovery to userspace policy. A storage path is a chain from controller through namespace, partition, device mapper, volume manager, and filesystem.

The next chapter moves into early userspace, where those discovered devices are assembled into the root storage graph.

---

## Primary references

## 21. Case study: controller exists, disk does not

lspci shows an NVMe controller, and lspci -k shows the nvme driver bound. No /dev/nvme0n1 exists.

The PCI and binding boundaries succeeded. Continue inside the driver rather than reloading PCI support. Inspect kernel messages for controller reset, identify, queue allocation, timeout, fatal status, and namespace enumeration. Look under /sys/class/nvme for the controller and its state. A controller can be operational but expose zero active namespaces because firmware configuration, namespace attachment, or hardware failure differs from the expected disk model.

Check whether IOMMU faults or DMA-mask errors prevent admin commands. Compare cold power-on and warm reboot because controller firmware can retain bad state. A previous kernel comparison tests a driver change while holding hardware constant. Rebinding the driver changes state and is unsafe when any namespace carries live filesystems.

The key conclusion is narrow: PCI enumeration and driver match do not imply child block-device creation.

## 22. Case study: module loads, device stays unbound

modprobe returns success, lsmod lists the module, but a PCI device's driver symlink is absent.

Module initialization registered a driver; it did not guarantee a match. Compare the device modalias with modinfo aliases. The hardware revision may be newer than the driver's table, another driver may own it, driver_override may be set, or a firmware node property may disable it.

If aliases match, read probe logs. A failed probe normally leaves the device unbound. Error codes can represent missing resources, dependency deferral, unsupported configuration, or hardware failure. An EPROBE_DEFER is not final rejection, but perpetual deferral points to a supplier.

Do not force-bind by writing sysfs merely because the class name looks close. Matching tables encode compatibility knowledge. For development, add the exact ID to a test build or supported new_id interface only with driver documentation and disposable hardware.

## 23. Case study: driver works after real root but not in initramfs

Boot drops to an early shell. Manually copying the driver module from the mounted real root and loading it makes the disk appear.

This proves hardware, current kernel driver, and basic probe can work. The early archive is missing the module, dependency, firmware, or configuration. Compare uname-equivalent kernel release with /lib/modules inside initramfs. List the archive and locate the module, dependency indexes, and device firmware.

A common error is rebuilding initramfs while /boot is not mounted, producing a correct file on the root filesystem that the loader never reads. Another is updating the kernel without installing its modules before generating the archive. A third is host-only generation before migrating from SATA to virtio or NVMe.

Rebuild a versioned image, inspect it, confirm the loader's exact path, and retain the old entry. The fact that manual modprobe works is a mechanism-level diagnostic, not a reason to add a blind sleep.

## 24. Sysfs path-reading exercise

Choose a noncritical device and resolve its chain:

~~~sh
dev=/sys/class/net/eth0
readlink -f "$dev"
udevadm info --attribute-walk --path="$(readlink -f "$dev")"
~~~

The real path encodes parent devices. At each parent, inspect subsystem, modalias, driver link, NUMA node, resources, and power state. Do not write attributes.

Now map the userspace name back to the kernel object. Predict which parent would disappear if the PCI function were removed, which driver owns each level, and which uevent created the class link. This exercise makes the device model a tree rather than a pile of /dev names.

## 25. Driver initialization review checklist

For code review, ask:

- Is every probe allocation checked and unwound in reverse order?
- Can an interrupt fire before state publication?
- Are DMA APIs used instead of physical-address guesses?
- Does probe return EPROBE_DEFER only for a real supplier dependency?
- Are firmware filenames stable and packaged for early boot?
- Does remove stop new work before freeing queues and IRQs?
- Do suspend, resume, shutdown, and hotplug callbacks preserve ordering?
- Is module init merely registering a driver rather than probing global hardware directly?
- Does asynchronous work hold a valid device reference?
- Can built-in initialization run before the filesystem or userspace helper it assumes?

These questions connect boot correctness to the full device lifecycle. A probe path that works once on a developer machine can still fail during deferred probe, initramfs boot, hot removal, or resume.

## 21. Three boundary case studies

**Controller exists, disk does not.** lspci shows an NVMe controller and its driver, but no namespace block device exists. PCI enumeration and binding succeeded. Inspect controller reset, identify, queue allocation, IOMMU faults, timeouts, and namespace enumeration. A controller may expose zero active namespaces. Reloading the PCI core targets an already successful layer.

**Module loads, device stays unbound.** modprobe success means module initialization registered a driver. Compare the device modalias with modinfo aliases. If they match, inspect probe failure or deferral. If they do not, the revision may be unsupported or another driver intended. Forcing bind does not make incompatible hardware compatible.

**Driver works after real root but not in initramfs.** The early archive may lack module, dependency, firmware, signature, or index. Confirm the archive's /lib/modules release matches the kernel. A common mistake rebuilds an image while /boot is unmounted, leaving a correct file where the loader never reads it.

Each case starts from the last proof and moves one edge forward.

## 22. Sysfs path exercise

Choose a noncritical device and resolve its real path:

~~~sh
readlink -f /sys/class/net/eth0
udevadm info --attribute-walk --path="$(readlink -f /sys/class/net/eth0)"
~~~

At each parent inspect subsystem, modalias, driver link, NUMA node, resources, and power state without writing. Predict which children disappear if a parent is removed and which driver owns each level. This turns /sys from a pile of files into the device-lifecycle tree.

## 23. Driver review checklist

Ask whether probe publishes state before unmasking interrupts; every allocation unwinds in reverse order; DMA uses the DMA API; deferred probe represents a real supplier; required firmware ships in early userspace; remove stops new work before freeing queues; asynchronous callbacks hold valid references; and suspend/resume/shutdown mirror probe ordering.

Also ask whether a built-in driver assumes a userspace helper or filesystem not yet available. A module can work perfectly after login and fail as built-in because its execution phase changed.

## 24. Driver-core tracing lab

In a VM, enable initcall_debug and only the driver-core tracepoints relevant to probe. Boot once with the controller driver built in and once as a module included in initramfs. Record when the bus registers, device appears, driver registers, match occurs, probe begins, class object appears, and udev handles the event.

The functional result can be identical while timing and ownership differ. Built-in code is available before initramfs, but modular loading lets userspace policy choose. Compare logs without assuming earlier is always better: early probe can lengthen the serialized kernel phase, while asynchronous or modular probe can race root policy if dependencies are incomplete.

## 25. Deferred-probe case study

An embedded storage controller's driver matches but returns EPROBE_DEFER because its clock provider is not registered. The clock provider itself waits for an I2C regulator whose Device Tree node is disabled by mistake. The visible root symptom is "storage device missing," but retrying the storage probe cannot create the regulator.

Read devices_deferred reasons, follow firmware phandles or ACPI dependencies from consumer to supplier, and find the first supplier that failed permanently. Correct the description or supplier driver. Adding rootdelay merely gives an impossible dependency more time.

Deferred probe is therefore a graph, not a queue of slow devices. A leaf failure propagates through consumers. Device links improve ordering and power management, but firmware must describe relationships accurately.

## 26. Naming and stable identity

Kernel enumeration names such as eth0, sda, and card0 can vary with probe order. udev creates stable policy names from topology, MAC address, filesystem UUID, WWN, or administrator rules. Stability depends on the chosen property: a topology-based network name changes when a card moves slots; a MAC-based name follows the card; a filesystem UUID follows cloned bytes unless changed.

Boot policy should select the identity whose failure semantics match the deployment. Using /dev/sda because it was first once turns harmless parallel probe timing into root selection risk.

Finally, remember that probe success is provisional. Runtime power management, reset, hotplug, and error recovery revisit much of the same state. A driver that survives boot only because firmware initialized the device is incomplete; test after a function-level reset and cold start so Linux proves it owns initialization.

- Driver model overview: https://docs.kernel.org/driver-api/driver-model/overview.html
- Driver binding: https://docs.kernel.org/driver-api/driver-model/binding.html
- Device links: https://docs.kernel.org/driver-api/device_link.html
- Firmware loader: https://docs.kernel.org/driver-api/firmware/
- Driver core source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base
- initcall source and macros: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/init.h
- PCI documentation: https://docs.kernel.org/PCI/
- DMA API: https://docs.kernel.org/core-api/dma-api.html
- devtmpfs configuration: https://docs.kernel.org/admin-guide/devices.html
