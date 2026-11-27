# Roadmap

> Plans are cheap. A stage counts only when its exit criterion is a command that
> fails without it.

---

## Where we are

```
M0  ████████████████████ 100%   ✅
M1  ████████████████████ 100%   ✅
M2  ████████████████████ 100%   ✅
M3  ███████████████░░░░░  75%   🟡
M4  █████████████░░░░░░░  65%
M5  ████████████░░░░░░░░  60%
M6  ███████████████████░  95%
M7  ░░░░░░░░░░░░░░░░░░░░   0%
M8  ░░░░░░░░░░░░░░░░░░░░   0%
M9  ░░░░░░░░░░░░░░░░░░░░   0%
M10 ███████░░░░░░░░░░░░░  37%
M11 ░░░░░░░░░░░░░░░░░░░░   0%
M12 ░░░░░░░░░░░░░░░░░░░░   0%
M13 ░░░░░░░░░░░░░░░░░░░░   0%
M14 ░░░░░░░░░░░░░░░░░░░░   0%

Overall  ████████████████████████████████░░░░░░░░░░░░  kernel 74% · project 45% · PE 8%
```

The boot chain works end to end: `SPOTBOOT.EFI` chain-loads `SPOTLDR.EFI`, which
loads `ntoskrnl.exe`; the kernel reaches HAL initialization under QEMU/OVMF, and
the smoke run reports `SPOTBOOT true`, `SPOTLDR true` and the kernel marker `true`.

| Stage | Scope | State |
|---|---|---|
| **M0** | UEFI app in V, no libc, no runtime | ✅ |
| **M1** | loader, PE32+, `ExitBootServices`, own CR3, handoff | ✅ |
| **M2** | COM1 + klog, GDT/TSS/IDT, exceptions, KPCR, PIC/PIT, bugcheck | ✅ |
| **M3** | ACPI/MADT, LAPIC/IOAPIC, IRQL, spinlocks, timers, PCI, SMP | 🟡 75% |
| **M4** | Mm: PFN, pool, page tables, VAD, sections | 🟡 65% |
| **M5** | Ob/Ps: objects, processes, threads, APC | 🟡 60% |
| **M6** | syscalls, ring-3 process, dispatch | 🟡 95% |
| **M7** | Io/PnP/Po — **gate G2** | ⬜ |
| **M8** | Cm: registry, hives | ⬜ |
| **M9** | Se: security | ⬜ |
| **M10** | storage, NTFS, Cc | 🟡 37% |
| **M11** | ntdll, ALPC, console | ⬜ |
| **M12** | Win32 layer, `.exe` corpus → **L3** | ⬜ |
| **M13** | Track A: real MS drivers and userland | 🔬 |
| **M14** | Spotlight X 1.0: image, installer | ⬜ |

### M2, in detail — closed

* COM1 16550 polling driver with loopback self-test.
* `vrt` log sink, `fmt_hex`/`fmt_dec`, `info_kv`/`err_kv`.
* GDT with kernel/user selectors, TSS + IST stacks, `ltr`.
* IDT 0..31, 32 asm stubs, `ke_exception_dispatch` → `EXC <n>`.
* KPCR/KPRCB at `GS` base (`GS:[0x188]` = CurrentThread).
* `bugcheck(code, subsys, msg)` and `assert_failed`.
* 8259 remap, PIT 100 Hz, 16 IRQ stubs with `iretq`, `sti`; `pit_ticks=0x3`.

### M3, in detail — 75%

Done:

* **ACPI**: RSDP → XSDT/RSDT → MADT parser, host-tested on synthetic tables.
  Signatures are inline literals (pointer consts are null without `_vinit`).
* **MADT values from QEMU**: LAPIC `0xfee00000`, IOAPIC `0xfec00000`, 5 ISO overrides.
* **LAPIC**: enable, ID, EOI, periodic timer API (not yet the active timer).
* **IOAPIC**: init, redirection table, masking; version `0x170020` read back.
* **IRQL**: CR8-based raise/lower, `current_irql`; measured `0 → 2 → 0`.
  QEMU stores IRQL in the low nibble (`CR8 = 2`, not `0x20`).
* **Spinlocks**: `Kspin_lock` with `lock cmpxchg`, IRQL-paired acquire/release.
* **HPET**: capabilities, counter, frequency, tick/us conversions (driver only).
* **LAPIC timer takeover**: periodic vector `0x40`, APIC EOI routing, 8259 and PIT
  masked; `lapic_timer-ok` and `lapic_timer_ticks=0xa` observed.
* **APIC bus calibration**: one-shot countdown measured against HPET (PIT fallback);
  QEMU q35 measures ~998 MHz instead of the assumed 100 MHz.
* **IOAPIC end-to-end**: ISA IRQ0 → GSI 2 → vector `0x30`; 5 ticks observed while
  the 8259 is masked.
* **PCI enumeration**: mechanism #1 on bus 0; 6 functions, host bridge `8086:29c0`.

Remaining:

* SMP: per-CPU KPCR, INIT-SIPI-SIPI, AP bring-up.
* Adopt IRQL/spinlocks in klog and the scheduler paths.
* ECAM/MCFG, BAR discovery and device-to-GSI wiring for real drivers.
* HPET as a stable time source (currently calibration only); TSC deadline timer.

### M4–M6 and M10, in detail

Done:

* **M4 (65%)**: PFN database, kernel pool, own page tables with CR3 switch, VAD,
  page-fault handling, anonymous, file-backed and `SEC_IMAGE` NT sections with
  demand paging, and a user VA allocator above 8 GiB.
* **M5 (65%)**: EPROCESS/KTHREAD skeletons, process CR3 switching, 32-priority
  ready queues, preemptive round-robin driven by the LAPIC timer, reference-counted
  handle duplication (`NtDuplicateObject`), thread/process termination plus
  alert state (`test-exit`), user APC delivery through a ring-3 trampoline
  (`test-apc`: `apc-routine-ok`, `apc-resume-ok`), dynamic user threads
  (`NtCreateThreadEx` 0xc2: own user stack, ETHREAD slot, ready-queue insertion
  and a ring-3 entry; `test-thread-create`), and thread exit through the kind-12
  handle (`NtTerminateThread` records the exit code and dequeues; a
  non-consumptive wait returns immediately after exit; `ThreadBasicInformation`
  reports `ExitStatus`/`UniqueThread`; `test-thread-exit`).
* **M6 (96%)**: SYSCALL entry with stack arguments, 95 NT handlers covering
  file (incl. path-based attribute queries, `NtCreateFile` disposition modes,
  rename and delete-on-close), memory (incl. `NtReadVirtualMemory`/
  `NtWriteVirtualMemory`, lock/unlock/flush), event, mutant, semaphore and timer
  objects with named opens, scheduler-blocking waits (single- and multi-object
  `WaitAll`/`WaitAny`, finite timeout resume on the caller's syscall stack), an in-memory registry, the object
  namespace (directories, symbolic links, named sections, seeded `\KnownDlls`/
  `\BaseNamedObjects`/`\Device`/`\??`), I/O completion ports, object,
  process/thread control (terminate, alerts, priority class via
  `NtSetInformationProcess`), time, token, thread, process and system
  queries, plus pseudo-handle opens, timer resolution and instruction-cache
  flush; PE32+ loader with IAT/TEB/PEB at CPL3. `NtWaitForSingleObject` covers
  signaled, zero-timeout and scheduler-blocking finite waits; the multi-object
  calls add `WaitAll`/`WaitAny` over events, mutants, semaphores and timers
  (`test-sync`); `test-blocking-wait` proves zero poll iterations and signal/timeout
  resume on the same syscall stack. Single-object waits acquire an unowned mutant
  or one owned by the current thread, decrement a positive semaphore count and
  fire on timer expiry. User APCs are
  delivered (`NtQueueApcThread` 0x45 → `NtTestAlert`/alertable wait → ring-3
  trampoline) but only `RAX`/`RBP` are restored. `NtCreateThreadEx` 0xc2 creates
  a thread with its own user stack and ETHREAD, inserts it into the ready queue
  and starts it at CPL3 (`test-thread-create`); its kind-12 handle then supports
  exit/dequeue, post-exit wait and `ThreadBasicInformation`
  (`test-thread-exit`). File handles carry DOS creation/write timestamps, and
  `FileBasicInformation`/`FileAllInformation`/`FileNetworkOpenInformation`
  return them as FILETIMEs (`test-file`). SSNs were re-checked against
  19045 (`NtOpenThread` 0x12f, `NtResetEvent` 0x17a); the earlier documented count
  was an undercount (token/thread SSNs 0x24, 0x2f, 0x41 and 0xc2).
* **M10 (37%)**: polling AHCI (`READ`/`WRITE DMA`, SPOTFAT port selection), FAT32
  read and in-place write with cluster allocation on growth, path resolution
  through subdirectories for open/read/query (creation and writes stay
  root-only), rename and delete-on-close, DOS creation/write timestamps in
  directory entries, a published VFS volume
  over a 64 KiB disk image, write
  through the cached volume, `NtFlushBuffersFile` synchronously flushing the whole
  image to disk (`test-flush`), and the NTFS parser.

Remaining:

* M4: working sets, copy-on-write, section sharing.
* M5: priorities and quantum, idle thread, scheduler-integrated APC, join/stack
  reclamation, per-thread TEB and TSS `RSP0`.
* M6: full APC register-context save, remaining syscall semantics.
* M10: block I/O abstraction, ext4 backend, cache manager with write-back, partial/async flush, nested-path creation and writes.

Boot tests `test-disk`, `test-file`, `test-flush`, `test-section`, `test-event`,
`test-namespace`, `test-iocp`, `test-sync`, `test-blocking-wait`, `test-exit`, `test-registry`,
`test-svc`, `test-svc3`, `test-misc`, `test-preempt`, `test-apc`,
`test-thread-create`, `test-thread-exit` and the host unit/FAT32
suites are green (2026-11-27) — 28 QEMU test targets in total.

---

## The ladder

| Level | Definition | State |
|---|---|---|
| L0 | our loader and kernel boot on UEFI | ✅ |
| L1 | ring-3 process, syscall round-trip | ⬜ |
| L2 | our ntdll + kernel32 + CRT, our programs | ⬜ |
| L3 | **20 third-party `.exe` binaries pass checklists** | ⬜ |
| L4 | real Microsoft drivers load and work | ⬜ |
| L5 | real Windows 10 userland on our kernel | 🔬 |

> **L3 is the release.** L4/L5 are research tracks.

---

## Gate G2

After M7 the project makes a factual decision, not an emotional one:

| Question | Threshold |
|---|---|
| Do MS `acpi.sys` and `pci.sys` load unmodified? | yes |
| Does `storport` + `disk` + `ntfs` work on our Io? | yes |
| Is `win32k` estimated at ≤ 9 months? | yes |
| Otherwise | freeze Track A, ship Track B |

Track **B** (our kernel + our userland, Wine-derived DLLs where allowed) is the
main line. Track **A** (real Microsoft userland) probes ABI accuracy.

---

## Honest timeline

With 5–6 senior engineers plus AI agents:

| Milestone | Calendar |
|---|---|
| L1 | 9–12 months |
| L3 | 30–40 months |
| L4 | +12–18 months |
| L5 | 7+ years, no guarantees |

The hard parts are ABI archaeology, debugging drivers you did not write, and
hardware diversity. AI accelerates volume, not truth.

---

## Principles of planning

1. Vertical slices, not subsystems on paper.
2. Every stage ends with a command, a log line, or a test.
3. Estimates are revised monthly and publicly.
4. Scope creep is rejected with a link to this page.
5. "Works on QEMU" ≠ "works everywhere"; hardware labs start at M3.
