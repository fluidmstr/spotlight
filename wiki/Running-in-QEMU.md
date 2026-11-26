# Running in QEMU

> The only supported machine today is QEMU with OVMF. Real hardware comes after
> the storage stack exists (M10).

---

## One command

```powershell
v run mk/build.vsh -- run-efi
```

It builds `SPOTLDR.EFI` and `ntoskrnl.exe`, lays out an ESP directory, copies a
fresh OVMF variable store, starts QEMU headless, waits 20 seconds, kills the VM
and prints the serial log.

To run the harness directly:

```powershell
v run tools/vm/qemu.vsh -- run-efi --timeout 15
```

---

## What the harness does

1. Creates `build/esp/EFI/BOOT/BOOTX64.EFI` from `SPOTLDR.EFI`.
2. Creates `build/esp/EFI/SPOTLIGHT/ntoskrnl.exe` from the built kernel.
3. Copies the OVMF vars template to `build/vars.fd`.
4. Runs:

```
qemu-system-x86_64 -machine q35 -m 2048 -cpu qemu64 \
  -drive if=pflash,format=raw,unit=0,readonly=on,file=<OVMF_CODE> \
  -drive if=pflash,format=raw,unit=1,file=build/vars.fd \
  -drive format=raw,file=fat:rw:build/esp \
  -display none -monitor none \
  -serial file:build/serial.log \
  -debugcon file:build/debugcon.log \
  -no-reboot
```

5. Waits `--timeout` seconds, terminates QEMU, prints both logs.

No disk image is required: QEMU exposes a host directory as a FAT drive.

---

## Logs

| File | Contents |
|---|---|
| `build/serial.log` | loader banner and kernel `klog` lines (`I [boot] ...`) |
| `build/debugcon.log` | loader milestones (`EXIT`, `PGON`, `JUMP`) and `EXC <n>` |

The kernel's first observable line is:

```
I [boot] KiSystemStartup entered
```

If you see `EXIT / PGON / JUMP` in debugcon but nothing in serial, the loader
worked and the kernel died before or inside serial setup.

---

## Environment overrides

| Variable | Default | Meaning |
|---|---|---|
| `SPOT_QEMU` | `C:\Program Files\qemu\qemu-system-x86_64.exe` | QEMU binary |
| `SPOT_OVMF_CODE` | `C:\Program Files\qemu\share\edk2-x86_64-code.fd` | firmware code |
| `SPOT_OVMF_VARS` | `C:\Program Files\qemu\share\edk2-i386-vars.fd` | variable store template |

---

## Hardware assumptions

* x86-64, UEFI (no legacy BIOS), Secure Boot **off**.
* 2 GiB RAM is enough for now.
* AHCI is polled for disk I/O; NVMe is not used yet; the ESP is a FAT drive.

---

## Interactive control

There is none yet. The VM boots, prints, and halts. That is intentional: the
kernel has no shell, no scheduler idle loop with input, no drivers.

Stop QEMU early with `Ctrl+C` if you started it yourself; the harness kills it
for you.

---

## Next

* [Boot Testing](Boot-Testing.md) — make the boot assert itself.
* [Troubleshooting](Troubleshooting.md) — when serial is empty.
