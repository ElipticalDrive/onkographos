# Octree Child Index Computation

Also called: Octant Offset Computation, Child Index Calculation

## Overview
Octree Child Index Computation determines the 0–7 octant offset (and thus the child index) of a specific child node relative to its parent, given the parent's current depth and a target destination coordinate [S1, unofficial]. It solves the problem of efficiently mapping a 3D spatial query to a specific child slot in a sparse voxel octree without performing expensive arbitrary bit shifts or full top-down traversals [S1, unofficial]. Unlike [Point Location](2076-point-location.md), which finds the leaf node containing a point, this method yields only the offset of a child relative to a current node [S1, unofficial]. It is distinct from [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md) in that it operates on depth-masked coordinates rather than raw geometric bounds [S1, unofficial].

## How it works
**Inputs:**
- `depth`: The current depth of the parent node in the octree (integer).
- `max_depth`: The maximum depth of the octree (integer).
- `origin`: The 3D ray origin or current node position (integer coordinates).
- `destination`: The 3D target coordinate defining the child's location (integer coordinates).

**Outputs:**
- `child_index`: An integer in the range [0, 7] representing the octant offset of the child relative to the parent.

**Data Structures:**
- `child_bit_depth`: A precomputed array of integers where `child_bit_depth[d]` stores the bit mask for depth `d`.
  - Layout: `child_bit_depth[d] = 1 << (max_depth - d - 1)` for each depth `d` from 0 to `max_depth`.
  - Size: `max_depth + 1` integers.

**Steps:**
1. **Precompute Masks:** Calculate `child_bit_depth[d] = 1 << (max_depth - depth - 1)` for the current `depth` [S1, unofficial].
2. **Mask Coordinates:** Apply the mask to both `origin` and `destination` coordinates. This effectively truncates the lower bits of the coordinates to align with the current node's resolution [S1, unofficial].
   - `masked_origin = origin & child_bit_depth[depth]`
   - `masked_destination = destination & child_bit_depth[depth]`
3. **Compute Octant:** Determine the target child octant using binary AND and integer inequality operations on the masked coordinates [S1, unofficial].
   - The octant index is derived by comparing the bits of `masked_destination` against `masked_origin` (or zero, if origin is the node base) [S1, unofficial].
   - Specifically, the method uses the sign of the difference or inequality checks to determine the X, Y, and Z components of the offset [S1, unofficial].
4. **Bit Packing:** Combine the X, Y, and Z components into a single scalar index.
   - `index = (x_component << 2) | (y_component << 1) | z_component`
   - Where each component is 0 or 1, derived from the inequality checks [S1, unofficial].
5. **Return:** Return `index` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_child_index(depth, max_depth, origin, destination):
    # Precompute the bit depth mask for the current depth
    # child_bit_depth[d] = 1 << (max_depth - d - 1)
    mask = 1 << (max_depth - depth - 1)
    
    # Apply mask to origin and destination
    # This aligns coordinates to the current node's grid
    masked_origin = origin & mask
    masked_destination = destination & mask
    
    # Compute the target child octant
    # Using binary & and integer inequality operations
    # Determine the direction (0 or 1) for each axis
    # If destination bit is greater than origin bit, direction is 1, else 0
    # Note: The source describes using inequality operations on masked values
    x_dir = 1 if (masked_destination.x > masked_origin.x) else 0
    y_dir = 1 if (masked_destination.y > masked_origin.y) else 0
    z_dir = 1 if (masked_destination.z > masked_origin.z) else 0
    
    # Combine into octant index (0-7)
    # Standard Morton-like packing: X is MSB, Z is LSB
    child_index = (x_dir << 2) | (y_dir << 1) | z_dir
    
    return child_index
```

**Parameters:**
- `max_depth`: Typically determined by the volume resolution (e.g., 10 for 1024^3 volume).
- `depth`: Current traversal depth, 0 to `max_depth`.

**Edge Cases and Pitfalls:**
- **Precision:** Integer coordinates must be used. Floating-point coordinates must be converted to integer grid coordinates before masking [S1, unofficial].
- **Boundaries:** If `destination` is outside the parent's bounds, the computed index may be invalid. The caller must ensure `destination` is within the parent's spatial extent [S1, unofficial].
- **Leaf Nodes:** When encountering a leaf, the method returns the scalar value (either a scalar leaf in an interior node or a voxel within a cap node) [S1, unofficial].

## Subtopics

### [Point Location](2076-point-location.md)
Point Location is an accelerated hashing scheme that maps a 3D integer coordinate to a specific leaf node in a canonical sparse voxel octree defined on the interval $[0, 2^{d_{max}}]$. It solves the problem of finding the octree cell containing a given point without traversing parent pointers or performing expensive arbitrary bit shifts. Unlike sibling methods that may rely on Morton codes or spatial partitioning bounds, Point Location uses precomputed depth masks to determine the child octant index (0–7) directly from the destination coordinates using constant bitwise operations. It is distinct from Octree Child Index Computation in that it performs the full top-down traversal to the leaf, whereas the sibling method yields only the offset of a child relative to a current node.

### [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md)
Fast Projection for Octree Nodes computes the projected center of an octree node from its parent's projected center using a simple 2D translation. It preprocesses eight constant 2D translation vectors for every octree level to quickly determine subdivisions, affected pixels, and recursively update occlusion maps (HOMs). The same translation approach is used to compute the depth component for fast depth cueing.

### [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md)
Voxel Child Address Calculation determines the storage index of a child voxel within a parent's child array, given a query point and the parent's spatial bounds. It solves the problem of mapping a 3D geometric position to a specific slot in a potentially sparse or compacted child payload array. Unlike Point Location, which finds the leaf node containing a point, this method operates at the internal node level to identify which of the 8 octants contains the ray intersection or sample point. It differs from Ordered Child Traversal by focusing on the arithmetic mapping of coordinates to indices rather than the sequence of traversal.

### [Direction-dependent Morton Order Mapping](2662-direction-dependent-morton-order-mapping.md)
Direction-dependent Morton Order Mapping is a remapping function that converts a standard Octree Morton code into a sorting key aligned with the near-to-far rendering order for a specific ray direction. It solves the problem of correct occlusion culling and blending order in sparse voxel rasterization by ensuring that voxels are processed from closest to furthest relative to the camera, regardless of the viewing angle. Unlike Point Location which determines spatial containment, or Fast Projection for Octree Nodes which estimates bounding volumes, this method specifically addresses the topological sorting requirement for volume rendering by adapting the Morton space-filling curve to the ray's sign vector.

### [Ordered Child Traversal](3008-ordered-child-traversal.md)
Ordered Child Traversal determines the visitation sequence of a node's eight children during ray traversal, ensuring they are processed in front-to-back order relative to the ray origin. It differs from Point Location by focusing on traversal order rather than spatial indexing, and from Fast Projection for Octree Nodes by explicitly handling the depth-sorting logic required for correct compositing [G].

### Octree Child Bounding Box Computation
Octree Child Bounding Box Computation computes the AABB of a child node given its parent bounds and child index (0-7 for octree, 0-63 for wide) [S2, unofficial]. It uses Morton/Z-order bit packing for child indexing [S2, unofficial].
**Steps:**
1. Compute `mid = (min + max) * 0.5` [S2, unofficial].
2. Extract bits from `child_index`: `high_x = (child_index & 1) != 0`, `high_y = (child_index & 2) != 0`, `high_z = (child_index & 4) != 0` [S2, unofficial].
3. Child min = `(high_x ? mid.x : min.x, high_y ? mid.y : min.y, high_z ? mid.z : min.z)` [S2, unofficial].
4. Child max = `(high_x ? max.x : mid.x, high_y ? max.y : mid.y, high_z ? max.z : mid.z)` [S2, unofficial].
**Layout:** Child index bit packing: bit 0 = x, bit 1 = y, bit 2 = z. Supports standard 8-way octree (indices 0-7) and extends to 4x4x4 wide nodes (indices 0-63) by reusing the same bit logic for the first 3 dimensions [S2, unofficial].

## Operating characteristics
- **Time Complexity:** O(1) per child index computation, assuming precomputed masks and constant-time bitwise operations [S1, unofficial].
- **Memory Complexity:** O(max_depth) for storing precomputed bit masks [S1, unofficial].
- **Performance:** Precomputing `child_bit_depth` avoids expensive arbitrary bit shifts, resulting in noticeably better performance compared to dynamic shifting [S1, unofficial].
- **Strengths:** Efficient, hardware-friendly bitwise operations; no branching for mask computation; suitable for GPU parallel execution [S1, unofficial].
- **Limitations:** Requires integer coordinates; does not handle floating-point precision directly; limited to octree structures with regular 2x2x2 subdivisions [S1, unofficial].

## Background
The method originates from the work on Interactive Isosurface Ray Tracing of Large Octree Volumes, where efficient child index computation was critical for performance [S1, unofficial]. The approach was developed to replace expensive arbitrary bit shifts with precomputed constant shifts, optimizing the traversal loop for large volumes [S1, unofficial]. It is widely used in sparse voxel octree rendering systems, including those based on GigaVoxels and other hierarchical grid traversal techniques [S1, unofficial][S2, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S2] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
