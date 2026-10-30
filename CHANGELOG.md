# Changes from TISEAN 3.0.1

Changes are listed by program.
**Output** says whether the program's results differ from 3.0.1.
Each entry corresponds to a commit after the tag `upstream-3.0.1`, whose message gives the details; those that came from hctsa carry an `hctsa-commit:` trailer.

## Unreleased

### c1 (and its subroutine d1.f)

- **Fixed a hang** when the number of embedded points is a power of two for some requested embedding dimension (e.g. 520 points, `-d2`, `-M5`): the mass loop could reach 1, which asked for one more neighbor than exists.
  Also skips embedding dimensions longer than the series, and stops searching after 200 box enlargements as a safety net.
  Output: unchanged wherever c1 previously finished.
- **Fixed out-of-range memory use** when shuffling the centers: the shuffle partner `int(rand*nmax-(m-1)*d)+1` could be zero or negative for m ≥ 2, so the result depended on whatever was in memory.
  The number of centers is also capped at the number of embedded points.
  Output: changes.
- **Deterministic centers**: the centers are the first points of a fixed, evenly spread ordering (a golden-ratio lattice) instead of a random permutation, so the result no longer depends on the seed `-I`.
  Output: changes.
- **Fixed a bias in the dimension estimates**: the sum of log radii was divided by `ncmin-(m-1)*d` instead of by the number of centers, which shrank every estimate, more so at large m·d and with few centers (on iid noise at d=10, m=4: 2.35 / 3.17 / 3.32 for 100 / 500 / 2000 centers, now 3.36 / 3.37 / 3.37).
  Output: changes.
- **Repeated points**: a center that coincides with its k-th neighbor (to within 1e-12 of the data range) makes the mean log distance minus infinity, so that mass is skipped, instead of flooring the distance at 1e-20.
  Output: changes only for data with repeated points.

### c2g, c2t

- **Double precision** throughout.
  In single precision the interpolation prefactor in c2g overflowed on steep local slopes, giving Inf/NaN.
- **c2g no longer integrates a stale point**: the counter was incremented before the check that ends each block, so an uninitialized point, or one left from the previous embedding dimension, entered the integral.
- Output: changes.
  Inf/NaN results become finite; otherwise results agree closely.

### d2

- Writes correlation sums and slopes with `%.17g` instead of `%e`, so they are not rounded to six significant digits.
  Output: same values, more digits.

### false_nearest

- **Fixed a segfault** when `(M+1)·d` is at least the data length (unsigned loop bounds wrapped around); the program now exits with status 54 and a message.
  Output: unchanged wherever it previously ran.
- **The delay `-d` now sets the lag between embedding coordinates for a scalar series.**
  In 3.0.1 the scalar case always embedded consecutive samples, and `-d` only changed which points were used.
  Output: changes for `-d` > 1; identical for `-d1`.
- **Ties**: distances within 1e-12 (data rescaled to [0,1]) count as equal, pairs closer than that are repeated points rather than neighbors, and every neighbor tied with the nearest is tested, each point contributing the fraction of them that are false.
  The result no longer depends on rounding or on the order of the box search, and is unchanged by affine transforms and negation of the series.
  Output: unchanged when nearest neighbors are unique; changes with ties.
- **A fifth output column**: the number of points behind each fraction.
  Scripts that expect exactly four columns need updating.

### lyap_r

- **Fixed an endless loop** on data where some reference point has no neighbor within the acceptance radius (coarsely quantized series, or a few outliers squashing the rest into one box): epsilon grew until it overflowed.
  The search now stops, and if any reference point is unresolved lyap_r writes no output and exits with status 54, rather than reporting an estimate from only some of the points.
  Output: unchanged wherever lyap_r previously finished, except that such data now give exit status 54 and no output.
- **Ties**: every neighbor tied with the nearest (to within 1e-12, repeated points excluded) contributes with equal weight, instead of whichever was found first, and pairs that merge within the steps followed are left out at every step.
  Output: unchanged when nearest neighbors are unique and never merge; changes with ties.

### lyap_spec

- **New option `-t`**, a Theiler window: neighbors closer in time than this to the reference point are not used in the local linear fits.
  Output: unchanged by default (`-t0`).

### rank.f (used by c1)

- No longer reads `x(0)` at the end of a list (Fortran does not short-circuit `.and.`), which stopped bounds-checked builds.
  Output: unchanged.

### Build

- `configure` finds C compilers again: its test program declared `main` without a return type, which clang 16+ and GCC 14+ reject, so no C programs were built.
- `configure` tries `gfortran`.
- `cluster.f` compiles with gfortran (a format with no field width).
- `make install` creates the install directory if needed, and no longer stops before installing the C programs when no Fortran compiler is found.
- Hollerith constants in format strings replaced by quoted strings, and two C warnings fixed.
- Regression tests (`tests/`) and continuous integration on Linux and macOS.
