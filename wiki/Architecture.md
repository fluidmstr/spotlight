# Architecture

> Three repositories, one machine. The kernel owns the ABI; everything else obeys it.

---

## Boot chain

```
UEFI firmware
  └─ SPOTBOOT.EFI        boot manager: menu, config, chainload          (planned)
       └─ SPOTLDR.EFI    loader: PE32+, paging, ExitBootServices
            └─ ntoskrnl.exe  Spotlight NT
                 └─ Spotlight X userland (ntdll, Win32, GUI)
```

Today `SPOTLDR.EFI` is started directly as `\EFI\BOOT\BOOTX64.EFI`. The boot
manager is designed but not implemented.

### What the loader does

1. `HandleProtocol(LoadedImage)` → device handle.
2. `SimpleFileSystem.OpenVolume` → `\EFI\SPOTLIGHT\ntoskrnl.exe`.
3. Read the image into pool memory.
4. Parse PE32+ (our parser, no Windows code), validate AMD64/native/sections.
5. `AllocatePages(AnyPages, LoaderCode)` and copy headers + sections; zero BSS.
6. Apply DIR64 relocations for the actual base.
7. `GetMemoryMap` with retries (the map grows as we allocate).
8. Build identity page tables for 4 GiB (2 MiB pages), allocate a kernel stack
   and a `Loader_block`.
9. `ExitBootServices`, load CR3, `spot_jump(entry, stack_top, block)`.

Everything after step 9 uses only port `0xE9` until the kernel's COM1 driver runs.

---

## Kernel components

| Component | Responsibility | State |
|---|---|---|
| **Ke** | GDT/IDT/TSS, IRQL, DPC/APC, scheduler, threads | 🟡 descriptors, KPCR, preemptive round-robin |
| **HAL** | CPU, PIC/PIT → APIC/HPET, PCI, ACPI, SMP | 🟡 COM1, ACPI/APIC, PCI, HPET |
| **Mm** | PFN database, paging, sections, pools, MDL | 🟡 PFN, pool, paging, VAD, sections |
| **Ob** | object manager, handles, namespaces | 🟡 event/mutant/semaphore/timer objects, named opens, duplicate; directory and symbolic-link objects, named sections |
| **Ps** | processes, threads, ETHREAD/EPROCESS | 🟡 processes, threads, CR3, scheduler, termination/alert |
| **Io** | IRPs, device stacks, PnP, power | ⬜ IRPs; I/O completion objects (queue/set/remove/query) only |
| **Cm** | registry, hives | ⬜ hives; in-memory key/value store only |
| **Se** | tokens, SIDs, ACL, integrity | ⬜ token queries only |
| **Lpc** | ALPC ports | ⬜ |
| **Cc** | file cache | ⬜ |
| **Syscall** | LSTAR/KiSystemCall64, SSDT | 🟡 91 handlers, multi-arg entry |

---

## ABI contracts

The kernel is only "NT-compatible" if the invisible contracts hold. Each one has
a specification and a test:

| Contract | Spec | Status |
|---|---|---|
| KPCR/KPRCB (`GS:[0x188]`) | `docs/05-abi/kpcr-kprcb.md` | assumed |
| Syscall ABI (LSTAR, R10, shadow space) | `docs/05-abi/syscall-abi.md` | assumed |
| Loader handoff block | `docs/05-abi/loader-block.md` | **verified** |
| PE32+ / SEH | `docs/05-abi/pe-coff-seh.md` | assumed |
| RTL types | `docs/05-abi/rtl-types.md` | assumed |
| IRP / IO_STACK_LOCATION | `docs/05-abi/irp-io-stack.md` | assumed |
| OBJECT_HEADER | `docs/05-abi/object-header.md` | assumed |
| TEB/PEB | `docs/05-abi/teb-peb.md` | assumed |
| GDT/IDT/TSS | `docs/05-abi/idt-gdt-tss.md` | assumed |

"Assumed" is a status, not an insult: it means the layout comes from public
documentation and must be confirmed on build 19045 before drivers depend on it.

---

## Repositories

| Repository | Contains | Depends on |
|---|---|---|
| `spotlight-nt` | `boot/`, `kernel/`, `drivers/`, `compat/`, `tools/`, `tests/`, `docs/` | — |
| `spotlight-x` | `user/`, `distro/`, tools, application corpus | `spotlight-nt` (submodule) |
| `spotlight-pe` | installer environment: console, explorer, installer | `spotlight-nt` (submodule) |

Versions are identical across all three: `YY.BUILD.0.1`.

---

## Design rules

1. **No allocation before Mm exists.** Early boot uses static buffers or
   `AllocatePages` from the loader.
2. **No V runtime.** No strings, no GC, no `error()`, no `println`.
3. **Every ABI field has an offset assert** or a host test.
4. **Observability before features.** A new subsystem must be visible in a log.
5. **No lying stubs.** Unimplemented returns `STATUS_NOT_IMPLEMENTED`.
6. **One ABI.** Windows 10 x64 19045; changes require an ADR.

---

## Next

* [Roadmap](Roadmap.md)
* [Compatibility](Compatibility.md)
* Kernel deep dive: [`spotlight-nt/docs/01-architecture.md`](https://github.com/fluidmstr/spotlight-nt/blob/main/docs/01-architecture.md)
