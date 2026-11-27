# Installation

> Status: **developer preview**. There is no installer for Spotlight X yet.
> This page gets you from a clean Windows machine to a booting kernel in QEMU.

---

## 1. What you need

| Component | Version | Why |
|---|---|---|
| Windows 10/11 x64 | any recent | development host (Linux/macOS untested) |
| [V](https://vlang.io) | **0.5.2** exactly | the kernel and tooling are written in V |
| LLVM / clang + lld | 18+ (23 tested) | V's C backend, PE linking |
| QEMU | 8+ | the only supported machine right now |
| OVMF | ships with QEMU | UEFI firmware for the VM |
| Git | any | submodules |
| Python 3 | 3.10+ | ABI and tooling scripts |

Check what you already have:

```powershell
v version
clang --version
lld-link --version
qemu-system-x86_64 --version
```

If any of these are missing, install them first — the build fails early and loudly.

---

## 2. Install the toolchain

### V 0.5.2

The project pins V in `.vvmrc`. Other versions may work, but they are not tested.

```powershell
Invoke-WebRequest -UseBasicParsing `
  https://github.com/vlang/v/releases/download/0.5.2/v_windows.zip `
  -OutFile v.zip
Expand-Archive v.zip -DestinationPath C:\v
C:\v\v\v.exe version        # must print 0.5.2
```

Add `C:\v\v` to `PATH`, then reopen the terminal.

### LLVM

```powershell
winget install LLVM.LLVM
```

Verify that `clang.exe` and `lld-link.exe` are in `PATH`. The build scripts also
look for the default install path `C:\Program Files\LLVM\bin` as a fallback.

### QEMU and OVMF

```powershell
winget install SoftwareFreedomConservancy.QEMU
```

OVMF firmware ships with QEMU at `C:\Program Files\qemu\share\edk2-x86_64-code.fd`.
The scripts use that path by default and can be overridden (see
[Running in QEMU](Running-in-QEMU.md)).

---

## 3. Get the sources

The kernel is a submodule of the OS repository:

```powershell
git clone --recursive https://github.com/fluidmstr/spotlight-x.git
cd spotlight-x
```

If you cloned without `--recursive`:

```powershell
git submodule update --init --recursive
```

You should now have:

```
spotlight-x/
├─ kernel/      <- spotlight-nt (submodule)
├─ user/        <- userland (future)
├─ distro/      <- image and installer (future)
└─ mk/
```

Working on the kernel itself? Clone `spotlight-nt` directly and use its own
`mk/build.vsh`; everything below applies there too.

---

## 4. Build

```powershell
cd kernel
v run mk/build.vsh -- test
```

You should see host tests pass and three artifacts appear:

```
build/bin/SPOTBOOT.EFI     # boot manager
build/bin/SPOTLDR.EFI      # loader
build/bin/ntoskrnl.exe     # kernel
```

If the build fails, jump to [Troubleshooting](Troubleshooting.md) — the first
five entries cover almost every fresh-machine problem.

---

## 5. First boot

```powershell
v run mk/build.vsh -- run-efi
```

This builds everything, prepares `build/esp/EFI/BOOT/BOOTX64.EFI`, starts QEMU
with OVMF and waits 20 seconds. The tail of `build/serial.log` should contain:

```
I [boot] KiSystemStartup entered
I [boot] loader block accepted
I [boot] image_base=0x...
I [ke] GDT, TSS and IDT loaded; exceptions report to debugcon
I [ke] KPCR installed at GS base
I [serial] loopback self-test passed
```

The kernel ends in `cli; hlt`. QEMU is killed by the script after the timeout.
That is success.

---

## 6. Prove the machine

Three tests turn "it printed something" into evidence:

```powershell
v run mk/build.vsh -- test-boot        # serial expectations, 8/8
v run mk/build.vsh -- test-exception   # expects EXC 3 on debugcon
v run mk/build.vsh -- test-bugcheck    # expects BUGCHECK 0x50000001
```

Each target rebuilds the kernel with a test define, boots it, and fails with a
non-zero exit code if the marker is missing.

---

## 7. What you will not get (yet)

* No desktop, no shell, no file manager.
* No third-party Windows `.exe` support. That is L3 and it is far away — see
  [Compatibility](Compatibility.md).
* No install to a physical disk. The installer environment (Spotlight PE) is 11%:
  its host-tested installer core, deterministic GPT/FAT32 image builder and
  file-backed apply path exist. Built-in images and ISOs carry the freestanding
  `\SPOTPE\INIT.EXE` diagnostic seed, but no boot stage launches it and there is
  no PE userland session or working physical-disk installer yet.

If a guide on the internet promises otherwise, it is not about this project.

---

## Next

* [Building](Building.md) — what each target does and why the flags look like that.
* [Running in QEMU](Running-in-QEMU.md) — firmware, logs, knobs.
* [Architecture](Architecture.md) — how the pieces fit.
