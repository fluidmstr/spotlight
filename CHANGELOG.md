# Changelog

All notable milestones. Versions follow `YY.BUILD.0.1` (for example `26.4173.0.1`).

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
  - VAD table: insert/find/remove over 64 ranges with host tests and a boot
    smoke test (`vad-ok`).
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
