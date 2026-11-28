# Spotlight documentation

> Three repositories, one machine. Start here.

---

```
Spotlight NT     ██████████████░░░░░░  85%   kernel: M0–M2 closed, M3 75%, M4 72%, M5 92%, M6 99%, M10 37%
Spotlight X      █░░░░░░░░░░░░░░░░░░░  12%   full GPT image auto-launches hello; 98 ntdll stubs
Spotlight PE     █░░░░░░░░░░░░░░░░░░░  13%   installer core, payload launch, rollback, console, ISO, answer plans
Spotlight        ████████░░░░░░░░░░░░  55%   the project as a product (target: L3)
```

**Current state (build 26.1.5061.0.1-dev):** M0–M2 are closed; M3 is at 75%, M4 at 72%, M5 at 92%, M6 at
99% and M10 at 37%. Our own UEFI boot manager chain-loads our loader, which boots
a PE32+ kernel written in V, exits boot services, builds identity page tables and
enters `KiSystemStartup`; the QEMU smoke run reports `SPOTBOOT true`,
`SPOTLDR true` and the kernel marker `true`. 5061 adds QEMU-proven sptdm graphical boot supervision (SPDR 2, pe-auto-ok), config-driven SPOTPE.CFG/KUSER multi-service policy markers, and svcprobe SVC1.EXE groundwork (real process launch blocked by active VFS FAT32 write issue, reverted to marker-only). The kernel brings up COM1, GDT/TSS/IDT,
exception handling, KPCR, hardware interrupts, a bugcheck path, ACPI/MADT,
LAPIC/IOAPIC, IRQL, spinlocks, a
calibrated LAPIC timer replacing the 8259/PIT, one IOAPIC line routed end-to-end
and PCI bus enumeration. It maps anonymous, file-backed and `SEC_IMAGE` NT
sections with demand paging, runs a preemptive round-robin scheduler, and loads
a PE32+ image with IAT/TEB/PEB into CPL3. Storage is a polling AHCI driver over FAT32 with in-place writes, exposed
as a published VFS volume; `NtWriteFile` writes through to the cached volume and
`NtFlushBuffersFile` synchronously flushes the whole 64 KiB image to the SATA
disk. FAT32 directory entries carry DOS creation/write timestamps, surfaced as
FILETIMEs through `FileBasicInformation`, `FileAllInformation` and
`FileNetworkOpenInformation`. The kernel also has event, mutant, semaphore and timer objects with named
opens, scheduler-blocking finite timed and multi-object waits, an object namespace with directory
and symbolic-link objects, I/O completion ports, direct virtual-memory
read/write, user APC delivery through a ring-3 trampoline, an in-memory registry,
timed delays and an NTFS parser.

**Spotlight X:** 12% — a deterministic GPT/FAT32 image boots the kernel and automatically launches
`\X\hello.exe` through the primary-GPT/sector-backed ESP path. A linked `ntdll.dll` exposes 98
host-validated syscall trampolines. The PE completes `NtQuerySystemTime` and the `XELL` marker;
ntdll loader/heap/RTL, Win32 and GUI remain absent.

**Spotlight PE:** 13% — host-tested installer core (83 installer tests + 14 answer tests + 15 image
tests + 12 ISO tests + 9 payload tests + 24 init tests), durable journal, dry-run executor,
deterministic GPT/FAT32 image builder, file-backed apply, journal-driven `no-gpt` rollback, host
console/init, unsigned ISO 9660/El Torito media, and deterministic unattended answer plans.
Built-in images and ISOs carry the freestanding `\SPOTPE\INIT.EXE` payload; NT
`test-pe-payload-launch` loads it with exact `INIT.CFG`, observes `PEP2` → `PEP5` → `PEP6` → `PEP4` and returns to the
parent. Normal boot honors an opted-in `\SPOTPE.CFG`; the real 64 MiB GPT server image reaches
`pe-auto-ok`. A full PE userland session, interactive installer and ISO auto-launch remain absent.

**Recent kernel progress:** 98 NT syscalls, up from 74 (the earlier count missed the
token/thread SSNs 0x24, 0x2f, 0x41 and the new 0xc2). The internal executable-child
bootstrap creates a child PML4 and initial thread and completes parent → child → parent
with CR3/PID/PEB/TSS/GS publication, rollback and cleanup (`test-exec-process`);
public `NtCreateProcess` keeps null-section object-only lifecycle and creates zero-thread
`SEC_IMAGE` child DTBs (`test-process-image`). Identity-based public `NtCreateThreadEx` handles complete
parent → child → parent with child-only stack/KUSER mappings (`test-public-process-thread`).
`NtCreateUserProcess` publishes real child PEB/TEB/process parameters and `NtResumeThread`
starts the suspended initial thread; `test-create-user-process` covers System and ring-3
non-System parents. The Spotlight PE payload runs through `PEP2` → `PEP5` → `PEP6` → `PEP4` (`test-pe-payload-launch`), and normal boot honors an opted-in `\SPOTPE.CFG` (`test-pe-auto-boot`). Recent work includes file and path-based
attribute queries (including rename via `FileRenameInformation`,
delete-on-close via `FileDispositionInformation`, `NtCreateFile` disposition
modes with FAT32 cluster growth, and subdirectory path resolution for
open/read/query), memory and sections (`NtReadVirtualMemory`/`NtWriteVirtualMemory`,
lock/unlock/flush), event/mutant/semaphore/timer objects with named
create/open/duplicate (`test-event`),
`NtOpenEvent`/`NtOpenMutant`/`NtOpenSemaphore`/`NtOpenTimer`/
`NtDuplicateObject`, process/thread control (`NtTerminateProcess`,
`NtTerminateThread`, alert-state calls; `NtQueueApcThread` queues user APCs,
delivered at `NtTestAlert` or an alertable wait through a ring-3 trampoline; the
routine runs with 3 arguments and execution resumes at the interrupted RIP —
`test-apc`; `NtCreateThreadEx` 0xc2 creates a user thread with its own stack and
ETHREAD and starts it in ring 3, `test-thread-create`; after exit the kind-12
handle is waitable without consuming the signal and
`NtQueryInformationThread` class 0 reports `ExitStatus` and
`UniqueThread = 0x100 + tid`, `test-thread-exit`), the object namespace
(`NtCreateDirectoryObject`/`NtOpenDirectoryObject`/`NtQueryDirectoryObject`,
symbolic-link objects, named sections with one-level link resolution, seeded
`\KnownDlls`, `\BaseNamedObjects`, `\Device`, `\??`; `test-namespace`),
I/O completion ports (`test-iocp`), in-memory registry keys and values seeded
from `HKLM\Software\Microsoft\Windows NT\CurrentVersion`, token queries for
classes 8/10/18/20, timing and thread information, current-process/thread
handle opens, timer resolution and instruction-cache flush.
`NtWaitForSingleObject` now blocks on the caller's own syscall stack for a
non-zero timeout and resumes on a signal or the TSC deadline; delay/yield share
the scheduler hooks. `NtWaitForMultipleObjects` (0x5b) / `NtWaitForMultipleObjects32` (0x1a) implement
`WaitAll`/`WaitAny` over events, mutants, semaphores and timers (`test-sync`);
signaled and zero-timeout waits acquire an unowned mutant or one owned by the
current thread, decrement a positive semaphore count and fire on timer expiry.
`test-blocking-wait` observes zero poll iterations plus signal and timeout resume.
Syscall numbers were re-checked against 19045, fixing `NtOpenThread` to 0x12f
and `NtResetEvent` to 0x17a. APC delivery restores `RAX` (NTSTATUS), `RBP`,
`RDX`, `R8`, `R9` and `R10`; XMM/FPU and exception context are still missing.
Also: a CPL3 PE image making a multi-argument `SYSCALL` including stack arguments
(`test-svc3`), AHCI + FAT32 read/write behind a VFS volume with a synchronous
whole-image flush, anonymous, file-backed and `SEC_IMAGE` NT sections with demand
paging, and a user VA allocator above 8 GiB. 39 QEMU test targets plus the host
FAT32 and unit suites are green. There is no third-party Windows `.exe` (L3),
no runnable ntdll/Win32/GUI or full PE userland session, no per-thread TEB,
no SMP, and no COW/pagefile yet; nested-path file creation and writes are not implemented.

---

## Start here

| Page | Read it if you want to… |
|---|---|
| [Installation](Installation.md) | go from a clean machine to a booting kernel |
| [Building](Building.md) | understand targets, flags and artifacts |
| [Running in QEMU](Running-in-QEMU.md) | control the VM, read the logs |
| [Boot Testing](Boot-Testing.md) | make a boot assert itself |
| [Architecture](Architecture.md) | see how the pieces fit |
| [Roadmap](Roadmap.md) | know what comes next and when |
| [Compatibility](Compatibility.md) | understand the `.exe` promise |
| [Troubleshooting](Troubleshooting.md) | fix a failed build or boot |
| [Contributing](Contributing.md) | send a change that gets merged |
| [Legal](Legal.md) | licensing, clean room, symbols |
| [FAQ](FAQ.md) | short answers to hard questions |

---

## The machine today

```
UEFI firmware
  └─ SPOTBOOT.EFI     boot manager: menu, config, chain-loads the loader
       └─ SPOTLDR.EFI      reads ntoskrnl.exe from the ESP, PE32+, relocations
            └─ ntoskrnl.exe   KiSystemStartup: COM1, GDT/TSS/IDT, KPCR, bugcheck
                 └─ (self-tests, then halts)
```

The QEMU smoke run reports `SPOTBOOT true`, `SPOTLDR true` and the kernel marker
`true`.

Proof, straight from QEMU:

```
I [boot] KiSystemStartup entered
I [boot] loader block accepted
I [boot] image_base=0x7bd37000
I [ke] GDT, TSS and IDT loaded; exceptions report to debugcon
I [ke] KPCR installed at GS base
I [ke] irql=0x0
I [ke] irql_raised=0x2
I [acpi] lapic_base=0xfee00000
I [acpi] ioapic_base=0xfec00000
I [hal] lapic_id=0x0
I [hal] ioapic_version=0x170020
I [hal] PIC and PIT ready; enabling interrupts
I [hal] pit_ticks=0x3
I [hal] PIC and PIT masked; LAPIC timer is the time source
I [hal] lapic_timer-ok
I [hal] lapic_timer_ticks=0xa
I [hal] lapic_bus_hz=0x3b735060
I [hal] ioapic_gsi0=0x2
I [hal] ioapic_irq-ok
I [hal] ioapic_irq_ticks=0x5
I [pci] devices=0x6
I [pci] first=0x29c08086
I [mm] free_pages=0x7ac8f
I [mm] paging-on
I [mm] cr3=0x100000
I [mm] alloc-free-ok
I [mm] pool-ok
I [mm] vad-ok
I [serial] loopback self-test passed
```

Self-tests: `test-boot` (38/38 lines), `test-lapic` (LAPIC timer), `test-timer` (10 ticks), `test-exception` (`EXC 3`),
`test-bugcheck` (`BUGCHECK 0x50000001`), `test-exe`/`test-exe-file` (PE + IAT/TEB/PEB), `test-disk`/`test-file`/`test-flush` (AHCI + FAT32 + VFS + persistent flush, rename, delete-on-close, subdirectory paths, DOS creation/write timestamps),
`test-section` (NT sections), `test-event` (named events), `test-namespace` (directory/symbolic-link objects), `test-iocp` (I/O completion),
`test-sync` (scheduler-blocking waits), `test-blocking-wait` (zero polls, signal/timeout resume), `test-x-hello` (FAT32 `\X\hello.exe` → process → syscall round-trip), `test-x-image` (full GPT image → automatic launch), `test-process` (null-section process lifecycle), `test-process-image` (`SEC_IMAGE` child DTB, refcount, rollback and cleanup), `test-public-process-thread` (public handles → child → parent → cleanup), `test-create-user-process` (SSN 0xc9, System + ring-3 non-System parent), `test-pe-payload-launch` (`INIT.CFG` → `PEP2` → `PEP5` → `PEP6` → `PEP4` → parent → cleanup), `test-pe-auto-launch`/`test-pe-auto-boot` (opted-in normal boot → `PEP2` → `PEP5` → `PEP6` → `PEP4`), `test-exec-process` (child PML4 + initial thread → parent → child → parent → cleanup), `test-apc` (APC delivery and resume), `test-thread-create` (thread creation and ring-3 start), `test-thread-exit` (exit, dequeue, post-exit wait and query), `test-exit` (thread/process exit), `test-registry` (in-memory keys and values),
`test-svc`/`test-svc3` (multi-argument SYSCALL); host unit and FAT32 tests. 39 QEMU test targets in total.

---

## Repositories

| Repository | Role |
|---|---|
| [spotlight-nt](https://github.com/fluidmstr/spotlight-nt) | kernel, boot chain, drivers, ABI specs |
| [spotlight-x](https://github.com/fluidmstr/spotlight-x) | userland, Win32, GUI, distribution |
| [spotlight-pe](https://github.com/fluidmstr/spotlight-pe) | installer and recovery environment |

Version: `26.1.4280.0.1-dev` (`YY.SEMVER.BUILD.0.1[-dev]`), tag
`v26.1.4280.0.1`.

---

> **A kernel is not a feature. It is a contract with every program you will ever run.**

> **No magic. If we cannot explain why it works, it does not work.**
