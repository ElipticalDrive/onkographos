# Flood Fill Voxelization
Also called: Flood Fill, Seed Fill

## Overview
Flood Fill Voxelization is a GPU-based algorithm that completes an incomplete binary voxelization by identifying all "outer" voxels (voxels whose centers lie outside the target geometry) that are not visible from infinity along the six orthogonal axes [S1]. It builds on [GPU and Parallel Acceleration Voxelization](6878-gpu-and-parallel-acceleration-voxelization.md) by using a parallel seed-fill approach to resolve hidden surfaces and interior primitives that orthogonal projection methods miss [S1]. Unlike [Per-Pixel Linked List Voxelization](1187-per-pixel-linked-list-voxelization.md) which relies on depth sorting, or [MSAA Voxelization](7412-msaa-voxelization.md) which uses supersampling, this method explicitly propagates a frontier of known outer voxels to fill the volume [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** $T_o$, a 3D texture representing outer voxels identified by orthogonal fill (binary: 1 if outer, 0 otherwise); $T_b$, a 3D texture representing boundary voxels from primitive voxelization (binary: 1 if part of a surface primitive, 0 otherwise).
*   **Output:** $T_o$, updated to include all outer voxels found by the flood fill.
*   **Data Structures:**
    *   $T_o$: 3D texture (volume), bit-packed or single-channel byte, size $(N_x, N_y, N_z)$.
    *   $T_b$: 3D texture, same layout as $T_o$.
    *   $T_{new}$: 3D texture, temporary storage for voxels identified as outer in the current iteration.
    *   $T_{mask}$: 3D texture, temporary storage for candidate voxels to check in the next iteration.
    *   All textures use 1-bit or 8-bit per voxel representation. The algorithm assumes a 6-connected neighborhood (face-sharing neighbors).

**Definitions:**
*   A voxel $(i, j, k)$ is an **outer voxel** if its center is outside the Minkowski sum of the input geometry [S1].
*   A voxel is a **boundary voxel** if it is marked in $T_b$ [S1].
*   Neighbors are defined as voxels sharing a face (max 6 neighbors: $\pm x, \pm y, \pm z$) [S1].

**Steps:**

1.  **Preprocessing (Barrier Removal):**
    Voxels marked as 1 in both $T_o$ and $T_b$ represent boundary voxels that were also identified as outer by orthogonal fill. These must be reset to 0 in $T_o$ to prevent the flood fill from incorrectly penetrating into the object's interior through the boundary surface [S1].
    $$ T_o(i, j, k) \leftarrow T_o(i, j, k) \land \neg T_b(i, j, k) $$
    [G] In GPU implementation, this is typically done via a fragment shader performing a logical AND with the inverse of $T_b$.

2.  **Initialization:**
    Create temporary textures $T_{new}$ and $T_{mask}$, initialized to 0 [S1].
    Populate $T_{mask}$ with the neighbors of all currently identified outer voxels in $T_o$. If a voxel $(i, j, k)$ has at least one neighbor with value 1 in $T_o$, set $T_{mask}(i, j, k) = 1$ [S1].

3.  **Iterative Flood Fill:**
    Repeat the following loop until no new voxels are found:
    a.  **Clear $T_{new}$:** Set all voxels in $T_{new}$ to 0 [S1].
    b.  **Expand Frontier:** For every voxel $(i, j, k)$ where $T_{mask}(i, j, k) = 1$:
        *   If $T_o(i, j, k) = 0$ AND $T_b(i, j, k) = 0$:
            *   Set $T_o(i, j, k) = 1$ (mark as outer).
            *   Set $T_{new}(i, j, k) = 1$ (record for next mask generation).
    c.  **Clear $T_{mask}$:** Set all voxels in $T_{mask}$ to 0 [S1].
    d.  **Generate Next Mask:** For every voxel $(i, j, k)$:
        *   If at least one neighbor has value 1 in $T_{new}$, set $T_{mask}(i, j, k) = 1$ [S1].
    e.  **Termination Check:** Use an occlusion query (or atomic counter reduction) to count the number of set bits in $T_{new}$. If the count is 0, terminate [S1].

4.  **Cleanup:**
    Delete $T_{new}$ and $T_{mask}$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def flood_fill(T_o, T_b, dimensions):
    # Step 1: Preprocessing - Remove boundary voxels from outer set
    # T_o[i,j,k] = T_o[i,j,k] AND NOT T_b[i,j,k]
    for i, j, k in all_voxels(dimensions):
        if T_o[i, j, k] == 1 and T_b[i, j, k] == 1:
            T_o[i, j, k] = 0

    # Step 2: Initialization
    T_new = zeros(dimensions)
    T_mask = zeros(dimensions)
    
    # Populate mask with neighbors of current outer voxels
    for i, j, k in all_voxels(dimensions):
        if has_neighbor_with_value(T_o, i, j, k, 1):
            T_mask[i, j, k] = 1

    # Step 3: Iterative Loop
    while True:
        # Clear T_new
        T_new = zeros(dimensions)
        
        # Expand: Check voxels in mask
        for i, j, k in all_voxels(dimensions):
            if T_mask[i, j, k] == 1:
                if T_o[i, j, k] == 0 and T_b[i, j, k] == 0:
                    T_o[i, j, k] = 1
                    T_new[i, j, k] = 1
        
        # Clear T_mask
        T_mask = zeros(dimensions)
        
        # Generate next mask from neighbors of newly found voxels
        for i, j, k in all_voxels(dimensions):
            if has_neighbor_with_value(T_new, i, j, k, 1):
                T_mask[i, j, k] = 1
        
        # Termination check
        # Count set bits in T_new using occlusion query or reduction
        new_count = count_set_bits(T_new)
        if new_count == 0:
            break

    # Step 4: Cleanup
    # T_new and T_mask are discarded
    return T_o

def has_neighbor_with_value(texture, i, j, k, val):
    # Check 6 neighbors
    neighbors = [
        (i+1, j, k), (i-1, j, k),
        (i, j+1, k), (i, j-1, k),
        (i, j, k+1), (i, j, k-1)
    ]
    for ni, nj, nk in neighbors:
        if is_valid_coord(ni, nj, nk) and texture[ni, nj, nk] == val:
            return True
    return False

def is_valid_coord(i, j, k):
    # Check bounds against dimensions
    return 0 <= i < Nx and 0 <= j < Ny and 0 <= k < Nz
```

**Parameters:**
*   **Resolution:** $(N_x, N_y, N_z)$ determined by the bounding box and desired voxel size [S1].
*   **Connectivity:** 6-connected (face-sharing) [S1].

**Edge Cases and Pitfalls:**
*   **Hidden Surfaces:** If a large portion of the boundary is invisible from all orthogonal directions, the initial seed set from orthogonal fill is small, causing the flood fill to traverse a larger volume and take more time [S1].
*   **Interior Penetration:** Failing to reset voxels present in both $T_o$ and $T_b$ causes the flood fill to incorrectly penetrate into the object's interior [S1].
*   **Precision:** The method relies on binary (0/1) voxelization. Non-binary values are not supported by this specific algorithm [S1].

## Subtopics

### GPU-based Voxelization via Orthogonal Fill and Flood Fill
This is the complete pipeline described in the source, where Flood Fill is the final stage [S1]. It generates a volumetric representation of the Minkowski sum using binary voxelization [S1]. It avoids parity checks (which fail with interior primitives) by using flood fill to find outer voxels [S1]. It differs from Karabassi et al. [S1] by handling hidden surfaces (Karabassi requires complete visibility from six directions) and from Fang and Chen [S1] by projecting along three directions and encoding multiple voxels per texel rather than rendering slice-by-slice, which saves memory and handles parallel surfaces [S1].

## Operating characteristics
*   **Time Complexity:** Dominated by the number of iterations required to fill the volume. Performance degrades if the boundary is largely invisible from orthogonal directions, as the flood fill frontier must traverse more voxels [S1].
*   **Memory Complexity:** Requires storage for $T_o$, $T_b$, $T_{new}$, and $T_{mask}$, totaling four 3D textures of resolution $(N_x, N_y, N_z)$ [S1].
*   **Measured Performance:** On a Quadro FX 5800 GPU, the method can compute Minkowski sums for complex models (tens/hundreds of thousands of triangles) within one minute [S1]. The CUDA implementation provides a 25–30x speedup over CPU implementation [S1].
*   **Strengths:** Runs completely on the GPU, avoiding expensive CPU-GPU readbacks [S1]. Handles hidden surfaces and interior primitives that break parity-based methods [S1].
*   **Limitations:** Memory intensive due to multiple 3D textures [S1]. Performance is sensitive to the visibility of the boundary from orthogonal axes [S1].

## Background
The method was proposed by Dong et al. in 2009 [S1]. It was developed to solve problems in 3D Minkowski sum computation where previous methods like Karabassi et al. [16] and Fang and Chen [7] failed due to hidden surfaces or high memory costs [S1]. The core insight is that all outer voxels are connected, and any neighbor of an outer voxel is either another outer voxel or a boundary voxel [S1].

## Sources
[S1] A GPU-based voxelization approach to 3D Minkowski sum computation, https://doi.org/10.1145/1839778.1839783, fetched 2026-11-05
