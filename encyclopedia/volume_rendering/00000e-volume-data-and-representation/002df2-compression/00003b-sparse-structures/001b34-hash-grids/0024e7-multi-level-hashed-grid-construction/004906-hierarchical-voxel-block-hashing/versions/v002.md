# Hierarchical Voxel Block Hashing

Also called: HVBH, Hierarchical hashing data structure

## Overview
Hierarchical Voxel Block Hashing (HVBH) is a sparse volumetric representation that combines the memory efficiency of hash-based block indexing with the adaptability of multi-resolution grids [S1]. Unlike standard voxel block hashing which uses a single resolution, HVBH maintains a fixed hierarchy of $L$ levels, allowing coarse blocks to point to finer sub-blocks only where detail is required [S1]. This approach bypasses the time-consuming tree traversal of octrees while avoiding the memory waste of uniform high-resolution grids, achieving state-of-the-art accuracy with approximately 40% memory savings compared to fixed grids [S1].

## How it works
**Inputs:**
*   Current camera pose $T = (R, t)$ [S1].
*   Depth image $ID$ [S1].
*   Intrinsic matrix $K$ [S1].
*   Truncation band parameter $\mu$ [S1].
*   Base voxel size $s$ at the finest resolution level [S1].
*   Number of hierarchy levels $L$ [S1].
*   Hash table size $H_l$ and coefficients $h_1, h_2, h_3$ for each level $l$ [S1].

**Outputs:**
*   Updated Truncated Signed Distance Function (TSDF) volume stored in the hierarchical hash structure [S1].

**Data Structures:**
1.  **Voxel Block:** A fixed-size data block containing $8 \times 8 \times 8$ voxels [S1].
2.  **Hash Table Entry:** Stored in a hash table bucket. Contains either a pointer to a Voxel Block array (storing actual voxel data) or a specific flag indicating that additional information is stored at a finer level [S1].
3.  **Resolution Levels:** The structure maintains $L$ levels [S1]. At level $l$, voxels represent the TSDF with resolution $2^l s$ [S1]. Level 0 is the coarsest, and level $L-1$ is the finest [S1].

**Algorithm Steps:**

1.  **Access a Voxel:**
    To retrieve or update a voxel at a specific 3D coordinate:
    1.  Determine the block coordinate $\mathbf{b} = (b_x, b_y, b_z)$ at the coarsest resolution level (level 0) that contains the target voxel [S1].
    2.  Compute the hash index $h(\mathbf{b})$ using the formula [S1]:
        $$h(\mathbf{b}) = (h_1 b_x \oplus h_2 b_y \oplus h_3 b_z) \bmod H_l$$
        where $\oplus$ is bitwise XOR, and $H_l$ is the hash table size at level $l$ [S1].
    3.  Check the hash table bucket at index $h(\mathbf{b})$ for an entry matching block $\mathbf{b}$ [S1].
    4.  If no matching entry is found, abort the search; no observations are stored for this location [S1].
    5.  If an entry is found:
        *   If the entry contains a pointer to a voxel block, access the voxel within that block [S1].
        *   If the entry contains the 'finer level' flag, compute the block coordinate $\mathbf{b}'$ at the next finer level (level $l+1$) [S1].
        *   Compute the hash $h(\mathbf{b}')$ for level $l+1$ using the formula with $H_{l+1}$ [S1].
        *   Repeat the lookup at level $l+1$ [S1].

2.  **Integration Allocation:**
    To integrate new geometry from a depth image $ID$ with pose $T$:
    1.  For each pixel $p$ in $ID$, create a 3D line segment $\mathcal{L}$ within the truncation band $\pm \mu$ around the measured depth [S1]. The segment is defined by:
        $$\mathcal{L}: \left[ \mathbf{T}^{-1} \tilde{\mathbf{s}} \left( 1 - \frac{\mu}{\|\mathbf{s}\|} \right), \mathbf{T}^{-1} \tilde{\mathbf{s}} \left( 1 + \frac{\mu}{\|\mathbf{s}\|} \right) \right]$$
        where $\mathbf{s} = ID(p) K^{-1} \tilde{p}$ and $\tilde{\mathbf{s}}, \tilde{p}$ are homogeneous equivalents [S1].
    2.  Identify the set of blocks $B$ at the coarsest level (level 0) that the line segment $\mathcal{L}$ passes through [S1].
    3.  For each block $\mathbf{b} \in B$:
        *   Compute the hash value $h(\mathbf{b})$ for level 0 [S1].
        *   Check the hash table [S1].
        *   If the block is not associated with voxel data (no entry or only a finer-level flag), allocate a new block from a pool and modify the hash table to point to it [S1].
        *   If the hash table indicates information exists at a finer level, proceed to allocate/verify blocks at that finer level recursively [S1].
    4.  Once all required blocks are allocated, integrate the depth information into the TSDF values within the voxels, following the standard weighted sum update procedure [S1].

**Parameters:**
*   $s$: Base voxel size at the finest resolution level [S1].
*   $L$: Fixed number of levels in the resolution hierarchy [S1].
*   $\mu$: Truncation band parameter defining the range $\pm \mu$ around the 3D surface [S1].
*   $h_1, h_2, h_3$: Predefined hash coefficients [S1].
*   $H_l$: Hash table size at level $l$ [S1].

**Edge Cases and Pitfalls:**
*   **Hash Collisions:** The hash function uses a modulo operation, and entries falling into the same bucket are stored in a linked list [S1]. Implementers must traverse the linked list to find the exact block coordinate [S1].
*   **Missing Data:** If a lookup fails at any level, the search aborts immediately, assuming no data exists [S1]. This requires ensuring all necessary blocks are allocated along the integration ray before TSDF updates [S1].
*   **Precision:** The source specifies that $h_1, h_2, h_3$ are predefined hash coefficients but does not provide specific numerical values [S1].

## Operating characteristics
*   **Time Complexity:** Access and manipulation functions are highly parallelizable and provide efficient access [S1]. The method avoids the time-consuming tree traversal required by octrees [S1].
*   **Memory Complexity:** Memory usage is proportional to the number of non-empty blocks across all levels [S1]. The structure achieves approximately 40% memory savings compared to a fixed-resolution grid [S1].
*   **Performance:** The method demonstrated state-of-the-art representation accuracy at typical framerates around 100 Hz on a Titan X GPU [S1].
*   **Strengths:** Efficient access without tree traversal; adaptive resolution reduces memory waste in flat or simple areas while preserving detail in complex regions [S1].
*   **Limitations:** The fixed number of levels $L$ must be chosen appropriately for the scene scale and detail requirements [S1].

## Background
Hierarchical Voxel Block Hashing was introduced in a paper by the authors of the cited source [S1]. It builds upon ideas from voxel block hashing [11, 12] and tree-based structures [8, 9, 10] [S1]. The method was embedded in a system for simultaneous localization and mapping (SLAM) from RGB-D imagery, specifically extending the KinectFusion approach [1, 2] [S1]. The work was presented in the IEEE Robotics and Automation Letters (LRA) [S1].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
