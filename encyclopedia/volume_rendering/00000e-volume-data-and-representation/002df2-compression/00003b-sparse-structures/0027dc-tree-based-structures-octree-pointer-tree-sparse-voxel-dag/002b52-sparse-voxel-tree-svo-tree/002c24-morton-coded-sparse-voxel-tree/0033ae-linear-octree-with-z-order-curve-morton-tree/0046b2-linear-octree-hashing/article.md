# Linear Octree Hashing

Also called: Gargantini's linear octree, Sorted leaf array lookup

## Overview
Linear Octree Hashing is a pointerless sparse octree storage method where only leaf nodes are stored in a contiguous memory array, sorted by their Morton (Z-order) code to enable binary search [S1, unofficial]. It solves the problem of memory overhead in traditional octrees by eliminating interior nodes and pointers, trading lookup speed for significant storage reduction [S1, unofficial]. Unlike its parent [Linear Octree with Z-order curve (Morton tree)](7054-linear-octree-with-z-order-curve-morton-tree.md), which may retain structural metadata or use different traversal mechanisms, this variant relies strictly on the sorted leaf array for all spatial queries [S1, unofficial].

## How it works
**Inputs:** A set of 3D voxel coordinates $(x, y, z)$ representing occupied leaf nodes within a bounded cubic domain of depth $D$.
**Outputs:** A sorted array of leaf nodes, where each entry contains the Morton code and associated voxel data.

**Data Structures:**
The core structure is a linear array `Leaves`. Each element `Leaves[i]` contains:
- `morton_code`: An integer representing the Z-order curve index of the leaf node.
- `data`: The voxel value or metadata associated with the leaf.

**Steps:**
1. **Morton Encoding:** For each occupied leaf node at coordinates $(x, y, z)$, compute its Morton code $M$. The Morton code interleaves the bits of the coordinates. For a coordinate system with depth $D$, the code is calculated as:
   $$ M = \sum_{k=0}^{D-1} (x_k 2^{2k} + y_k 2^{2k+1} + z_k 2^{2k+2}) $$
   where $x_k, y_k, z_k$ are the $k$-th bits of $x, y, z$ respectively [S1, unofficial].
2. **Collection:** Store pairs $(M, \text{data})$ for all leaf nodes in a temporary list.
3. **Sorting:** Sort the list of pairs based on the `morton_code` in ascending order. This step ensures that spatially adjacent nodes are stored contiguously in memory [S1, unofficial].
4. **Storage:** Write the sorted list into the contiguous `Leaves` array.

**Point Location (Query):**
To find the node containing a query point $P(x_q, y_q, z_q)$:
1. Compute the Morton code $M_q$ for the query point [S1, unofficial].
2. Perform a binary search on the `Leaves` array for $M_q$.
3. If $M_q$ is found, return the associated data. If not, the point lies in an empty region (or requires further interpolation/empty-space skipping logic not defined in the basic structure) [S1, unofficial].

**Parameters:**
- $D$: Depth of the octree. Typical values depend on the volume resolution (e.g., 10–16 for high-res medical data) [S1, unofficial].
- $L$: Number of leaf nodes.

**Edge Cases and Pitfalls:**
- **Empty Regions:** The structure does not store empty interior nodes. Queries for points in empty space will result in a "not found" during binary search [S1, unofficial].
- **Neighbor Finding:** Finding spatial neighbors (e.g., for filtering or gradient calculation) is difficult because neighbors may not be adjacent in the sorted array. Additional data structures or complex geometric calculations are required [S1, unofficial].
- **Precision:** Morton codes can grow large for deep trees. Ensure integer types are large enough to hold $3D$ bits (e.g., 64-bit integers for $D \le 21$) [S1, unofficial].

## Subtopics

### [Linear Level Octree (LLO)](16467-linear-level-octree-llo.md)
Extends the linear octree by using a deterministic coordinate-based key system for direct geometric calculations, avoiding hashing overhead and facilitating spatial relation estimation.

## Operating characteristics
- **Time Complexity:** Node lookup is $O(\log_2(L))$, where $L$ is the number of leaf nodes [S1, unofficial]. This is slower than pointer octrees ($O(\log_8(N))$) for general cases [S1, unofficial].
- **Memory Complexity:** $O(L)$, storing only leaf nodes. No memory is wasted on interior nodes or pointers [S1, unofficial].
- **Performance:** Generally slower to hash than pointer octrees, except in highly vertical trees with few leaf nodes relative to interior nodes [S1, unofficial].
- **Strengths:** Minimal memory footprint; suitable for applications where storage is critical and only leaf data is needed [S1, unofficial].
- **Limitations:** Unsuitable for applications requiring traversal of non-leaf structures or efficient neighbor finding without additional data [S1, unofficial].

## Background
The linear octree was originally proposed by Gargantini for quadtrees [Gar82] and later extended to octrees using Morton's interleaved base-8 code [Mor66] [S1, unofficial]. It is a pointerless structure designed to minimize storage overhead by eliminating interior nodes and pointers [S1, unofficial]. While not directly applied to volume rendering in the cited survey, it is noted for its potential in compression-based adaptive octree methods [S1, unofficial].

## Sources
[S1] A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
