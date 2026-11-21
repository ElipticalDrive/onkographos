# RLE Index Computation

Also called: ComputeIndex

## Overview
RLE Index Computation generates a lookup structure that maps 3D voxel coordinates to their corresponding positions within a Run-Length Encoded (RLE) data stream. It solves the problem of efficiently locating non-zero voxel data without scanning the entire compressed volume. It differs from raw RLE storage by providing direct coordinate-to-offset mapping, enabling $O(1)$ access to run headers, whereas sibling methods like [Voxel Indexing and Bit Compression](1707-voxel-indexing-and-bit-compression.md) focus on the bit-level packing of the index itself.

## How it works
The method constructs a `VoxelLocation` array where each entry corresponds to a voxel in the 3D grid. Each entry stores the offset to the start of the RLE run containing that voxel and the position within that run.

**Inputs:**
- `rle_voxels`: The RLE-encoded volume data, containing an array of run lengths.
- `ilen`, `jlen`, `klen`: The dimensions of the voxel grid along the x, y, and z axes, respectively.

**Outputs:**
- `voxel_index`: An array of `VoxelLocation` structs, one for each voxel in the grid. If the grid is too large for the fixed-size index, this is set to `NULL`.

**Data Structures:**
The index entry is defined as `VoxelLocation` with the following fields:
- `run_count`: `int`. The number of voxels remaining in the current run.
- `len_offset`: `unsigned char`. The byte offset into the `run_lengths` array where the current run's length is stored.
- `data_offset`: `unsigned char`. The byte offset into the voxel data array where the current run's data begins.
  - Bit Packing: The least significant bit of `data_offset` is a flag `INDEX_RUN_IS_ZERO`. If set, the run consists of zero-valued voxels.

**Algorithm:**
1. Initialize `ilen`, `jlen`, `klen` from the dimensions of `rle_voxels`.
2. Initialize a pointer `RLElen` to the start of `rle_voxels->run_lengths`.
3. Initialize `run_is_zero = 0` and `run_count = 0`.
4. Loop `k` from 0 to `klen - 1`:
   5. Loop `j` from 0 to `jlen - 1`:
      6. Assert `run_is_zero == 0` and `run_count == 0`. This ensures that the previous voxel's run has been fully consumed.
      7. Initialize `len_offset = 0` and `data_offset = 0`.
      8. Loop `i` from 0 to `ilen - 1`:
         9. Check if `len_offset > 256`. If true, the index cannot fit in the allocated byte-sized fields. Deallocate `voxel_index`, set it to `NULL`, and return `VPERROR_LIMIT_EXCEEDED`.
         10. Store `index->run_count = run_count`.
         11. Store `index->len_offset = len_offset`.
         12. Set `index->data_offset` based on `run_is_zero`. If `run_is_zero` is true, OR `INDEX_RUN_IS_ZERO` into `data_offset`.
         13. Increment the `index` pointer to the next voxel location.
         14. While `run_count == 0`:
             15. Read `run_count = *RLElen++`.
             16. Toggle `run_is_zero` (0 becomes 1, 1 becomes 0).
             17. Increment `len_offset`.
         18. Decrement `run_count`.
         19. If `!run_is_zero`, add `voxel_size` to `data_offset`.

**Parameters:**
- `INDEX_RUN_IS_ZERO`: A bit flag (typically `0x01`) OR-ed into the `data_offset` field to indicate zero-valued runs.

**Edge Cases and Pitfalls:**
- **Index Overflow:** If the number of RLE runs exceeds 256, `len_offset` will exceed the capacity of an `unsigned char`. The method handles this by returning an error (`VPERROR_LIMIT_EXCEEDED`) and setting the index to `NULL` [S1, unofficial].
- **Run Consumption:** The algorithm assumes that runs are consumed exactly as the voxel coordinates are traversed. The assertion `run_is_zero == 0` and `run_count == 0` at the start of each (j, k) loop ensures that no partial runs are carried over incorrectly between slices or rows.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N = ilen \times jlen \times klen$. Each voxel is visited once, and RLE pointers are advanced linearly.
- **Memory Complexity:** $O(N)$ for the index array, with each entry being 3 bytes (1 byte `run_count`, 1 byte `len_offset`, 1 byte `data_offset`).
- **Limitations:** The fixed-size `unsigned char` fields limit the number of distinct RLE runs to 256. This is a hard constraint of the original VolPack implementation [S1, unofficial].

## Background
This method is part of the VolPack volume rendering library, originally developed by Phil Lacroute at Stanford University in 1994 [S1, unofficial]. It was designed to provide efficient access to sparse volume data encoded with Run-Length Encoding, allowing renderers to skip zero-valued regions quickly.

## Sources
[S1] ferdymercury/volpack, https://github.com/ferdymercury/volpack, fetched 2026-10-27 (unofficial)
