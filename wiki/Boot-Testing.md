# Boot Testing

> A boot is not "it worked on my machine". It is a set of lines that must appear,
> enforced by a command that fails when they do not.

---

## The commands

```powershell
v run mk/build.vsh -- test          # host unit tests + ABI tests
v run mk/build.vsh -- test-boot     # serial expectations
v run mk/build.vsh -- test-lapic    # LAPIC timer takeover
v run mk/build.vsh -- test-timer    # 10 sustained timer interrupts
v run mk/build.vsh -- test-exception
v run mk/build.vsh -- test-bugcheck
v run mk/build.vsh -- test-file      # rename and delete-on-close
v run mk/build.vsh -- test-event     # named event create/open/duplicate
v run mk/build.vsh -- test-sync      # single- and multi-object waits
v run mk/build.vsh -- test-exit      # thread/process exit
v run mk/build.vsh -- test-registry  # in-memory registry keys and values
```

| Command | Asserts |
|---|---|
| `test` | PE parser, page tables, ssdt-gen, symgen, Loader_block offsets |
| `test-boot` | loader and kernel milestones on COM1 |
| `test-lapic` | LAPIC timer takes over, PIC/PIT masked |
| `test-timer` | 10 sustained PIT interrupts (`pit_timer-ok`) |
| `test-exception` | a deliberate `int3` reaches the IDT dispatcher (`EXC 3`) |
| `test-bugcheck` | `ke.bugcheck()` prints `BUGCHECK 0x50000001` and halts |
| `test-file` | FAT32/VFS rename (`FileRenameInformation`) and delete-on-close (`FileDispositionInformation`) |
| `test-event` | named event create, open and duplicate |
| `test-sync` | finite timed single-object and multi-object waits |
| `test-exit` | thread and process termination paths |
| `test-registry` | in-memory keys and values under `HKLM\Software\Microsoft\Windows NT\CurrentVersion` |

Exit code is non-zero on failure, so all of them are CI-ready.

---

## How expectations work

`tests/boot/m2_serial.lines` is a plain list. One substring per line, `#` starts
a comment:

```
Spotlight loader 26.4173.0.1
loader: image at
I [boot] KiSystemStartup entered
I [boot] loader block accepted
I [boot] image_base=0x
I [ke] GDT, TSS and IDT loaded
I [ke] KPCR installed at GS base
I [ke] irql=0x0
I [ke] irql_raised=0x2
I [acpi] lapic_base=0xfee00000
I [acpi] ioapic_base=0xfec00000
I [hal] lapic_id=0x
I [hal] ioapic_version=0x
I [hal] PIC and PIT ready
I [hal] pit_ticks=0x
I [hal] PIC and PIT masked
I [hal] lapic_timer-ok
I [hal] lapic_timer_ticks=0x
I [hal] lapic_bus_hz=0x
I [hal] ioapic_gsi0=0x2
I [hal] ioapic_irq-ok
I [hal] ioapic_irq_ticks=0x
I [pci] devices=0x
I [pci] first=0x
I [mm] free_pages=0x
I [mm] paging-on
I [mm] cr3=0x
I [mm] alloc-free-ok
I [mm] pool-ok
I [mm] vad-ok
I [serial] loopback self-test passed
```

The harness strips ANSI escapes from `build/serial.log` and checks that each
substring is present:

```powershell
v run tools/vm/qemu.vsh -- run-efi --expect tests/boot/m2_serial.lines
```

Output on success:

```
boot test passed: 33 expectations matched
```

On failure it prints every missing line and exits 1.

---

## Adding a regression

1. Add the new log line to the kernel (via `vrt.info`, never `println`).
2. Append the expected substrings to the `.lines` file.
3. Run `v run mk/build.vsh -- test-boot`.
4. Commit both changes together.

A test that passes even when the feature is removed is not a test. If a line
cannot be observed by design, do not add it — add an in-kernel self-test instead
(planned `test-kernel` target).

---

## Test-only builds

Kernel tests that must crash the machine are compiled behind defines:

```powershell
v run mk/build.vsh -- test-exception    # -d spot_test_exception
v run mk/build.vsh -- test-bugcheck     # -d spot_test_bugcheck
```

The normal build never contains those paths: V's `$if flag ? { ... }` removes
them at compile time.

---

## What is still missing

* In-kernel test runner (`test-kernel`): suites loaded through the loader block.
* Fault injection: fail N-th page fault / IRP / allocation.
* Multi-CPU boot tests (SMP arrives with M3).
* Real hardware matrix (M3+).

---

## Next

* [Running in QEMU](Running-in-QEMU.md)
* [Troubleshooting](Troubleshooting.md)
