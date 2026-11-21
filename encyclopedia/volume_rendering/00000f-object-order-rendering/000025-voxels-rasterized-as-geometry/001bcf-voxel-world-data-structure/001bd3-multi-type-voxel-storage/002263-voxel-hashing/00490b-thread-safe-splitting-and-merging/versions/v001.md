# Thread-Safe Splitting and Merging

Also called: Atomic Hash Bucket Maintenance, Parallel Block Refinement

## Overview
This method manages the dynamic refinement and coarsening of voxel blocks within a thread-safe voxel hash table, enabling parallel updates to sparse volumetric representations like TSDFs. It differs from non-parallel approaches by using atomic compare-and-swap operations on linked list entries to handle concurrent block allocation and deallocation without locks, and from purely static hashing by supporting runtime structural changes. It builds on [Voxel Hashing](2702-voxel-hashing.md) by adding the logic for maintaining consistency during topology changes.

## How it works
**Inputs:** A voxel hash table with linked lists for collision resolution, a pool of empty voxel block entries, a pool of empty linked list elements, and a complexity criterion (see Subtopics) determining which blocks to split or merge.
**Outputs:** Updated hash table structure with new child blocks (split) or consolidated parent blocks (merge), and updated TSDF data.

**Data Structures:**
1.  **Voxel Block Entry:** Contains a pointer to the voxel block array (or a flag indicating an empty block).
2.  **Linked List Element:** Contains a pointer to the next linked list element and a pointer to a voxel block entry.
3.  **Hash Table:** Array of linked list heads.
4.  **Pools:** Free lists for voxel block entries and linked list elements.

**Steps for Splitting:**
1.  Identify a voxel block $b$ to split based on the complexity criterion.
2.  For each of the 8 new child blocks $b_i$:
    a. Claim a new entry $v$ from the pool of empty voxel blocks [G: ensure $v$ is marked as used].
    b. Compute the hash index $h(b_i)$ for the new block.
    c. Traverse the linked list at $h(b_i)$ to find an invalid pointer.
    d. Use an atomic compare-and-swap (CAS) operation:
       - Compare: Check if the current entry's voxel block pointer is invalid (e.g., null or a specific sentinel).
       - Swap: If valid, overwrite the invalid pointer with $v$.
    e. If no invalid pointers are found in the existing list:
       i. Claim a new linked list element $e$ from the pool.
       ii. Set $e$'s voxel block pointer to $v$.
       iii. Use CAS to find the current end of the linked list and replace its next pointer with $e$.
3.  Update TSDF data: Perform bilinear interpolation from the parent block to the 8 children. This step is trivially parallelized across the children [S1].

**Steps for Merging:**
1.  Identify a parent block $p$ and its 8 children $c_i$ to merge based on the complexity criterion.
2.  Modify the entry of the parent block $p$ to point to a new empty voxel block (or reuse the parent's block if it remains valid, but the source specifies pointing to a *new* empty block to facilitate the merge logic) [S1].
3.  For each child $c_i$:
    a. Replace the pointer to the voxel block array in the child's entry with a flag indicating an empty block.
    b. If the child's entry is at the end of its linked list in a hash bucket, shorten the list by updating the previous element's next pointer to null (or the list head if it was the only element). This shortening only at the ends ensures no race conditions [S1].
4.  Update TSDF data: Perform subsampling from the 8 children into the new parent block. This step is trivially parallelized [S1].

**Pitfalls:**
-   Unlike the initial allocation step, no two threads will ever attempt to allocate the same voxel block during splitting, allowing implicit exploitation of this property to avoid contention on the block pool itself [S1].
-   Linked list modifications must be restricted to the ends to prevent race conditions during traversal [S1].

## Subtopics

### Complexity Criterion for Splitting and Merging
A heuristic criterion to decide whether to split or merge voxel blocks based on surface roughness, avoiding storage of all input observations required by probabilistic model selection [S1]. The complexity measure $c(\mathbf{b})$ for a block $\mathbf{b}$ is the determinant of the covariance matrix of surface normals within the block:
$$c(\mathbf{b}) = \det \left( \sum_{\mathbf{x}} (\nabla F_{\mathbf{b}}(\mathbf{X}) \nabla F_{\mathbf{b}}(\mathbf{X})^{\top}) - \left( \sum_{\mathbf{x}} \nabla F_{\mathbf{b}}(\mathbf{X}) \right) \left( \sum_{\mathbf{x}} \nabla F_{\mathbf{b}}(\mathbf{X}) \right)^{\top} \right)$$
where $F_{\mathbf{b}}$ is the signed distance function stored in block $\mathbf{b}$ and $\nabla$ is the gradient operator [S1].
-   **Split:** Schedule block $\mathbf{b}$ for splitting if $c(\mathbf{b}) > t_s$ (threshold) and it is not on the finest level of the hierarchy [S1].
-   **Merge:** Schedule block $\mathbf{b}$ for merging if: 1) it is marked as having been split previously, 2) none of its children are marked as being split, and 3) the complexity of each child $b_0$ is below threshold $t_m$ ($c(b_0) < t_m$) [S1].

## Operating characteristics
-   **Time Complexity:** Splitting and merging operations are $O(1)$ amortized per block if the hash table load factor is controlled, but linked list traversal can degrade to $O(N)$ in worst-case hash collisions. The atomic operations add constant overhead.
-   **Memory Complexity:** Requires additional memory for linked list elements and pools. The complexity criterion requires storing surface normal statistics per block.
-   **Performance:** The method enables parallel processing of block updates, significantly speeding up dynamic scene reconstruction compared to sequential approaches. The use of atomic operations ensures thread safety without locks.
-   **Strengths:** Thread-safe, dynamic resolution, no need to store raw observations for decision making.
-   **Limitations:** Heuristic may not always produce optimal resolution; linked list management adds complexity; potential for hash collisions.

## Background
Originated in the context of online 3D reconstruction from depth images, specifically for efficient integration of TSDF volumes. The method was introduced to handle the dynamic nature of voxel blocks in a parallel environment, avoiding the prohibitive cost of storing all input observations for probabilistic model selection. It is used in frameworks that require real-time or near-real-time surface extraction and refinement.

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
