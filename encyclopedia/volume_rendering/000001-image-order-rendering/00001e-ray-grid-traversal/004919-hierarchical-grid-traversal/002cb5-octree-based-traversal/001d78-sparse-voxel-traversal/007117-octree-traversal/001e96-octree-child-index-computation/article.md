# Octree Child Index Computation


## Overview
Octree Child Index Computation determines the 0–7 octant offset (and thus the child index) of a specific child node relative to its parent, given the parent's current depth and the target destination coordinates [S1, unofficial]. It solves the problem of efficiently mapping a 3D spatial query to a local child slot in a sparse voxel octree without performing expensive arbitrary bit shifts [S1, unofficial]. Unlike [Point Location](2076-point-location.md), which finds the absolute leaf node containing a point, this method yields only the relative offset of a child within a specific parent node [S1, unofficial]. It is distinct from [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md) by operating on precomputed depth masks rather than geometric bounds, and from [Direction-dependent Morton Order Mapping](2662-direction-dependent-morton-order-mapping.md) by focusing on local indexing rather than global sorting [S1, unofficial].

## How it works
**Inputs:**
- `destination`: The 3D coordinate of the target point or ray end.
- `depth`: The current depth level of the parent node in the octree.
- `max_depth`: The maximum depth of the octree.

**Outputs:**
- `child_index`: An integer in the range [0, 7] representing the octant offset of the child containing the destination relative to the parent [S1, unofficial].

**Data Structures:**
- `child_bit_depth`: A precomputed array of integers where `child_bit_depth[d] = 1 << (max_depth - d - 1)` for each depth `d` [S1, unofficial]. This array replaces arbitrary left-shifts with constant shifts [S1, unofficial].

**Steps:**
1. **Precompute Masks:** Calculate `child_bit_depth[d]` for all relevant depths $d$ [S1, unofficial].
2. **Compute Octant Offset:** Determine the target child octant using binary `&` and integer inequality operations [S1, unofficial].
3. **Mask and Shift:** `&-mask` the precomputed value with the destination coordinates and bit-shift by constants corresponding to the X, Y, and Z components [S1, unofficial].
4. **Return:** Return the scalar value when encountering a leaf; either a scalar leaf in an interior node, or a voxel within a cap node [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_child_index(destination, depth, max_depth, child_bit_depth):
    """
    Computes the 0-7 octant offset of the child containing destination
    relative to the parent at the given depth.
    
    Args:
        destination: tuple (dx, dy, dz) - Target point coordinates
        depth: int - Current octree depth
        max_depth: int - Maximum octree depth
        child_bit_depth: list[int] - Precomputed array where child_bit_depth[d] = 1 << (max_depth - d - 1)
    
    Returns:
        int: Child index (0-7)
    """
    # Step 1: Retrieve the precomputed mask for the current depth
    m = child_bit_depth[depth]
    
    # Step 2 & 3: &-mask destination coordinates and bit-shift by constants
    # The shift amount corresponds to the bit position of the current subdivision level
    k = max_depth - depth - 1
    bit_x = (destination[0] & m) >> k
    bit_y = (destination[1] & m) >> k
    bit_z = (destination[2] & m) >> k
    
    # Step 4: Combine bits to form the 0-7 octant offset
    child_index = bit_x | (bit_y << 1) | (bit_z << 2)
    
    return child_index
```

**Parameters:**
- `max_depth`: Typically determined by the volume resolution and minimum voxel size. Not given in the sources.
- `child_bit_depth`: Precomputed array. Size is `max_depth + 1`.

**Edge Cases and Pitfalls:**
- **Precision:** The algorithm relies on integer coordinate representation and bitwise operations; floating-point coordinates must be quantized to the grid first [G].
- **Boundaries:** The method assumes a regular octree structure aligned to powers of two [S1, unofficial].
- **Leaf Detection:** The method itself only computes the offset; traversal logic must handle leaf termination and payload retrieval (scalar or voxel within a cap node) [S1, unofficial].

## Subtopics

### [Point Location](2076-point-location.md)
Point Location is an accelerated hashing scheme that maps a 3D integer coordinate to a specific leaf node in a canonical sparse voxel octree defined on the interval $[0, 2^{d_{max}}]$. It solves the problem of finding the octree cell containing a given point without traversing parent pointers or performing expensive arbitrary bit shifts. Unlike sibling methods that may rely on Morton codes or spatial partitioning bounds, Point Location uses precomputed depth masks to determine the child octant index (0–7) directly from the destination coordinates using constant bitwise operations. It is distinct from [Octree Child Index Computation](1807-octree-child-index-computation.md) in that it performs the full top-down traversal to the leaf, whereas the sibling method yields only the offset of a child relative to a current node.

### [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md)
Fast Projection for Octree Nodes computes the projected center of an octree node from its parent's projected center using a simple 2D translation. It preprocesses eight constant 2D translation vectors for every octree level to quickly determine subdivisions, affected pixels, and recursively update occlusion maps (HOMs). The same translation approach is used to compute the depth component for fast depth cueing.

### [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md)
Voxel Child Address Calculation determines the storage index of a child voxel within a parent's child array, given a query point and the parent's spatial bounds. It solves the problem of mapping a 3D geometric position to a specific slot in a potentially sparse or compacted child payload array. Unlike [Point Location](2076-point-location.md), which finds the leaf node containing a point, this method operates at the internal node level to identify which of the 8 octants contains the ray intersection or sample point. It differs from [Ordered Child Traversal](3008-ordered-child-traversal.md) by focusing on the arithmetic mapping of coordinates to indices rather than the sequence of traversal.

### [Direction-dependent Morton Order Mapping](2662-direction-dependent-morton-order-mapping.md)
Direction-dependent Morton Order Mapping is a remapping function that converts a standard Octree Morton code into a sorting order aligned with the near-to-far rendering direction of a specific ray. It solves the problem that standard Morton codes provide a fixed spatial traversal order, which does not guarantee correct alpha blending (back-to-front) for arbitrary viewing angles. It sets itself apart from [Ordered Child Traversal](3008-ordered-child-traversal.md) by operating on pre-computed global Morton codes via bitwise permutation rather than dynamic tree traversal, and from [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md) by focusing on global sorting keys rather than local child indexing.

### [Ordered Child Traversal](3008-ordered-child-traversal.md)
Ordered Child Traversal determines the visitation sequence of a node's eight children during ray traversal, ensuring they are processed in front-to-back order relative to the ray origin. It differs from [Point Location](2076-point-location.md) by focusing on traversal order rather than spatial indexing, and from [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md) by explicitly handling the depth-sorting logic required for correct compositing [G].

### Octree Child Bounding Box Computation
Octree Child Bounding Box Computation computes the AABB of a child node given its parent bounds and child index (0-7 for octree, 0-63 for wide) [S2, unofficial]. It uses Morton/Z-order bit packing for child indexing [S2, unofficial]. The child index bit packing maps bit 0 to x, bit 1 to y, and bit 2 to z [S2, unofficial]. This supports standard 8-way octree (indices 0-7) and extends to 4x4x4 wide nodes (indices 0-63) by reusing the same bit logic for the first 3 dimensions [S2, unofficial]. The child min is computed as `(high_x ? mid.x : min.x, high_y ? mid.y : min.y, high_z ? mid.z : min.z)` and child max as `(high_x ? max.x : mid.x, high_y ? max.y : mid.y, high_z ? max.z : mid.z)`, where `mid = (min + max) * 0.5` and `high_*` bits are extracted from `child_index` [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per node visit, assuming precomputed `child_bit_depth` array. The operations involve constant-time bitwise shifts and comparisons [S1, unofficial].
- **Performance:** Precomputing `child_bit_depth` avoids expensive arbitrary left-shifts, resulting in noticeably better performance on hardware where variable-shift instructions are costly [S1, unofficial].
- **Strengths:** Efficient, branch-minimal computation of child indices; suitable for GPU and CPU implementations [S1, unofficial].
- **Limitations:** Requires precomputation of depth masks [S1, unofficial].

## Background
The method is described in the context of octree traversal and neighbor finding for sparse voxel volumes [S1, unofficial]. It addresses the performance bottleneck of arbitrary bit shifts by using precomputed constants [S1, unofficial]. Details are in Appendix B.1 of the source [S1, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S2] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S3] Eine adaptive, hybride Datenstruktur für Volumen Datensätze auf der GPU, https://doi.org/10.34726/hss.2016.26562, fetched 2026-10-26 (unofficial)
