# Early Memory Management

Linux needs memory in order to build the machinery that manages memory.

That circular dependency is one of the central boot problems. The kernel arrives with a firmware-provided map and several objects already occupying physical addresses: its own image, boot parameters, firmware runtime regions, an initramfs, platform tables, and possibly reserved crash or device areas. It must allocate page tables and metadata before the normal page allocator exists. It must then describe every usable page, divide memory into topology and allocation domains, initialize free lists, build small-object caches, and finally retire the temporary mechanisms.

This chapter follows that transition. The key distinction is between **discovering physical address ranges**, **reserving ownership**, **creating mappings**, and **making pages allocatable**. A byte can be present in DRAM but unavailable to Linux; mapped but reserved; free but outside a requested zone; or managed by another NUMA node.

---

## 1. Physical memory is not one flat pool

Firmware reports ranges with types and attributes. Some ranges are ordinary RAM. Others contain firmware code, runtime data, ACPI tables, persistent memory, device windows, security-reserved storage, or holes where no DRAM exists.

The physical address space also includes memory-mapped I/O. A CPU load to an MMIO address can access a device register, not a DRAM cell. Adding every address below installed capacity to a free list would be catastrophic.

Linux therefore begins with a map, not a number.

On x86, legacy and UEFI information is converted into an e820-style internal map and architecture resources. On Device Tree systems, memory nodes and reserved-memory entries contribute. ACPI can describe NUMA relationships and special-purpose regions.

After boot:

~~~sh
cat /proc/iomem
journalctl -b -k | grep -Ei 'e820|BIOS.*map|EFI.*mem|reserved|NUMA'
~~~

Address visibility may be restricted. The map shows ownership labels, not whether every page is currently resident in a process.

---

## 2. Installed, addressable, usable, and available

Several memory totals coexist:

- **Installed memory:** hardware capacity physically present.
- **Firmware-usable memory:** ranges firmware offers as ordinary RAM.
- **Kernel-managed memory:** usable ranges after kernel reservations and architecture exclusions.
- **Currently available memory:** a dynamic estimate after allocations, caches, reclaimable pages, and workload.

A machine with 16 GiB installed will not show exactly 16 GiB as free after boot. Firmware, devices, page metadata, the kernel image, initramfs, and persistent reservations consume or hide portions. The difference is not automatically a leak.

Commands:

~~~sh
grep -E 'MemTotal|MemFree|MemAvailable|CmaTotal|HugePages' /proc/meminfo
free -h
~~~

MemFree is not the amount applications can safely allocate. Linux uses spare memory for caches and reports MemAvailable as a better estimate of allocation headroom.

---

## 3. memblock: the early allocator and ownership map

memblock is a boot-time mechanism representing physical memory and reserved regions. Architecture setup adds usable ranges to the memory collection and marks occupied ranges reserved. Early initialization can then allocate aligned physical areas before the buddy allocator is ready.

Conceptually, memblock tracks:

~~~text
memory:   ranges Linux may potentially manage
reserved: ranges that must not be handed out
~~~

An allocation finds a suitable portion of memory, honors alignment and limits, and reserves it. The API supports NUMA-aware selection and architecture constraints.

Objects allocated this way include early page tables, page metadata support, per-CPU regions, and other foundational structures. Allocation policy may prefer low or high addresses depending on architecture and current phase.

memblock is not a general runtime allocator. Most configurations discard or reduce its role after boot once pages are released to the normal allocator. Debug parameters can make its operations visible, but output is extremely verbose.

---

## 4. Reserving what already exists

Before freeing pages, Linux reserves all live boot objects:

- physical kernel image and its sections;
- initramfs location;
- command line and parameter structures until copied;
- firmware runtime code and data;
- ACPI and Device Tree regions as required;
- framebuffers and special platform ranges;
- crash-kernel reservation;
- initrd and microcode data;
- DMA pools and contiguous-memory areas;
- architecture trampoline or low-memory structures;
- persistent or secure-world regions.

Reservation is an ownership statement. It prevents the allocator from giving a page to an unrelated user while its current consumer still expects stable bytes.

A bug in reservation can remain invisible until the page is reused. Early boot may succeed, and later allocation overwrites a firmware table or initramfs buffer. This delayed corruption is why exact size and alignment in boot protocols matter.

---

## 5. Page tables and the kernel address space

The MMU translates virtual addresses through page tables. Early entry created temporary mappings; memory initialization builds the longer-lived kernel layout.

Common x86-64 regions include:

- the kernel text, read-only data, writable data, and BSS mapping;
- a direct mapping of physical memory into a high kernel virtual range;
- vmalloc space for nonphysically contiguous mappings;
- module space;
- fixmap slots;
- per-CPU mappings;
- vmemmap or other storage for page descriptors;
- architecture and sanitizer regions.

Exact addresses vary with configuration, paging level, KASLR, and architecture.

The direct map lets the kernel translate many physical pages to kernel virtual addresses cheaply. It does not mean every physical address is safe RAM, and permissions can be adjusted for hardening or special memory.

Large pages can reduce page-table overhead for broad linear mappings. Later splitting may be needed to set fine-grained permissions or remove mappings.

---

## 6. struct page: describing physical pages

The kernel needs metadata for each managed page frame: flags, reference counts, mapping state, allocator linkage, compound-page information, and other unioned fields. The central type is struct page, with newer development exploring folio abstractions for groups of pages at higher layers.

Metadata itself consumes memory. For large systems, describing every small page requires substantial space. Architectures and configurations store this metadata in schemes such as FLATMEM, DISCONTIGMEM historically, or SPARSEMEM with optional virtual memmap arrangements.

Sparse representations handle holes and hot-pluggable sections without allocating a dense descriptor array for every possible physical address.

The bootstrapping challenge appears again: Linux allocates and initializes page descriptors before those pages can be managed by the allocator they describe. memblock and carefully ordered setup solve it.

---

## 7. NUMA topology

Non-Uniform Memory Access systems attach memory with different access costs to different CPUs. Firmware tables such as ACPI SRAT describe affinity domains; SLIT or HMAT can describe relative distance and performance attributes. Device Tree can describe NUMA nodes on applicable platforms.

Linux builds nodes and assigns physical ranges and CPUs. Allocation policy tries to use local memory while falling back when needed.

NUMA affects boot because metadata and per-node allocators must exist before secondary CPUs and general allocation. Missing or incorrect firmware topology can collapse a machine into one node or assign memory strangely.

Inspect:

~~~sh
lscpu | grep -i numa
numactl --hardware 2>/dev/null
find /sys/devices/system/node -maxdepth 1 -name 'node*' -print
~~~

A virtual machine can expose configurable virtual NUMA independent of the host's physical layout. Bad virtual topology can hurt performance even when boot succeeds.

---

## 8. Zones

Not every physical page is equally usable by every device or kernel path. Linux groups pages into zones representing allocation constraints.

Historical and current zone names can include:

- **DMA:** low-address memory reachable by devices with severe DMA limits;
- **DMA32:** memory below a 32-bit address boundary for devices unable to address higher RAM;
- **Normal:** ordinary directly mapped kernel memory;
- **HighMem:** on 32-bit systems, memory not permanently mapped into kernel virtual space;
- **Movable:** pages intended to support migration and memory hot removal;
- **Device:** special device-managed memory in applicable configurations.

The exact zone set is architecture and configuration dependent. A 64-bit server commonly has DMA, DMA32, and Normal; HighMem is mainly a 32-bit concern.

Allocations include GFP flags describing context and constraints. The allocator chooses a suitable zone and may fall back according to zonelists. A system can have free memory overall and still fail an allocation requiring scarce DMA32 or a high-order contiguous block.

Inspect zone state:

~~~sh
cat /proc/zoneinfo
cat /proc/buddyinfo
~~~

These are detailed diagnostic interfaces, not simple health scores.

---

## 9. The buddy allocator

The buddy allocator manages physically contiguous blocks in power-of-two page counts. Order 0 is one page, order 1 is two contiguous pages, and so on.

When a block is freed, the allocator checks whether its same-size buddy is free. If so, they merge into the next order. When a smaller block is requested, a larger block can be split repeatedly.

During boot, Linux initializes zone free lists and releases eligible pages from memblock reservations into the buddy system. From then on, alloc_pages and related APIs can provide pages according to GFP policy.

External fragmentation can leave many free individual pages but no large contiguous block. Compaction and reclaim help at runtime. Boot-time contiguous reservations such as CMA reserve flexibility for devices needing large DMA buffers.

/proc/buddyinfo displays counts by zone and order. A row with many low-order blocks and zero high-order blocks is not necessarily failure, but it explains why a high-order request may struggle.

---

## 10. Slab-family allocators

Kernel code usually wants objects, not raw page blocks: an inode, socket, task structure, dentry, or fixed-size private structure. Slab-family allocators obtain pages from the buddy allocator and divide them into object caches.

Linux commonly uses SLUB, though configuration can differ. Caches provide:

- efficient reuse of initialized object-sized regions;
- reduced internal fragmentation compared with dedicating pages per object;
- per-CPU fast paths;
- debugging features for corruption, red zones, poisoning, and leaks;
- constructors for selected caches.

kmalloc maps size classes onto general caches, while kmem_cache_create builds specialized caches.

Initialization is staged. Very early caches support the structures needed to create the full cache hierarchy. Code running before slab availability must use explicit early mechanisms.

Inspect:

~~~sh
head /proc/slabinfo
slabtop -o 2>/dev/null | head
~~~

Permissions and configuration may restrict these interfaces because allocation patterns can expose security-relevant information.

---

## 11. vmalloc and noncontiguous memory

vmalloc reserves a contiguous virtual range and maps potentially noncontiguous physical pages underneath. This is useful for large kernel buffers that do not require physically contiguous DMA memory.

It has overhead: page tables must be populated, translation locality can be lower, and virtual address space is consumed. kmalloc returns physically contiguous memory for sizes within practical limits and is preferred for many objects.

The vmalloc subsystem needs page allocation and mapping infrastructure, so it appears after foundational memory setup. Kernel modules commonly use vmalloc-like space for executable sections, subject to permission transitions.

Confusing virtual contiguity with physical contiguity causes driver bugs. A device performing DMA needs addresses mapped through the DMA API, not a raw vmalloc virtual pointer.

---

## 12. Boot memory reclamation

Several regions are needed only during initialization:

- functions marked __init;
- data marked __initdata;
- temporary page tables;
- discarded alternatives or relocation metadata;
- original initramfs buffer after unpacking when safe;
- memblock metadata or reserved ranges whose owner has finished.

Linker scripts group init sections. After initcalls complete, free_initmem() and related paths release eligible pages. Calling an __init function afterward is a use-after-free bug; the memory may already contain unrelated data.

Kernel logs often report memory freed from unused kernel image sections. This is planned reclamation, not pressure.

The __ro_after_init annotation handles a related pattern: data is writable during setup and made read-only afterward, reducing the attack surface without freeing it.

---

## 13. initramfs memory ownership

The loader places the compressed initramfs in physical memory and reports its range. The kernel reserves that range so general allocation cannot overwrite it. Later, unpacking writes files into rootfs, allocating pages through normal kernel mechanisms.

After successful unpacking and when no consumer needs the original archive, the input pages can be released. The unpacked files consume their own memory. A large initramfs can therefore temporarily require both compressed input and expanded output at once.

This matters on small systems. An archive that fits in storage may fail during boot because peak memory includes kernel, decompressor, archive, unpacked files, page metadata, and early allocations.

Use a smaller host-only image, a faster/lower-memory compression algorithm, or more RAM after measuring which phase fails.

---

## 14. Memory hotplug and special-purpose memory

Boot constructs the initial memory topology, but servers can add or remove memory later. Sparse memory models, movable zones, and device/firmware coordination support hotplug.

Not every firmware-described range becomes ordinary System RAM. Persistent memory, device-private memory, CXL capacity, secure-world reservations, and platform-specific regions may be managed by drivers or exposed through specialized APIs.

The label "memory" therefore does not imply eligibility for anonymous process pages. Early boot classification determines which subsystem may claim it.

Firmware options such as memory mirroring, interleaving, NUMA clustering, and specific-purpose marking can change Linux topology without changing installed DIMMs.

---

## 15. Kernel memory protections

Initialization gradually tightens permissions:

- executable text becomes read-only and executable;
- read-only data becomes nonwritable;
- writable data becomes nonexecutable where architecture supports it;
- init sections are freed;
- ro_after_init data becomes read-only;
- direct-map aliases may receive restricted permissions;
- page-table protections can prevent accidental modification.

The W^X principle avoids mappings that are writable and executable simultaneously. Temporary transitions are needed when loading modules or applying alternatives, but final permissions should be narrow.

Logs may report NX availability and write-protect tests. Secure Boot does not replace these runtime protections; they defend against different attack steps.

---

## 16. Worked lifecycle of one range

Suppose firmware marks 0x40000000 through 0x7fffffff as conventional RAM.

1. Architecture setup adds the range to memblock memory.
2. A subrange containing initramfs is marked reserved.
3. Another subrange is chosen for page-descriptor metadata and reserved.
4. NUMA discovery associates the range with node 0.
5. Zone calculation places eligible pages in Normal.
6. Page structures are initialized.
7. Unreserved pages enter buddy free lists.
8. The slab allocator requests pages and builds object caches.
9. Initramfs is unpacked into pages obtained through normal allocation.
10. The original initramfs range is unreserved and freed.
11. Runtime allocations reuse those physical pages for unrelated purposes.

At step 2, the RAM is real but not allocatable. At step 7, it is managed and free. At step 9, other pages are allocated even though the source remains reserved. At step 10, ownership ends. "Is this memory free?" requires a time and owner.

---

## 17. Diagnostics and controlled experiments

Capture:

~~~sh
dmesg | grep -Ei 'Memory:|memblock|NUMA|zone|freeing.*memory|CMA|reserved'
cat /proc/meminfo
cat /proc/iomem
cat /proc/buddyinfo
cat /proc/zoneinfo
~~~

In a VM, vary assigned RAM while keeping kernel and initramfs constant. Observe:

- firmware map changes;
- page metadata overhead;
- zone sizes;
- initramfs peak-memory behavior;
- KASLR placement;
- whether low-memory thresholds change failure mode.

Then reserve memory with a documented command-line parameter in a disposable guest and compare /proc/iomem. Do not use mem= or memmap= casually on production: incorrect values can hide RAM the kernel needs or expose unsafe regions.

For development kernels, CONFIG_DEBUG_MEMORY_INIT, page poisoning, KASAN, kmemleak, and slab debugging can expose errors at substantial performance and memory cost.

---

## 18. Failure patterns

### Firmware reports less memory after an upgrade

Check training, disabled DIMMs, error logs, reserved/specific-purpose policy, and topology. Compare firmware inventory with the Linux map.

### Kernel panics during early allocation

The map may be malformed, reservations may consume candidates, alignment may be impossible, or the image/initramfs may be too large. Preserve the exact map and earliest log.

### Plenty of free memory but DMA allocation fails

The request may require a constrained zone or contiguous order. Inspect DMA32 and buddy state, device DMA mask, IOMMU configuration, and CMA policy.

### High-order allocation warnings

Runtime fragmentation can prevent contiguous blocks. Determine whether the caller can fall back, whether the order is justified, and whether compaction or CMA is appropriate. Total free memory alone is insufficient.

### Crash only with KASLR enabled

Investigate incorrect reserved ranges, relocation, firmware mappings, and drivers assuming addresses. Disabling KASLR is a comparison, not a final security-neutral fix.

---

## 19. Mechanism summary

Early memory management converts a firmware description into kernel-owned allocation systems. memblock records usable and reserved physical ranges and serves allocations before the normal allocator exists. Page tables create the kernel virtual layout. struct page metadata describes managed pages. NUMA nodes and zones encode locality and constraints. The buddy allocator manages page blocks, and slab-family allocators turn pages into efficient object caches. Initialization-only memory is then freed or protected.

The important vocabulary is now precise:

- present DRAM is not automatically usable;
- usable RAM is not automatically free;
- a free page may not satisfy zone or contiguity constraints;
- a virtual contiguous range may not be physically contiguous;
- reservation is a time-scoped ownership claim;
- initramfs can create a significant temporary memory peak.

The next chapter adds concurrency. It explains how exception handling, interrupt controllers, clocks, scheduler state, per-CPU data, and secondary processors turn the single boot CPU into a functioning SMP system.

---

## Primary references

## 20. Worked allocation example

Suppose a network driver requests a 64 KiB coherent DMA buffer. The request is not equivalent to asking for sixteen arbitrary free 4 KiB pages.

The driver first establishes a DMA mask describing addresses the device can generate. The DMA API and IOMMU policy decide whether physical memory anywhere can be translated into the device's reachable address space. Without a suitable IOMMU translation, a 32-bit device may require pages from DMA32.

The allocator calculates an order sufficient for physically contiguous pages. Sixty-four KiB with 4 KiB base pages is order 4. The selected zonelist is searched according to GFP flags and NUMA policy. Watermarks protect reserves needed for forward progress. If no order-4 block exists, reclaim or compaction may run when context permits. An atomic interrupt-context request cannot sleep for the same work.

If allocation succeeds, the DMA layer establishes or verifies a device-visible mapping and returns both a CPU virtual address and a DMA address. Those two numbers are not interchangeable. With an IOMMU, the DMA address can be unrelated to physical address. The driver programs the device with the DMA address and accesses memory through the CPU pointer.

Now imagine /proc/meminfo reports gigabytes available but DMA allocation fails. The system may have free pages scattered across order-0 lists, while no order-4 block exists in the constrained zone. Or the IOMMU aperture is exhausted. Or the device mask was set incorrectly. "Out of memory" is the outcome of a constrained request, not always exhaustion of total bytes.

This example connects boot-time zone construction to runtime driver behavior. Incorrect firmware reservations or zone boundaries can surface much later as a DMA failure.

## 21. Huge pages and boot reservation

Huge pages reduce translation overhead by mapping larger regions with one TLB entry. Transparent Huge Pages assemble them dynamically where possible. Explicit hugetlb pages can be reserved at boot, when physical memory is unfragmented.

Parameters such as hugepagesz, default_hugepagesz, and hugepages influence pools according to architecture. Reserving too many removes memory from ordinary allocation; reserving too few may make later contiguous allocation fail.

Inspect:

~~~sh
grep -E 'Huge|AnonHuge' /proc/meminfo
find /sys/kernel/mm/hugepages -maxdepth 2 -type f -print -exec cat {} \; 2>/dev/null
~~~

A hugetlb page is not ordinary free memory while in its pool. Applications need matching mappings and limits. NUMA placement also matters: a global count can hide shortage on the node where a workload runs.

## 22. Out-of-memory is later policy

Boot constructs allocators; runtime pressure adds reclaim and the out-of-memory killer. Reclaim writes dirty pages, discards clean cache, and invokes shrinkers. Memory cgroups impose local limits. If progress remains impossible, OOM policy selects a victim or panics according to configuration.

Early boot can fail before normal OOM recovery exists. An allocation needed to initialize reclaim, work queues, or filesystems cannot rely on those same facilities to recover. Reserved pools and careful GFP contexts preserve forward progress.

This is why fault-injection and tiny-memory VM tests are valuable. A kernel that boots with abundant RAM can contain invalid assumptions about allocation success. Test with constrained memory, large initramfs, and debug allocation failure only in disposable environments, then ensure error paths release every reservation.

## 23. Memory-map audit exercise

Capture the firmware-derived map from the earliest kernel log, /proc/iomem after boot, NUMA node ranges, zone sizes, and MemTotal. Reconcile them instead of expecting identical totals.

Start with firmware-usable ranges. Account for the kernel image, initrd during its early lifetime, firmware runtime pages, ACPI data that stays reserved, crashkernel, CMA, persistent-memory classifications, page descriptors, and page tables. Compare the resulting order of magnitude with the memory initialization line and /proc/meminfo.

Exact byte reconciliation is difficult because reservations change over time and logs round units. The exercise still exposes large unexplained gaps or a forgotten reservation. Repeat in a VM after adding one GiB, then enable a crashkernel reservation and observe its ownership. "Linux stole memory" becomes an auditable ledger.

## 24. Fragmentation thought experiment

Imagine a zone with 1024 free order-0 pages but no adjacent pair. It has four MiB free with four-KiB pages, yet an order-1 request fails. Reclaim cannot help if each allocated separator is unreclaimable. Compaction can migrate movable pages, while pinned DMA, page tables, and some slabs block it.

Now reserve a large Movable zone. It helps migration and memory hot removal, but kernel allocations still need Normal. Too much Movable memory can pressure Normal while total free remains high. This is why /proc/buddyinfo and zone watermarks complement /proc/meminfo: capacity, contiguity, locality, and mobility are independent.

## 25. Stale mappings and security

Freeing physical pages is not enough if an unintended writable or executable alias remains. Kernel hardening removes broad early mappings, marks text read-only, and flushes TLBs when permissions change. Module loading transitions freshly populated memory from writable to executable rather than keeping W+X.

A writable alias to executable text can bypass protection. Auditing early page-table teardown is therefore security work, not only cleanup. KASLR obscures placement; strict permissions constrain modification after placement is learned.

## 26. CMA case study

A camera device requires large physically contiguous DMA buffers. Without planning, runtime memory becomes fragmented and high-order allocation fails after hours of use. The Contiguous Memory Allocator reserves a region at boot whose pages can serve movable ordinary allocations until a device needs a contiguous range. The allocator migrates movable occupants and returns the assembled span.

This is a compromise: capacity is not permanently wasted, but unmovable allocations must not pollute the area. Too small a CMA region starves the camera; too large can constrain Normal memory and workloads with pinned pages.

Inspect CmaTotal and CmaFree in /proc/meminfo and the boot log. Device Tree reserved-memory or cma= can set size. Change it only after measuring peak buffers and the rest of the system. A successful allocation immediately after boot does not prove long-running reclaim works.

The example illustrates why early physical placement influences a driver hours later. Boot creates zones and reservations that runtime allocation cannot cheaply reinvent.

## 27. Review questions

Why can RAM appear in /proc/iomem but not MemTotal? Which allocations need physical contiguity? Why does a 32-bit DMA mask matter on a 64-bit CPU? When may initramfs source pages be freed? Which metadata describes a page that stores the metadata itself? How does NUMA change a "free page" request? What guarantee is lost by booting with mem= smaller than hardware?

Answer each using a concrete range and owner. If the answer is only "the kernel reserves it," find the exact subsystem and lifetime.

A useful final diagram has four columns: physical range, current owner, virtual mappings, and release event. Update it at loader handoff, allocator initialization, initramfs unpack, and init-memory release. The diagram makes aliases and stale reservations visible and forces every "free" claim to include a moment in time.

Repeat the diagram after hibernation resume or kexec. Those paths inherit or reconstruct memory ownership differently from a cold boot, so an assumption that is safe after firmware reset may be unsafe when another kernel prepared the handoff.

- Boot-time memory management: https://docs.kernel.org/core-api/boot-time-mm.html
- Physical memory model: https://docs.kernel.org/mm/physical_memory.html
- Page allocation: https://docs.kernel.org/mm/page_allocation.html
- Memory management index: https://docs.kernel.org/mm/
- Linux kernel memory source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm
- x86 memory setup source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/kernel/setup.c
- NUMA memory policy: https://docs.kernel.org/admin-guide/mm/numa_memory_policy.html
- SLUB documentation: https://docs.kernel.org/admin-guide/mm/slab.html
- Kernel address sanitizers: https://docs.kernel.org/dev-tools/kasan.html
