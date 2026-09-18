# Changelog

All notable changes to ndarray-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-09-18

First implementation of the interface published as 0.0.1. Every `pub
fn` has a body, no `todo()` remains under `src/`, and every signature
is the one that was published — nothing moved, and nothing was added.
`stability = "experimental"`.

### Added

- Bodies for all 129 published functions across `ndshape`, `ndfault`,
  `ndbool`, `ndfloat` and `ndint`.
- `tests/semantics_tests.nv` — 19 tests whose expected answers are
  NumPy's, taken from NumPy's documentation where NumPy prints them:
  the broadcasting table, `arange` and `linspace`, the first position
  on an `argmax` tie, NaN through `sum` and `mean` and around `min` and
  `max`, and `matmul`'s promote-then-drop. The five departures from
  NumPy each have a case that asserts the departure.
- `tests/coverage_tests.nv` — 10 tests that reach 850 of 850 lines and
  every function under `src/`, with no `// cov: skip` marker anywhere
  in the package.
- `tests/property_tests.nv` — 12000 checks of twenty-odd laws over 200
  shapes generated from the case number. It is also a program: `novo
  run` and `novo run --interp` print the same digest line.
- `tests/bench_matmul.nv` — a program, not a test. On the machine this
  release was built on, a 64-by-64 `ndfloat.matmul` takes 0.59 ms and
  an `ndint.matmul` 0.32 ms, which is 2.26 and 1.23 ns per
  multiply-add. It is here to be a baseline, not a target.

### Changed

- `to_list`, and therefore every operation that reads an array, hands
  the buffer straight back when the array is packed and starts at
  offset 0, instead of walking the strides position by position. The
  answer is the same — `ndshape.is_packed` documents that a packed
  array's buffer IS its elements in row-major order — and a 64-by-64
  matrix multiply is 2.3 times faster for it.
- Every loop in the package is written with the stdlib's own form:
  `for i in 0..n` for a counter, `list.map` for an accumulator that
  only pushes, and `list.flat_map` for the walk that expands one axis
  at a time. No behaviour changed.

### Decided

Two questions the interface left open, answered here. Neither changes a
signature, and both are visible to a caller, so both are written down.

- **`argmin` and `argmax` skip NaN**, the way `min` and `max` already
  documented that they do. The consequence is that `at(a, argmin(a))`
  is `min(a)` for every array. NumPy's `argmin` propagates the NaN
  instead, so this is a sixth departure from NumPy and it is in the
  README's list.
- **An array that is nothing but NaN** answers a NaN from `min` and
  `max`, and position zero from `argmin` and `argmax`. It is not an
  empty array, so there is nothing for `NdEmptyReduction` to report.

### Known

- **One published test assertion is red on novo-lang 0.9.1, and the
  package is not the cause.** In `tests/ndfloat_tests.nv`, the case "gt
  is the one a notebook writes" compares a list of truths against a
  list literal. On this toolchain `[Bool] == [Bool]` answers false for
  equal lists whenever a list of `Float` was bound earlier in the same
  function, on the compiled and the interpreted backend alike. The mask
  is correct and printing it shows the right values; the comparison is
  what is wrong. Filed against the toolchain as
  `cross-backend/bool-list-equality-answers-false-after-a-float-list-in-the-same-function`.
  The suite passes under `novo test tests/ndfloat_tests.nv --isolate`,
  which gives every test its own process. No workaround was written
  into this package.
- **`ndbool` has no `broadcast_to`**, so a mask cannot be stretched to
  a shape by a caller. `both`, `either` and `ndfloat.where` stretch it
  internally. This was true of the interface and is unchanged.
- **The two concrete arrays are still two.** A generic function over a
  generic struct is still not emitted across a module boundary, so
  `NdFloat` and `NdInt` continue to mirror each other member for
  member, and the private machinery in each module mirrors the other's
  as well. They collapse into one type by deletion when that closes.
- No `@tier(embedded)` claim, and none is intended.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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
