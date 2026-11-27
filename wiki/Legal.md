# Legal

> This page is an engineering policy, not legal advice. Before a public release,
> consult a lawyer.

---

## Licence

Spotlight is distributed under **GPL-2.0-or-later**.

Why:

* compatibility with **Wine** (LGPL-2.1+) and, where useful, ReactOS modules
  (GPL-2.0+/LGPL-2.1+);
* the kernel stays open, including for derivatives;
* GPLv2 is simpler to combine with existing components than v3.

The alternative — Apache-2.0 with everything written from scratch — was
considered and rejected: it adds 12–18 months to L3 because LGPL-licensed
Wine/ReactOS code could not be combined with a permissive kernel.

### Reuse policy (Wine / ReactOS)

* **Kernel-facing layers are original:** PE/DLL loader, registry/CM, GDI and
  framebuffer, Win32 ABI inside the kernel. Wine/ReactOS are behavioural
  references, not sources for `kernel/`.
* **Userland may reuse Wine (LGPL-2.1+)** only in isolated directories
  (`third_party/wine/`, `user/wine/`), with LGPL notices, source availability
  and a provenance entry. The combined work is distributed under
  GPL-2.0-or-later (LGPL-2.1 §3).
* **Headers, constants and tests** may be used as references; copied material
  keeps its licence notice.
* **Microsoft binaries and assets remain forbidden.**

Full decision and component table:
[ADR-0009](https://github.com/fluidmstr/spotlight-nt/blob/main/docs/adr/0009-wine-reuse.md).

---

## What the project does not contain

* Microsoft binaries: `.exe`, `.dll`, `.sys`, `.mui`, hives, ISOs, fonts.
* Leaked source code, including the Windows Research Kernel.
* Decompiled code or mechanical translations of it.
* Patched Microsoft drivers.
* Activation, signature or DRM bypasses.

The repository contains only original code, documentation, and manifests
(names, versions, hashes) of files the user supplies themselves.

---

## Clean room

For contested areas (for example `win32k` contracts) the project uses a
two-room process:

1. A research group studies public documentation and observed behaviour and
   writes a **specification** (facts, no code).
2. An implementation group writes code **only** from that specification.
3. The journal records who studied what and when; the spec is versioned.

---

## Symbols and PDB files

Microsoft public symbols are useful for *verifying* our structure layouts, but
their licence is aimed at debugging.

Project policy:

* PDB files are used **locally** by `tools/pdb-abi` to produce a **report**.
* Structures committed to the repository are written from public documentation.
* Generating shipped code from PDB type information is not allowed without a
  written legal review.

---

## Track A and user-provided Windows

Track A (real Microsoft userland and drivers) requires a licensed Windows
installation. The user provides it locally; the project never distributes it and
never publishes ready-made Track A images.

---

## Trademarks

"Windows" and "Microsoft" are trademarks of Microsoft Corporation. They are used
here descriptively, to state the compatibility target. Spotlight is not
affiliated with, endorsed by, or sponsored by Microsoft. The project does not use
Microsoft logos or trade dress.

---

## Dependencies

| Component | Licence | Use |
|---|---|---|
| V | MIT | compiler |
| LLVM/Clang/lld | Apache-2.0 with LLVM exceptions | toolchain |
| QEMU | GPL-2.0 | development only, not shipped |
| OVMF/EDK2 | BSD-2-Clause | test firmware only |
| Wine | LGPL-2.1+ | reference; isolated userland reuse only (ADR-0009) |
| ReactOS | GPL-2.0+/LGPL-2.1+ | reference / optional reuse, case by case |
| Mesa (RADV/ANV/NVK) | MIT | userland Vulkan ICD, upstream |
| DXVK / D8VK | zlib | userland D3D translation |
| VKD3D-Proton | LGPL-2.1+ | isolated userland D3D12 translation |
| Linux ext4/btrfs code (if reused) | GPL-2.0 | optional backend reference; combined work under GPLv2, provenance logged |
| FreeType | FTL/GPLv2 | fonts in the GUI, planned |

New dependencies require a licence check and a row in `third_party/README.md`.

---

## Reference installations

A licensed Windows installation may be used as a local **oracle** for ABI
verification, syscall numbers, registry/NTFS fixtures and behavioural probes.
It never enters the repository: only facts (offsets, ordinals, error codes) in our
own words. Visual assets are recreated, not extracted. The full policy and
workflow are in [Reference-Windows.md](Reference-Windows.md).

---

## Next

* [Contributing](Contributing.md)
* Kernel legal chapter: [`spotlight-nt/docs/06-risks-legal.md`](https://github.com/fluidmstr/spotlight-nt/blob/main/docs/06-risks-legal.md)
