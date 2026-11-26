# Using a reference Windows installation

> Status: **policy + workflow** · 2026-11-26 · owner: arch

Spotlight is a clean-room reimplementation. A licensed Windows 10 installation is
an extremely useful **oracle**: it tells us what the ABI actually is and how the
system behaves. It is not a source of code, binaries, assets or visual identity.

This page defines what we may and may not do with such an installation.

---

## 1. What we use it for

| Use | Output | Where it goes |
|---|---|---|
| **ABI verification** | real offsets/sizes of `KPCR`, `KPRCB`, `EPROCESS`, `ETHREAD`, `IRP`, `OBJECT_HEADER` | `compat/abi/offsets.json` + spec rows updated to `verified` |
| **Syscall table** | `Nt*` → ordinal mapping for 19045 | `compat/ssdt/` (generated, text only) |
| **Registry fixtures** | real `SYSTEM`/`SOFTWARE` hives | test fixtures, **never committed** |
| **NTFS fixtures** | small disk images with known files | local test data, **never committed** |
| **Behaviour probes** | error codes, wait semantics, loader rules | specification notes in `docs/05-abi/` |
| **UI reference** | screenshots of layout, spacing, flows | design notes only; assets are recreated |

The installation stays on the developer's machine. Nothing extracted from it is
published except **facts** (numbers, offsets, names) expressed in our own words.

---

## 2. What we never do

* Copy or commit Microsoft binaries (`.exe`, `.dll`, `.sys`, `.mui`, hive files,
  fonts, icons, wallpapers, ISO images).
* Decompile Microsoft code and translate it into our source.
* Copy code from leaked sources, the WRK, or decompiler output.
* Ship a ready-made "Windows on Spotlight" image.
* Reuse Microsoft visual assets or logos.

Reverse engineering for **interoperability** is permitted in many jurisdictions
under conditions; we stay on the conservative side: observe behaviour, verify
documented structures, implement independently.

---

## 3. Local layout

Everything related to a reference installation lives outside version control:

```
F:\openNT\ref\                 # gitignored
├─ win10_22h2_x64.iso
├─ vm\                         # QEMU/Hyper-V disk, snapshots
├─ pdb\                        # symbols fetched from the public symbol server
├─ fixtures\
│  ├─ SYSTEM                   # hive copy
│  ├─ SOFTWARE
│  └─ ntfs-small.img
└─ notes\                      # observations, screenshots (not assets)
```

`.gitignore` in every repository contains `ref/`, `*.iso`, `*.vhd`, `*.vhdx`,
`*.hive`, `*.regf`.

---

## 4. Reference VM

1. Create a VM (QEMU q35 or Hyper-V Gen2), 4 vCPU, 4 GiB RAM, 60 GiB disk.
2. Install **Windows 10 x64 22H2, build 19045** — the frozen ABI target.
3. Take a clean snapshot; never install our tooling into it.
4. For ABI work, attach WinDbg (or use `llvm-pdbutil` offline with PDBs) and
   record only numbers.

### ABI oracle run

```
pdb-abi dump --pdb ntoskrnl.pdb --type _KPCR --type _KPRCB --type _EPROCESS
pdb-abi report --json abi.json --md report.md
pdb-abi compare --json abi.json --spec compat/abi/offsets.json
```

`compare` reports `missing / extra / moved`. A moved field is either a wrong
assumption in our spec or a genuine build difference — both must be resolved
before any driver depends on it.

### Syscall table run

The extractor reads `ntdll.dll` from the reference installation (or from a
mounted ISO image) on the developer machine and produces a text listing:

```
NtCreateFile 0x0055
NtReadFile   0x0006
...
```

`tools/ssdt-gen build --input listing.txt` turns it into the table we ship.

---

## 5. Visual identity

"Looks familiar" is a design goal; "copies Microsoft assets" is not allowed.

| Element | Rule |
|---|---|
| Layout, spacing, flows | may be recreated from observation |
| Icons | drawn by us; no extracted `.ico`/`.png` |
| Fonts | open fonts only — **Selawik** is metric-compatible with Segoe UI and SIL OFL |
| Wallpapers | original artwork |
| Logos and names | no Windows logo, no Microsoft marks; product is "Spotlight X" |
| Sounds | original or licensed open assets |

Screenshots of a reference installation may be kept as private design notes.
They are never published as part of the product.

---

## 6. What the oracle unlocks, in order

1. **M4/M5 ABI**: verify `KPCR/KPRCB` (GS offsets), `KTHREAD/ETHREAD`,
   `KPROCESS/EPROCESS`, `IRP`, `OBJECT_HEADER` — the contracts that make processes,
   syscalls and drivers possible.
2. **M6**: exact syscall numbers for 19045 and the `KiServiceTable` shape.
3. **M8**: real hive fixtures for the registry parser.
4. **M10**: NTFS images with known content for the file-system driver.
5. **M12**: the application corpus, installed from the user's own media.

---

## 7. Legal note

This document is engineering policy, not legal advice. Before publishing anything
derived from a reference installation, review `docs/Legal.md` and
`spotlight-nt/docs/06-risks-legal.md`.
