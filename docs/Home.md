# Spotlight documentation

> Three repositories, one machine. Start here.

---

```
Overall   ████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░  ~11% to L3
```

**Current state:** M0 and M1 are closed, M2 is in progress. Our own UEFI loader
boots a PE32+ kernel written in V, exits boot services, builds identity page
tables and enters `KiSystemStartup`. The kernel brings up COM1, GDT/TSS/IDT,
exception handling, KPCR, hardware interrupts (PIC/PIT) and a bugcheck path.

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
I [hal] PIC and PIT ready; enabling interrupts
I [hal] pit_ticks=0x3
I [serial] loopback self-test passed
```

Self-tests: `test-boot` (11/11 lines), `test-timer` (10 ticks), `test-exception` (`EXC 3`);
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
