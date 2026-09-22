# I-know-tt

A shared, continuously growing knowledge base for the **Tenstorrent (TT)** stack.

This repository is consumed as a git submodule by `tt-loop-scheduler`, mounted at
`thirdparty/I-know-tt`. It exists so that knowledge gained while working in a
consuming project is captured once, in one place, instead of being rediscovered.

## Where to put knowledge

Anything learned about Tenstorrent goes here, routed by topic:

| Folder            | Scope                                | Typical contents                                                                                              |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `isa/`            | TT **hardware** and ISA              | Tensix core architecture; RISC-V roles (BRISC/NCRISC/TRISC); NoC; circular buffers; `Dst` registers; tile layout; instruction semantics; memory maps |
| `tt-metal/`       | The **`tt-metal`** runtime & kernels | Kernel APIs; JIT/build; `cb_*` and `noc_*` semantics; profiler and Tracy; watcher; build flags; debugging      |
| `tt-npe/`         | **TT-NPE**                           | NPE usage, traces, performance modelling, what-if analysis                                                     |
| `unclassified/`   | Everything else                      | Tooling, environment setup, links, notes that don't fit above                                                  |

If a note genuinely spans topics, put it where it is most useful and cross-link
rather than duplicating it.

### `isa/` is not `tt-isa-documentation/`

- `tt-isa-documentation/` is the **upstream** Tenstorrent ISA documentation,
  vendored as a nested submodule. Treat it as a read-only reference.
- `isa/` is **ours**: distilled, task-driven notes written from what we actually
  ran into.

Prefer **linking** into `tt-isa-documentation/` over copying text out of it.
Upstream files carry their own licenses (see `LICENSE-APACHE` and
`LICENSE-CC BY-ND 4.0` inside that directory) — copying content here would
change its licensing, linking does not.

### `tt-metal/` here is not `thirdparty/tt-metal`

The `tt-metal` submodule at the consuming project's `thirdparty/tt-metal` is the
**source code**. The `tt-metal/` folder in this repository is **notes about it**.
Same name, different thing.

## Writing notes

- One topic per file, `kebab-case.md`.
- Lead with the concrete finding, then the evidence that supports it.
- **Record the version you observed it on.** Behaviour differs between tt-metal
  releases and between Wormhole and Blackhole. A note without a version is a
  liability.
- Distinguish **measured** from **assumed**. Say which is which explicitly.
- Keep it short. A note nobody reads is worth less than no note.

## Layout

```text
.
├── isa/                   # TT hardware / ISA notes
├── tt-metal/              # tt-metal runtime & kernel notes
├── tt-npe/                # TT-NPE notes
├── unclassified/          # everything else
└── tt-isa-documentation/  # upstream ISA docs (nested submodule, read-only)
```

The four knowledge folders carry a `.gitkeep` so git tracks them while empty.
Delete the `.gitkeep` once a folder has real content.

## Consuming this repository

In a fresh checkout of the consuming project:

```bash
git submodule update --init --recursive thirdparty/I-know-tt
```

`--recursive` is required: `tt-isa-documentation/` is a nested submodule and is
not populated by a non-recursive update.

### Adding knowledge from a consuming project

This repository is a submodule, so a file written into `thirdparty/I-know-tt/`
is **not** part of the consuming project's history. Committing it there does
nothing for anyone else. The full sequence is:

```bash
# 1. commit and push in the knowledge base itself
cd thirdparty/I-know-tt
git add -A && git commit -m "isa: document DstRowValid behaviour for MVMUL"
git push origin main

# 2. move the consuming project's pointer to the new commit
cd ../..
git add thirdparty/I-know-tt
git commit -m "Update I-know-tt submodule pointer"
```

Skipping step 2 leaves the consuming project pinned to an older commit, and the
new note silently will not appear for anyone else. Skipping step 1 means the
note exists only in your working copy.
