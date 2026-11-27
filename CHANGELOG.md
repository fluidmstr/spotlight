# Changelog

All notable milestones. Versions follow `YY.SEMVER.BUILD.0.1[-dev]`: `YY` is
the year, `SEMVER` is the API/ABI major line, `BUILD` is a random four-digit
build number, the `0.1` tail is fixed, and `-dev` marks a prerelease (for example
`26.1.4280.0.1-dev`, tag `v26.1.4280.0.1`).

---

## [Unreleased] - blocking waits, ABI checks, bootstrap images (2026-11-28)

### Added

- **DOS creation/write timestamps.** FAT32 directory entries now carry DOS
  creation and last-write date/time, set on create, overwrite and write. VFS
  file handles expose them as FILETIMEs through `FileBasicInformation`,
  `FileAllInformation` and `FileNetworkOpenInformation`; path-based
  `NtQueryAttributesFile`/`NtQueryFullAttributesFile` return the same values.
  `test-file` asserts a non-zero write time and a non-zero directory-creation
  time.
- **Thread lifecycle.** A created thread's kind-12 handle supports
  `NtTerminateThread` (exit code + ready-queue removal), a non-consumptive
  `NtWaitForSingleObject` once the thread has exited, and
  `NtQueryInformationThread` class 0 (`ThreadBasicInformation`: `ExitStatus`,
  `UniqueThread = 0x100 + tid`, current process id). New `test-thread-exit`
  covers the pre-exit wait timeout, terminate, dequeue, immediate post-exit
  wait and the query. `ps` publishes the lifecycle callbacks so `ke` stays
  `ps`-free; a late-SSN audit in `main` runs after `ps_init`
  (`syscall-slot-missing`), and `tests/abi/syscall_ssn_test.v` locks the 95
  documented SSN rows against the 19045 table.

### Changed

- **Scheduler-blocking waits.** Unsignaled non-zero-timeout waits dequeue the
  thread and resume on its own syscall stack after a signal or TSC deadline;
  delay/yield share the same hooks. `test-blocking-wait` reports zero poll
  iterations and proves both signal and timeout resume.
- **ABI assertions.** Host tests cover RTL structures and `OBJECT_HEADER`; boot
  refuses a KPCR/KPRCB layout that drifts from the documented GS offsets.
- **Bootstrap media.** X builds a tested GPT/FAT32 image that boots to HAL init;
  PE adds file-backed rollback, host `spotpe-init`, and unsigned ISO 9660/El
  Torito media that also boots to the kernel. Neither runs a userland session yet.
- **ntdll syscall surface.** X generates 95 naked x64 trampolines, an SSN table and PE exports
  from the kernel constants plus the 19045 CSV, links `ntdll.dll`, and validates every export and
  11-byte stub on the host (3494 assertions). No loader or process calls it yet.
- **Readiness scales:** Spotlight NT **72% → 74%** (M5 **60% → 65%**,
  M6 **95% → 96%**), the project overall **44% → 45%**, Spotlight X **5% → 8%**;
  Spotlight PE stays **8%**; M0–M2 100%, M3 75%, M4 65%, M10 37%, M11/M14 5%, all
  others 0%.
- **Tests:** 29 QEMU targets (adds `test-thread-exit` and `test-blocking-wait`).

### Verified

| Check | Result |
|---|---|
| `test-file` | non-zero DOS creation/write timestamps through `FileBasicInformation` |
| `test-thread-exit` | wait timeout, terminate, dequeue, wait 0, `ThreadBasicInformation` |
| `test-blocking-wait` | block (0 polls), signal resume, timeout resume |
| `tests/abi` | 7 suites: SSN/status/layout/list/RTL/OBJECT_HEADER/PS checks |
| X `test-ntdll` | 95 exports/stubs; 3494 table/CSV/kernel-constant/byte assertions |
| `test` (host) | ABI/SSN suite green, 95 SSN rows |

---

## [Unreleased] — historical: user thread creation, 95 SSNs, kernel 70%, project 42% (2026-11-27)

### Added

- **User thread creation.** `NtCreateThreadEx` (0xc2) allocates its own user
  stack (`mm.user_alloc`), takes a dynamic `ETHREAD` slot, is inserted into the
  ready queue and enters ring 3 (`spot_user_thread_start` →
  `spot_enter_user3_arg`) with the start routine and argument. New
  `test-thread-create` proves the scheduler switch and a syscall from the new
  thread (`thread-create-ok`). Remaining: thread teardown/join, per-thread TEB
  and TSS `RSP0`, and kernel-mode thread starts.

### Changed

- **Syscall count corrected to 95 SSNs:** the earlier token/thread SSNs
  `0x24`, `0x2f`, `0x41` and the new `0xc2` were not counted.
- **Readiness scales:** Spotlight NT **69% → 70%** (M5 **55% → 58%**,
  M6 **94% → 95%**), the project overall **41% → 42%**. Spotlight X stays
  **5%**, Spotlight PE stays **8%**; M0–M2 100%, M3 75%, M4 65%, M10 35%, all
  others 0%.
- **Tests:** 27 QEMU targets (adds `test-thread-create`).

### Verified

| Check | Result |
|---|---|
| `test-thread-create` | `thread-create-ok` (ring-3 start + syscall from the new thread) |
| QEMU smoke boot chain | `SPOTBOOT true`, `SPOTLDR true`, kernel marker `true` |

---

## [Unreleased] — historical: user APC delivery, kernel 69%, project 41% (2026-11-27)

### Added

- **User APC delivery.** `NtQueueApcThread` (0x45) queues a user APC;
  `NtTestAlert` (0x1c2) or an alertable wait arms it. On the return path
  `SYSRET` enters a mapped user trampoline that calls the routine with three
  arguments, restores `RAX`/`RBP` and resumes the interrupted RIP. New
  `test-apc` target proves both halves (`apc-routine-ok`, `apc-resume-ok`).
  Remaining limit: the full volatile register context is not saved — only
  `RAX`/`RBP`.

### Changed

- **Readiness scales:** Spotlight NT **68% → 69%** (M5 **52% → 55%**,
  M6 **93% → 94%**), the project overall **40% → 41%**. Spotlight X stays
  **5%**, Spotlight PE stays **8%**; M0–M2 100%, M3 75%, M4 65%, M10 35%, all
  others 0%.
- **Tests:** 26 QEMU targets (adds `test-apc`).

### Verified

| Check | Result |
|---|---|
| `test-apc` | `apc-routine-ok`, `apc-resume-ok` |
| QEMU smoke boot chain | `SPOTBOOT true`, `SPOTLDR true`, kernel marker `true` |

---

## [Unreleased] — historical: APC delivery, boot chain, PE apply path and refreshed scales (2026-11-27)

### Changed

- **Readiness scales:** Spotlight NT **68% → 69%**, Spotlight PE **6% → 8%**, the
  project overall **39% → 41%**, Spotlight X is **7%**. Milestones: M0–M2 100%,
  M3 75%, M4 65%, M5 **52% → 55%**, M6 **93% → 94%**, M10 35%, all others 0%.
- **APC delivery:** `NtQueueApcThread` queues user APCs; `NtTestAlert` or an alertable wait delivers
  them through a ring-3 trampoline (`test-apc`), restoring RAX/RBP and resuming the interrupted RIP.
- **Boot chain:** `SPOTBOOT.EFI` chain-loads `SPOTLDR.EFI`, `SPOTLDR` loads
  `ntoskrnl.exe`, and the kernel reaches HAL initialization under QEMU/OVMF.
- **Spotlight PE:** documented the host-tested installer core (59 tests), the
  durable journal, the dry-run executor, the deterministic GPT/FAT32 image
  builder with our boot artifacts, and the file-backed apply path (GPT
  primary/backup, FAT32 ESP, payload copy, read-back, journal). There is still
  no PE userland, console or interactive installer.

### Verified

| Check | Result |
|---|---|
| QEMU smoke boot chain | `SPOTBOOT true`, `SPOTLDR true`, kernel marker `true` |
| Spotlight PE host tests | 59 installer tests + 13 image tests green |

---

## [Unreleased] — 95-SSN surface, object namespace, I/O completion, direct VM access (2026-11-27)

### Added

- **95 NT syscalls / 95 SSNs** (the earlier documented count was an undercount;
  see the top entry). New since the 74-SSN pass:
  - **Waits:** `NtWaitForSingleObject` 0x04 now polls to a TSC deadline for finite
    timeouts; `NtWaitForMultipleObjects` 0x5b and `NtWaitForMultipleObjects32`
    0x1a implement `WaitAll`/`WaitAny` over events, mutants, semaphores and
    timers (`test-sync`). Blocking scheduler waits still return
    `STATUS_NOT_IMPLEMENTED`.
  - **Named objects and handles:** `NtOpenEvent` 0x40, `NtOpenMutant` 0x125,
    `NtOpenSemaphore` 0x12c, `NtOpenTimer` 0x130; `NtDuplicateObject` 0x3c
    creates reference-counted aliases.
  - **Process/thread:** `NtTerminateProcess` 0x2c and `NtTerminateThread` 0x53
    park the target after logging; `NtTestAlert` 0x1c2, `NtAlertThread` 0x6f and
    `NtAlertThreadByThreadId` 0x70 manage alert state. `NtQueueApcThread` 0x45
    was registered but returned `STATUS_NOT_IMPLEMENTED` at this pass; user APC
    delivery was added in the entry above.
  - **Object namespace:** `NtCreateDirectoryObject` 0xa7,
    `NtOpenDirectoryObject` 0x58, `NtQueryDirectoryObject` 0x144,
    `NtCreateSymbolicLinkObject` 0xc1, `NtOpenSymbolicLinkObject` 0x12e,
    `NtQuerySymbolicLinkObject` 0x15f, named sections and `NtOpenSection` 0x37
    with one-level link resolution. The namespace is seeded with `\KnownDlls`,
    `\BaseNamedObjects`, `\Device` and `\??` (`test-namespace`).
  - **I/O completion:** `NtCreateIoCompletion` 0xad, `NtSetIoCompletion` 0x1a3,
    `NtRemoveIoCompletion` 0x09, `NtQueryIoCompletion` 0x153 (`test-iocp`).
  - **Memory:** `NtReadVirtualMemory` 0x3f, `NtWriteVirtualMemory` 0x3a,
    `NtLockVirtualMemory` 0x10e, `NtUnlockVirtualMemory` 0x1cd,
    `NtFlushVirtualMemory` 0xec; `NtSetInformationProcess` 0x1c handles priority
    class and execute flags.
  - **Files:** `NtCreateFile` honours `FILE_SUPERSEDE`, `FILE_OPEN`,
    `FILE_CREATE`, `FILE_OPEN_IF`, `FILE_OVERWRITE` and `FILE_OVERWRITE_IF`, with
    FAT32 cluster allocation and growth; volume classes 7/8 are accepted. FAT32
    path resolution walks subdirectories for open/read/query; creation and writes
    are still root-only.
  - **Registry:** the in-memory store gained `NtQueryKey` 0x16, `NtDeleteKey`
    0xd4, `NtDeleteValueKey` 0xd7 and `NtFlushKey` 0xea alongside create/open/
    query/enumerate/set, seeded from
    `HKLM\Software\Microsoft\Windows NT\CurrentVersion`.
- **SSN corrections** against the Windows 10 19045 reference: `NtOpenThread`
  0x12f and `NtResetEvent` 0x17a.
- **Test target** additions: `test-namespace`, `test-iocp`; 25 QEMU targets and
  the host FAT32/unit suites run green.

### Verified

| Check | Result |
|---|---|
| 25 QEMU boot/test targets | green |
| `test-namespace` (directories, symbolic links, named sections) | green |
| `test-iocp` (completion queue set/remove/query) | green |
| `test-sync` (finite timed, `WaitAll`/`WaitAny`) | green |
| `test-file` (disposition modes, rename, delete-on-close) | green |
| host unit and FAT32 suites | green |

---

## [Unreleased] — 74-SSN surface, waits, named objects, registry (2026-11-27)

### Added

- **74 NT syscalls (up from 51).** File: `NtCreateFile` 0x55, `NtReadFile` 0x06,
  `NtWriteFile` 0x08, `NtQueryInformationFile` 0x11, `NtSetInformationFile` 0x27,
  `NtQueryAttributesFile` 0x3d, `NtQueryFullAttributesFile` 0x147,
  `NtFlushBuffersFile` 0x4b, `NtQueryDirectoryFile` 0x35.
  `NtSetInformationFile` now handles rename (`FileRenameInformation`) and
  delete-on-close (`FileDispositionInformation`), proven by `test-file`.
  Memory: `NtAllocateVirtualMemory` 0x18, `NtFreeVirtualMemory` 0x1e,
  `NtProtectVirtualMemory` 0x50, `NtCreateSection` 0x4a,
  `NtQuerySection` 0x51, `NtMapViewOfSection` 0x28, `NtUnmapViewOfSection` 0x2a. Events:
  `NtCreateEvent` 0x48, `NtOpenEvent` 0x40 (named create/open/duplicate,
  `test-event`), `NtSetEvent` 0x0e, `NtResetEvent` 0x7a,
  `NtQueryEvent` 0x56. Timing and waits: `NtWaitForSingleObject` 0x04
  (finite timed polling), `NtWaitForMultipleObjects` 0x5b,
  `NtWaitForMultipleObjects32` 0x1a (`test-sync`),
  `NtDelayExecution` 0x34, `NtQueryPerformanceCounter` 0x31,
  `NtQuerySystemTime` 0x5a, `NtQueryTimerResolution` 0x163,
  `NtSetTimerResolution` 0x1b2 (15.625 ms). Queries: `NtQueryInformationProcess` 0x19,
  `NtQuerySystemInformation` 0x36, `NtQueryVirtualMemory` 0x23,
  `NtQueryVolumeInformationFile` 0x49, `NtYieldExecution` 0x46.
  Objects: `NtClose` 0x0f (also accepts the -1/-2 current-process/thread
  pseudo-handles), `NtQueryObject` 0x10 (`ObjectBasicInformation`:
  attributes, granted access, handle/pointer counts, creation time),
  `NtOpenProcess` 0x26 and `NtOpenThread` 0x2f (validated `ClientId`, return the
  current pseudo-handles), `NtDuplicateObject` 0x3c. Named opens:
  `NtOpenEvent` 0x40, `NtOpenMutant` 0x125, `NtOpenSemaphore` 0x12c,
  `NtOpenTimer` 0x130.
  Process/thread control: `NtTerminateProcess` 0x2c, `NtTerminateThread` 0x53,
  `NtTestAlert` 0x1c2, `NtAlertThread` 0x6f, `NtAlertThreadByThreadId` 0x70
  (`test-exit`); `NtQueueApcThread` 0x45 is registered but returns
  `STATUS_NOT_IMPLEMENTED`. Mutants: `NtCreateMutant` 0xb4 (optional initial
  owner), `NtReleaseMutant` 0x20 (returns the previous count) and
  `NtQueryMutant` 0x156 (`MutantBasicInformation`: count, owned-by-caller,
  abandoned). Semaphores: `NtReleaseSemaphore` 0x0a (returns the previous
  count), `NtCreateSemaphore` 0xc0 (initial/maximum count) and
  `NtQuerySemaphore` 0x15e (`SemaphoreBasicInformation`: current/maximum).
  Timers: `NtCreateTimer` 0xc3 (notification/synchronization),
  `NtSetTimer` 0x62 (relative 100 ns due time, optional period),
  `NtCancelTimer` 0x61 (returns the previous state) and
  `NtQueryTimer` 0x38 (`TimerBasicInformation`: remaining time, state).
  Registry (in-memory, seeded from
  `HKLM\Software\Microsoft\Windows NT\CurrentVersion`; no hives):
  `NtOpenKey` 0x12, `NtCreateKey` 0x1d, `NtQueryKey` 0x16,
  `NtQueryValueKey` 0x17, `NtEnumerateKey` 0x32,
  `NtEnumerateValueKey` 0x13, `NtSetValueKey` 0x60, `NtDeleteKey` 0xd4,
  `NtDeleteValueKey` 0xd7, `NtFlushKey` 0xea (`test-registry`).
  Tokens: `NtOpenProcessToken` 0x129, `NtOpenProcessTokenEx` 0x30 and
  `NtQueryInformationToken` 0x21 (classes 8/10/18/20 return `TokenPrimary`,
  `TokenStatistics`, `TokenElevationTypeDefault` and a not-elevated state).
  Cache: `NtFlushInstructionCache` 0xe9 (validated
  no-op; x86 I-cache is coherent). Threads:
  `NtSetInformationThread` 0x0d, `NtQueryInformationThread` 0x25.
  `NtWaitForSingleObject` covers signaled, zero-timeout and finite timed waits,
  acquires an unowned mutant or one owned by the current thread, decrements a
  positive semaphore count and fires on timer expiry; blocking scheduler waits
  return `STATUS_NOT_IMPLEMENTED`. APC delivery was not implemented at this pass
  (added later; see the top entry).
- **Info-class coverage.** `NtQueryInformationFile` accepts file classes 4–9, 14,
  16, 17, 21, 22, 34, 35 and 39; `NtQueryInformationProcess` accepts 0, 2, 3, 4,
  5, 7, 12, 18, 20–22, 24–34, 36 and 43; `NtQuerySystemInformation` accepts 0, 1,
  3, 5, 7, 8, 9, 23, 35 and 103; `NtQueryInformationToken` accepts 8, 10, 18
  and 20 (`TokenType`/`TokenPrimary`, `TokenStatistics`,
  `TokenElevationType`/`TokenElevationTypeDefault`, `TokenElevation`/not
  elevated). `NtSetInformationFile` class 4
  (`FileBasicInformation`) updates mutable file attributes.
- **Storage**: polling AHCI (`READ DMA`/`WRITE DMA`, SPOTFAT port selection),
  FAT32 read and in-place write, rename (`FileRenameInformation`) and
  delete-on-close (`FileDispositionInformation`, `test-file`), and a published
  VFS volume over a 64 KiB disk image. Writes go through to the cached volume;
  `NtFlushBuffersFile` flushes the
  whole image to the SATA disk (`test-flush` re-reads sector 34).
- **Memory**: anonymous, file-backed and `SEC_IMAGE` NT sections with demand
  paging; user VA allocator above 8 GiB.
- **Ring 3**: PE loader with IAT/TEB/PEB at CPL3; multi-argument `SYSCALL` entry
  including arguments passed on the stack.
- **Scheduling**: preemptive round-robin.

### Verified

| Check | Result |
|---|---|
| `test-boot`, `test-exe`, `test-exe-file`, `test-disk` | green |
| `test-preempt`, `test-svc`, `test-svc3`, `test-sync` | green |
| `test-file` (rename, delete-on-close), `test-section` | green |
| `test-event` (named create/open/duplicate) | green |
| `test-exit` (thread/process termination) | green |
| `test-registry` (in-memory keys and values) | green |
| `test-misc` (timer resolution, pseudo-handle opens, I-cache flush) | green |
| `test-flush` (whole-image flush, device re-read) | green |
| host unit tests | green |

---

## [26.1.4280.0.1] — 2026-11-26

First bootable milestone. The kernel is not usable as an OS yet; it is a proven
foundation.

### Added

- **M0 — UEFI application on V.** `SPOTBOOT.EFI` prints a banner and the UEFI
  memory map in QEMU/OVMF. Built with zero libc and zero V runtime.
- **M1 — loader.** `SPOTLDR.EFI` reads `ntoskrnl.exe` from the ESP over
  `SimpleFileSystem`, parses PE32+, allocates pages, applies DIR64 relocations,
  retrieves the memory map, calls `ExitBootServices`, builds identity page tables
  for 4 GiB, switches CR3 (with readback verification), installs a kernel stack
  and passes `Loader_block`.
- **M2 — kernel bring-up (closed).** COM1 16550 driver with loopback self-test,
  klog sink and formatter, GDT with user selectors, TSS with IST stacks, IDT with
  vectors 0..31, 32 exception stubs, `ke_exception_dispatch`, KPCR/KPRCB at the GS
  base (`GS:[0x188]`), `bugcheck`/`assert_failed`.
- 8259 PIC remap, PIT at 100 Hz, 16 IRQ stubs with a correct `iretq` frame;
  interrupts enabled and `pit_ticks=0x3` observed in QEMU.
- **M3 (partial).**
  - ACPI RSDP → XSDT/RSDT → MADT parser with host tests on synthetic tables.
  - MADT values verified in QEMU: LAPIC `0xfee00000`, IOAPIC `0xfec00000`, 5 ISO
    overrides.
  - LAPIC enable/ID/EOI/timer API; IOAPIC init and redirection table (version
    `0x170020` read back).
  - IRQL via CR8 with `raise_irql`/`lower_irql`; measured `0 → 2 → 0`.
    QEMU stores IRQL in the low nibble (`CR8 = 2`), correcting the earlier
    `IRQL << 4` assumption.
  - Spinlocks with `lock cmpxchg` and IRQL pairing.
  - HPET driver (capabilities, counter, frequency, conversions) — not yet wired
    as the active timer.
  - LAPIC timer takeover: periodic vector `0x40`, APIC EOI routing, 8259 and PIT
    masked; `lapic_timer-ok` / `lapic_timer_ticks=0xa` observed in QEMU.
  - APIC bus frequency calibrated against HPET (PIT fallback): QEMU q35 measures
    ~998 MHz, not the assumed 100 MHz.
  - IOAPIC routed end-to-end: ISA IRQ0 → GSI 2 → vector `0x30`, 5 ticks with the
    8259 masked.
  - PCI bus-0 enumeration via mechanism #1: 6 functions, host bridge `8086:29c0`.
- **M4 (started) — memory manager.** PFN bitmap built from the loader memory map
  (conventional pages only, first 1 MiB reserved), allocation bitmap so reserved
  pages cannot be freed, contiguous allocator, and a first-fit kernel pool.
  QEMU: `free_pages=0x7ac8f` (~1.9 GiB), alloc/free round-trip and pool reuse pass.
  - Own page tables: identity map of 4 GiB with 2 MiB pages, CR3 switch with
    readback (`cr3=0x100000`, `paging-on`).
  - Page-fault handler: logs CR2 and the error code, then bugchecks; `test-pf`
    faults at `0x100000000` and observes `PF cr2/err`.
  - **Demand paging**: anonymous section objects, 4 KiB map/unmap on the kernel
    PML4, and a `#PF` path that installs the backing page and retries. The first
    attempt exposed two real bugs: the ISR stub did not preserve the
    general-purpose registers, and `iretq` was not dropping the CPU-pushed error
    code, so a repaired store retried with a clobbered `rax`. Both are fixed;
    `test-demand` touches three pages above 4 GiB (`demand-ok`, PTE `A/D` set).
  - ISR stubs now save/restore all 15 GPRs and drop the error code only for
    `err=1` vectors (`docs/07-subsystems/exceptions.md` §5.1).
  - **Syscall table (19045).** `tools/ntdll-ssn` extracts 472 exported stubs;
    `tools/kernel-ssdt` reads `KiServiceTable` (`0xc79f0`), `KiServiceLimit`
    (`0x1d9` = 473) and resolves names through `llvm-symbolizer` (OMAP-aware).
    All 472 ntdll names match the kernel; `0x005a` is `NtQuerySystemTime` —
    implemented in user mode, hence no ntdll stub. 15 slots share stubs
    (12 return `STATUS_NOT_IMPLEMENTED`, 3 return success). Artifacts:
    `compat/ssdt/ntoskrnl_19045.*` and `verify_19045.md`.
  - **M6 (started) — syscall entry and ring 3.** `EFER.SCE`, `STAR/LSTAR/FMASK`
    with readback, `swapgs`, a 16 KiB kernel syscall stack, `SYSRETQ` return and
    an SSN limit check: valid numbers return `STATUS_NOT_IMPLEMENTED`,
    out-of-range numbers return `STATUS_INVALID_SYSTEM_SERVICE`. `test-syscall`
    covers the policy; `test-ring3` enters CPL3 with a 29-byte static stub, does
    `SYSCALL` → kernel → `SYSRETQ` back to user, then a second `SYSCALL` with a
    marker (`syscall-user-ok`), proving the round-trip. The GDT was migrated to
    the 19045 selectors after reading `IA32_STAR` in `HalInitSystem`
    (kernel `0x10`, user `0x33`/`0x2B`, TSS `0x40`), and `FMASK` is `0x4700`.
  - **First syscall handler.** `NtQuerySystemTime` (0x5a) reads
    `KUSER_SHARED_DATA` through the kernel alias `0xFFFFF780_00000000`,
    validates the user pointer (`U/S` on every level) and copies the FILETIME.
    `test-ring3` now does `NtQuerySystemTime(buf)`, reads `*buf` in CPL3 and
    sends the value back through a second `SYSCALL`; the test compares it with
    `kshared-time=`. `KUSER_SHARED_DATA` is at the kernel alias only; the user
    address `0x7FFE0000` needs 2 MiB identity-page splitting.
  - **M5 (started) — EPROCESS/KTHREAD skeleton.** Verified packed layouts
    (`KTHREAD 0x430`, `ETHREAD 0x898`, `KPROCESS 0x438`, `EPROCESS 0xa40`) in
    `kernel/include/ps.v`, host ABI asserts, and a static System process
    (`PID 4`) / thread (`TID 4`) with `DirectoryTableBase = CR3`. The syscall
    stack lives after the ETHREAD in `ps.Spot_thread`; `ps-ready` and the
    per-thread stack address are in the boot log.
  - **SSN handler table.** `ke_syscall_dispatch` now indexes
    `syscall_slots[0x1d9]` filled by `syscall_register`; duplicates are
    rejected. `NtQuerySystemTime` is the first registered handler.
  - **CR3 + cooperative threads.** `ps_switch_process` loads
    `EPROCESS.Pcb.DirectoryTableBase` through `mm.switch_cr3` (with readback);
    `spot_swap_context` saves/restores the nonvolatile registers, and a second
    static thread starts from a synthetic frame. `test-threads` proves
    `0x100000 → 0x12b000 → 0x100000` and A→B→A (`thread-b-running`,
    `thread-a-resumed`). A 32-priority ready-queue module (`kernel/ps/sched`)
    now backs the choice: `sched-tid=0x1` comes from `pop_highest`, and B puts
    A back before switching. No preemption yet.
  - **RTC time source.** `kernel/hal/rtc.v` reads the CMOS RTC (BCD/12h/UIP,
    bounded retries) and fills `KUSER_SHARED_DATA` with the real boot time;
    the fixed 2026-01-01 value remains as a fallback.
  - **Preemptive scheduler.** The LAPIC timer (vector 0x40, 100 Hz) calls a
    `Tick_hook` after EOI; `ps_schedule` round-robins between threads via the
    ready queues and `spot_swap_context`. `test-preempt` proves two volatile
    counters advance without explicit switches (`preempt-a`, `preempt-b`,
    `preempt-ok`). Priorities/quantum/idle are still fixed.
  - **PE32+ parser.** `kernel/image/pe.v` validates DOS/COFF/optional headers,
    walks sections, maps RVA→file offset and iterates DIR64 relocations; host
    tests cover truncated/malformed images.
  - **PE user loader.** `kernel/loader/loader.v` maps sections with `U/S=1`,
    zero-fills tails and applies DIR64 deltas. `test-pe-user` builds a 0x400-byte
    PE32+, loads it at `0x140000000`, jumps to `0x140001000` in CPL3 and receives
    `pe-exec-ok` from its `SYSCALL` marker.
  - **IAT imports, TEB/PEB, user process.** `test-exe` resolves an
    `IMAGE_IMPORT_DESCRIPTOR` to a built-in ntdll stub (`mov r10,rcx; mov
    eax,0x5a; syscall; ret`), creates a new PML4 (`DTB 0x12c000`), maps TEB
    `0x170000000` / PEB `0x170001000`, sets `KERNEL_GS_BASE=TEB` and enters CPL3
    as PID `0x100`. The PE calls the IAT, reads the time from its buffer, checks
    `GS:[0x30]` and sends a marker (`exe-ok`).
  - **FAT32 read + file-backed .exe.** `kernel/fs/fat32.v` parses the BPB, root
    directory and FAT chains (8.3, read-only, partial reads); host tests cover
    parse/find/read and malformed images. `test-exe-file` builds a 64 KiB FAT32
    RAM disk with `SPOT.EXE`, reads it through the filesystem parser and runs the
    bytes through the PE loader (`file-fat32=0x800`, `exe-file-ok`). The disk is
    a RAM buffer; AHCI/NVMe and VFS are next.
  - **NTFS parser.** `kernel/fs/ntfs.v` (module `fs`) reads the VBR, FILE
    record, USA fixup, attributes and resident FILE_NAME; 32 host tests. NTFS
    is a compatibility backend, not the root.
  - **ABI helpers.** NTSTATUS u32 codes (`status.v`) with i32 aliases in
    `ntstatus.v`, and `LIST_ENTRY` operations (`list.v`) with host tests.
  - **Licence policy.** ADR-0009 and `docs/11-legal.md`: kernel-facing layers
    (PE/DLL, registry, GDI, Win32 ABI) stay original; Wine (LGPL-2.1+) may be
    reused only in isolated userland with notices.
  - **Product decisions.** `X:\` is the native root with `C:\` as a per-process
    Win32 alias; the VFS targets ext4 (root), FAT32 (ESP), NTFS (compat) and
    btrfs, not NTFS-only (ADR-0010). `spotinit` falls back to `spotterm` when the
    desktop is absent (ADR-0012). Graphics uses own kernel drivers + a DRM-like
    UAPI for Mesa RADV/ANV/NVK and DXVK/D8VK/VKD3D; WDDM is not the path
    (ADR-0011).
  - VAD table: insert/find/remove over 64 ranges with host tests and a boot
    smoke test (`vad-ok`).
  - **ABI oracle**: a licensed Windows 10 22H2 image + public PDB symbols were
    used to verify real layouts. `KPCR.Prcb = 0x180`, `KPRCB.CurrentThread = 0x008`
    (`GS:[0x188]`), `EPROCESS.UniqueProcessId = 0x440`, `ETHREAD.Cid = 0x478`,
    `KTHREAD.Teb = 0x0f0`. Our `KPRCB.Number` was wrong (0x020) and is now 0x024.
    Details: `spotlight-nt/docs/05-abi/verified-19045.md`.
- `Loader_block` ABI specification and offset tests (`sizeof == 0x70`).
- Tooling: `ssdt-gen`, `symgen`, `pdb-abi`, version stamping, QEMU harness with
  serial expectations.
- Documentation: 12 guides plus kernel architecture, ABI and legal chapters.

### Verified

| Check | Result |
|---|---|
| `mk/build.vsh -- test-boot` | 31/31 serial expectations |
| `mk/build.vsh -- test-lapic` | `lapic_timer-ok`, PIC/PIT masked |
| `mk/build.vsh -- test-timer` | 10 sustained interrupts (`timer-ok`) |
| `mk/build.vsh -- test-exception` | `EXC 3` on debugcon |
| `mk/build.vsh -- test-bugcheck` | `BUGCHECK 0x50000001` |
| `mk/build.vsh -- test` | 5/5 host suites (incl. ACPI) |

### Not yet

No scheduler, no memory manager, no processes, no file systems, no `.exe`
support, no installer, no real hardware support. The APIC timer is initialized
but the PIT still drives time; PCI and SMP are ahead.
