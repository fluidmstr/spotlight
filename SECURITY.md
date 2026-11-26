# Security policy

Spotlight is an operating system kernel project. Security reports are taken
seriously even at this stage.

## Reporting

Do **not** open a public issue for a vulnerability. Use GitHub's private
vulnerability reporting for this repository, or contact the maintainers directly.

Include:

- affected repository and commit,
- a description of the impact,
- reproduction steps (QEMU configuration, image, commands),
- logs (`build/serial.log`, `build/debugcon.log`) if relevant.

## Scope

In scope today:

- loader: PE parsing, memory allocation, page-table construction, handoff;
- kernel: descriptor tables, exception handling, KPCR, serial/bugcheck paths;
- build system and tooling that produces artifacts.

Out of scope today (not implemented): networking, userland, drivers, installer.

## Expectations

- We aim to acknowledge reports within 7 days.
- Please give us time to fix before public disclosure.
- Reports about third-party components (V, LLVM, QEMU, Wine) should go to their
  upstreams.
