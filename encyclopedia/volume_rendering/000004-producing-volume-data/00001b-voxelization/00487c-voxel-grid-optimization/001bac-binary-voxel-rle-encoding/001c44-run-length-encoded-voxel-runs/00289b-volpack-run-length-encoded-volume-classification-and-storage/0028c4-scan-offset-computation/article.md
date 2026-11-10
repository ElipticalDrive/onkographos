# Scan Offset Computation

Also called: ComputeScanOffsets, RLE Scan Offset Computation

## Overview
Scan Offset Computation recomputes the `scan_offsets` array for a classified volume, where each `ScanOffset` entry stores the first length offset and first data offset for a specific scanline $(j, k)$ plane [S1, unofficial]. Unlike [ComputeIndex](52-compute-index.md), which maps every voxel coordinate to a specific run position, this method only stores the starting offsets for each scanline $(j, k)$, requiring the renderer to traverse the RLE runs sequentially along the $i$-axis to locate voxels [S1, unofficial].

## How it works
The method iterates through the volume dimensions in the order $k$, then $j$, accumulating the lengths of RLE headers and data payloads to compute cumulative offsets [S1, unofficial].

**Inputs and Outputs:**
*   **Input:** A classified RLE volume structure (`rle_voxels`) containing:
    *   `ilen`, `jlen`, `klen`: The dimensions of the volume along the $i$, $j$, and $k$ axes respectively [S1, unofficial].
    *   `run_lengths`: A pointer to the RLE header data, consisting of pairs of integers: `zerocount` (number of zero-valued voxels) and `nonzerocount` (number of non-zero voxels) for each run [S1, unofficial].
    *   `voxel_size`: The size in bytes of a single voxel's data payload [S1, unofficial].
    *   `mmapped`: A boolean flag indicating if the data is memory-mapped [S1, unofficial].
*   **Output:** An array of `ScanOffset` structures, indexed by $(j, k)$, where each entry contains the starting byte offset for the RLE headers (`first_len`) and the starting byte offset for the voxel data (`first_data`) [S1, unofficial].

**Data Structures:**
The `ScanOffset` structure is defined as:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```c
typedef struct {
    unsigned first_len;   // Offset into the RLE header array (run_lengths)
    unsigned first_data;  // Offset into the voxel data array
} ScanOffset;
```
The output array is allocated as `ScanOffset*` of size `klen * jlen` [S1, unofficial]. The indexing into this array for a specific scanline $(j, k)$ is `scan_offsets[k * jlen + j]` [G].

**Algorithm Steps:**
1.  **Guard Check:** If `rle_voxels->mmapped` is true, return `VPERROR_IO` immediately to prevent modification of memory-mapped data [S1, unofficial].
2.  **Initialization:**
    *   Let $I = \text{rle\_voxels->ilen}$, $J = \text{rle\_voxels->jlen}$, $K = \text{rle\_voxels->klen}$ [S1, unofficial].
    *   Let $V_{size} = \text{rle\_voxels->voxel\_size}$ [S1, unofficial].
    *   Initialize `len_offset = 0` and `data_offset = 0` [S1, unofficial].
    *   Initialize a pointer `RLElen` to the start of `rle_voxels->run_lengths` [S1, unofficial].
3.  **Iteration:** Loop $k$ from $0$ to $K-1$:
    *   Loop $j$ from $0$ to $J-1$:
        *   Set `scan_offsets[k * J + j].first_len = len_offset` and `scan_offsets[k * J + j].first_data = data_offset` [S1, unofficial].
        *   Initialize `i = 0` as a counter for processed voxels [S1, unofficial].
        *   While `i < I`:
            *   Read `zerocount = *RLElen++` and `nonzerocount = *RLElen++` [S1, unofficial].
            *   Increment `len_offset` by $2$ (each run contributes two integers to the header stream) [S1, unofficial].
            *   Add `nonzerocount * V_size` to `data_offset` [S1, unofficial].
            *   Add `zerocount + nonzerocount` to `i` [S1, unofficial].
        *   Assert `i == I` to verify the RLE data exactly covers the scanline length [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_scan_offsets(rle_voxels):
    if rle_voxels.mmapped:
        return "VPERROR_IO"

    ilen = rle_voxels.ilen
    jlen = rle_voxels.jlen
    klen = rle_voxels.klen
    voxel_size = rle_voxels.voxel_size

    scan_offsets = [None] * (klen * jlen)
    run_lengths = rle_voxels.run_lengths
    rle_idx = 0

    len_offset = 0
    data_offset = 0

    for k in range(klen):
        for j in range(jlen):
            scan_offsets[k * jlen + j] = {
                'first_len': len_offset,
                'first_data': data_offset
            }

            voxels_processed = 0
            while voxels_processed < ilen:
                zerocount = run_lengths[rle_idx]
                nonzerocount = run_lengths[rle_idx + 1]
                rle_idx += 2

                len_offset += 2
                data_offset += nonzerocount * voxel_size
                voxels_processed += zerocount + nonzerocount

            if voxels_processed != ilen:
                raise AssertionError("RLE data mismatch for scanline")

    return scan_offsets
```

**Parameters:**
*   `voxel_size`: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Memory-Mapped Data:** If `rle_voxels->mmapped` is true, the function returns `VPERROR_IO` immediately to prevent modification of read-only memory [S1, unofficial].
*   **RLE Consistency:** The sum of `zerocount` and `nonzerocount` for all runs in a scanline must exactly equal `ilen`. An assertion verifies this condition at the end of each scanline [S1, unofficial].
*   **Offset Overflow:** If the volume is very large, `len_offset` or `data_offset` may exceed the capacity of `unsigned` integers. The sources use `unsigned`, implying a limit of $2^{32}-1$ bytes/entries. Not given in the sources.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of RLE runs in the volume, as every RLE entry must be read exactly once to compute offsets [G].
*   **Memory Complexity:** $O(J \times K)$ for the `scan_offsets` array, where each entry is 8 bytes (two `unsigned` integers) [G].
*   **Strengths:** Simple to implement; low memory overhead for the offset table; enables fast ray casting by providing direct access to the start of each scanline's data [G].
*   **Limitations:** Does not provide direct random access to individual voxels; requires sequential traversal of RLE runs for each ray [S1, unofficial].
*   **When to Use:** When rendering volumes via ray casting along the $i$-axis (or other axes with appropriate permutation) and the primary access pattern is scanline-based [G].
*   **When Not to Use:** When random voxel access is required (use [ComputeIndex](52-compute-index.md) instead) [S1, unofficial].

## Background
Scan Offset Computation is part of the Volpack library, a widely used volume rendering toolkit [S1, unofficial]. It was developed to optimize the rendering of large volumetric datasets by reducing the overhead of locating data during ray casting [G]. The method is specific to the Volpack RLE format, which classifies voxels and stores them in a run-length encoded structure optimized for three orthogonal viewing axes [S1, unofficial].

## Sources
[S1] ferdymercury/volpack, https://github.com/ferdymercury/volpack, fetched 2026-10-27 (unofficial)
