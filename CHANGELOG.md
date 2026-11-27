# Changelog

All notable milestones. Versions follow `YY.BUILD.0.1` (for example `26.4173.0.1`).

---

## [Unreleased] — kernel I/O and syscall progress (2026-11-26, updated 2026-11-27)

### Added

- **38 NT syscalls.** File: `NtCreateFile` 0x55, `NtReadFile` 0x06,
  `NtWriteFile` 0x08, `NtQueryInformationFile` 0x11, `NtSetInformationFile` 0x27,
  `NtQueryAttributesFile` 0x3d, `NtQueryFullAttributesFile` 0x147,
  `NtFlushBuffersFile` 0x4b, `NtQueryDirectoryFile` 0x35.
  Memory: `NtAllocateVirtualMemory` 0x18, `NtFreeVirtualMemory` 0x1e,
  `NtProtectVirtualMemory` 0x50, `NtCreateSection` 0x4a,
  `NtQuerySection` 0x51, `NtMapViewOfSection` 0x28, `NtUnmapViewOfSection` 0x2a. Events:
  `NtCreateEvent` 0x48, `NtSetEvent` 0x0e, `NtResetEvent` 0x7a,
  `NtQueryEvent` 0x56. Timing and waits: `NtWaitForSingleObject` 0x04,
  `NtDelayExecution` 0x34, `NtQueryPerformanceCounter` 0x31,
  `NtQuerySystemTime` 0x5a, `NtQueryTimerResolution` 0x163,
  `NtSetTimerResolution` 0x1b2 (15.625 ms). Queries: `NtQueryInformationProcess` 0x19,
  `NtQuerySystemInformation` 0x36, `NtQueryVirtualMemory` 0x23,
  `NtQueryVolumeInformationFile` 0x49, `NtYieldExecution` 0x46.
  Objects: `NtClose` 0x0f (also accepts the -1/-2 current-process/thread
  pseudo-handles), `NtQueryObject` 0x10 (`ObjectBasicInformation`:
  attributes, granted access, handle/pointer counts, creation time),
  `NtOpenProcess` 0x26 and `NtOpenThread` 0x2f (validated `ClientId`, return the
  current pseudo-handles). Cache: `NtFlushInstructionCache` 0xe9 (validated
  no-op; x86 I-cache is coherent). Threads:
  `NtSetInformationThread` 0x0d, `NtQueryInformationThread` 0x25.
  `NtWaitForSingleObject` covers signaled and zero-timeout waits; blocking waits
  return `STATUS_NOT_IMPLEMENTED`.
- **Info-class coverage.** `NtQueryInformationFile` accepts file classes 4–9, 14,
  16, 17, 21, 22, 34, 35 and 39; `NtQueryInformationProcess` accepts 0, 2, 3, 4,
  5, 7, 12, 18, 20–22, 24–34, 36 and 43; `NtQuerySystemInformation` accepts 0, 1,
  3, 5, 7, 8, 9, 23, 35 and 103. `NtSetInformationFile` class 4
  (`FileBasicInformation`) updates mutable file attributes.
- **Storage**: polling AHCI (`READ DMA`/`WRITE DMA`, SPOTFAT port selection),
  FAT32 read and in-place write, and a published VFS volume over a 64 KiB disk
  image. Writes go through to the cached volume; `NtFlushBuffersFile` flushes the
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
| `test-file`, `test-section`, `test-event` | green |
| `test-misc` (timer resolution, pseudo-handle opens, I-cache flush) | green |
| `test-flush` (whole-image flush, device re-read) | green |
| host unit tests | green |

---

## [26.4173.0.1] — 2026-11-26

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
