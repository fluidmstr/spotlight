# Spotlight documentation

> Three repositories, one machine. Start here.

---

```
Spotlight NT     ██████░░░░░░░░░░░░░░  30%   kernel: M0–M2 closed, M3 75%, M4 30%
Spotlight X      █░░░░░░░░░░░░░░░░░░░   5%   userland, Win32, GUI, distro
Spotlight PE     █░░░░░░░░░░░░░░░░░░░   5%   installer environment
Spotlight        ███░░░░░░░░░░░░░░░░░  18%   the project as a product (target: L3)
```

**Current state:** M0 and M1 are closed, M2 is closed, M3 is at 75% and M4 has
started. Our own UEFI loader boots a PE32+ kernel written in V, exits boot
services, builds identity page tables and enters `KiSystemStartup`. The kernel
brings up COM1, GDT/TSS/IDT, exception handling, KPCR, hardware interrupts
(PIC/PIT), a bugcheck path, the first M3 pieces — ACPI/MADT, LAPIC/IOAPIC, IRQL,
spinlocks, a calibrated LAPIC timer replacing the 8259/PIT, one IOAPIC line
routed end-to-end, PCI bus enumeration — and the first M4 pieces: a PFN page
database, kernel pool, own page tables with CR3 switch, a page-fault handler and
a VAD table.

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
  └─ SPOTLDR.EFI      reads ntoskrnl.exe from the ESP, PE32+, relocations
       └─ ntoskrnl.exe   KiSystemStartup: COM1, GDT/TSS/IDT, KPCR, bugcheck
            └─ (halts)
```

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

Self-tests: `test-boot` (31/31 lines), `test-lapic` (LAPIC timer), `test-timer` (10 ticks), `test-exception` (`EXC 3`);
`test-bugcheck` (`BUGCHECK 0x50000001`).

---

## Repositories

| Repository | Role |
|---|---|
| [spotlight-nt](https://github.com/fluidmstr/spotlight-nt) | kernel, boot chain, drivers, ABI specs |
| [spotlight-x](https://github.com/fluidmstr/spotlight-x) | userland, Win32, GUI, distribution |
| [spotlight-pe](https://github.com/fluidmstr/spotlight-pe) | installer and recovery environment |

Version: `26.4173.0.1` (`YY.BUILD.0.1`), tag `v26.4173.0.1`.

---

> **A kernel is not a feature. It is a contract with every program you will ever run.**

> **No magic. If we cannot explain why it works, it does not work.**
