# Bingo documentation

How Betty's bingo is organised — the scheduling model, the backend contract, and how the
mini bingo widget renders it.

These documents are self-contained and cross-reference only each other, so this folder
can be lifted out into its own repository as-is.

## Start here

| Document | For | Covers |
|---|---|---|
| [**HOW_BINGO_WORKS.md**](./HOW_BINGO_WORKS.md) | anyone, no technical background | What rooms, sessions and rounds are, how they fit together, and a real evening start to finish |
| [**BACKEND_MODEL.md**](./BACKEND_MODEL.md) | engineers | The entities, the endpoints, which fields each returns, the round-selection rules and the real production intervals |
| [**MINI_BINGO_WIDGET.md**](./MINI_BINGO_WIDGET.md) | engineers, design, product | Every state of the mini bingo pill and its panel, the join prompt, and why each decision was made |

If you are new to bingo here, read them in that order — each assumes the one above it.

## Why these exist

Our mock bingo server was originally written from the client's own validation schemas,
which are permissive: almost every field is optional. That let the mock return data
production never sends, and client behaviour was then built on top of it — costing
several rounds of debugging before anyone checked the backend contract itself.

`BACKEND_MODEL.md` records that contract so it can be checked rather than guessed at.
`MINI_BINGO_WIDGET.md` records the design decisions that followed, including the ones we
got wrong first and why.

## A note on references

The documents cite source files from two repositories:

- **`Betty-Gaming/Bingo`** — the bingo backend (C#). Paths like
  `src/Bingo.Domain/Room.cs`. Note that mini bingo lives on the unmerged
  `BETTY-9470/Mini_Bingo` branch, not `main`.
- **the client monorepo** — paths like
  `features/bingo/lib/components/MiniBingo/…` and `packages/bingo-mock-server/`.

Claims are labelled by where they come from: read from source, observed in a production
response, or inferred. The inferences are collected at the end of
`BACKEND_MODEL.md` — worth confirming with the backend team before relying on them.
