# ndarray-nv

An N-dimensional array is a block of numbers of one type, addressed by a
list of coordinates. A one-dimensional array is a vector, a
two-dimensional one is a matrix, and the number of coordinates is called
the array's rank. This package brings that data structure, and the
operations a notebook performs on one, to novo-lang. Its reference is
[NumPy](https://numpy.org/doc/stable/reference/index.html) for what each
operation means, and the Rust crate
[ndarray](https://docs.rs/ndarray) for the layout that makes a transpose
free.

Eight other packages on the registry are defined over these arrays:
[linalg-nv](https://novo-lang.org/packages/linalg-nv) (matrix
decompositions), [stats-nv](https://novo-lang.org/packages/stats-nv)
(summaries, distributions and hypothesis tests),
[fft-nv](https://novo-lang.org/packages/fft-nv) (two-dimensional Fourier
transforms), [cluster-nv](https://novo-lang.org/packages/cluster-nv)
(k-means, DBSCAN and agglomerative clustering),
[embeddings-nv](https://novo-lang.org/packages/embeddings-nv) (vector
arithmetic over a token-embedding matrix),
[interpolate-nv](https://novo-lang.org/packages/interpolate-nv)
(interpolation over a regular grid),
[plot-nv](https://novo-lang.org/packages/plot-nv) (charts), and
[dataframe-nv](https://novo-lang.org/packages/dataframe-nv), whose
numeric columns convert to and from them.

**Status: implemented, and experimental.** Version 0.1.0 is the first
working release: every function published as an interface in 0.0.1 and
0.0.2 now has a body, and every signature is the one that was
published. The package is marked experimental because the surface may
still change between 0.1.x releases. A change that breaks a caller will
be named under a "Breaking" heading in the CHANGELOG.

Everything the interface declared is implemented. What is not here is
listed under "What is not included", and none of it is a stub: a
function that is absent is absent, not present and broken.

## What an N-dimensional array is here

An array is three things: a flat buffer of numbers, a **shape**, and an
**offset**. The shape is two lists of integers. **Dims** gives the extent
along each axis, outermost first, so a 3-by-4 matrix has dims `[3, 4]`.
**Strides** gives how far to step in the flat buffer to move one place
along that axis. The offset says where in the buffer the array's first
element sits. Element `[i, j, k]` of an array with offset `o` lives at
`o + i*strides[0] + j*strides[1] + k*strides[2]`, and that line is the
whole layout.

A **view** is a new shape over a buffer that already exists. Because a
view copies nothing, transposing a million-element matrix costs a list of
two integers reversed. NumPy's views alias memory that another holder can
write through, so a NumPy user has to know which operations view and
which copy. That hazard does not exist here. A novo-lang list is a value,
and no function in this package writes into an array in place, so there
is no writer for a view to expose.

**Broadcasting** is the rule that lets two arrays of different shapes
take part in one arithmetic operation. The shapes are lined up from the
right. An axis of extent 1 is stretched to meet the other array's extent.
Any other disagreement is refused. NumPy's
[broadcasting rules](https://numpy.org/doc/stable/user/basics.broadcasting.html)
are the rule this package follows.

A **rank-0 array** is an array with no axes at all: a single number with
an empty dims list. It broadcasts against every shape. That is why the
arithmetic here always takes two arrays, and why there is no
`add_scalar` beside `add`.

A **mask** is an array of truths with a shape, which is what a comparison
answers. Comparing a 3-by-4 matrix against a number gives a 3-by-4 mask,
not a flat list of twelve truths.

The package publishes two array types, `NdFloat` over `Float` and `NdInt`
over `Int`, rather than one array with a type parameter. See "How to
choose an entry point" for which to reach for, and "What is not
included" for why there is only one element type each.

## Install

```
novo pkg add ndarray-nv
```

## Example

```novo
use ndfloat
use ndbool

fn main() [io]
    // Six values laid out as a 2-by-3 matrix, one row after the other.
    match ndfloat.of_list([1.0, 2.0, 3.0, 4.0, 5.0, 6.0], [2, 3])
        Err(e) => println(e.message())
        Ok(a) =>
            // A transpose is a new shape over the same buffer, so nothing is copied.
            let t = ndfloat.transpose(a)
            println("the 3-by-2 transpose sums to ${ndfloat.sum(t)}")

            // `scalar` is a rank-0 array, and a rank-0 array broadcasts
            // against every shape, so this adds 10 to every element.
            match ndfloat.add(t, ndfloat.scalar(10.0))
                Err(e) => println(e.message())
                Ok(b)  => println("with 10 added to each, ${ndfloat.sum(b)}")

            // A comparison answers a mask of the same shape, not a flat list.
            match ndfloat.gt(a, ndfloat.scalar(3.0))
                Err(e) => println(e.message())
                Ok(m)  => println("${ndbool.count(m)} elements are above 3.0")
```

Build and test with `novo pkg build` and `novo test tests/`.

## What the package contains

| Module | Contents |
| --- | --- |
| `ndshape` | The layout: dims, strides, the element count, the flat address of one coordinate, the broadcasting rule, and the axis operations a view is built from. |
| `ndfault` | The one error type for the whole package. Nine variants, each carrying the shapes or the numbers it compared. |
| `ndbool` | `NdMask`, the shaped array of truths a comparison answers, and what a mask can be asked on its own: all, any, count, count along an axis, and the three logical combinations. |
| `ndfloat` | The array of `Float`: construction, the views, broadcast arithmetic and comparison, the element-wise functions, reductions over the whole array and along one axis, selection, `dot` and `matmul`. |
| `ndint` | The array of `Int`, with the same members under the same names, plus `take`, which gathers positions along an axis. |

## How to choose an entry point

**Use `ndfloat` for measurements and `ndint` for counts, positions and
labels.** The two surfaces carry the same members under the same names,
so code written against one reads the same against the other.

They differ in four places, and each difference is a property of the
element type.

| Difference | `ndfloat` | `ndint` |
| --- | --- | --- |
| Division by zero | answers an infinity | refused as an error |
| Remainder | not present | `rem`, truncating |
| `mean`, `sqrt`, `exp`, `ln`, `linspace` | present | absent, because each answers a number that is not an integer |
| `take`, the gather along an axis | not present | present |

`ndint.to_float` and `ndfloat.to_int` convert between the two.
`ndbool.to_int` and `ndbool.of_int` convert a mask to and from an integer
array of zeroes and ones.

**Most programs never name `ndshape` directly.** It is there for a
program that computes a layout before it has an array, and for reading
the dims and strides off one it has.

## The rules a user needs

1. **Broadcasting lines the shapes up from the right.** An axis of
   extent 1 stretches to meet the other extent. An axis missing from the
   shorter shape counts as extent 1. Any other disagreement is
   `NdNotBroadcastable`, which carries both shapes. The rule is written
   once, in `ndshape.broadcast`.
2. **Arithmetic takes two arrays, never an array and a number.** Wrap the
   number with `ndfloat.scalar` or `ndint.scalar`. A rank-0 array
   broadcasts against every shape, so that call is the scalar case of
   the ordinary operation.
3. **`transpose`, `swap_axes`, `slice_axis`, `index_axis`,
   `broadcast_to` and a `reshape` of a packed array are views.** They
   copy nothing and cost a new shape. Everything else materialises a new
   buffer.
4. **A view keeps its whole buffer alive.** Slicing three elements out of
   a large array and keeping only the slice keeps the large array too.
   `pack` is the escape: it copies a view into a fresh, contiguous
   buffer of exactly the right size. It is the only function here whose
   purpose is to copy.
5. **Nothing is written in place.** `with` answers a new array with one
   element changed. There is no `set`.
6. **Axes are numbered from 0 and count from the outermost.** The legal
   range is 0 up to the rank, not including it. A negative axis number is
   refused as `NdAxisOutOfRange` rather than counted from the end, which
   is where this package departs from NumPy.
7. **A reduction along an axis drops that axis.** Summing a 3-by-4 matrix
   along axis 0 answers a rank-1 array of four elements, not a 1-by-4
   matrix. This too departs from NumPy, which can keep the axis at
   extent 1.
8. **`min` and `max` skip NaN.** They behave as NumPy's `nanmin` and
   `nanmax`, not as its `min` and `max`.
9. **`argmin` and `argmax` answer the first position on a tie.** This
   follows NumPy.
10. **`arange` excludes its stop value and `linspace` includes it.** A
    `linspace`'s last element is exactly the stop value. This follows
    NumPy.
11. **`matmul` promotes a rank-1 operand to a matrix and drops the
    promoted axis from the result.** This follows NumPy. `dot` answers a
    single number and is for two rank-1 arrays.
12. **A comparison answers an `NdMask`, not a list of truths.** Feed the
    mask to `select`, which keeps the elements it marks, or to `where`,
    which chooses element by element between two arrays.
13. **Every failure is one type, `NdFault`, and every variant carries the
    numbers.** A message prints both shapes it compared, so a reader does
    not have to guess which of four arrays in an expression was wrong.

## What is not included

- **Complex numbers.** A complex array is a second element type with its
  own arithmetic. [fft-nv](https://novo-lang.org/packages/fft-nv) carries
  complex data as an interleaved buffer of real and imaginary parts
  instead.
- **32-bit float arrays.** The buffer here is `Float`, which is 64-bit.
- **One generic array type over an element type parameter.** This
  release publishes two concrete arrays instead. `NdFloat` and `NdInt`
  carry the same members, in the same order, under the same names, so a
  later release can merge them into one type without moving any
  argument. The four places they differ are in "How to choose an entry
  point".
- **Sorting.** `ndint.take` is the gather that a sort's output feeds.
- **`einsum`**, the index-notation contraction.
- **Matrix decompositions.** LU, QR, Cholesky, the eigendecomposition and
  the singular value decomposition are in
  [linalg-nv](https://novo-lang.org/packages/linalg-nv).
- **The Fourier transform.** It is in
  [fft-nv](https://novo-lang.org/packages/fft-nv).
- **Random number generation.** No function in this package draws
  entropy. [stats-nv](https://novo-lang.org/packages/stats-nv) takes
  random draws as an argument instead.
- **In-place mutation, and an iterator over elements.** See rule 5.
- **A microcontroller build.** Every operation here allocates a fresh buffer.
  Adding two arrays of a thousand elements allocates a thousand-element list.
  A device with no heap allocator needs a fixed-size buffer the caller
  supplies, which is a different interface rather than a flag on this one.
  Nothing here is claimed to build for a device with no heap allocator, and
  there is no `tests/embedded_probe.nv`.

## Related packages

- [linalg-nv](https://novo-lang.org/packages/linalg-nv) factors these
  matrices and solves systems over them. Take it when you need a
  determinant, an inverse, a least-squares fit or a singular value
  decomposition.
- [stats-nv](https://novo-lang.org/packages/stats-nv) reads these arrays
  as samples. Take it for a mean with a variance beside it, a
  distribution, or a hypothesis test.
- [fft-nv](https://novo-lang.org/packages/fft-nv) transforms them. Its
  one-dimensional transforms take a plain interleaved list; its
  two-dimensional transforms take these arrays.
- [dataframe-nv](https://novo-lang.org/packages/dataframe-nv) adds a
  name, a null mask and non-numeric element types on top. A numeric
  column converts to an array of this package and back.
- [interpolate-nv](https://novo-lang.org/packages/interpolate-nv) uses
  these arrays for the regular two-dimensional grid its surface
  interpolation reads.
- `std.array` in the standard library has an `ArrayF64` over a foreign
  allocation. Every one of its methods declares the `[io]` effect,
  because reading one element is a call out of novo-lang. This package
  is a flat novo-lang list instead, and every function here declares no
  effect at all.

## Tests

```bash
novo test tests/ndshape_tests.nv      # 10 tests: the layout and the broadcasting rule
novo test tests/ndbool_tests.nv       #  7 tests: the mask
novo test tests/ndfloat_tests.nv      # 23 tests: the array of Float
novo test tests/ndint_tests.nv        # 11 tests: the array of Int
novo test tests/semantics_tests.nv    # 19 tests: the numbers, against NumPy
novo test tests/coverage_tests.nv     # 10 tests: every line of src/
novo test tests/property_tests.nv     #  1 test:  the laws, over 200 shapes
```

The first four suites were published with the interface and are
unchanged. They say what the shape of each answer is: that a transpose
does not copy, that a broadcast refusal names both shapes, that a
reduction drops its axis, that an empty reduction is refused rather
than answering zero, that a negative axis is refused, and that the four
documented differences between `ndfloat` and `ndint` hold.

`semantics_tests.nv` says what the numbers are. Its expected answers
are NumPy's, taken from NumPy's own documentation where NumPy prints
them: the broadcasting table from the broadcasting page, `arange` and
`linspace` as the reference manual shows them, the first position on an
`argmax` tie, NaN propagating through `sum` and `mean` and being skipped
by `min` and `max`, and `matmul` promoting a rank-1 operand and then
dropping the promoted axis. The five places this package departs from
NumPy each have a case that asserts the departure.

`coverage_tests.nv` reaches every line of `src/`. Measured with
`novo test tests/coverage_tests.nv --cov`, it executes 1030 of 1030
source lines and reaches all 184 functions. No line is excused with a
`// cov: skip` marker.

`property_tests.nv` checks laws rather than vectors: that packing an
array does not change what it reads, that transposing twice is the
array back, that `at` agrees with `to_list` at every position, that a
reduction along each axis reduced again is the reduction over the whole
array, that gathering every position of an axis in order is the array,
and that the transpose of a matrix product is the product of the
transposes in the other order. It runs 12000 such checks over 200
shapes built from the case number rather than from a random source, so
every run is the same run. It is also a program, and

```bash
novo run tests/property_tests.nv
novo run tests/property_tests.nv --interp
```

print the same line from the compiled and the interpreted backend.

`tests/bench_matmul.nv` is a program, not a test. It times twenty
64-by-64 matrix multiplies and prints the per-multiply figure, so a
later release has something to compare against.

**One published assertion is red on this toolchain, and the package is
not the reason.** In `tests/ndfloat_tests.nv`, the case "gt is the one
a notebook writes" compares a list of truths against a list literal.
On novo-lang 0.9.1 a `[Bool] == [Bool]` comparison answers false for
equal lists whenever a list of `Float` was bound earlier in the same
function, on both the compiled and the interpreted backend. The mask
the test builds is correct and printing it shows the right values; the
comparison is what is wrong. The defect is filed against the toolchain.
Running that suite with `novo test tests/ndfloat_tests.nv --isolate`
gives every test its own process and all 23 pass.

## Implementation status

Every declaration below is implemented.

| Item | Implemented |
| --- | --- |
| `ndshape.NdShape`, `ndbool.NdMask`, `ndfloat.NdFloat`, `ndint.NdInt`, `ndfault.NdFault` | yes |
| `ndshape.of_dims`, `.scalar`, `.rank`, `.size`, `.is_packed`, `.fits` | yes |
| `ndshape.flat_index`, `.broadcast`, `.swap_axes`, `.reverse_axes`, `.drop_axis` | yes |
| `ndfault`'s nine variants and its `Error` implementation | yes |
| `ndbool.of_list`, `.filled`, `.shape`, `.size`, `.at`, `.to_list` | yes |
| `ndbool.all`, `.any`, `.count`, `.count_axis`, `.both`, `.either`, `.invert` | yes |
| `ndbool.swap_axes`, `.pack`, `.to_int`, `.of_int` | yes |
| `ndfloat.of_list`, `.scalar`, `.zeros`, `.ones`, `.filled`, `.arange`, `.linspace`, `.eye` | yes |
| `ndfloat.shape`, `.rank`, `.size`, `.at`, `.with`, `.to_list` | yes |
| `ndfloat.reshape`, `.transpose`, `.swap_axes`, `.slice_axis`, `.index_axis`, `.broadcast_to`, `.pack` | yes |
| `ndfloat.add`, `.sub`, `.mul`, `.div`, `.pow`, `.neg`, `.abs`, `.sqrt`, `.exp`, `.ln`, `.map` | yes |
| `ndfloat.eq`, `.ne`, `.lt`, `.le`, `.gt`, `.ge`, `.select`, `.where` | yes |
| `ndfloat.sum`, `.mean`, `.min`, `.max`, `.argmin`, `.argmax` | yes |
| `ndfloat.sum_axis`, `.mean_axis`, `.min_axis`, `.max_axis`, `.argmin_axis`, `.argmax_axis` | yes |
| `ndfloat.dot`, `.matmul`, `.to_int` | yes |
| `ndint`'s forty-six functions, mirroring `ndfloat` and adding `.rem`, `.take`, `.to_float` | yes |

Two answers this release fixes that the interface left open, both
stated here because a caller can see the difference:

| Question the interface did not answer | This release |
| --- | --- |
| What do `argmin` and `argmax` do with a NaN? | They skip it, the way `min` and `max` do, so `at(a, argmin(a))` is always `min(a)`. NumPy's `argmin` propagates the NaN instead. |
| What does `min` answer for an array that is nothing but NaN? | A NaN, and `argmin` answers position zero. The array is not empty, so there is nothing to refuse. |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
