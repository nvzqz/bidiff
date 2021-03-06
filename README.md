
# bidiff

`bidiff` is a set of rust crates that generate and apply patches for arbitrary
binary files. 

**NOTE: bidiff has not been thoroughly tested and should be considered ALPHA-GRADE SOFTWARE.**

**We do not guarantee compatibility of patches between versions until v1.0.**

It is particularly well-suited to deploying software updates, and:

  * Is written in 100% safe Rust
  * Has been fuzzed thoroughly
  * Has reasonable memory requirements
  * Supports large (>2GiB) inputs
  * Supports multiple compression formats (brotli, zstd, etc.) via [comde][] 

[comde]: https://crates.io/crates/comde

## License

This project (`bidiff`, `bipatch` and `bic`) is licensed under either of

 * Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
 * MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.

## Benchmarks

In this section, we compare using bidiff to generate patches against two versions
of well-known software, versus just compressing the newer version with `zstd` (hereafter
referred to as "naive").

All measurements are done on an	Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz, with 12 partitions and 1 MiB chunks.

`zstd` compression at level 21 is used in both "naive" and "bidiff" runs:

| older                         | newer         | newer size | naive size | naive time  | bidiff size    | bidiff time     |
|-------------------------------|---------------|------------|------------|-------------|----------------|-----------------|
| Wine 4.18 sources             | 4.19 sources  | 201 MiB    | 21.96 MiB  | 81.300s     | **182.26 KiB** | **5.220s**      |
| Linux 5.3.13 sources          | 5.4 sources   | 894.68 MiB | 106.73 MiB | 448.145s    | **6.14 MiB**   | **57.799s**     |
| Chromium 78 binary (build 97) | (build 108)   | 257.04 MiB | 79.73 MiB  | **116.975** | **12.81 MiB**  | 158.867s        |
| Firefox 71 binary (build 11)  | (build 12)    | 192.49 MiB | 54.64 MiB  | **78.039s** | **6.56 MiB**   | 81.847s         |

## Repository organization

This repository contains three crates:

  * `crates/bidiff` contains the diff algorithm, and has an optional `enc` feature
  for serialization.
  * `crates/bipatch` contains code that reads and applies patches generated by
  `bidiff`'s `enc` feature.
  * `crates/bic` is a *demonstration* command-line interface to the above two
  crates, which uses `comde` for compression.

The essential part is contained in the `bidiff` crate itself. The
serialization/deserialization code is provided as a (practical) example of how
to store patch files. It is very simplistic: a magic number, a version number,
and then a series of ADD, COPY and SEEK instructions, with variable-length
integer encoding.

> Note: `bidiff` and `bipatch` do not concern themselves with compression, but
patch files **MUST**  be compressed. Uncompressed, they are slightly larger
than the "newer" file (but lower-entropy).

Diffing is a memory- and CPU-intensive task. The requirements are discussed
below. Patching, however, can be done in streaming fashion and requires very
little memory (only small buffers to perform addition between the "older" file
and parts of the "patch" file).

---

## Prior art: `bsdiff`

`bidiff` is based on the `bsdiff` algorithm as described in [Naïve
Differences in Executable Code (2003)][] by Colin Percival, but the
implementation differs in a few key areas.

[Naïve Differences in Executable Code (2003)]: https://www.daemonology.net/papers/bsdiff.pdf

The key idea behind `bsdiff` is to find approximate matches between an "older"
and a "newer" file, and then stored the *difference* between those, which
ends up having lower entropy than the newer data itself. After compression,
the resulting patch file is usually much smaller than the newer file would be
if compressed.

The algorithm works in two phases:

  * **Sorting**, in which all suffixes of the "older" file are sorted, and their
  rank is stored in a suffix array. This is done using a SACA (Suffix Array Construction Algorithm).
  * **Scanning**, in which the suffix array is used to find approximate matches.
  The most promising approximate matches are selected using a simple scoring system.

The original `bsdiff` implementation tends to be quite time and memory-hungry.

In particular:

  * It requires `max(17*n,9*n+m)+O(1)` bytes of memory (where `n` and `m` are the size of the older and newer file respectively)
  * It stores "control", "difference" and "extra" data separately for "better compression"
  * It uses `bzip2` as a compression method.
  * It uses a rather old SACA, described in [Faster Suffix Sorting (1999)][] by Larsson and Sadakane.

[Faster Suffix Sorting (1999)]: http://www.larsson.dogma.net/ssrev-tr.pdf

## What makes `bidiff` different

`bidiff` generates patches of similar size compared to `bsdiff`, but is generally
faster and requires less memory, thanks to the following changes.

`bidiff` does suffix sorting using a [hand-ported Rust
version][divsufsort-rust] of [divsufsort][] by Yuta Mori, a highly-optimized
suffix sorting algorithm that has been repeatedly mentioned in research,
despite the emergence of newer sorting methods.

[divsufsort]: https://github.com/y-256/libdivsufsort

Whilst not "the latest" in suffix sorting, it performs surprisingly well in
practice (in near-linear time) and has modest memory requirements of `O(5*n)`
(where `n` is the size of the "older" file).

[divsufsort-rust]: https://github.com/fasterthanlime/stringsearch/tree/master/crates/divsufsort

Switching to `divsufsort` alone is already a significant improvement on the
original `bsdiff` in terms of memory usage and time efficiency, but `bidiff`
does not stop there.

For large file support, `bidiff` does not use the `divsufsort64` variant,
which would require `O(9*n)` memory (plus a larger constant). Instead, it
introduces "partitioned suffix arrays". The "older" file is split into
multiple partitions, each suffix sorted separately.

This achieves two things:

  * The maximum input size is no longer limited to 2**31-1 (due to using `i32`
  as an index type)
  * Suffix sorting can be performed in parallel

In practice, suffix sorting in `bidiff` *is* performed in parallel
using [rayon][], as long as the number of partitions is greater than one.

[rayon]: https://crates.io/crates/rayon

> Note: switching to `u32` as an index type to raise the original limit from
2GiB to 4GiB is not an option. `divsufsort` uses negative numbers abundantly
while sorting, so the only options are `i32` and `i64`.

Using partitioned suffix arrays has a downside: the cost of searching for
the "longest matching substring" in the original text is no longer `O(log n)`,
but `O(p*log n)`, where `p` is the number of partitions.

Since that operation is performed a great number of times while diffing,
using a large number of partitions slows the **scanning** phase more than
it speeds up the **sorting** phase.

For sample 20MiB Windows executable files:

| partitions | sort time | scan time | total diff time |
|------------|-----------|-----------|-----------------|
| 1 | 1.2965049s | 726.9464ms | 2.031168s |
| 2 | 739.773ms | 1.0506574s | 1.798709s |
| 3 | 480.5651ms | 1.4029111s | 1.8912619s |
| 4 | 369.3306ms | 1.7776965s | 2.1549498s |
| 5 | 302.6711ms | 2.3324006s | 2.6430686s |
| 6 | 280.6257ms | 2.69022s | 2.9789148s |

> Another downside is that, due to partitioning, *very occasionally*,
worse matches are found, for substrings that cross the border between
two partitions. In practice, this only very slightly affects patch size,
as the rest of the match is found immediately thereafter, resulting in
two "ADD" operations rather than one, which still compresses nicely.

Seeing these results, one may consider performing binary searches in each
suffix array *in parallel*. However, experimentation has shown that, since
those are very short operations repeated many times, the cost of
synchronization overshadows any gains one might have hoped for.

`bidiff` addresses this another way, by parallelizing the **scanning** phase.
The "newer" file is split into "chunks", and each of them are diffed in
parallel (by as many workers as is efficient given the number of cores
available). The results are a series of approximate matches, which are
reordered and translated into a patch file by another worker.

> Note that dividing the newer file into chunks also contributes to
generating *very slightly* worse patch files: cross-chunk matches will
result in two ADD operations, where they would only be one before. However,
in practice, the difference is negligible.

This technique results in notable speedups.

For the same inputs as before:

| partitions | chunk size | sort time | scan time | total time |
|------------|------------|-----------|-----------|------------|
| 1 | 32 MiB | 1.2896446s | 669.5845ms | 1.967119s |
| 1 | 16 MiB | 1.2831451s | 501.6626ms | 1.7926627s |
| 1 | 8 MiB | 1.2936768s | 397.7302ms | 1.6991965s |
| 1 | 4 MiB | 1.2878612s | 327.009ms | 1.622483s |
| 1 | 2 MiB | 1.2888983s | 225.2414ms | 1.5213945s |
| 1 | 1 MiB | 1.2881292s | 176.2079ms | 1.4714389s |

When parallelizing scanning, sort time begins to dominate. Using a combination
of both techniques allow optimizing for core utilization and overall runtime:

| partitions | chunk size | sort time | scan time | total diff time |
|------------|------------|-----------|-----------|-----------------|
| 1 | 32 MiB | 1.3027385s | 665.7531ms | 1.9764095s |
| 1 | 4 MiB | 1.2900258s | 352.3892ms | 1.6501501s |
| 1 | 512 KiB | 1.3043592s | 143.3714ms | 1.4548654s |
| 2 | 32 MiB | 706.5673ms | 1.0025483s | 1.7168672s |
| 2 | 4 MiB | 723.699ms | 471.5766ms | 1.2028935s |
| 2 | 512 KiB | 703.7808ms | 201.0271ms | 911.9539ms |
| 4 | 32 MiB | 358.5569ms | 1.7222684s | 2.0884749s |
| 4 | 4 MiB | 356.8114ms | 828.6099ms | 1.1930674s |
| 4 | 512 KiB | 356.1849ms | 330.2086ms | 693.5397ms |
| 6 | 32 MiB | 273.307ms | 2.582831s | 2.8640321s |
| 6 | 4 MiB | 264.5068ms | 1.1823795s | 1.4542472s |
| 6 | 512 KiB | 240.1568ms | 444.4659ms | 691.7743ms |

These are one-off measurement on relatively small files (20 MiB) but this is
representative of the tradeoff that's done here - performance varies similarly
for larger files (and various kinds of files).

In this case, the overall speedup achieved is almost 3x, on a 6-core machine.
The optimal parameters for "partitions" and "chunk size" depend on the machine
used to diff, but are generally along the lines of:

  * `partitions = (num_cores - 1)` (this leaves a core for bookkeeping and compression)
  * `chunk size = newer_size / (num_cores * k)`, where `k` is between 2 and 4;

> Note: efforts to parallelize `divsufsort` *and still have a single suffix array*
have been attempted. However, the speedups are rather disappointing - the parallel
variant needs 4 to 8 cores before it starts beating the sequential version. [See for yourself][parallel-divsufsort].

[parallel-divsufsort]: https://github.com/jlabeit/parallel-divsufsort/blob/master/benchmarks/OVERVIEW.md

Choosing a chunk size that's too large will result in suboptimal core utilization, whereas
choosing a chunk size that's too small will result in increased memory usage for diminishing
returns:

| partitions | chunk size | sort time | scan time | total diff time |
|------------|------------|-----------|-----------|-----------------|
| 6 | 512 KiB | 246.8454ms | 460.2141ms | 714.4462ms |
| 6 | 128 KiB | 240.9415ms | 408.1361ms | 656.3144ms |
| 6 | 64 KiB | 243.5352ms | 426.3133ms | 676.7965ms |
| 6 | 16 KiB | 253.8723ms | 420.6165ms | 681.7169ms |
| 6 | 4 KiB | 257.9093ms | 424.8433ms | 689.8088ms |
| 6 | 2 KiB | 254.1255ms | 454.6246ms | 715.69ms |
| 6 | 1 KiB | 250.7823ms | 497.2212ms | 755.1542ms |
