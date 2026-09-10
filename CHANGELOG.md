# Changelog

All notable changes to ndarray-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-10

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `ndshape` — dims, strides and an offset, and the two rules a reader
  has to know: the right-aligned broadcasting rule, written once, and
  the answer to "does a transpose copy?", which is no. A view is free
  AND safe here, because a novo-lang list is a value and nothing in this
  package writes in place.
- `ndfault` — one error enum for the whole package, nine variants, each
  carrying the shapes it compared.
- `ndbool` — `NdMask`, the shaped array a comparison produces, so that
  `a > 0` on a matrix is a matrix-shaped answer rather than a flat list
  of truths.
- `ndfloat` — the array of Float: construction, the views, broadcast
  arithmetic and comparison, the whole-array and per-axis reductions,
  `select` and `where`, `dot` and `matmul`.
- `ndint` — the array of Int, mirroring `ndfloat` member for member,
  plus `take` (the gather a group-by and a join are built out of) and
  minus everything that would answer with a non-integer.

### Known

- **Two concrete arrays rather than one `NdArray<T>`, and it was
  measured.** A generic function over a generic struct is not emitted
  for a call from another module — three faces of the same defect,
  filed under `bugs/codegen-llvm/` — so a generic array would be
  unreachable from outside this package. `NdFloat` and `NdInt` mirror
  each other deliberately: when that closes, they collapse into one
  type by deletion.
- **`xs[i] ?? d` on a list of Float or Int does not build**, filed under
  `bugs/codegen-llvm/`. Nothing here reaches it — every body is a
  `todo()` — and the implementation lane uses `list.get(xs, i) ?? d`,
  which is the more honest spelling anyway.
- No `@tier(embedded)` claim, and none is intended: every operation
  allocates a fresh buffer, and a package whose cheapest call is an
  allocation has no business in 64 KB of RAM.
