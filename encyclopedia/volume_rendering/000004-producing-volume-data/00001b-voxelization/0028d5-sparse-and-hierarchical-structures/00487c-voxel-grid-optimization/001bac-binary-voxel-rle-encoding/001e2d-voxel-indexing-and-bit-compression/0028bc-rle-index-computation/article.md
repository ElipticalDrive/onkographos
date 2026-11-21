# RLE Index Computation

Also called: ComputeIndex

## Overview
RLE Index Computation computes an index mapping 3D voxel coordinates to RLE run data [S1, unofficial]. The index stores byte offsets to the beginning of the run containing each voxel, plus a count indicating the voxel's position within that run [S1, unofficial]. It differs from raw RLE storage by providing direct coordinate-to-offset mapping [S1, unofficial].

## How it works
**Inputs:** `rle_voxels` (RLE-encoded volume data containing a `run_lengths` array), `ilen`, `jlen`, `klen` (grid dimensions along the x, y, and z axes), and `voxel_size` (size of a single voxel in bytes) [S1, unofficial].

**Outputs:** `voxel_index` (array of `VoxelLocation` structs, one per voxel) [S1, unofficial]. If the grid exceeds the index capacity, `voxel_index` is set to `NULL` and `VPERROR_LIMIT_EXCEEDED` is returned [S1, unofficial].

**Data Structures:** The index entry is the `VoxelLocation` struct with fields `run_count` (`int`), `len_offset` (`unsigned char`), and `data_offset` (`unsigned char`) [S1, unofficial]. The `data_offset` field packs a flag: `INDEX_RUN_IS_ZERO` is OR-ed into it to indicate zero-valued runs [S1, unofficial].

**Numbered Steps:**
1. Initialize `ilen`, `jlen`, `klen` from the dimensions of `rle_voxels` [S1, unofficial].
2. Initialize a pointer `RLElen` to the start of `rle_voxels->run_lengths` [S1, unofficial].
3. Initialize `run_is_zero = 0` and `run_count = 0` [S1, unofficial].
4. Loop `k` from 0 to `klen - 1` [S1, unofficial].
5. Loop `j` from 0 to `jlen - 1` [S1, unofficial].
6. Assert `run_is_zero == 0` and `run_count == 0` [S1, unofficial].
7. Initialize `len_offset = 0` and `data_offset = 0` [S1, unofficial].
8. Loop `i` from 0 to `ilen - 1` [S1, unofficial].
9. Check if `len_offset > 256`; if so, deallocate `voxel_index`, set it to `NULL`, and return `VPERROR_LIMIT_EXCEEDED` [S1, unofficial].
10. Store `index->run_count = run_count` and `index->len_offset = len_offset` [S1, unofficial].
11. Set `index->data_offset` based on the `run_is_zero` flag [S1, unofficial].
12. Increment the `index` pointer to the next voxel location [S1, unofficial].
13. While `run_count == 0`, read `run_count = *RLElen++`, toggle `run_is_zero`, and increment `len_offset` [S1, unofficial].
14. Decrement `run_count` [S1, unofficial].
15. If `!run_is_zero`, add `voxel_size` to `data_offset` [S1, unofficial].

**Formulas:**
- Index entry layout: `VoxelLocation { int run_count; unsigned char len_offset; unsigned char data_offset; }` [S1, unofficial].
- Data offset encoding: `data_offset = base_offset | (INDEX_RUN_IS_ZERO if run_is_zero else 0)` [S1, unofficial].

**Parameters:**
- `INDEX_RUN_IS_ZERO`: A bit flag OR-ed into the `data_offset` field to indicate that the current run consists of zero-valued voxels [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Index Overflow:** If `len_offset` exceeds 256, the index cannot be stored in the allocated byte-sized fields [S1, unofficial]. The method handles this by deallocating `voxel_index`, setting it to `NULL`, and returning `VPERROR_LIMIT_EXCEEDED` [S1, unofficial].
- **Run Consumption:** The assertion `run_is_zero == 0` and `run_count == 0` at the start of each `j` loop ensures that the previous voxel's run has been fully consumed before processing the next slice [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_rle_index(rle_voxels, ilen, jlen, klen, voxel_size):
    RLElen = rle_voxels.run_lengths
    run_is_zero = 0
    run_count = 0
    voxel_index = allocate_array(ilen * jlen * klen)
    index_ptr = voxel_index
    INDEX_RUN_IS_ZERO = 0x01

    for k in range(klen):
        for j in range(jlen):
            assert run_is_zero == 0 and run_count == 0
            len_offset = 0
            data_offset = 0
            for i in range(ilen):
                if len_offset > 256:
                    deallocate(voxel_index)
                    return None, VPERROR_LIMIT_EXCEEDED
                index_ptr.run_count = run_count
                index_ptr.len_offset = len_offset
                index_ptr.data_offset = data_offset | (INDEX_RUN_IS_ZERO if run_is_zero else 0)
                index_ptr += 1
                while run_count == 0:
                    run_count = RLElen[0]
                    RLElen += 1
                    run_is_zero = 1 - run_is_zero
                    len_offset += 1
                run_count -= 1
                if not run_is_zero:
                    data_offset += voxel_size
    return voxel_index, VP_OK
```

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N = ilen \times jlen \times klen$ [S1, unofficial]. Each voxel is visited once, and RLE pointers are advanced linearly [S1, unofficial].
- **Memory Complexity:** $O(N)$ for the index array, with each entry occupying 3 bytes (1 byte `run_count`, 1 byte `len_offset`, 1 byte `data_offset`) [S1, unofficial].
- **Limitations:** The fixed-size `unsigned char` fields limit the number of distinct RLE runs to 256 [S1, unofficial].

## Background
This method is part of the VolPack volume rendering library, originally developed by Phil Lacroute at Stanford University in 1994 [S1, unofficial]. It was designed to provide efficient access to sparse volume data encoded with Run-Length Encoding [S1, unofficial].

## Sources
[S1] ferdymercury/volpack, https://github.com/ferdymercury/volpack, fetched 2026-10-27 (unofficial)
