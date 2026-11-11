# Group Prefix Sum for Shell Lists
## Overview
Group Prefix Sum for Shell Lists is a parallel shell list compaction method used in GPU-based isosurface extraction [S1, unofficial]. It replaces sequential processing by thread 0 with a parallel prefix sum approach [S1, unofficial]. It coordinates with the octree to derive block offsets from the first active octree nodes (AOs) in each shell [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Shell lists for a block of 128 functions containing AO offsets and coefficient data [S1, unofficial].
- Output: Compacted shell lists with updated AO offsets [S1, unofficial].
- Auxiliary: Octree block offsets derived from the first AO in each shell [S1, unofficial].
**Data Structures:**
- Shell list entries: Contain AO offsets and coefficient matrix rows (C) [S1, unofficial].
- Groupshared memory: Stores chunks of at most 32 AOs, with coefficient rows staged for simultaneous thread access [S1, unofficial].
- Octree offsets: Derived from the first AO of each shell in a block [S1, unofficial].
**Steps:**
1. Pack shell data into chunks of at most 32 AOs per block of 128 functions [S1, unofficial].
2. Stage rows of coefficient matrices (C) into groupshared memory for simultaneous reading by all threads [S1, unofficial].
3. Upload all 16 field records per thread [S1, unofficial].
4. Clamp loads from groupshared indices to handle inactive lanes (llvmpipe compatibility) [S1, unofficial].
5. Compute the parallel prefix sum (`iso_group_prefix`) across the shell list entries [S1, unofficial].
6. Extract AO offsets from the first AOs of each shell to determine octree block offsets [S1, unofficial].
7. Update shell list entries with compacted positions based on the prefix sum results [S1, unofficial].
**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def group_prefix_sum_shell_lists(shell_data, group_size=128, chunk_size=32):
    # 1. Pack shell data into chunks of at most 32 AOs per block of 128 functions
    chunks = [shell_data[i:i+chunk_size] for i in range(0, len(shell_data), chunk_size)]
    # 2. Stage rows of coefficient matrices (C) into groupshared memory
    # Not given in the sources.
    groupshared_C = [chunk[1] for chunk in chunks]
    # 3. Upload all 16 field records per thread
    # Not given in the sources.
    records = [row for chunk in groupshared_C for row in chunk[:16]]
    # 4. Clamp loads from groupshared indices to handle inactive lanes (llvmpipe)
    # Not given in the sources.
    clamped = [max(0, min(idx, len(groupshared_C)-1)) for idx in range(len(groupshared_C))]
    # 5. Compute parallel prefix sum (iso_group_prefix) across shell list entries
    # Not given in the sources.
    ao_offsets = [chunk[0] for chunk in chunks]
    flat_offsets = [ao for chunk in ao_offsets for ao in chunk]
    prefix_sums = [sum(flat_offsets[:i]) for i in range(len(flat_offsets))]
    # 6. Extract AO offsets from the first AOs of each shell for octree block offsets
    block_offsets = [chunk[0][0] for chunk in chunks]
    # 7. Update shell list entries with compacted positions
    compacted = []
    for i, chunk in enumerate(chunks):
        for j, ao in enumerate(chunk[0]):
            compacted.append((ao + prefix_sums[i * chunk_size + j], chunk[1]))
    return compacted, block_offsets
```
**Parameters:**
- Group size: 128 functions per block [S1, unofficial].
- Chunk size: Maximum 32 AOs per chunk [S1, unofficial].
- Field records: 16 per thread [S1, unofficial].
**Edge Cases:**
- Inactive lanes: Loads from groupshared memory must be clamped to prevent out-of-bounds access [S1, unofficial].
- llvmpipe compatibility: Special handling required for inactive lane loads [S1, unofficial].
- Orbital padding: Orbital sets no longer pad their functions to even indices [S1, unofficial].
- Empty shells: Not given in the sources.
## Subtopics

### [Fenwick Tree (Binary Indexed Tree)](6390-fenwick-tree-binary-indexed-tree.md)
A compact data structure storing partial sums for $O(\log n)$ point updates and prefix-sum queries. It differs from standard complete binary trees by storing only $1.5n$ elements instead of $2n$, since prefix-sum queries from index 0 never require right children, halving memory and update costs. Choose it for dynamic environments requiring incremental light transport integration in volume rendering.

## Operating characteristics
Parallel processing replaces sequential thread-0 compaction, improving throughput [S1, unofficial]. Groupshared memory staging reduces global memory bandwidth by allowing simultaneous thread access to coefficient matrix rows [S1, unofficial]. Inactive lane clamping introduces minor overhead for llvmpipe compatibility [S1, unofficial]. Not given in the sources: measured performance metrics, scaling behavior, or specific hardware benchmarks.
## Background
Originates from the `md_gto_iso_gpu` optimization in the `mdlib` repository by Scanberg [S1, unofficial]. It shifts shell list compaction from sequential thread-0 processing to a parallel prefix sum approach in GPU-based isosurface extraction [S1, unofficial]. The implementation handles llvmpipe compatibility via index clamping and uses environment variables `GTO_ISO_RELAX_FIRST=0` and `GTO_ISO_CORNERS_V2=0` to revert to legacy forms [S1, unofficial].
## Sources
[S1] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
