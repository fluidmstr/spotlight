# Troubleshooting

> Most failures on a fresh machine are one of five things. Check them in order.

---

## Build problems

| Symptom | Cause | Fix |
|---|---|---|
| `v: command not found` | V not in `PATH` | add `C:\v\v`, reopen the terminal |
| `v: warning: .vvmrc requests V 0.4.12` | wrong V version | install **0.5.2**; the pin is deliberate |
| `clang`/`lld-link` not found | LLVM missing | `winget install LLVM.LLVM`, new shell |
| `undefined symbol: __chkstk` | large stack frame | we provide `__chkstk`; make sure `boot/common/rt/memory.S` is linked |
| `undefined symbol: memcpy/memset` | runtime object missing | link `rt_asm.o` (the build does) |
| `_fltused`, SSE instructions | `f32`/`f64` in kernel code | remove floats |
| `"@vlib" is not recognized as a command` | `-path` used `|` and `v test` did not quote it | use `;` on Windows: `-path "kernel;@vlib"` |
| `unknown function: <sibling>` | compiled one file of a multi-file module | build the module directory via V→C→clang |
| `c'X'` rejected as `u8` | `c'...'` is `&u8` in V 0.5.2 | use backticks: `` `X` `` |
| `condition is always false` notice | leftover `if false` | use `$if define ? { ... }` instead |

---

## Boot problems

| Symptom | Meaning | What to check |
|---|---|---|
| `no serial log produced` | QEMU wrote nothing | is `-serial file:` in the command? did QEMU start at all? |
| serial has loader lines, no kernel lines | died before COM1 init | `build/debugcon.log` for `EXIT/PGON/JUMP` |
| debugcon empty | loader never ran | wrong `BOOTX64.EFI` on the ESP, or firmware didn't boot it |
| `EXC <n>` on debugcon | CPU exception reached the handler | IDT works; now fix the cause (n = vector) |
| `BUGCHECK 0x...` | deliberate or real stop | code tells the subsystem; see `docs/07-subsystems/bugcheck.md` |
| QEMU exits immediately with `-no-reboot` | triple fault | missing IST for `#DF`, bad IDT/TSS, or CR3 without identity map |
| `BdsDxe: failed to start ... Unsupported` | our EFI app returned an error status | read serial: the app prints why |

---

## QEMU / OVMF

| Symptom | Fix |
|---|---|
| `Could not open 'C:\Program'` | path with spaces passed unquoted; copy firmware to a space-free path |
| OVMF not found | set `SPOT_OVMF_CODE` / `SPOT_OVMF_VARS` |
| Boots to UEFI shell instead of our app | `\EFI\BOOT\BOOTX64.EFI` missing in `build/esp` |
| Black screen, no output | expected: `-display none`; use the logs |

---

## Kernel debugging

* Serial is the primary channel: `build/serial.log`.
* Port `0xE9` is the fallback before/after boot services: `build/debugcon.log`.
* For a crash, run QEMU with `-d int,cpu_reset -D build/qemu.log` and look at the
  last exception.
* WinDbg over KD is not implemented yet (M6).

---

## Still stuck?

Open an issue with:

1. the exact command,
2. `v version`, `clang --version`,
3. `build/serial.log` and `build/debugcon.log`,
4. the last 50 lines of the build output.

"No output" without those four things is not actionable.

---

## Next

* [Installation](Installation.md)
* [Boot Testing](Boot-Testing.md)
