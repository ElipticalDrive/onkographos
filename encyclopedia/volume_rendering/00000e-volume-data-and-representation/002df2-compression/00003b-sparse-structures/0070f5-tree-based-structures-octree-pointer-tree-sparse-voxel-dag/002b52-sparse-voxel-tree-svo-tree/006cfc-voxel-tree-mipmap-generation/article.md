# Voxel Tree Mipmap Generation


## Overview
Voxel Tree Mipmap Generation constructs a hierarchical representation of a Sparse Voxel Octree (SVO) by downsampling occupancy data from one level to the next, enabling efficient ray marching and culling at varying distances [S1, unofficial]. It differs from surface-based simplification methods like [Least Squares Surface Simplification](11029-least-squares-surface-simplification.md) by operating strictly on volumetric occupancy bits rather than geometric meshes, and from [GigaVoxels Data Synthesis](16702-gigavoxels-data-synthesis.md) by typically generating the entire hierarchy in memory rather than streaming it [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A source SVO tree structure containing occupancy bits and optional color data.
*   **Output:** A target SVO tree structure with dimensions halved in each axis, containing the downsampled occupancy and color data.

**Data Structures:**
The source and target trees use a bit-packed array for occupancy. The exact layout depends on the implementation, but typically involves a linear array where bits represent voxel existence. Color data may be stored in separate arrays (`voxel_col1`, `voxel_col2`) or interleaved. The source tree dimensions are $(S_x, S_y, S_z)$. The target tree dimensions are $(T_x, T_y, T_z) = (S_x/2, S_y/2, S_z/2)$.

**Algorithm Steps:**
1.  **Initialization:** Create the target tree with dimensions $T_x, T_y, T_z$.
2.  **Mesh Check:** If the source tree contains explicit mesh faces (indicated by `faces_arr.size() > 0`), render them immediately using a specific draw function (e.g., `drawPLY(i)`) and terminate the mipmap generation process [S1, unofficial].
3.  **Iteration:** Loop through each voxel $(i, j, k)$ in the target tree, where $0 \le i < T_x$, $0 \le j < T_y$, $0 \le k < T_z$.
4.  **Sub-block Mapping:** Calculate the starting coordinates of the corresponding $2 \times 2 \times 2$ sub-block in the source tree:
    $$ ii = i \times 2, \quad jj = j \times 2, \quad kk = k \times 2 $$
5.  **Occupancy Scan:** Initialize a counter `mincnt = 3`. Iterate through the 8 voxels in the $2 \times 2 \times 2$ sub-block using offsets $subx, suby, subz \in \{0, 1\}$.
    *   Compute absolute source coordinates:
        $$ src\_x = ii + subx, \quad src\_y = jj + suby, \quad src\_z = kk + subz $$
    *   Check occupancy in the source tree. The bit index is calculated as:
        $$ bit\_index = (src\_x + src\_y \times S_x + src\_z \times 4 \times S_x \times S_y) \gg 3 $$
        $$ bit\_mask = 1 \ll (src\_x \& 7) $$
        $$ is\_occupied = (source\_tree.voxel[bit\_index] \& bit\_mask) \neq 0 $$
    *   If `is_occupied` is true, decrement `mincnt`.
    *   If `mincnt` reaches 0, set a flag `bit0 = 1`. If color data exists, copy the relevant color bits (e.g., `bit1`, `bit2`) from the source to the target. Break the inner loop.
6.  **Target Update:** If `bit0` is true:
    *   Set the occupancy bit in the target tree at $(i, j, k)$:
        $$ target\_index = (i + j \times T_x + k \times 4 \times T_x \times T_y) \gg 3 $$
        $$ target\_mask = 1 \ll (i \& 7) $$
        $$ target\_tree.voxel[target\_index] \ |= \ target\_mask $$
    *   If color channels exist, copy the stored color bits to `target_tree.voxel_col1` and `target_tree.voxel_col2` using the same bit-packing logic as the occupancy bits.
7.  **Continue:** If `bit0` is false, the target voxel remains empty; proceed to the next voxel.

**Parameters:**
*   **Threshold:** The algorithm uses a threshold of 3 occupied voxels out of 8 to determine if the parent voxel is occupied (`mincnt` starts at 3, so 3 or more occupied children trigger the parent). This is a specific implementation detail from the source [S1, unofficial].
*   **Color Copy:** Color data is copied only if the parent voxel is determined to be occupied. The specific bits copied (`bit1`, `bit2`) are implementation-dependent [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Mesh Faces:** If the tree contains mesh faces, the mipmap generation is bypassed in favor of direct rendering [S1, unofficial].
*   **Precision:** Bit-shifting operations must handle large indices correctly to avoid overflow in the bit index calculation.
*   **Boundaries:** The source tree dimensions must be even to allow perfect $2 \times 2 \times 2$ subdivision. If not, padding or truncation is required (not specified in sources, marked as [G]).
*   **Color Data:** If color data is not present in the source tree, the color copying steps are skipped [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_mipmap(source_tree, target_tree):
    # Initialize target tree dimensions
    target_tree.vx = source_tree.vx // 2
    target_tree.vy = source_tree.vy // 2
    target_tree.vz = source_tree.vz // 2
    
    # Check for mesh faces
    if source_tree.faces_arr.size() > 0:
        for i in range(source_tree.faces_arr.size()):
            drawPLY(i)
        return

    sx = source_tree.vx
    sy = source_tree.vy
    sz = source_tree.vz
    
    # Iterate over target voxels
    for i in range(target_tree.vx):
        for j in range(target_tree.vy):
            for k in range(target_tree.vz):
                ii = i * 2
                jj = j * 2
                kk = k * 2
                mincnt = 3
                bit0 = 0
                color_bits = None
                
                # Scan 2x2x2 sub-block
                for subx in range(2):
                    for suby in range(2):
                        for subz in range(2):
                            src_x = ii + subx
                            src_y = jj + suby
                            src_z = kk + subz
                            
                            # Calculate bit index
                            bit_index = (src_x + src_y * sx * 2 + src_z * 4 * sx * sy) >> 3
                            bit_mask = 1 << (src_x & 7)
                            
                            # Check occupancy
                            if (source_tree.voxel[bit_index] & bit_mask):
                                mincnt -= 1
                                if mincnt == 0:
                                    bit0 = 1
                                    # Copy color bits if available
                                    if source_tree.voxel_col1:
                                        # Implementation-specific color bit extraction
                                        color_bits = extract_color_bits(source_tree, src_x, src_y, src_z)
                                    break
                        if bit0: break
                    if bit0: break
                
                # Set target voxel if occupied
                if bit0:
                    target_index = (i + j * sx + k * 4 * sx * sy) >> 3
                    target_mask = 1 << (i & 7)
                    target_tree.voxel[target_index] |= target_mask
                    
                    # Copy color bits
                    if color_bits:
                        # Implementation-specific color bit packing
                        pack_color_bits(target_tree, i, j, k, color_bits)
```

## Subtopics

### Large Sparse Volume (SV Mode)
A rendering mode that increases the maximum volume dimensions to 2048x2048x1024 while keeping the total count of solid voxels limited, enabling efficient rendering of sparse large scenes [S2]. It works exclusively with cubic voxels [S2]. The total solid voxel count remains strictly limited regardless of volume dimensions, preventing memory overflow but capping scene density [S2].

## Operating characteristics
*   **Time Complexity:** $O(N)$, where $N$ is the number of voxels in the target tree (or $O(N_{source}/8)$). Each source voxel is visited a constant number of times.
*   **Memory Complexity:** $O(N_{target})$ for the new tree structure.
*   **Performance:** The algorithm is efficient for generating lower-resolution levels of an SVO. The specific implementation in [S1, unofficial] is optimized for bit-packing.
*   **Strengths:** Simple logic, low memory overhead for the hierarchy, enables fast culling.
*   **Limitations:** The threshold-based downsampling (3/8) may introduce artifacts or lose detail compared to other methods. The mesh face bypass is a specific edge case [S1, unofficial].
*   **When to use:** When building an SVO hierarchy for ray marching where lower levels are needed for distant rendering.
*   **When not to use:** When high-fidelity surface representation is required at lower levels (consider [Least Squares Surface Simplification](11029-least-squares-surface-simplification.md)).

## Background
The method is described in the context of a Sparse Voxel Octree Raycasting implementation [S1, unofficial]. The Large Sparse Volume mode is associated with MagicaVoxel [S2].

## Sources
[S1] sp4cerat/Sparse-Voxel-Octree-Raycasting, https://github.com/sp4cerat/Sparse-Voxel-Octree-Raycasting, fetched 2026-10-24 (unofficial)
[S2] MagicaVoxel .vox, https://ephtracy.github.io/, fetched 2026-10-26
