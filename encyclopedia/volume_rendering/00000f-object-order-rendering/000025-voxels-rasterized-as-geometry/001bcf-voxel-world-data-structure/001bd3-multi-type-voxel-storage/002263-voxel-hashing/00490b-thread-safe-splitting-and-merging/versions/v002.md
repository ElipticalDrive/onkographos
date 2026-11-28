# Thread-Safe Splitting and Merging

Also called: Atomic Hash Bucket Maintenance, Parallel Block Refinement

## Overview
This method manages the dynamic refinement and coarsening of voxel blocks within a thread-safe voxel hash table, enabling parallel updates to sparse volumetric representations like TSDFs [S1]. It uses atomic compare-and-swap operations on linked list entries to handle concurrent block allocation and deallocation without locks [S1]. It builds on [Voxel Hashing](2702-voxel-hashing.md) by adding logic for maintaining consistency during topology changes [S1].

## How it works
**Inputs:** A voxel hash table with linked lists for collision resolution, a pool of empty voxel block entries, a pool of empty linked list elements, and a complexity criterion (see Subtopics) determining which blocks to split or merge [S1].
**Outputs:** Updated hash table structure with new child blocks (split) or consolidated parent blocks (merge), and updated TSDF data [S1].

**Data Structures:**
1. Voxel Block Entry: Contains a pointer to the voxel block array (or a flag indicating an empty block) [S1].
2. Linked List Element: Contains a pointer to the next linked list element and a pointer to a voxel block entry [S1].
3. Hash Table: Array of linked list heads [S1].
4. Pools: Free lists for voxel block entries and linked list elements [S1].

**Steps for Splitting:**
1. Identify a voxel block $b$ to split based on the complexity criterion [S1].
2. For each of the 8 new child blocks $b_i$:
   a. Claim a new entry $v$ from the pool of empty voxel blocks [S1].
   b. Compute the hash index $h(b_i)$ for the new block [S1].
   c. Traverse the linked list at $h(b_i)$ to find an invalid pointer [S1].
   d. Use an atomic compare-and-swap (CAS) operation: compare checks if the current entry's voxel block pointer is invalid, and swap overwrites the first invalid pointer with $v$ [S1].
   e. If no invalid pointers are found in the existing list: claim a new linked list element $e$ from the pool, set $e$'s voxel block pointer to $v$, and use CAS to find the current end of the linked list and replace its next pointer with $e$ [S1].
3. Update TSDF data: Perform bilinear interpolation from the parent block to the 8 children [S1]. This step is trivially parallelized [S1].

**Steps for Merging:**
1. Identify a parent block $p$ and its 8 children $c_i$ to merge based on the complexity criterion [S1].
2. Modify the entry of the parent block $p$ to point to a new empty voxel block [S1].
3. For each child $c_i$: replace the pointer to the voxel block array in the child's entry with a flag indicating an empty block [S1]. If the child's entry is at the end of its linked list in a hash bucket, shorten the list accordingly [S1]. Shortening linked lists only at the ends ensures that there are no race conditions [S1].
4. Update TSDF data: Perform subsampling from the 8 children into the new parent block [S1]. This step is trivially parallelized [S1].

**Formulas:**
The complexity measure $c(\mathbf{b})$ for a block $\mathbf{b}$ is the determinant of the covariance matrix of surface normals within the block:
$$c(\mathbf{b}) = \det \left( \sum_{\mathbf{x}} (\nabla F_{\mathbf{b}}(\mathbf{X}) \nabla F_{\mathbf{b}}(\mathbf{X})^{\top}) - \left( \sum_{\mathbf{x}} \nabla F_{\mathbf{b}}(\mathbf{X}) \right) \left( \sum_{\mathbf{x}} \nabla F_{\mathbf{b}}(\mathbf{X}) \right)^{\top} \right)$$
where $F_{\mathbf{b}}$ is the signed distance function stored in block $\mathbf{b}$ and $\nabla$ is the gradient operator [S1].

**Parameters:**
Typical values for the splitting threshold $t_s$ and merging threshold $t_m$ are not given in the sources.

**Pitfalls:**
- Unlike the initial allocation step, no two threads will ever attempt to allocate the same voxel block during splitting, allowing implicit exploitation of this property to avoid contention on the block pool itself [S1].
- Linked list modifications must be restricted to the ends to prevent race conditions during traversal [S1].

## Subtopics

### Complexity Criterion for Splitting and Merging
A heuristic criterion to decide whether to split or merge voxel blocks based on surface roughness, avoiding storage of all input observations required by probabilistic model selection [S1]. The criterion schedules block $\mathbf{b}$ for splitting if $c(\mathbf{b}) > t_s$ (threshold) and it is not on the finest level of the hierarchy [S1]. It schedules block $\mathbf{b}$ for merging if: 1) it is marked as having been split previously, 2) none of its children are marked as being split, and 3) the complexity of each child $b_0$ is below threshold $t_m$ ($c(b_0) < t_m$) [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** Not given in the sources.
- **Strengths:** Thread-safe parallel implementation using atomic operations without locks [S1]. Avoids storing all input observations for decision making [S1].
- **Limitations:** Heuristic may not always produce optimal resolution [S1]. Linked list management adds complexity [S1]. Potential for hash collisions [S1].

## Background
Originated in the context of online 3D reconstruction from depth images, specifically for efficient integration of TSDF volumes [S1]. The method was introduced to handle the dynamic nature of voxel blocks in a parallel environment, avoiding the prohibitive cost of storing all input observations for probabilistic model selection [S1]. It is used in frameworks that require real-time or near-real-time surface extraction and refinement [S1].

## Sources
[S1] Hierarchical Voxel Block Hashing for Efficient Integration of Depth Images, https://doi.org/10.1109/lra.2015.2512958, fetched 2026-11-08
