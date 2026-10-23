# Interrupts, Timers, the Scheduler, and SMP Startup

Linux enters on one processor and eventually manages many processors executing unrelated work while devices interrupt asynchronously and timers expire with high precision.

That transformation cannot happen all at once. Before interrupts are enabled, the kernel must install exception entries and initialize interrupt controllers. Before a timer interrupt can request rescheduling, clocks and scheduler structures must exist. Before a secondary CPU can run, it needs per-CPU memory, a stack, descriptor state, local interrupt-controller setup, and a safe rendezvous with the boot CPU.

This chapter connects those dependencies. The exact controllers differ across architectures—APIC on conventional x86, GIC on many ARM systems, IMSIC/APLIC arrangements on RISC-V—but the reasoning remains: create the routing and state before allowing asynchronous execution.

---

## 1. Exceptions, interrupts, and traps

An **exception** is generated synchronously by the current instruction stream: page fault, divide error, invalid opcode, system call, breakpoint, or general-protection fault. An **external interrupt** is generated asynchronously by a device or controller. Kernel literature sometimes uses **trap** as a general or architecture-specific category.

The CPU needs a table mapping event vectors to entry code. On x86 this is the Interrupt Descriptor Table. Entries define target code, privilege rules, and gate properties. Early architecture code installs enough handling to survive faults; later setup builds the full entry environment.

Entry code must preserve registers, switch stacks where required, account context, manage interrupt state, and eventually call C handlers. Returning restores the interrupted context.

The distinction matters for debugging. A page fault at a reproducible instruction is synchronous. A network interrupt can arrive between almost any two instructions after enablement. The first is tied to current execution; the second introduces concurrency.

---

## 2. Interrupt controllers

Devices do not normally wire every signal directly to a unique CPU pin. Interrupt controllers collect, prioritize, route, mask, and deliver events.

On modern x86:

- each CPU has a local APIC;
- I/O APICs route platform interrupt inputs;
- PCI devices often use Message Signaled Interrupts, writing a special message rather than asserting a shared line;
- MSI-X lets devices expose multiple independently steerable vectors.

Firmware tables describe controllers and topology. Linux initializes them, assigns vectors, and programs routing. The generic IRQ layer presents descriptors independent of controller details.

On ARM systems, a GIC distributes interrupts and has per-CPU interfaces. Newer versions add redistributors and support large systems. Device Tree or ACPI describes IDs and trigger properties.

Interrupt-controller initialization must precede normal device probe that requests interrupts.

---

## 3. IRQ descriptors and handlers

The generic IRQ subsystem represents an interrupt with an irq_desc and associated irq_data, flow handler, chip operations, affinity, actions, and statistics.

A driver calls request_irq() or a managed equivalent with:

- interrupt number obtained from firmware or bus resources;
- handler function;
- trigger and sharing flags;
- name and device context.

The subsystem programs controller behavior and links the action. When the event arrives, architecture entry dispatches to the descriptor and registered handler.

Hard-interrupt context has constraints: it cannot perform operations that sleep, and it should finish quickly. Work can be deferred to threaded interrupts, softirqs, tasklets in legacy code, work queues, or driver-specific mechanisms.

After boot:

~~~sh
cat /proc/interrupts
cat /proc/softirqs
~~~

Counts show delivery by CPU. A zero count can mean an idle device, wrong routing, masked interrupt, polling mode, or that a different vector is used.

---

## 4. Interrupt context and preemption

When an interrupt preempts a task, the CPU executes kernel code on behalf of the event, not as a schedulable process that can sleep freely. Locks used in both process and interrupt context must prevent deadlock by handling local interrupt state appropriately.

Preemption models determine when kernel code can be preempted by higher-priority work. Real-time kernels thread more interrupt work to reduce latency and make scheduling explicit.

Early boot is simpler because interrupts and preemption are constrained. Enabling them expands possible interleavings. Initialization code must publish fully constructed state before a handler can observe it.

A common driver pattern is:

1. allocate and initialize state;
2. program hardware while interrupt is masked;
3. register handler;
4. make state visible;
5. unmask device interrupt.

Reversing the last steps can let an event reach a half-initialized handler.

---

## 5. Clocksource, clock event, and persistent clock

Three time concepts are often mixed.

### Clocksource

A clocksource is a monotonically increasing counter Linux reads to measure elapsed time. Examples include the x86 TSC, HPET, ACPI PM timer, and architecture counters. Important properties are frequency, width, access cost, stability, and behavior across CPUs and power states.

### Clock-event device

A clock-event device can generate an interrupt at a requested future time. Local APIC timers and architecture timers are examples. The kernel uses them for scheduler ticks and high-resolution timers.

### Persistent or wall clock

An RTC or firmware time service provides calendar time across power cycles. Linux reads it to initialize wall time, then maintains time using a clocksource and later synchronizes through NTP or another service.

A source can be excellent for elapsed time and know nothing about the date. A battery-backed RTC knows approximate date but is too slow or coarse for scheduling nanosecond intervals.

---

## 6. Calibrating and selecting clocks

Early code may need delays before full timekeeping exists. Architecture loops, firmware-reported frequencies, or counters provide provisional timing. Linux registers available clocksources, rates them, and selects a suitable one.

The clocksource watchdog can compare a preferred source with a reference and mark it unstable if drift or discontinuity exceeds expectations. Virtualization can complicate counters; hypervisors provide paravirtual clocks to avoid emulation and migration problems.

Inspect:

~~~sh
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
cat /sys/devices/system/clocksource/clocksource0/available_clocksource
journalctl -b -k | grep -Ei 'clocksource|tsc|timer|hpet'
~~~

Changing clocksource manually is a diagnostic experiment, not a generic performance tweak. A slower stable source can be better than a fast counter that jumps.

---

## 7. The scheduler's starting point

The boot CPU begins in the statically defined initial task. sched_init() creates scheduling foundations and a run queue for each possible CPU. Scheduling classes register logic for deadline, real-time, fair, and idle work according to configuration.

Every CPU eventually has an idle task. Idle is selected when no runnable task exists and can enter architecture power-saving states.

The scheduler tracks runnable tasks, priorities, CPU affinity, utilization, deadlines, and load balancing. But during the earliest startup there are no normal user tasks to choose among. The infrastructure must exist before kernel_init, kthreadd, worker threads, and interrupt-driven wakeups begin.

### Need-reschedule path

A timer or wakeup can mark that the current task should be rescheduled. At a safe preemption or return boundary, the scheduler selects another runnable task and performs a context switch. Context switching changes register state, stack, memory context where needed, and accounting.

---

## 8. Ticks and tickless operation

Traditional kernels program periodic timer interrupts at a configured HZ frequency. Each tick updates accounting and checks timers and scheduling.

Modern Linux can suppress unnecessary periodic ticks:

- **NO_HZ_IDLE** stops the tick on idle CPUs;
- **NO_HZ_FULL** can reduce ticks on selected CPUs running a single task, with operational constraints;
- high-resolution timers program one-shot events for exact next deadlines.

Tickless does not mean no interrupts. Devices, inter-processor messages, timers, and housekeeping still generate events.

During boot, periodic or one-shot clock events begin only after timekeeping and interrupt delivery are safe. Early delay loops are not evidence that the final tick is running.

---

## 9. CPU topology

Linux distinguishes possible, present, online, and active CPUs.

- **possible:** CPUs the kernel can support in this boot configuration;
- **present:** CPUs physically or virtually present;
- **online:** CPUs initialized and accepting work;
- **active:** CPUs participating fully in scheduling and migration decisions.

Firmware provides processor entries and relationships. CPUID or architecture registers provide package, core, thread, and cache details. Hypervisors synthesize topology.

Inspect:

~~~sh
lscpu -e
cat /sys/devices/system/cpu/possible
cat /sys/devices/system/cpu/present
cat /sys/devices/system/cpu/online
~~~

Topology affects scheduling, cache sharing, NUMA locality, and security mitigations. A virtual machine claiming two sockets with one core each is not equivalent to one socket with two cores for all policies.

---

## 10. Bringing up secondary x86 CPUs

The bootstrap processor initializes the kernel. Application processors begin stopped or in firmware-defined waiting states. Linux starts them through an architecture sequence involving startup inter-processor interrupts and low-memory trampoline code.

A simplified path:

1. Allocate or prepare the CPU's kernel stack and per-CPU data.
2. Populate architecture startup parameters and page-table references.
3. Copy or map trampoline code at an address reachable from the processor's reset startup mode.
4. Send INIT and Startup IPIs through the local APIC.
5. The secondary processor begins in a constrained mode at the trampoline.
6. Assembly transitions modes, installs descriptors and page tables, and reaches C startup.
7. The CPU initializes its local APIC, timers, topology, mitigations, and per-CPU subsystems.
8. It signals the boot CPU that it is alive and eventually becomes online.

Synchronization protects every handoff. The boot CPU must not reuse startup data before the secondary has consumed it, and the secondary must not schedule general work before initialization completes.

---

## 11. ARM64 and firmware CPU-start methods

On ARM64, firmware interfaces such as PSCI commonly provide CPU_ON operations. The kernel supplies an entry address and context; firmware or a higher privilege level releases the target processor. Older systems may use spin tables described in Device Tree.

The secondary enters architecture code, enables the MMU and caches as required, installs per-CPU state, initializes its GIC interface and timers, and joins the scheduler.

The contrast with x86 startup IPIs is implementation-specific. The invariant is the same: an external mechanism releases a processor into a tiny entry path that must converge on the kernel's shared SMP state.

---

## 12. Inter-processor interrupts

Once CPUs are online, they must coordinate. IPIs are interrupts sent from one CPU to another for tasks such as:

- rescheduling;
- TLB shootdown after page-table changes;
- function calls on selected CPUs;
- stopping CPUs during panic or shutdown;
- moving timers or work;
- maintaining RCU and scheduler state.

A TLB shootdown illustrates why SMP changes memory management. If CPU 0 removes a mapping, CPU 1 may retain a cached translation. CPU 0 sends an IPI asking CPU 1 to invalidate the relevant TLB entry and waits as required before reusing the memory.

SMP is not simply "more CPUs run the same code." It adds coherence protocols in software around per-CPU caches and shared state.

---

## 13. Memory ordering and synchronization

Processors can observe memory operations in orders different from source code unless architecture rules and barriers constrain them. Locks, atomic operations, RCU primitives, completion objects, and explicit barriers publish initialization safely.

During secondary startup, a flag such as "CPU online" must not become visible before the data structures it promises are initialized. Release and acquire semantics establish that ordering.

x86 has a relatively strong memory model; ARM and RISC-V permit more reordering. Portable kernel code uses Linux memory-model primitives instead of relying on accidental x86 behavior.

The kernel memory-barriers documentation is the authoritative conceptual reference. Casual use of volatile is not a substitute for synchronization.

---

## 14. Softirqs and deferred work

Hard interrupt handlers should perform the minimum work needed to acknowledge hardware and capture state. Deferred mechanisms complete heavier processing.

Softirqs are fixed kernel mechanisms used by networking, timers, RCU, block processing, and other subsystems. They can run on interrupt return or in per-CPU ksoftirqd threads under load. Work queues run functions in kernel-thread process context and can sleep.

Threaded interrupts move much handler work into schedulable threads, especially important in real-time kernels.

Choosing the wrong context causes bugs:

- sleeping in hard interrupt context;
- allocating with flags that can block;
- holding a lock across work that waits on the same interrupt;
- doing unbounded processing and starving other events.

Boot enables these mechanisms in stages after scheduler and per-CPU infrastructure exist.

---

## 15. SMP and initcalls

Many initcalls run after secondary CPUs can be brought online, and drivers may create worker threads or receive interrupts during probe. The kernel uses initcall ordering, locks, work queues, and synchronization to keep this safe.

Some initialization must run on every CPU through smp_call_function or CPU-hotplug callbacks. Rather than baking all per-CPU setup into boot, the CPU hotplug framework defines states and callbacks used both for initial SMP startup and later online/offline operations.

This unifies lifecycle: if a subsystem needs per-CPU queues, it registers a callback at an appropriate hotplug state. Booting CPU 7 and later re-onlining CPU 7 follow compatible setup logic.

---

## 16. CPU hotplug

Linux can offline and online CPUs for power management, maintenance, virtualization, and fault isolation. Offlining requires migrating tasks and interrupts, draining per-CPU work, moving timers, updating scheduler domains, and satisfying subsystem callbacks.

CPU 0 historically had special constraints, though architecture support has evolved. Some interrupts or timers cannot move on a given platform.

Inspect and, only in a disposable environment, control through sysfs:

~~~sh
cat /sys/devices/system/cpu/cpu1/online 2>/dev/null
echo 0 | sudo tee /sys/devices/system/cpu/cpu1/online
echo 1 | sudo tee /sys/devices/system/cpu/cpu1/online
~~~

This changes system state and can disrupt workloads. Do not perform it on production merely as a reading exercise.

Hotplug failures can expose assumptions that worked only during boot ordering.

---

## 17. Interrupt affinity

Interrupts can be steered to CPUs. Affinity affects cache locality, NUMA placement, throughput, and latency. Multi-queue devices use multiple MSI-X vectors so receive, transmit, or storage queues can align with CPU and memory topology.

View assignments:

~~~sh
cat /proc/irq/IRQ_NUMBER/smp_affinity_list
grep -E 'eth|nvme|virtio' /proc/interrupts
~~~

irqbalance or driver logic may change affinity dynamically. Manually pinning all interrupts away from one CPU can help isolation experiments and harm throughput or overload another CPU.

At boot, default affinity and controller capabilities establish the initial distribution; userspace policy may refine it later.

---

## 18. Watchdogs and lockups

Hardware watchdogs can reset a machine if software fails to make progress. Kernel softlockup and hardlockup detectors use scheduler and performance-monitoring mechanisms to detect CPUs stuck without scheduling or handling interrupts.

During early boot, not every watchdog or detector is initialized. Firmware may have its own watchdog that the loader or kernel must service or disable according to protocol. An unexplained reboot at a consistent interval can be a watchdog, not a direct CPU fault.

Kernel logs after recovery, BMC event logs, pstore, and watchdog configuration help distinguish deliberate reset from power loss.

---

## 19. Worked example: one secondary CPU fails

Logs show CPUs 0 through 7 discovered, but CPU 6 never becomes online and startup times out.

Evidence path:

1. Firmware tables described CPU 6, so discovery worked.
2. Boot CPU and several APs started, so global trampoline and controller paths mostly work.
3. The failure may be CPU-specific, topology-specific, microcode-related, or a race.
4. Booting with maxcpus=6 avoids attempting later CPUs but does not repair the cause.
5. Firmware diagnostics, microcode versions, APIC IDs, and exact timeout messages matter.

Compare cold boot, warm reboot, firmware version, and a previous kernel. In a VM, verify the declared topology is valid. Do not conclude "bad CPU" solely from one kernel timeout; incorrect firmware tables can describe nonexistent or duplicate IDs.

---

## 20. Worked example: clocksource instability

The kernel reports that TSC is unstable and switches to HPET or another source. Applications observe no backward time, but performance changes.

The timekeeping watchdog detected disagreement. Possible causes include unsynchronized counters, virtualization migration, power-management behavior, firmware, or hardware errata.

Collect:

~~~sh
journalctl -b -k | grep -Ei 'clocksource|tsc|hpet'
cat /sys/devices/system/clocksource/clocksource0/current_clocksource
lscpu
systemd-detect-virt
~~~

Forcing TSC may hide the warning and reintroduce incorrect time. Fix the hypervisor CPU model, firmware, or platform when possible. A slower fallback preserves correctness.

---

## 21. Boot diagnostics

Useful command-line comparisons in a lab:

~~~text
maxcpus=1
nosmp
noapic
nolapic
clocksource=...
tsc=...
irqpoll
~~~

These options are architecture-specific and can drastically reduce function or performance. Use one at a time to classify a problem, consult current kernel parameter documentation, and remove it afterward.

General evidence:

~~~sh
journalctl -b -k | grep -Ei 'SMP|CPU[0-9]|APIC|GIC|irq|clocksource|timer|watchdog'
cat /proc/interrupts
cat /proc/timer_list 2>/dev/null | head -n 100
cat /sys/devices/system/cpu/online
~~~

Complete logs preserve ordering better than grep alone.

---

## 22. Mechanism summary

Linux starts with one CPU and tightly controlled asynchronous state. Exception tables make faults survivable. Interrupt controllers and generic descriptors create routing. Clocksources measure time; clock-event devices schedule events. Scheduler run queues and idle tasks make task selection possible. Per-CPU storage gives each processor independent hot-path state.

Secondary processors enter through architecture-specific trampolines or firmware calls, initialize local state, synchronize with the boot CPU, and become online. IPIs then coordinate scheduling, TLB invalidation, and cross-CPU work. Softirqs, threaded interrupts, and work queues move expensive activity out of hard interrupt context.

The ordering is the lesson: do not enable an asynchronous producer before its consumer and data are ready. That principle reappears in driver initialization, the subject of the next chapter.

---

## Primary references

## 23. Timeline of one device interrupt

Follow a completed NVMe request on CPU 3.

The controller writes completion information into a DMA queue and emits an MSI-X message whose address/data target a configured interrupt vector. The interrupt-remapping/IOMMU layer may translate and validate the message. The local interrupt controller presents the vector to CPU 3 when priority and masking permit.

Architecture entry code switches into the kernel entry context, preserves registers, and identifies the vector. The generic IRQ layer finds the descriptor and invokes the NVMe handler. The handler reads completion entries, acknowledges work, updates request state, and schedules any deferred completion processing. It avoids sleeping because hard IRQ context is not a normal task.

Completion wakes a process waiting for I/O. Scheduler code marks that task runnable on an allowed CPU and may send a reschedule IPI if another CPU should reconsider its current task. On return from interrupt, preemption checks decide whether CPU 3 should continue the interrupted task or switch.

This one event crosses DMA memory ordering, interrupt routing, per-CPU state, hard IRQ constraints, block-layer completion, wait queues, and scheduler policy. If /proc/interrupts shows counts but requests time out, delivery occurred and the failure may be queue processing. If the completion memory changed but counts remain zero, routing or masking becomes a lead. If counts concentrate on one CPU, affinity or queue mapping explains load imbalance.

## 24. Scheduler domains and topology

The scheduler groups CPUs into domains reflecting shared caches, cores, packages, and NUMA nodes. Load balancing at a small domain can move tasks cheaply between sibling CPUs; cross-node balancing weighs remote-memory cost.

Firmware and CPUID-derived topology therefore influence performance policy. If a hypervisor exposes each virtual CPU as a separate socket, the guest can build inefficient domains. Inspect lscpu -e and scheduler debug data when permitted, then correct the VM definition rather than pinning every task manually.

SMT siblings share core resources. Security policy may disable SMT or avoid placing tasks from different trust domains together. Scheduler core-scheduling features can coordinate sibling selection. These decisions occur after CPU discovery but derive from topology created at boot.

## 25. A safe latency lab

In a VM, assign four vCPUs and capture /proc/interrupts, /proc/softirqs, and scheduler traces at idle. Generate a controlled network or block workload, capture again, and calculate per-CPU deltas. Do not compare cumulative counters from machines with different uptime.

Next pin the workload process to one CPU without changing interrupt affinity. Observe cache movement and wakeup IPIs. Then align one queue's interrupt and process to the same CPU, if the virtual device exposes multiple vectors. Measure throughput and tail latency across repeated runs.

Finally offline one nonboot CPU and repeat. The kernel migrates interrupts and tasks through hotplug callbacks. Any device that stops working exposes a broken lifecycle assumption.

The lesson is not that manual pinning is always faster. It is that boot-time topology, IRQ routing, and scheduler placement form one system. Optimizations must be measured under the real workload and leave housekeeping capacity for timers, RCU, and unbound work.

## 26. Failure-injection questions

Use a virtual machine and change one property at a time.

Limit the guest to one CPU. This removes AP startup and most cross-CPU synchronization while retaining interrupts and scheduling. A failure that disappears may depend on SMP or timing; it is not automatically an APIC bug.

Expose four vCPUs with an unusual socket/core/thread topology. Compare scheduler domains and enumeration. A performance change without functional error can reveal topology-sensitive balancing.

Change the virtual device model so it exposes fewer interrupt vectors. Compare /proc/interrupts and queue counts. Throughput loss can arise from reduced parallel queues rather than scheduler regression.

## 27. Interrupt-storm reasoning

An interrupt storm consumes CPU and can starve normal work. Verify one vector rising rapidly, high IRQ or softirq CPU time, and relevant driver messages.

Causes include a level-triggered source not acknowledged, a device repeatedly reporting work, broken hardware, link flapping, or a shared handler claiming the wrong status. Masking the IRQ stops symptoms and may disable root storage or networking.

Capture vector, driver, device, rate, affinity, and hardware errors before changing it. MSI-X reduces unrelated sharing but does not prevent a device-generated storm.

## 28. Real-time perspective

PREEMPT_RT threads much interrupt handling and changes many locks, allowing high-priority tasks to preempt work that would be nonpreemptible on a standard kernel. It does not eliminate hard IRQ entry or every latency source.

Boot ordering still matters: threaded handlers need scheduler infrastructure, and early polling can create long nonpreemptible sections. CPU isolation must leave housekeeping capacity for timers, RCU, unbound work, and interrupts. Otherwise isolated CPUs receive emergency work or another part of the system stalls.

## 29. Boot CPU and shutdown symmetry

CPU startup has a mirror at shutdown and crash. The kernel stops ordinary scheduling, migrates or terminates work, sends stop requests to secondary CPUs, disables interrupts in controlled order, and returns through architecture restart, poweroff, firmware runtime, or kexec mechanisms.

If one CPU is stuck with interrupts disabled, a stop IPI may not complete. Panic code uses bounded waits and records which CPUs failed to stop. A watchdog or firmware reset may ultimately restart the platform.

This symmetry matters for warm-reboot-only bugs. A driver or CPU that was not quiesced can leave device DMA or interrupt state for the next kernel. kexec intensifies the requirement because firmware does not fully reset hardware. Compare cold boot to reboot and kexec; three results reveal how much cleanup the preceding kernel and platform supplied.

## 30. Scheduler startup review

Trace one kernel thread from creation to its first run. Which task called kernel_thread or kthread_create? Which run queue receives it? What wakes it? Can it migrate before CPU topology domains are complete? Which resources must exist before its function sleeps?

Then trace a timer callback that wakes the thread. The clock-event interrupt enters hard IRQ context, timer softirq expires the timer, a wait queue marks the thread runnable, and return or IPI causes scheduling. The path joins every major subject in this chapter.

If a boot hang shows a thread waiting, identify the missing wakeup producer. Increasing a timeout changes observation; proving that the interrupt, timer, work item, or completion never fires identifies the mechanism.

Keep one principle in view: concurrency starts at boundaries. Enabling interrupts, onlining a CPU, publishing a runnable task, and unmasking a device each permit a new observer. Initialization must be complete before that observer is released, and teardown must revoke the observer before freeing state. The same rule explains many boot-only and shutdown-only races.

When logs seem reordered, consider concurrency before assuming timestamp corruption. Two CPUs can print valid lines in an order different from conceptual dependency, and deferred consoles can replay buffered records. Use trace events and explicit synchronization points for causal claims.

Always record the CPU identifier alongside an event when the logging format provides it. Cross-CPU analysis without CPU identity can mistake parallel progress for a single impossible sequence.

- Generic IRQ documentation: https://docs.kernel.org/core-api/genericirq.html
- x86 APIC documentation: https://docs.kernel.org/arch/x86/topology.html
- Scheduler documentation: https://docs.kernel.org/scheduler/
- NO_HZ documentation: https://docs.kernel.org/timers/no_hz.html
- Timekeeping documentation: https://docs.kernel.org/core-api/timekeeping.html
- CPU hotplug state machine: https://docs.kernel.org/core-api/cpu_hotplug.html
- Kernel memory barriers: https://docs.kernel.org/core-api/wrappers/memory-barriers.html
- ARM PSCI: https://developer.arm.com/documentation/den0022/latest/
- Linux SMP source: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/smp.c
