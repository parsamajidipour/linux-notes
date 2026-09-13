# Linux Notes

**Mechanism first, commands second.**

A growing collection of deep, mechanism-level notes on Linux — written while digging into topics that raised a question, a debugging session that didn't add up, or a "how does this actually work?" moment. This isn't a tutorial and it isn't a cheat sheet. Every series here tries to derive *why* Linux behaves the way it does before it ever tells you *what to type*, so the commands stop being incantations and start being consequences of a mechanism you actually understand.

> Work in progress. Some series here are complete end to end; others are still being written, in order of dependency rather than in any fixed schedule.

---

## Philosophy

Most Linux documentation answers "how do I run this command?" These notes try to answer a harder question: **what is actually happening, and why does the system do it this way rather than some other way?**

A few rules show up across every series in this repository:

- **Mechanism before rules.** A hardening rule, a config directive, or a troubleshooting step is only trustworthy once you can trace it back to the mechanism that makes it true.
- **Observation over assertion.** Claims are meant to be checked, not taken on faith — against `strace`, `/proc`, packet captures, source code, and man pages, not against folklore.
- **Everything is falsifiable.** Where a note disagrees with the kernel, the source, or the RFC, the primary source wins. Corrections are the whole point of this format.
- **Chapters build on chapters.** Each series is written as one continuous argument, not a pile of disconnected snippets — later chapters lean on the mechanism the earlier ones established.

---

## What's Inside

This repository is organized into topic-based series, each living in its own folder. Broadly, the notes cover:

- **Core Linux internals** — the permission and ownership model, the capability system that replaced the all-or-nothing root model, and other kernel-level mechanisms.
- **Identity and access** — how authentication actually happens under the hood (pluggable authentication frameworks, privilege-escalation tools), traced from the API down to the exact system call that changes a process's credentials.
- **Networking** — a packet's full journey through the kernel, from the wire to a socket and back, including routing, firewalling, the transport layer, DNS, virtual networking, tunnels, and traffic control.
- **System services and scheduling** — how the init system boots and supervises everything else, and how time-based job scheduling actually executes and isolates work.
- **Storage and memory-backed filesystems** — from physical devices up through partitioning, filesystems, volume management, and the in-memory filesystems used for runtime state.
- **Observability and logging** — how the system's own activity is captured, stored, rotated, and audited.
- **Remote access** — the protocols and mechanisms that make secure remote administration possible.

Each folder is self-contained with its own `README.md` that goes into far more depth than this summary — scope, prerequisites, how to read that particular series, and a full chapter index. Because the topic list keeps growing, this top-level README intentionally doesn't enumerate folder names or chapter counts — browse the repository itself for the current, accurate list.

---

## How to Use This Repository

1. **Browse the repository** to see the current list of topics — new series get added over time, so this file won't try to track that list.
2. **Open a folder's own README first.** Every series documents its own scope, prerequisites, and reading order before diving into chapters.
3. **Read in order the first time.** Almost every series is written so later chapters assume the ones before them.
4. **Come back as a reference.** Several series are explicitly organized so you can jump straight to the chapter you need once you've read it once.
5. **Verify, don't trust.** Nearly every chapter ends with commands you can run yourself to check the claim against your own system, rather than taking it on faith.

## Who This Is For

Linux and platform engineers, system administrators, DevOps/SRE, security researchers, and anyone studying for Linux certification tracks — as well as anyone who wants to actually understand Linux instead of memorizing commands for it. Most series assume comfort with processes, the shell, and basic networking; each README states its own prerequisites explicitly.

## Contributing

These are personal study notes, but corrections matter more than politeness here. If something disagrees with the kernel source, a man page, or an RFC, that's worth an issue or a pull request — accuracy is the entire point of this format.

## License

Educational resource for the Linux community. Feel free to read, share, and learn from it.

---

*Part of an ongoing effort to write things down before they're forgotten — and to make them useful to whoever else stumbles onto the same question.*
