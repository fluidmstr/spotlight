# FAQ

### Does it run `.exe` files today?

No. The kernel boots, prints to a serial port and halts. Running ordinary Windows
programs is **L3**, the 1.0 target.

### Is this ReactOS?

No. ReactOS reimplements the NT architecture with a strong focus on binary
compatibility and has been in development since 1996. Spotlight starts from a
different point: a new kernel written in V, a frozen ABI target (Windows 10 x64
19045), and a documented, test-first process. Some ReactOS/Wine components may be
reused where the licence allows — as libraries, not as a code dump.

### Is this a Linux distribution?

No. There is no Linux anywhere: our own loader, our own kernel, our own boot
chain. QEMU is only a development machine.

### Why V?

* memory safety and compile-time checks without a borrow checker;
* very fast edit-build-run cycle;
* C backend gives us full control over the generated code;
* excellent C interop for the places where assembly is unavoidable.

The limitations are real and documented: no `no_std`, runtime assumptions, and
`-no-builtin` quirks. They are treated as engineering constraints, not surprises.

### Why NT and not a new, cleaner API?

Because the goal is to run existing Windows software. A new API would require
recompiling the world; the NT contract is the only path to ordinary `.exe` files.

### Is the ABI really Windows 10?

Yes: x64, build **19045** (22H2). Syscall numbers, structures and semantics are
frozen to that build. Other builds become separate ABI branches.

### Can I use my own Windows drivers?

Eventually, in Track A. It requires a licensed Windows installation on your side;
the project ships no Microsoft files and no patched binaries.

### What hardware is supported?

QEMU + OVMF today. Real hardware after the storage stack (M10): UEFI, x86-64,
Secure Boot **off**, AHCI/NVMe, linear framebuffer.

### Is AI used to write this?

Yes, heavily — with strict rules: specs before code, tests before merge, no
invented ABI, no copied code. The rules live in `AGENTS.md` in every repository.

### How can I help?

* verify ABI rows on a real Windows 10 19045 and file corrections;
* add boot-test expectations;
* write application checklists for the future corpus;
* improve documentation and examples;
* implement small kernel pieces with their tests.

Start with [Contributing](Contributing.md).

### When will it be ready?

Honest estimate: L1 in 9–12 months, L3 in 30–40 months with 5–6 engineers.
Anyone promising a desktop next year is selling something else.

---

[Back to Home](Home.md)
