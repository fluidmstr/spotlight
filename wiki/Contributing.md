# Contributing

> One rule above all: **compatibility is a contract, not a resemblance.**
> Anything that cannot be explained and tested is not merged.

---

## Read first

| Document | Why |
|---|---|
| [`spotlight-nt/AGENTS.md`](https://github.com/fluidmstr/spotlight-nt/blob/main/AGENTS.md) | hard rules, review checklist, agent roles |
| [`spotlight-nt/docs/04-v-kernel-style.md`](https://github.com/fluidmstr/spotlight-nt/blob/main/docs/04-v-kernel-style.md) | how to write kernel code in V |
| `spotlight-nt/docs/05-abi/` | the contracts you must not guess |
| `spotlight-x/AGENTS.md` | userland and Wine policy |

---

## Workflow

1. Pick a task with a measurable exit criterion.
2. Write the spec or update the existing one.
3. Add the test that fails without your change.
4. Keep the PR small (≤ 600 meaningful lines), one topic.
5. Update documentation in the same PR.

### Commit style

```
feat(boot): exit boot services and enter kernel on own page tables
fix(ke): keep Loader_block pointing at the latest memory map
docs(abi): specify and test the Loader_block handoff
test(boot): add expectation harness and boot regression
```

Types: `feat`, `fix`, `refactor`, `perf`, `test`, `docs`, `build`, `chore`, `abi`, `ci`.
Branches: `m04/mm-page-fault`, `docs/abi-syscall`.

---

## What gets rejected

| Change | Why |
|---|---|
| a stub that returns success | it hides missing functionality and poisons debugging |
| a hard-coded offset without a spec and a test | it silently breaks compatibility |
| copied code from a decompiler or leaked source | legal contamination |
| a giant "implement the kernel" PR | unreviewable |
| disabling bounds/pointer checks "to make it work" | security regression |
| Microsoft binaries in the repository | cannot be redistributed |

---

## Good first tasks

| Area | Task |
|---|---|
| Tooling | add a fixture to `tools/ssdt-gen` or `tools/symgen` |
| Docs | turn a `TODO(verify)` in `docs/05-abi/` into a verified row with a source |
| Boot | add an expectation to `tests/boot/m2_serial.lines` for a new kernel log |
| Kernel | implement `vrt.fmt` consumers, a `ktest` suite, or an IRQ self-test |
| Corpus | write a checklist for one application in `tests/apps` |

---

## ABI discipline

1. The source of truth is `docs/05-abi/`. Not memory, not a blog post.
2. Every structure visible to asm, a driver, or userland needs: a spec, offset
   asserts, and a status (`verified` / `assumed` / `todo`).
3. `@[packed]`, explicit padding, no V `string`/arrays/maps in ABI structs.
4. Changing an ABI requires an ADR and a migration note.

---

## Licensing of contributions

By submitting a PR you confirm the code is written from scratch and is not a
derivative work of proprietary sources (DCO). New dependencies need a licence
check and a row in `third_party/README.md`.

---

## Communication

* Issues: bug reports with logs; feature requests with an exit criterion.
* Discussions: architecture and roadmap.
* Security: private report, never a public issue.

---

## Next

* [Legal](Legal.md)
* [Architecture](Architecture.md)
