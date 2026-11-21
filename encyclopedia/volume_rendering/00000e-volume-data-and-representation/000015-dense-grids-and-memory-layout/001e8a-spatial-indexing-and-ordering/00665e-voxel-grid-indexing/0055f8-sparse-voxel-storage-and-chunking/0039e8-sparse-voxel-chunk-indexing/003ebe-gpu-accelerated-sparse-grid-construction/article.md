# GPU Accelerated Sparse Grid Construction

Also called: GPU Accelerated IndexGrid Construction, IndexGrid Construction on GPU

## Overview
This method constructs the **IndexGrid** acceleration structure on the GPU from a set of signed voxel coordinates, enabling the handling of large-scale inputs with a competitive memory footprint [S1]. Unlike standard NanoVDB which assumes a fixed topology, this algorithm supports dynamic sparsity patterns, such as those required for Level-of-Detail (LOD) hierarchies [S1]. It builds on [Sparse Voxel Chunk Indexing](8606-sparse-voxel-chunk-indexing.md) and implements the core construction logic for the [IndexGrid](9819-indexgrid.md) data structure [S1].

## How it works
The algorithm builds the hierarchical IndexGrid topology in a top-down manner using parallel sorting and run-length encoding to determine node counts and offsets [S1].

### Inputs and Outputs
*   **Input**: A set of signed voxel coordinates $(i, j, k)$ [S1].
*   **Output**: An allocated IndexGrid structure on the GPU device memory, containing upper, lower, and leaf nodes, with active voxels registered in leaf bitmasks [S1].

### Data Structures
The construction relies on two primary 64-bit integer keys for sorting and registration [S1].

1.  **Sort Key `a`**: Used to group coordinates by their coarse grid position [S1].
    $$ a = (k \gg 12) \mid ((j \gg 12) \ll 21) \mid ((i \gg 12) \ll 42) $$
    *   `i, j, k`: Signed voxel coordinates [S1].
    *   `>> 12`: Right shift by 12 bits extracts the coarse index [S1].
    *   `|`: Bitwise OR combines the shifted components [S1].
    *   Layout: Bits [0–20] store $k$'s coarse index, bits [21–41] store $j$'s coarse index, and bits [42–63] store $i$'s coarse index [S1].
    *   *Note: GPU kernels use unsigned 64-bit arithmetic to prevent sign-extension artifacts during shifts [G].*

2.  **Registration Key `b`**: Used to register nodes and voxels into their parent structures [S1].
    $$ b = (M \ll 36) \mid (\text{upper::off}(i,j,k) \ll 21) \mid (\text{lower::off}(i,j,k) \ll 9) \mid \text{leaf::off}(i,j,k) $$
    *   `M`: A multiplier identifying the run or group [S1].
    *   `upper::off(i,j,k)`: The offset of the upper node [S1].
    *   `lower::off(i,j,k)`: The offset of the lower node [S1].
    *   `leaf::off(i,j,k)`: The offset of the leaf node [S1].
    *   `b & 511`: Extracts the leaf offset (bits 0–8) for bitmask registration [S1].

### Algorithm Steps
1.  Input: signed voxel coordinates $(i, j, k)$ [S1].
2.  Define 64-bit keys $a$ for sorting: $a = (k \gg 12) \mid ((j \gg 12) \ll 21) \mid ((i \gg 12) \ll 42)$ [S1].
3.  Full radix sort of keys $a$ [S1].
4.  Run-Length-Encode sorted keys $a$ to identify runs [S1].
5.  For each run, define keys $b$: $b = (M \ll 36) \mid (\text{upper::off}(i,j,k) \ll 21) \mid (\text{lower::off}(i,j,k) \ll 9) \mid \text{leaf::off}(i,j,k)$ [S1].
6.  Partial radix sort of keys $b$ associated with each run [S1].
7.  Upper node count = number of runs [S1].
8.  Lower node count = number of unique keys $b \gg 21$ [S1].
9.  Leaf node count = number of unique keys $b \gg 9$ [S1].
10. Allocate device memory using counts [S1].
11. Build NanoGrid top-down:
    11.1 Register upper nodes into root table using $a$ [S1].
    11.2 Register lower nodes into parent nodes using $b \gg 21$ [S1].
    11.3 Register leaf nodes into parent nodes using $b \gg 9$ [S1].
    11.4 Register active voxels into leaf `mBitMask` using $b \& 511$ [S1].
12. Optionally add ML features as blind data [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_index_grid(voxel_coords):
    # Step 1: Input signed voxel coordinates (i, j, k)
    # Step 2: Define 64-bit keys a for sorting
    keys_a = []
    for i, j, k in voxel_coords:
        # [G] GPU kernels use unsigned 64-bit arithmetic; masking prevents sign-extension
        a = ((k >> 12) & 0x1FFFFF) | (((j >> 12) & 0x1FFFFF) << 21) | (((i >> 12) & 0x1FFFFF) << 42)
        keys_a.append(a)

    # Step 3: Full radix sort of keys a
    # [G] GPU implementation uses parallel radix sort; Python equivalent:
    sorted_indices = sorted(range(len(keys_a)), key=lambda idx: keys_a[idx])
    sorted_keys_a = [keys_a[idx] for idx in sorted_indices]
    sorted_coords = [voxel_coords[idx] for idx in sorted_indices]

    # Step 4: Run-Length-Encode sorted keys a to identify runs
    runs = []
    if not sorted_keys_a:
        return [], [], [], []
    current_run = [0]
    for idx in range(1, len(sorted_keys_a)):
        if sorted_keys_a[idx] == sorted_keys_a[idx - 1]:
            current_run.append(idx)
        else:
            runs.append(current_run)
            current_run = [idx]
    runs.append(current_run)

    # Step 5: For each run, define keys b
    # b = (M << 36) | (upper_off << 21) | (lower_off << 9) | leaf_off
    # [G] The source defines the key structure but does not specify the exact
    # formula for upper::off, lower::off, leaf::off. They are derived from
    # the hierarchical registration process.
    keys_b = []
    for run_idx, run_indices in enumerate(runs):
        M = run_idx
        for local_idx in run_indices:
            i, j, k = sorted_coords[local_idx]
            # [G] Offset calculation logic is not specified in sources
            upper_off, lower_off, leaf_off = 0, 0, 0
            b = (M << 36) | (upper_off << 21) | (lower_off << 9) | leaf_off
            keys_b.append(b)

    # Step 6: Partial radix sort of keys b associated with each run
    # [G] GPU implementation uses per-run parallel radix sort
    sorted_b_indices = []
    for run_indices in runs:
        run_b_indices = sorted(run_indices, key=lambda idx: keys_b[idx])
        sorted_b_indices.extend(run_b_indices)
    sorted_keys_b = [keys_b[idx] for idx in sorted_b_indices]

    # Step 7: Upper node count = number of runs
    upper_count = len(runs)

    # Step 8: Lower node count = number of unique keys b >> 21
    lower_offsets = [key >> 21 for key in sorted_keys_b]
    lower_count = len(set(lower_offsets))

    # Step 9: Leaf node count = number of unique keys b >> 9
    leaf_offsets = [key >> 9 for key in sorted_keys_b]
    leaf_count = len(set(leaf_offsets))

    # Step 10: Allocate device memory using counts
    # [G] GPU memory allocation primitives required
    upper_nodes = [None] * upper_count
    lower_nodes = [None] * lower_count
    leaf_nodes = [None] * leaf_count
    leaf_bitmasks = [0] * leaf_count  # 512-bit masks simulated as integers

    # Step 11: Build NanoGrid top-down
    # 11.1 Register upper nodes into root table using a
    # [G] GPU atomic operations or sorting-based registration required
    for idx, key in zip(sorted_indices, sorted_keys_a):
        upper_nodes[key & 0xFFFFFFFF] = idx
    # 11.2 Register lower nodes into parent nodes using b >> 21
    # [G] GPU atomic operations or sorting-based registration required
    for idx, key in zip(sorted_b_indices, sorted_keys_b):
        lower_nodes[key >> 21] = idx
    # 11.3 Register leaf nodes into parent nodes using b >> 9
    # [G] GPU atomic operations or sorting-based registration required
    for idx, key in zip(sorted_b_indices, sorted_keys_b):
        leaf_nodes[key >> 9] = idx
    # 11.4 Register active voxels into leaf mBitMask using b & 511
    # [G] GPU atomic operations or sorting-based registration required
    for idx, key in zip(sorted_b_indices, sorted_keys_b):
        leaf_bitmasks[key >> 9] |= (1 << (key & 511))

    # Step 12: Optionally add ML features as blind data
    return upper_nodes, lower_nodes, leaf_nodes, leaf_bitmasks
```

### Parameters and Edge Cases
*   **Shift Amounts**: The shift amounts (12, 21, 9) are fixed constants derived from the 12-bit coarse grid resolution and the 9-bit leaf bitmask resolution ($2^9 = 512$) [S1].
*   **Precision**: The use of 64-bit integers for keys `a` and `b` ensures that all necessary information fits without overflow, provided the number of nodes does not exceed $2^{36}$ for the `M` field [S1].
*   **Boundaries**: The algorithm assumes valid signed voxel coordinates. Negative coordinates must be handled correctly by the shift operations; GPU kernels use logical shifts on unsigned 64-bit types to preserve bit patterns [G].
*   **Degenerate Input**: If the input is empty, the algorithm returns empty structures. If all coordinates are identical, there will be one run, one upper node, etc. [G].

## Operating characteristics
*   **Time Complexity**: Dominated by the sorting steps. Full radix sort is $O(N)$ for fixed-width keys. Partial sorts are proportional to the size of each run [S1].
*   **Memory Complexity**: Proportional to the number of nodes (upper, lower, leaf) and active voxels. The IndexGrid structure is designed for a competitive memory footprint [S1].
*   **Performance**: The method is optimized for GPU execution, leveraging parallel sorting and registration. It enables rapid construction of grids with different topologies, which is crucial for dynamic sparsity patterns [S1].
*   **Strengths**: Supports dynamic sparsity, integrates with deep learning frameworks (PyTorch), and provides fast ray tracing kernels via HDDA [S1].
*   **Limitations**: Requires GPU memory for intermediate sorted arrays. The complexity of the registration step can be non-trivial to implement efficiently without atomic operations or parallel prefix sums [G].

## Background
The method was introduced as part of the **fVDB** framework, a deep-learning framework for sparse, large-scale, and high-performance spatial intelligence [S1]. It is based on the **IndexGrid** data structure, which is derived from **NanoVDB** [Museth 2021] [S1]. The key innovation is the ability to build IndexGrids on the GPU with dynamic topology, addressing the limitations of fixed-topology structures like standard NanoVDB [S1]. The framework is fully integrated with PyTorch and is used for tasks such as large-scale point-cloud segmentation, 3D generative modeling, and Neural Radiance Fields [S1].

## Sources
[S1] fVDB: A Deep-Learning Framework for Sparse, Large-Scale, and High-Performance Spatial Intelligence, https://research.nvidia.com/labs/prl/publication/williams2024fvdb/, fetched 2026-10-30
