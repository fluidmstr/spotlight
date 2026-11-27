# Building

> Everything is built by `mk/build.vsh`. If a command is not there, it is not
> supported; one-off scripts in the repository root are not accepted.

---

## Targets

Run from the repository root (`spotlight-nt`, or `spotlight-x/kernel`):

| Target | What it does |
|---|---|
| `all` | `boot` + `loader` + `kernel` |
| `boot` | builds `SPOTBOOT.EFI` |
| `loader` | builds `SPOTLDR.EFI` |
| `kernel` | builds `ntoskrnl.exe` (+ `.pdb`) |
| `run-efi` | builds loader + kernel, boots QEMU, prints serial |
| `test` | host unit tests + ABI tests |
| `test-boot` | boots and checks `tests/boot/m2_serial.lines` |
| `test-exception` | builds with `-d spot_test_exception`, expects `EXC 3` |
| `test-bugcheck` | builds with `-d spot_test_bugcheck`, expects `BUGCHECK 0x50000001` |
| `test-kernel` | kernel self-tests (planned) |
| `fuzz` | parsers and syscall fuzzing (planned) |
| `version` | prints `VERSION` |
| `stamp` | regenerates `kernel/include/version.v` |
| `clean` | removes `build/` |

```powershell
v run mk/build.vsh -- all
v run mk/build.vsh -- test-boot
```

---

## The freestanding profile

Kernel and boot images are compiled with a deliberately hostile set of flags:

```
v -gc none -skip-unused -no-bounds-checking -no-builtin \
  -cc clang \
  -cflags "-ffreestanding -fno-builtin -fno-stack-protector -mno-red-zone \
           -mno-sse -mno-mmx -mno-avx -fno-pic \
           -Wno-incompatible-pointer-types -Wno-int-conversion" \
  -o build/obj/spotboot.o boot/spotboot/main.v
```

| Flag | Reason |
|---|---|
| `-no-builtin` | removes the V runtime: strings, `println`, panic machinery, Windows console code |
| `-gc none` | no garbage collector, no allocator |
| `-skip-unused` | drops unreachable code |
| `-no-bounds-checking` | otherwise every index pulls in `panic` → `eprintln` → Win32 |
| `-mno-red-zone` | red zone is illegal in kernel/interrupt context |
| `-mcmodel=kernel` | kernel sits in the higher half |
| `-fno-builtin` | our `memcpy`/`memset` must not become recursive calls |

Consequences you must accept while writing kernel code:

* No V `string`, `[]T`, `map`, `error()`, `Result`, `println`.
* Byte literals are backticks: `` `A` ``. `c'...'` is a C string (`&u8`), not a `u8`.
* All text formatting is manual (`vrt.fmt_hex`, `vrt.fmt_dec`).

---

## Linking

```powershell
lld-link /subsystem:efi_application /entry:efi_main /nodefaultlib ...
lld-link /subsystem:native          /entry:KiSystemStartup /nodefaultlib ...
```

`/nodefaultlib` means the kernel links only against our own runtime objects
(`boot/common/rt/memory.S`, `kernel/asm/*.S`). If a symbol is missing, it is a
real missing dependency — not something to satisfy with libcmt.

---

## Artifacts

| File | Purpose |
|---|---|
| `build/bin/SPOTBOOT.EFI` | boot manager (UEFI application) |
| `build/bin/SPOTLDR.EFI` | kernel loader |
| `build/bin/ntoskrnl.exe` | kernel (PE32+, native subsystem) |
| `build/bin/*.pdb` | symbols for WinDbg / `llvm-pdbdump` |
| `build/esp/` | FAT directory exposed to QEMU as the ESP |
| `build/serial.log` | COM1 output |
| `build/debugcon.log` | port 0xE9 output (loader marks, exceptions) |

`build/` is gitignored.

---

## Module layout and the `-path` flag

V resolves modules by directory. The kernel uses:

```
kernel/main.v        module main
kernel/include/      module include
kernel/vrt/          module vrt
kernel/hal/          module hal
kernel/ke/           module ke
```

Build commands pass `-path "kernel;@vlib"`. **On Windows use `;`**, not `|`:
`v test` does not quote the value, and `cmd` treats `|` as a pipe. This is a
real, reproduced failure mode.

A single-file compile of a module does not see its siblings. For multi-file
modules the pipeline is `v -shared -o build/obj/x.c kernel/hal` followed by
`clang -c`.

---

## Reproducibility

* V is pinned by `.vvmrc` (0.5.2).
* Every generated version header comes from `VERSION`
  (`YY.SEMVER.BUILD.0.1[-dev]`: year, API/ABI major line, random four-digit
  build, fixed `0.1` tail, optional `-dev` prerelease).
* `v run mk/build.vsh -- version` must print the same value on every machine.

---

## Next

* [Running in QEMU](Running-in-QEMU.md)
* [Boot Testing](Boot-Testing.md)
* [Troubleshooting](Troubleshooting.md)
