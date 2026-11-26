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
  for 4 GiB, switches CR3, installs a kernel stack and passes `Loader_block`.
- **M2 (partial) — kernel bring-up.** COM1 16550 driver with loopback self-test,
  klog sink and formatter, GDT with user selectors, TSS with IST stacks, IDT with
  vectors 0..31, 32 exception stubs, `ke_exception_dispatch`, KPCR/KPRCB at the GS
  base (`GS:[0x188]`), `bugcheck`/`assert_failed`.
- `Loader_block` ABI specification and offset tests (`sizeof == 0x70`).
- Tooling: `ssdt-gen`, `symgen`, `pdb-abi`, `version` stamping, QEMU harness with
  serial expectations.
- Documentation: 12 guides plus kernel architecture, ABI and legal chapters.

### Verified

| Check | Result |
|---|---|
| `mk/build.vsh -- test-boot` | 8/8 serial expectations |
| `mk/build.vsh -- test-exception` | `EXC 3` on debugcon |
| `mk/build.vsh -- test-bugcheck` | `BUGCHECK 0x50000001` |
| `mk/build.vsh -- test` | 5/5 host suites |

### Not yet

No scheduler, no memory manager, no processes, no file systems, no `.exe`
support, no installer, no real hardware support.
