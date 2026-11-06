# Subgroup-based Radix Sort Partition

Also called: GLSL Subgroup Histogram Sort, Subgroup Radix Sort

## Overview
This method performs a single pass of a radix sort for a partition of elements (typically 4096) using GPU subgroup ballot operations to compute local offsets, thereby minimizing atomic contention on shared memory. It builds on [8-bit Radix Sort](2599-8-bit-radix-sort.md) by replacing per-element atomic updates with subgroup-level exclusive scans, making it suitable for high-throughput sorting in compute shaders where subgroup size is known and fixed.

## How it works
**Inputs:**
- `keys`: Array of 32-bit unsigned integers to sort.
- `values`: Array of 32-bit unsigned integers (or other data) to be permuted with keys.
- `element_count`: Total number of valid elements in the partition.
- `pass`: Current radix pass index (0–3 for 32-bit keys).
- `local_keys`, `local_values`: Shared memory buffers for keys and values.
- `local_histogram`: Shared memory buffer for histogram counts.
- `local_histogram_sum`: Shared memory buffer for reduced histogram sums.
- `global_histogram`: Global memory array storing cumulative histogram sums across partitions.
- `partition_histogram`: Global memory array storing histogram sums for the current partition.

**Outputs:**
- `keys`: Sorted keys written to global memory.
- `values`: Corresponding values written to global memory.

**Data Structures:**
- `local_histogram`: `uint` array of size `SHMEM_SIZE` (4096). Layout: `local_histogram[gl_NumSubgroups * radix + subgroup_index]`. Stores per-subgroup, per-radix bin counts.
- `local_histogram_sum`: `uint` array of size `RADIX` (256). Stores the sum of bin counts across subgroups for a specific radix.
- `local_keys`: `uint` array of size `PARTITION_SIZE` (4096).
- `local_values`: `uint` array of size `PARTITION_SIZE` (4096).
- `local_offsets`: `uint` array of size `PARTITION_SIZE` (4096). Stores local destination offsets for each element.
- `subgroup_histogram`: `uint` array of size `gl_NumSubgroups`. Stores per-subgroup histogram sums.

**Parameters:**
- `PARTITION_SIZE`: 4096.
- `RADIX`: 256 ($2^8$).
- `pass`: 0, 1, 2, or 3.

**Steps:**

1. **Initialize Local Histograms:**
   For each radix index $i$ from 0 to `RADIX`-1, iterate over all subgroups and set `local_histogram[gl_NumSubgroups * i + subgroup_index] = 0`.

2. **Load Keys and Values:**
   For each element $i$ in the partition:
   - Calculate global index: `global_index = gl_GlobalInvocationID.x + in_offset`.
   - If `global_index < element_count`:
     - `local_keys[i] = keys[key_index + in_offset]`
     - `local_values[i] = values[key_index + in_offset]`
   - Else:
     - `local_keys[i] = 0xffffffff`
     - `local_values[i] = 0`

3. **Extract Radix Digit:**
   For each element $i$:
   - `digit = bitfieldExtract(local_keys[i], pass * 8, 8)`

4. **Compute Subgroup Ballot Masks:**
   For each element $i$:
   - `mask = subgroupBallot(true)`
   - For each bit $j$ from 0 to 7:
     - `ballot = subgroupBallot(digit == (1 << j))`
     - `mask &= uvec4(digit - 1) ^ ballot`
   - This creates a mask where bits are set if the corresponding radix digit matches the current element's digit.

5. **Compute Subgroup Offset and Count:**
   For each element $i$:
   - `subgroup_offset = subgroupBallotExclusiveBitCount(mask)`
   - `radix_count = subgroupBallotBitCount(mask)`

6. **Reduce Local Histogram Across Subgroups:**
   For each index $i$ in `local_histogram` with stride `WORKGROUP_SIZE`:
   - `v = local_histogram[i]`
   - `sum = subgroupAdd(v)`
   - `excl = subgroupExclusiveAdd(v)`
   - `local_histogram[i] = excl`
   - If `thread_index == 0`:
     - `local_histogram_sum[i / gl_SubgroupSize] = sum`

7. **Reduce `local_histogram_sum` Across Subgroups (128 elements):**
   For each index $i$ from 0 to `intermediate_offset`-1 (where `intermediate_offset = RADIX * gl_NumSubgroups / gl_SubgroupSize`):
   - `v = local_histogram_sum[i]`
   - `sum = subgroupAdd(v)`
   - `excl = subgroupExclusiveAdd(v)`
   - `local_histogram_sum[i] = excl`
   - If `thread_index == 0`:
     - `local_histogram_sum[intermediate_offset + i / gl_SubgroupSize] = sum`

8. **Reduce `local_histogram_sum` Further (4 or 1 elements):**
   For each index $i$ from 0 to `intermediate_size`-1:
   - `v = local_histogram_sum[intermediate_offset + i]`
   - `excl = subgroupExclusiveAdd(v)`
   - `local_histogram_sum[intermediate_offset + i] = excl`

9. **Add Reduced Sums Back to `local_histogram_sum`:**
   For each index $i$ from 0 to `intermediate_offset`-1:
   - `local_histogram_sum[i] += local_histogram_sum[intermediate_offset + i / gl_SubgroupSize]`

10. **Add Reduced Sums to `local_histogram`:**
    For each index $i$ in `local_histogram` with stride `WORKGROUP_SIZE`:
    - `local_histogram[i] += local_histogram_sum[i / gl_SubgroupSize]`

11. **Post-Scan Stage: Update Local Offsets:**
    For each element $i$:
    - `local_offsets[i] = local_histogram[gl_NumSubgroups * radix + subgroup_index]`
    - If `subgroup_histogram[subgroup_index] > 0`:
      - `atomicAdd(local_histogram[gl_NumSubgroups * radix + subgroup_index], subgroup_histogram[subgroup_index])`

12. **Compute Global Offsets:**
    For each index $i$ from 0 to `RADIX`-1:
    - `v = (i == 0) ? 0 : local_histogram[gl_NumSubgroups * i - 1]`
    - `local_histogram_sum[i] = global_histogram[RADIX * pass + i] + partition_histogram[RADIX * partition_index + i] - v`

13. **Rearrange Keys into `local_histogram` (Aliasing Memory):**
    For each element $i$:
    - `local_histogram[local_offsets[i]] = local_keys[i]`
    - *Note: A barrier is required here to ensure all histogram data is consumed before overwriting.*

14. **Binning: Write Keys to Global Memory:**
    For each element $i$:
    - `radix = bitfieldExtract(local_histogram[i], pass * 8, 8)`
    - `dst_offset = local_histogram_sum[radix] + i`
    - If `dst_offset < element_count`:
      - `keys[dst_offset + out_offset] = local_histogram[i]`
    - `local_keys[i / WORKGROUP_SIZE] = dst_offset`

15. **Rearrange Values into `local_histogram`:**
    For each element $i$:
    - `local_histogram[local_offsets[i]] = local_values[i]`

16. **Binning: Write Values to Global Memory:**
    For each element $i$:
    - `value = local_histogram[i]`
    - `dst_offset = local_keys[i / WORKGROUP_SIZE]`
    - If `dst_offset < element_count`:
      - `values[dst_offset + out_offset] = value`

**Pitfalls:**
- **Boundary Check:** When loading keys/values, if `key_index >= element_count`, load `0xffffffff` for keys and `0` for values to prevent out-of-bounds access and ensure invalid elements are sorted to the end.
- **Memory Aliasing:** `local_histogram` is reused for storing keys after the histogram reduction phase to save shared memory. This requires barriers to ensure all histogram data is consumed before overwriting.

## Operating characteristics
- **Time Complexity:** $O(N)$ per radix pass, where $N$ is the partition size. The use of subgroup ballot operations reduces the constant factor compared to atomic-based approaches.
- **Memory Complexity:** $O(PARTITION_SIZE)$ shared memory for keys, values, and histogram buffers.
- **Strengths:** Minimizes atomic contention by using subgroup-level exclusive scans. Efficient for small to medium partition sizes where subgroup operations are fast.
- **Limitations:** Requires fixed subgroup size. Performance may degrade if subgroup size is not a power of two or if the hardware does not support efficient ballot operations.
- **When to Use:** High-throughput sorting in compute shaders where subgroup size is known and fixed, and atomic contention is a bottleneck.
- **When Not to Use:** When subgroup size is variable or unknown, or when sorting very large partitions where global memory bandwidth becomes the bottleneck.

## Background
This method is derived from the implementation in the Godot Gaussian Splatting project, which uses subgroup ballot operations to optimize radix sort partitions. It is an adaptation of parallel radix sort techniques to leverage GPU subgroup primitives for improved performance.

## Sources
[S1] ReconWorldLab/godot-gaussian-splatting, https://github.com/ReconWorldLab/godot-gaussian-splatting, fetched 2026-10-27 (unofficial)
