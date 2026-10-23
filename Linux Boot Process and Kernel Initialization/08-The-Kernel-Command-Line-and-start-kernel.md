# The Kernel Command Line and start_kernel()

The decompressed kernel is executing, but the word "kernel" still describes a program under construction.

There is one boot CPU, interrupts are constrained, ordinary scheduling is not available, memory management is incomplete, and most drivers have never run. Architecture code has established enough state to call into common initialization. The function start_kernel() in init/main.c is the best-known center of that transition.

It is tempting to read start_kernel() as a checklist of function calls. A better reading asks why each call is legal at that point and what new assumptions it permits later calls to make. Boot order is a dependency graph serialized just long enough to build safe concurrency.

This chapter first explains how the command line reaches multiple consumers, then follows the major dependency groups around start_kernel(). Exact calls move between kernel versions; the invariants are more stable than line numbers.

---

## 1. Architecture code arrives with a machine description

Before start_kernel(), architecture-specific code has already performed critical work:

- entered the kernel's virtual-address environment;
- established bootstrap page tables and stacks;
- preserved boot parameters;
- prepared CPU features needed for C code;
- installed minimal exception state;
- identified enough platform information to continue.

The common kernel cannot be completely architecture-neutral. It calls setup_arch(), which imports and normalizes architecture data: memory maps, reserved ranges, command-line storage, firmware tables, initrd location, CPU topology hints, and architecture resources.

On x86, arch/x86/kernel/setup.c is central. On another architecture, setup_arch() has a different implementation and consumes Device Tree, ACPI, or firmware data according to that platform's contract.

The division is useful: common initialization can say "initialize the page allocator" after architecture code describes physical memory without knowing how an x86 E820 map or arm64 memory node was obtained.

---

## 2. The command line is shared transport

The loader supplies one command-line string, but several stages parse it.

### Early kernel parameters

Some values must affect initialization before the general parameter framework or target subsystem exists. Handlers registered as early parameters are processed during an early parse. Examples can influence debug output, memory layout, architecture behavior, or console setup.

### Normal kernel parameters

Subsystem setup handlers and module-style parameters consume values during later parsing and initialization. The kernel parameter documentation lists thousands of options, but availability depends on build configuration.

### Initramfs parameters

The kernel preserves the full command line in /proc/cmdline. Early userspace reads it and implements policies such as rd.luks, rd.lvm, network root, resume, or distribution-specific debug shells. The kernel does not need to understand those names.

### PID 1 parameters

Service managers can read /proc/cmdline and consume their own namespace, such as systemd options. Arguments not recognized by kernel parsing may also be passed toward init under defined rules.

The presence of a string does not prove the intended consumer accepted it. Diagnose with that component's logs and documentation.

---

## 3. Parsing syntax and common parameters

The command line is space-separated at its transport level, with quoting resolved according to the loader and kernel parser. Common examples:

~~~text
root=UUID=1111-2222 ro rootwait
console=tty0 console=ttyS0,115200n8
loglevel=7 ignore_loglevel
init=/bin/sh
panic=10
maxcpus=1
mem=4G
~~~

Each changes a different boundary:

- root= selects or describes the future root device;
- ro asks for the initial root mount to be read-only;
- rootwait waits indefinitely for an asynchronously appearing root block device;
- console= registers console candidates and chooses the preferred output/input console by ordering rules;
- loglevel= controls console visibility, not which messages exist in the ring buffer;
- init= replaces the normal init path and is dangerous as a persistent setting;
- panic= controls reboot delay after panic;
- maxcpus= limits CPUs brought online for diagnosis;
- mem= constrains usable memory and can hide the real problem if used casually.

Parameter spelling is exact. Some options accept suffixes or repeated forms. Search the documentation and source for the running kernel rather than copying a forum line from another architecture.

---

## 4. saved_command_line and the editable copy

The kernel needs both a preserved record and a mutable string for parsing. setup_command_line() arranges copies so later code can expose the original command line while setup handlers tokenize or modify a working copy.

This explains why /proc/cmdline remains readable after initialization and why internal parsing can safely insert terminators. Security-sensitive parameters remain visible to privileged and often unprivileged users, so secrets should not be placed on the command line. They may appear in logs, crash dumps, boot-loader configuration, and firmware variables.

Use key references, TPM policy, protected files, or interactive acquisition instead of literal encryption keys in root= or rd parameters.

---

## 5. The first operations in start_kernel()

At entry, the environment is fragile. start_kernel() performs or triggers operations such as:

- disabling local interrupts while state is inconsistent;
- setting the boot CPU identity;
- establishing stack canaries or early security state when possible;
- initializing lock-dependency tracking and debug infrastructure;
- preparing per-CPU areas;
- setting up the architecture and command line;
- establishing early memory allocators and page structures;
- creating the log buffer and console path;
- building scheduler, interrupt, timer, and RCU foundations;
- initializing caches and core subsystems;
- enabling interrupts only after handlers and controllers are ready;
- creating the first kernel tasks and moving later initialization into schedulable context.

The exact order contains architecture and configuration branches. The principle is stable: a function cannot use an allocator, lock validator, timer, work queue, or driver core that has not been initialized.

---

## 6. Interrupt state and why it matters

Early kernel code normally runs with interrupts disabled. Firmware or the loader may have left device state, but Linux has not installed its complete interrupt descriptor tables, interrupt-controller routing, per-CPU stacks, or handlers.

If an interrupt arrived too early, the CPU could jump through an invalid entry or a handler could use uninitialized state. start_kernel() tracks interrupt enablement carefully and warns when code enables interrupts unexpectedly.

Later, after trap handling, interrupt descriptors, controllers, timekeeping, and scheduling primitives are ready, the kernel enables interrupts. That moment allows asynchronous hardware events to preempt the boot thread. From then on, ordering bugs can become concurrency bugs.

Interrupts disabled does not mean no exceptions can occur. Page faults, invalid instructions, and machine checks are synchronous architectural events and require at least safe early handling.

---

## 7. setup_arch(): importing the physical world

setup_arch() is where common code asks the architecture to finalize its view of the platform.

Typical responsibilities include:

- parse and sanitize firmware memory maps;
- reserve the kernel, initrd, firmware runtime, and special regions;
- initialize architecture paging structures;
- find ACPI or Device Tree data;
- record CPU and NUMA topology;
- establish resources visible through /proc/iomem;
- set up early consoles and architecture debug facilities;
- prepare crash-kernel reservations;
- select architecture-specific mitigations;
- return the command line for common parsing.

setup_arch() does not fully probe every device. It creates a trustworthy platform foundation for generic memory and subsystem initialization.

The memory chapter follows these reservations in detail.

---

## 8. Per-CPU areas

The kernel stores frequently accessed state separately for each CPU: current task information, scheduler data, interrupt counters, allocator caches, timers, and architecture state. Code uses efficient per-CPU addressing rather than indexing a normal array with locks for every access.

Before secondary CPUs start, setup_per_cpu_areas() arranges storage and the boot CPU's offsets. Later CPU startup installs each processor's base and initializes its local state.

This is another bootstrap dependency. Scheduler initialization wants per-CPU run queues. Interrupt accounting wants per-CPU counters. SMP startup wants per-CPU stacks and descriptors. Therefore per-CPU layout precedes the concurrency it supports.

---

## 9. Boot-time memory initialization

The kernel begins with memblock or architecture-specific early allocation. It discovers physical regions, excludes reserved ranges, and allocates foundational tables. Later calls initialize page descriptors, zones, the buddy allocator, and slab-family object allocation.

The order is visible conceptually:

1. Import physical memory ranges.
2. Reserve objects already occupying memory.
3. Allocate page metadata and architecture structures through early mechanisms.
4. Initialize zones and free-page lists.
5. Release eligible physical pages to the normal allocator.
6. Initialize small-object caches.
7. Retire early allocator assumptions when no longer needed.

Calling kmalloc in a path that runs before slab initialization is invalid unless the API explicitly has early behavior. Init code annotations and boot-order documentation help developers choose legal facilities.

---

## 10. printk and the log buffer

Early messages can be emitted before a normal console driver exists. printk stores records in the kernel log buffer once its early form is available. Console drivers determine which records become visible on a display or serial port.

Therefore:

- no visible text does not mean no log record was created;
- quiet and loglevel affect console presentation more than record generation;
- earlycon can show records before a normal console registers;
- after a successful boot, dmesg or journalctl -k can reveal messages never shown during boot;
- a crash before persistence or a working console can still lose records unless pstore or an external console captures them.

Console initialization itself is staged. boot consoles are later replaced by real consoles; the kernel can replay buffered messages when a console registers.

---

## 11. Scheduler foundations

The scheduler eventually decides which task runs on each CPU, but early boot begins as a single direct control flow. sched_init() prepares run queues, scheduling classes, task structures, and per-CPU state for the boot CPU.

The initial task, historically called the swapper or idle task for CPU 0, is not an ordinary process created by fork. Its task structure is statically available so the kernel has a current context before dynamic task creation works.

Later, rest_init() creates kernel_init and kthreadd. The boot task becomes the idle task for CPU 0. kernel_init eventually runs the initcalls and executes userspace init; kthreadd becomes the ancestor/manager for many kernel threads.

The transition matters because later initialization can sleep, schedule, and run in process context. Before that, code cannot assume ordinary scheduling semantics.

---

## 12. RCU initialization

Read-Copy Update is a synchronization mechanism heavily used in the kernel for read-mostly data. It allows readers to access structures with very low overhead while updates wait for a grace period before freeing old versions.

RCU has staged boot behavior because grace periods and callbacks depend on CPUs, scheduler progress, interrupts, and timers. Early initialization supports limited use; later initialization transitions to normal operation as those dependencies appear.

This illustrates why subsystem initialization is not always one call. A facility may have an early bootstrap mode, a scheduler-aware mode, and a fully SMP-capable mode.

---

## 13. Traps, IRQ descriptors, and interrupt controllers

Architecture trap setup establishes exception entries. Generic IRQ initialization creates descriptors and data structures representing interrupt lines. Architecture code initializes controllers such as local and I/O APICs on x86 or GICs on ARM systems.

Drivers later request interrupts by number or firmware-described resource. At that point the generic layer can route events to handlers, manage affinity, mask and unmask sources, and account activity.

The hierarchy is:

~~~text
CPU exception entry
    -> architecture interrupt-controller handling
    -> generic IRQ descriptor
    -> registered device handler
~~~

A firmware table can describe an interrupt before Linux creates the descriptor, and the descriptor can exist before a device driver registers its handler. "The interrupt exists" has stage-dependent meaning.

---

## 14. Timekeeping and timers

Linux needs a clocksource to measure time and clock-event devices to schedule future interrupts. Early delays may use crude loops or architecture counters; normal timekeeping evaluates available sources for stability and resolution.

Initialization establishes:

- jiffies and tick foundations;
- clocksource registration and selection;
- timekeeper state;
- high-resolution timer infrastructure when supported;
- per-CPU clock events;
- timer softirq processing;
- wall-clock initialization from persistent time sources.

Boot timestamps before and after clocksource selection may have different accuracy. A clocksource watchdog can later mark a source unstable and switch. Performance analysis must know which clock generated each measurement.

---

## 15. VFS and core caches

Before mounting any filesystem, Linux initializes Virtual Filesystem structures, inode and dentry caches, mount infrastructure, file tables, and namespace foundations. It also initializes key object caches and security hooks.

The root filesystem used during early boot is a special in-memory rootfs. Initramfs unpacking populates it later. The VFS abstraction lets the kernel treat ramfs/tmpfs, ext4, procfs, sysfs, and other filesystems through shared operations despite different storage.

No disk root can be mounted until:

- VFS exists;
- the filesystem type is registered;
- the block device exists;
- lower storage layers are active;
- mount parameters are available.

That dependency explains the position of initramfs and initcalls later in the sequence.

---

## 16. Security framework initialization

Linux Security Module infrastructure allows security modules such as SELinux, AppArmor, Smack, Landlock-related hooks, and the default capability logic to participate in kernel authorization.

Security initialization is staged because credentials, filesystems, caches, and hooks become available at different times. The selected major LSM order can be influenced by build and command line. Early policy loading may occur from initramfs before untrusted userspace runs.

Boot parameters that disable a security module can substantially change the threat model. If the command line is attacker-controlled, a signed kernel may start with weaker enforcement unless policy binds those parameters.

Inspect runtime state through subsystem tools and kernel logs rather than assuming a compiled option is active.

---

## 17. initcalls and the move to kernel_init

Core start_kernel() code cannot explicitly call every built-in driver and subsystem. The kernel uses initcall sections: functions are placed into ordered levels at link time and invoked later by do_initcalls(). Levels express coarse dependencies such as early, core, postcore, arch, subsys, fs, device, and late initialization.

The main initcall execution happens in kernel_init context after scheduling is available. Individual initcalls can register buses, filesystems, drivers, network protocols, and platform facilities.

The next chapters examine memory, CPU/SMP, and initcalls separately. For the start_kernel() map, the key transition is that hard-coded foundational initialization creates an environment where link-time registered subsystems can initialize in staged order.

---

## 18. rest_init() and the first tasks

At the end of the earliest common sequence, rest_init() creates kernel_init and kthreadd with carefully controlled ordering.

kernel_init eventually waits for enough setup, runs kernel_init_freeable(), executes initcalls, frees init memory, and attempts the userspace init executable. kthreadd provides infrastructure for creating kernel threads.

The original boot task enters cpu_startup_entry() and becomes the idle loop for the boot CPU. "Idle" does not mean it spins wastefully; architecture idle mechanisms can place the CPU in power-saving states until work or interrupts arrive.

This is a profound change from entry: the direct boot control flow has transformed into scheduled tasks. Linux can now express work through processes, kernel threads, work queues, interrupts, softirqs, and timers.

---

## 19. Reading start_kernel() scientifically

Use a kernel tag matching the system or experiment:

~~~sh
git clone --depth=1 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
sed -n '/asmlinkage __visible void __init start_kernel/,/^}/p' init/main.c
~~~

For an installed distribution kernel, obtain its exact source package and configuration instead of assuming the newest upstream tree matches.

For every call, write three notes:

1. **Requires:** which facilities must already work?
2. **Provides:** which assumption becomes valid afterward?
3. **Phase:** can it allocate, sleep, receive interrupts, or run concurrently?

Example:

~~~text
sched_init
requires: per-CPU areas and basic memory structures
provides: boot CPU run queue and scheduler foundations
phase: still early; ordinary user tasks do not exist
~~~

This method converts a long call list into a dependency graph.

---

## 20. Diagnostic command-line experiments

Perform these only in a disposable VM with a known-good default entry.

### Increase visibility

~~~text
ignore_loglevel loglevel=7 initcall_debug
~~~

The first two increase console output; initcall_debug records initcall timing and return information. Output volume is large.

### Limit CPU count

~~~text
maxcpus=1
~~~

If a failure disappears, SMP startup or concurrency becomes a lead, not a proven cause.

### Override init

~~~text
init=/bin/sh
~~~

This can bypass normal userspace policy for rescue, but the shell has PID 1 responsibilities, filesystems may be read-only, devices and terminals may be incomplete, and exiting can panic. It does not bypass the need to find and mount root unless init exists in the active root.

### Break at early userspace

Initramfs frameworks provide their own debug and breakpoint parameters, such as rd.break in dracut environments. Those are not generic kernel parameters even though they travel on the kernel command line.

Always remove temporary weakening options after diagnosis.

---

## 21. Failure patterns

### Unknown kernel parameter

Check whether the name belongs to initramfs or PID 1. The kernel may pass it onward. A warning is not always fatal.

### Panic before initcalls

Focus on architecture setup, memory, traps, scheduler, timers, and core initialization. initramfs content is unlikely to explain a crash before unpacking or process creation.

### Hang after "smp" or CPU messages

Compare maxcpus=1, firmware/ACPI data, microcode, interrupt controller, and watchdog output. A single-CPU success narrows the stage but can also hide unrelated races.

### "No working init found"

Core kernel initialization succeeded far enough to attempt userspace. Check initramfs /init, final root paths, executable architecture, dynamic linker, permissions, and init=.

### Console output stops but machine responds

Console ownership may have moved from boot console to a misconfigured normal driver. Test serial and network reachability and inspect logs after recovery.

---

## 22. Mechanism summary

start_kernel() is a dependency-ordered construction site. Architecture code supplies machine-specific state. Command-line parsing feeds kernel, early userspace, and PID 1. Early memory mechanisms enable normal allocators; per-CPU areas enable scheduler and interrupt state; traps and controllers make asynchronous events safe; timekeeping supports timers; VFS and caches make filesystems possible; security hooks prepare policy; and rest_init() turns direct boot execution into scheduled kernel tasks.

The important skill is not recalling every function in order. It is recognizing which facilities exist at a given line and why moving one initialization earlier or later could be illegal.

The next chapter opens the memory portion of this graph. It follows firmware ranges through memblock, page descriptors, zones, the buddy allocator, slab caches, NUMA topology, and the freeing of init-only memory.

---

## Primary references

## 23. Annotated dependency walk through

Consider a simplified sequence and ask what would break if two steps were reversed.

The kernel begins by protecting the fragile entry state: interrupts remain disabled, the boot CPU is identified, and debug mechanisms that can operate without ordinary allocation are established. If interrupts were enabled now, a device event could enter through tables or controller state that Linux has not installed.

setup_arch() imports the firmware memory description and reserves the kernel, initramfs, platform data, and architecture regions. This must precede freeing physical memory. If zone initialization ran against an unfiltered firmware map, the page allocator might hand a firmware runtime page to an unrelated cache.

Command-line setup preserves an original copy and parses parameters that affect the earliest decisions. A mem= limit must influence memory topology before normal allocation. An early console must be selected before messages it is intended to expose. A normal device-driver parameter can wait until that driver's initcall.

Per-CPU areas are allocated while only the boot CPU runs. Scheduler and interrupt code then receive stable per-CPU locations. If a secondary CPU started first, it could access an uninitialized base and corrupt another CPU's data.

The kernel initializes page metadata, zones, and the buddy allocator, then object caches. From that boundary, code can use normal GFP allocation according to context. It still cannot necessarily sleep, because schedulable tasks and wakeup sources are not fully established.

Trap and IRQ foundations install safe entries; timekeeping creates a monotonic basis; sched_init builds run queues; RCU enters an appropriate boot mode. Each subsystem may have an early and normal phase. A function name ending in init does not guarantee the facility is completely ready for every API.

VFS caches and namespace structures make filesystem registration and mounting possible. Security hooks must be placed before untrusted filesystem and process activity if they are to mediate it. Work queues and kernel threads require scheduling; therefore their ordinary forms appear after scheduler foundations.

Only when interrupt descriptors, controllers, handlers, timer events, and scheduling are coherent does the kernel enable interrupts. The instant is a concurrency boundary. Before it, a data structure cannot be observed by a device IRQ. After it, publication and locking rules matter.

rest_init() creates kernel_init and kthreadd. Creating kernel_init before entering the idle loop gives later initialization a normal process context that can sleep. Creating kthreadd establishes a parent and service for kernel threads. The original task becomes CPU 0's idle task rather than exiting.

kernel_init runs the broad initcall levels. A filesystem registers before it can mount. A bus registers before its drivers bind. Root storage appears only after controller, child device, partition, and optional mapping layers. The kernel waits for asynchronous probes at defined points before declaring initialization complete.

Finally, init-only memory is freed and protections are tightened. At that point an __init function is no longer callable. The kernel attempts the configured userspace init paths. Successful exec proves every dependency in this walk through was sufficient to create a process and a root containing a usable executable.

### A command-line ownership table

Build a table for the running machine rather than treating /proc/cmdline as one configuration namespace:

| Parameter | Intended consumer | Evidence of consumption |
|---|---|---|
| console= | kernel console core | console registration lines, /dev/console behavior |
| root= | kernel direct-root or initramfs | mount source and early logs |
| rd.luks.uuid= | initramfs framework | cryptsetup/initrd logs and mapping |
| mitigations= | architecture kernel code | kernel mitigation messages |
| systemd.unit= | systemd PID 1 | selected target and manager logs |

For every custom option, locate documentation or source for its handler. A parameter silently ignored because the relevant feature was not compiled can look like an implementation bug. A parameter consumed by initramfs will not appear in the kernel parameter table. A module option for a built-in driver may need its module-name prefix, while a standalone module can receive options through modprobe configuration after loading. Precise ownership prevents hours of editing the correct string for the wrong parser.

## 24. Review exercise: locate the first legal user

Take these facilities—kmalloc, schedule(), request_irq(), queue_work(), filp_open(), and userspace exec—and locate the earliest phase where each is legal in the target kernel. The answer is not one universal line number. It depends on which initialization variant is called, execution context, configuration, and API.

An early allocator may satisfy a narrow request before the full slab system, but that does not make arbitrary kmalloc flags valid. Scheduler data may exist before sleeping is allowed in the current context. IRQ descriptors can exist before a device controller and handler are ready. Workqueue code initializes in stages before every queue exists.

Write the prerequisite chain for each API and verify it from source. This exercise catches boot-order regressions: a refactor can preserve compilation while moving a call before the facility that makes it legal.

### Version discipline

start_kernel() changes as subsystems evolve. When a guide lists calls in one order, treat it as a snapshot, not an ABI. Check the exact source tag and configuration. Compiler options, architecture hooks, sanitizers, and security modules can insert substantial work absent from a minimal build.

Document stable invariants separately from current implementation. "Normal page allocation must exist before unrestricted slab use" is a dependency. "Function X is the seventeenth call" is a version-specific observation. This separation keeps the mental model useful across kernel releases.

When reviewing a boot regression, diff both source order and configuration. A newly enabled sanitizer, LSM, allocator, or tracing feature can introduce initialization without an obvious change in init/main.c. Linker sections also change which initcalls are present. The executable kernel—not one source file—is the final sequence.

- Linux kernel init source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/init/main.c
- Kernel parameters: https://docs.kernel.org/admin-guide/kernel-parameters.html
- Linux kernel command-line API: https://docs.kernel.org/admin-guide/kernel-parameters.html
- Core API documentation: https://docs.kernel.org/core-api/
- Scheduler documentation: https://docs.kernel.org/scheduler/
- RCU documentation: https://docs.kernel.org/RCU/
- printk basics: https://docs.kernel.org/core-api/printk-basics.html
- Linux Security Module documentation: https://docs.kernel.org/security/lsm.html
- initcall debugging and boot configuration: https://docs.kernel.org/admin-guide/bootconfig.html
