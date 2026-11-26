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
considered and rejected: it adds 12–18 months to L3 because no Wine/ReactOS code
could be reused.

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
| Wine | LGPL-2.1+ | userland layer (Track B), planned |
| ReactOS | GPL-2.0+/LGPL-2.1+ | reference / optional reuse, case by case |
| FreeType | FTL/GPLv2 | fonts in the GUI, planned |

New dependencies require a licence check and a row in `third_party/README.md`.

---

## Next

* [Contributing](Contributing.md)
* Kernel legal chapter: [`spotlight-nt/docs/06-risks-legal.md`](https://github.com/fluidmstr/spotlight-nt/blob/main/docs/06-risks-legal.md)
