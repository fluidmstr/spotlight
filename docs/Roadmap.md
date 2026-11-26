# Roadmap

> Plans are cheap. A stage counts only when its exit criterion is a command that
> fails without it.

---

## Where we are

```
M0  ████████████████████ 100%   ✅
M1  ████████████████████ 100%   ✅
M2  ████████████████████ 100%   ✅
M3  ███████████░░░░░░░░░  55%   🟡
M4  ░░░░░░░░░░░░░░░░░░░░   0%
M5  ░░░░░░░░░░░░░░░░░░░░   0%
M6  ░░░░░░░░░░░░░░░░░░░░   0%
M7  ░░░░░░░░░░░░░░░░░░░░   0%
M8  ░░░░░░░░░░░░░░░░░░░░   0%
M9  ░░░░░░░░░░░░░░░░░░░░   0%
M10 ░░░░░░░░░░░░░░░░░░░░   0%
M11 ░░░░░░░░░░░░░░░░░░░░   0%
M12 ░░░░░░░░░░░░░░░░░░░░   0%
M13 ░░░░░░░░░░░░░░░░░░░░   0%
M14 ░░░░░░░░░░░░░░░░░░░░   0%

Overall  ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~15% to L3
```

| Stage | Scope | State |
|---|---|---|
| **M0** | UEFI app in V, no libc, no runtime | ✅ |
| **M1** | loader, PE32+, `ExitBootServices`, own CR3, handoff | ✅ |
| **M2** | COM1 + klog, GDT/TSS/IDT, exceptions, KPCR, PIC/PIT, bugcheck | ✅ |
| **M3** | ACPI/MADT, LAPIC/IOAPIC, IRQL, spinlocks, LAPIC timer, PCI, SMP | 🟡 55% |
| **M4** | Mm: PFN database, paging, sections, pools | ⬜ |
| **M5** | Ob/Ps: objects, processes, threads, APC | ⬜ |
| **M6** | syscalls, first ring-3 process | ⬜ |
| **M7** | Io/PnP/Po — **gate G2** | ⬜ |
| **M8** | Cm: registry, hives | ⬜ |
| **M9** | Se: security | ⬜ |
| **M10** | storage, NTFS, Cc | ⬜ |
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

### M3, in detail — 40%

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

Remaining:

* Route IOAPIC lines for external devices; keep the 8259 masked for good.
* PCI enumeration (ECAM + legacy), resource discovery.
* SMP: per-CPU KPCR, INIT-SIPI-SIPI, AP bring-up.
* Adopt IRQL/spinlocks in klog and the scheduler paths.
* Calibrate the APIC bus frequency instead of assuming 100 MHz.

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
