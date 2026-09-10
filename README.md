# ndarray-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

N-dimensional arrays for novo-lang: a flat buffer, a shape, strides and
an offset, plus the operations a notebook actually performs on one —
construction, reshape, transpose, slicing along an axis, broadcast
arithmetic and comparison, reductions along an axis and over the whole
array, and matrix multiply.

It is the numeric floor of the novobook tier.  dataframe-nv's numeric
columns are these arrays; stats-nv's summaries, distributions and tests
take them as input.  The subset here is the one measured off notebook
corpora rather than the whole of numpy: `arange`, `linspace`, `eye`,
`reshape`, `transpose`, a slice, `+ - * /`, a comparison, `sum`, `mean`,
`min`, `max`, `argmin`, `argmax` along an axis, `dot` and `matmul`.  What
is deliberately absent is in "What is not here" below.

```
novo pkg add ndarray-nv
novo pkg build
novo test --isolate
```

Every call panics with `not implemented` until the implementation lands,
so `novo test --isolate` is what a first-time reader runs: each `@test`
gets its own process and prints the function it stopped at.

## The one example that will work

```novo
use ndfloat
use ndbool

fn main() [io]
    // Twenty measurements, and the ones above the mean.
    match ndfloat.linspace(0.0, 19.0, 20)
        Ok(xs) =>
            match ndfloat.mean(xs)
                Ok(m) =>
                    match ndfloat.gt(xs, ndfloat.scalar(m))
                        Ok(above) => println("${ndbool.count(above)} of ${ndfloat.size(xs)} are above ${m}")
                        Err(e)    => println(e.message())
                Err(e) => println(e.message())
        Err(e) => println(e.message())
```

## The layer, and why

`core`.  An array is a list of numbers, a shape is two lists of
integers, and every function here is arithmetic over what the caller
already holds.  Nothing is read, nothing is written, no clock and no
entropy is consulted, and no function takes a stream — so the sans-IO
question that shapes most `core` packages does not arise here at all.
The whole surface is `[]`, and there are no `host_modules`.

**No `@tier(embedded)` claim.**  A core package's device claim is built,
not asserted, so making one means shipping `tests/embedded_probe.nv` and
having the audit build it for a microcontroller.  This package does not
make it, and the audit's `core-embedded` row passes and says so.  The
reason is honest rather than procedural: every operation here allocates
a fresh buffer — `add` of two 1,000-element arrays is a 1,000-element
list — and a package whose cheapest call is an allocation has no
business in 64 KB of RAM.  A device that wants this arithmetic wants a
fixed-size, caller-supplied buffer and a different interface, and that
would be a different package rather than a flag on this one.

## The load-bearing interface

**Two concrete arrays, `NdFloat` and `NdInt`, and not one generic
`NdArray<T>`.**  This is the decision every other one here follows from,
and it was measured rather than assumed.

A generic container and generic functions over it compile and run
correctly inside a single module.  Across a module boundary — which is
all a library is — three things go wrong on the toolchain this
interface is written against:

| what was tried | what happened |
| --- | --- |
| `pub fn nd_make<T>(data: [T], dims: [Int]) -> NdBox<T>` called from another module | internal compiler error `E6000` — "cannot resolve this call's return type: no typing_info entry, no registry signature, no user-fn signature" |
| `pub fn nd_len<T>(a: NdBox<T>) -> Int` called from another module | the call's result is lowered as a pointer; LLVM verification fails on `store i64 %t10` of a `ptr` |
| the same call with the destination annotated `let n: Int = …` | the same failure — the annotation does not rescue it |

Under that, `impl` blocks carry no type parameters and an `impl` target
must be a bare identifier, so `impl<T> NdArray<T>` and `impl Numeric for
[T]` cannot be spelled either — which is why there is no numeric trait
here that a single generic array could have been written against.

So a generic array would be a type nobody outside this package could
call, and this package publishes two concrete ones that work.  The two
surfaces MIRROR each other member for member and name for name, which
is the deliberate part: the day a type parameter can leave a module,
`NdFloat` and `NdInt` collapse into `NdArray<T>` by DELETION rather than
by redesign, and a caller's `ndfloat.sum` becomes `ndarray.sum` with the
same arguments in the same order.  The four places they honestly differ
— integer division refusing a zero divisor where float division returns
an infinity, a truncating `rem`, no `mean`/`sqrt`/`exp`/`ln`/`linspace`
on the integers, and `take` living with the positions it gathers by —
are exactly the four a generic version would still have to special-case.

Both faults are reported to the toolchain, and neither is worked around
here: this package publishes the surface that works rather than a
design bent to fit a defect.

The three rules a reader has to know, each written down once in the
code and pointed at from everywhere else:

- **Broadcasting** — `ndshape.broadcast`.  Right-aligned, size-1
  stretches, everything else refused.  A rank-0 scalar agrees with every
  shape, which is why the arithmetic takes two arrays and never an array
  and a number, and why there is no `add_scalar` beside `add`.
- **Views versus copies** — `ndshape`'s module comment.  `transpose`,
  `swap_axes`, `slice_axis`, `index_axis`, `broadcast_to` and a packed
  `reshape` are views and do not copy.  In numpy that answer needs a
  warning about aliasing; here it needs none, because a novo-lang list
  is a value and nothing in this package writes in place.  A view is
  free and it is safe, and those two facts have the same cause.  `pack`
  is the deliberate copy.
- **What an error is** — `ndfault`.  One enum, nine variants, every one
  carrying the shapes it compared.

## ndarray-nv and `orbit/ml`

The must-have plan says "ml's tensors are the base", so this is the
paragraph that says exactly how the two relate.

**They do not share a type today, and neither is a subset of the
other.**  `orbit/ml`'s numeric core is deliberately not an N-D tensor at
all: it is `WMat` — a read-only, row-major, packed-**f32** weight matrix
over `F32Buf`, always rank 2 — plus a bare `[Float]` f64 activation
vector, and its own module comment says the narrowness is the point
("deliberately only the two storage shapes a batch-1 LLM decoder
actually uses").  `NdFloat` is f64 at any rank, immutable, strided and
viewable.  So the three things that would have to agree do not:

| | `orbit/ml` | ndarray-nv |
| --- | --- | --- |
| element type | f32 in `WMat`, f64 in activations | f64 |
| rank | exactly 2 (`WMat`) or exactly 1 (`[Float]`) | any |
| layout | packed row-major, no strides, no offset | strides and an offset, so views are free |
| mutability | a weight is read-only after load; `F32Buf.release` frees it by hand | a value; nothing is written in place and nothing is released |

**Would ml depend on ndarray-nv?**  Not as it stands, and not for
`WMat`.  ml's f32 buffer exists so that a 7-billion-parameter model is
half the RSS it would be in f64 and so that a safetensors F32 payload
loads by `memcpy` with no conversion pass; widening it to `NdFloat`
would double the memory of the thing the package exists to run.  The
half that COULD move is the activation side — `ml/cpu.nv`'s `dot`,
`ewadd`, `ewmul`, `argmax` and `softmax` over `[Float]` are, function for
function, this package's rank-1 operations, and `ml` could take them
from here and delete its own.  That is a real, small, testable step and
it is what "ml's tensors are the base" cashes out to.

**What would have to change on either side for more than that.**  Three
things, in the order they bind:

1. **An f32 element type.**  ndarray-nv would need an `NdFloat32` over a
   packed f32 buffer, which is a third concrete array — or, once a type
   parameter can cross a module boundary, the third instantiation of one
   generic array.  This is the load-bearing decision above wearing a
   different hat: the generic-container limit is the same thing blocking
   both.
2. **A borrowed buffer.**  `WMat` is built over an mmap'd region through
   `F32Buf`; `NdFloat` owns a `[Float]`.  Until ndarray-nv can be
   constructed over a buffer it did not allocate, a loader cannot hand
   it a model file without copying it.
3. **`orbit/ml`'s layer.**  ml is an `app`, and a `core` package may not
   depend on one — the direction is only ever ml depending on
   ndarray-nv, never the reverse.  ml's `L0 core` would have to become
   its own `core` package first, which is the `gguf-nv` / `tokenizers-nv`
   split the plan already lists for ml.

Until then the honest statement, and the one this package makes: **ml's
`WMat` is the shape ndarray-nv is designed to be able to become, and the
activation half is the part that can converge first.**

## What is not here

Named so a reader stops looking: no complex numbers, no f32 arrays, no
sorting (`ndint.take` is the gather a sort's output feeds), no
`einsum`, no decompositions (linalg-nv), no FFT (fft-nv), no random
generation (a `core` package has no entropy — see stats-nv's README for
the shape that takes draws as an argument), no in-place mutation and no
iterator protocol over elements.

## The reference implementation

**numpy** (BSD-3-Clause) for the semantics a notebook expects, and
**ndarray** (MIT/Apache-2.0) for the shape-and-strides design that makes
views free.  The behaviours borrowed verbatim, so a reader can check the
port rather than trust it: the right-aligned broadcasting rule; `arange`
half-open and `linspace` closed with the last element exactly the stop;
`argmin`/`argmax` answering with the FIRST position on a tie; `matmul`
promoting a rank-1 operand and dropping the promoted axis again;
`transpose` of a rank-1 array being that array.  Where this package
departs, it says so in the doc comment: `min` and `max` skip NaN (numpy's
`nanmin`, not its `min`), a negative axis number is refused rather than
counted from the end, and a reduction along an axis DROPS that axis
rather than keeping it as 1.

numpy's own test suite is the oracle the implementation lane will run
the ported subset against.

## Status

Every function is `todo()`.  `novo test --isolate` is the readable form
of that verdict: each `@test` runs in its own process and prints the
function it stopped at.

| module | public functions | implemented |
| --- | --- | --- |
| `ndshape` | 11 | no |
| `ndfault` | the `NdFault` enum and its `Error` impl | no |
| `ndbool` | 17 | no |
| `ndfloat` | 55 | no |
| `ndint` | 46 | no |
