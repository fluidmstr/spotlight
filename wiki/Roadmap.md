# Roadmap

> Plans are cheap. A stage counts only when its exit criterion is a command that
> fails without it.

---

## Where we are

```
M0  ████████████████████ 100%   ✅
M1  ████████████████████ 100%   ✅
M2  ███████████████░░░░░  75%   🟡
M3  ░░░░░░░░░░░░░░░░░░░░   0%
...
Overall  ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~10% to L3
```

| Stage | Scope | State |
|---|---|---|
| **M0** | UEFI app in V, no libc, no runtime | ✅ |
| **M1** | loader, PE32+, `ExitBootServices`, own CR3, handoff | ✅ |
| **M2** | COM1 + klog, GDT/TSS/IDT, exceptions, KPCR, PIC/PIT, bugcheck | 🟡 75% |
| **M3** | APIC/IOAPIC/HPET, PCI, SMP | ⬜ |
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

### M2, in detail

Done:

* COM1 16550 polling driver with loopback self-test.
* `vrt` log sink, `fmt_hex`/`fmt_dec`, `info_kv`/`err_kv`.
* GDT with kernel/user selectors, TSS + IST stacks, `ltr`.
* IDT 0..31, 32 asm stubs, `ke_exception_dispatch` → `EXC <n>`.
* KPCR/KPRCB at `GS` base (`GS:[0x188]` = CurrentThread).
* `bugcheck(code, subsys, msg)`.\n* 8259 remap, PIT 100 Hz, 16 IRQ stubs with `iretq`, `sti`; `pit_ticks=0x3` observed.

Remaining:

* PIC remap + PIT 100 Hz + IRQ stubs (`iretq` frame), `sti`.
* Tick counter in the log, boot test for interrupts.
* APIC/HPET transition (M3).

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
main line. Track **A** (real Microsoft userland) is a probe into how accurate the
ABI really is.

---

## Honest timeline

With 5–6 senior engineers plus AI agents:

| Milestone | Calendar |
|---|---|
| L1 | 9–12 months |
| L3 | 30–40 months |
| L4 | +12–18 months |
| L5 | 7+ years, no guarantees |

The hard parts are not typing code. They are ABI archaeology, debugging drivers
you did not write, and hardware diversity. AI accelerates volume, not truth.

---

## Principles of planning

1. Vertical slices, not subsystems on paper.
2. Every stage ends with a command, a log line, or a test.
3. Estimates are revised monthly and publicly.
4. Scope creep is rejected with a link to this page.
5. "Works on QEMU" ≠ "works everywhere"; hardware labs start at M3.

---

## Next

* [Compatibility](Compatibility.md)
* [Architecture](Architecture.md)
