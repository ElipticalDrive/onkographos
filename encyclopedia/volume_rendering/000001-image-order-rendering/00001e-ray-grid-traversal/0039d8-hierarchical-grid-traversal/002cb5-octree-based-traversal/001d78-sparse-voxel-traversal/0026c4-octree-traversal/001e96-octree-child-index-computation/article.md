# Octree Child Index Computation

Also called: Octant Index Computation, Child Octant Offset Calculation

## Overview
Octree Child Index Computation determines the 0–7 octant offset (and thus the linear child index) of a specific child node within an octree parent, given the parent's depth and the target coordinates. It solves the problem of efficiently mapping spatial coordinates to a specific child node during traversal without performing expensive arbitrary bit shifts. It sets itself apart from generic tree traversal by relying on precomputed bitmasks and constant-time bitwise operations to derive the index, optimizing for the regular 2x2x2 subdivision structure of octrees.

## How it works
**Inputs:**
- `depth`: The current depth level of the parent node in the octree.
- `max_depth`: The maximum depth of the octree.
- `origin`: The 3D coordinate vector of the ray origin or traversal start point.
- `destination`: The 3D coordinate vector of the target point or ray direction endpoint.

**Outputs:**
- `child_index`: An integer in the range [0, 7] representing the octant index of the child node containing the destination relative to the parent.

**Data Structures:**
- `child_bit_depth`: A precomputed array where `child_bit_depth[d] = 1 << (max_depth - d - 1)`. This array maps each depth level `d` to a bitmask representing the spatial resolution at that level.

**Steps:**
1.  **Precompute Bitmasks:** Calculate `child_bit_depth[d]` for all relevant depths $d \in [0, \text{max\_depth}]$. This avoids runtime arbitrary shifts.
2.  **Mask Coordinates:** Apply the bitmask for the current depth to both `origin` and `destination` coordinates. This effectively quantizes the coordinates to the grid resolution of the current node.
3.  **Compute Octant Offset:** Determine which octant the `destination` falls into relative to the `origin` (or the parent's center) using binary AND (`&`) and integer inequality comparisons.
4.  **Extract Components:** Bit-shift the result by constants corresponding to the X, Y, and Z components to isolate the individual bit flags for each axis.
5.  **Combine:** Sum the shifted components to produce the final 0–7 index.
6.  **Leaf Handling:** If the current node is a leaf (or cap node), return the scalar value or voxel index instead of a child index.

**Formulas:**
Let $B_d$ be the bitmask for depth $d$:
$$ B_d = 1 \ll (\text{max\_depth} - d - 1) $$

Let $O_x, O_y, O_z$ be the origin coordinates and $D_x, D_y, D_z$ be the destination coordinates.

The masked coordinates are:
$$ O'_x = O_x \ \& \ B_d, \quad O'_y = O_y \ \& \ B_d, \quad O'_z = O_z \ \& \ B_d $$
$$ D'_x = D_x \ \& \ B_d, \quad D'_y = D_y \ \& \ B_d, \quad D'_z = D_z \ \& \ B_d $$

The octant bits are determined by comparing the masked destination to the masked origin (or parent bounds). Typically, the bit is set if the destination is greater than the origin/center in that dimension:
$$ b_x = (D'_x > O'_x) ? 1 : 0 $$
$$ b_y = (D'_y > O'_y) ? 1 : 0 $$
$$ b_z = (D'_z > O'_z) ? 1 : 0 $$

The final index is:
$$ \text{child\_index} = b_x + 2 \cdot b_y + 4 \cdot b_z $$

**Parameters:**
- `max_depth`: Typically determined by the voxel resolution and scene bounds. Not given in the sources.

**Edge Cases and Pitfalls:**
- **Precision:** Integer operations require coordinates to be scaled appropriately to avoid precision loss during masking.
- **Boundary Conditions:** Coordinates exactly on the boundary between octants must be handled consistently (e.g., using strict inequality or inclusive bounds) to ensure unique indexing.
- **Leaf Nodes:** The method returns a scalar or voxel index for leaves, not a child index. The caller must check for leaf status before invoking this computation.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_octree_child_index(depth, max_depth, origin, destination):
    """
    Computes the 0-7 octant index of the child node containing the destination
    relative to the parent node at the given depth.
    
    Args:
        depth (int): Current depth of the parent node.
        max_depth (int): Maximum depth of the octree.
        origin (tuple): (ox, oy, oz) coordinates of the parent origin or ray start.
        destination (tuple): (dx, dy, dz) coordinates of the target point.
        
    Returns:
        int: The child index (0-7).
    """
    # Precomputed bitmask for the current depth
    # child_bit_depth[d] = 1 << (max_depth - d - 1)
    mask = 1 << (max_depth - depth - 1)
    
    # Apply mask to coordinates
    ox, oy, oz = origin
    dx, dy, dz = destination
    
    # Masked coordinates
    mx_ox = ox & mask
    mx_oy = oy & mask
    mx_oz = oz & mask
    
    mx_dx = dx & mask
    mx_dy = dy & mask
    mx_dz = dz & mask
    
    # Determine octant bits using integer inequality
    # Bit 0: X component
    bit_x = 1 if mx_dx > mx_ox else 0
    
    # Bit 1: Y component
    bit_y = 1 if mx_dy > mx_oy else 0
    
    # Bit 2: Z component
    bit_z = 1 if mx_dz > mx_oz else 0
    
    # Combine bits to form index
    child_index = bit_x + (bit_y << 1) + (bit_z << 2)
    
    return child_index
```

## Subtopics

### [Point Location](2076-point-location.md)
Point Location maps a continuous 3D coordinate vector to a specific leaf node or voxel within a canonical octree. It solves the problem of efficiently retrieving data associated with a spatial position by traversing the tree from a given starting node to the target leaf. What sets this method apart from generic tree traversal is its reliance on precomputed bit masks to determine child indices using constant-time bitwise operations, avoiding the computational cost of arbitrary bit shifts. It builds on [Octree Child Index Computation](1807-octree-child-index-computation.md) to derive the 0-7 octant offset.

### [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md)
Fast Projection for Octree Nodes accelerates the mapping of 3D octree geometry to 2D screen space by computing the projected center of child nodes from their parents using precomputed 2D translation vectors. It solves the bottleneck of repeated full projection matrix multiplications during hierarchical traversal. Unlike [Point Location](2076-point-location.md) which finds spatial coordinates, or [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md) which handles linear memory offsets, this method specifically optimizes the screen-space footprint calculation for occlusion and depth cueing in object-order ray casting.

### [Voxel Child Address Calculation](2495-voxel-child-address-calculation.md)
Calculates the linear index (address) of a specific child octant within a compact block of valid children, based on a validity bitmask. It maps a logical octant index (0–7) to its physical position in a dense array that only stores active children.

### [Direction-dependent Morton Order Mapping](2662-direction-dependent-morton-order-mapping.md)
Direction-dependent Morton Order Mapping is a remapping function that converts a standard Octree Morton code into a sorting key aligned with the near-to-far rendering order for a specific ray direction. It solves the problem of correct occlusion culling and blending order in sparse voxel rasterization by ensuring that voxels are processed from closest to furthest relative to the camera, regardless of the viewing angle. Unlike [Point Location](2076-point-location.md) which determines spatial containment, or [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md) which estimates bounding volumes, this method specifically addresses the topological sorting requirement for volume rendering by adapting the Morton space-filling curve to the ray's sign vector.

### [Ordered Child Traversal](3008-ordered-child-traversal.md)
Ordered Child Traversal determines the visitation sequence of a node's eight children during ray traversal, ensuring they are processed in front-to-back order relative to the ray origin. This ordering is critical for correct alpha blending in volume rendering and for maintaining visibility order in rasterization-based approaches. It differs from [Point Location](2076-point-location.md) by focusing on traversal order rather than spatial indexing, and from [Fast Projection for Octree Nodes](2178-fast-projection-for-octree-nodes.md) by explicitly handling the depth-sorting logic required for correct compositing.

### Octree Child Bounding Box Computation
Computes the AABB of a child node given its parent bounds and child index (0-7 for octree, 0-63 for wide) [S2, unofficial]. Uses Morton/Z-order bit packing for child indexing [S2, unofficial]. Compute mid = (min + max) * 0.5. Extract bits from child_index: high_x = (child_index & 1) != 0, high_y = (child_index & 2) != 0, high_z = (child_index & 4) != 0. Child min = (high_x ? mid.x : min.x, high_y ? mid.y : min.y, high_z ? mid.z : min.z). Child max = (high_x ? max.x : mid.x, high_y ? max.y : mid.y, high_z ? max.z : mid.z) [S2, unofficial]. Child index bit packing: bit 0 = x, bit 1 = y, bit 2 = z. Supports standard 8-way octree (indices 0-7) and extends to 4x4x4 wide nodes (indices 0-63) by reusing the same bit logic for the first 3 dimensions [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(1)$ per node visit, assuming precomputed bitmasks.
- **Memory Complexity:** $O(\text{max\_depth})$ for storing the `child_bit_depth` array.
- **Performance:** Significantly faster than arbitrary bit shifts due to the use of precomputed constants and simple bitwise operations [S1, unofficial].
- **Strengths:** Efficient for regular octrees; minimal computational overhead per traversal step.
- **Limitations:** Requires precomputation; less flexible for non-standard tree structures or variable child counts without modification.

## Background
Originated in the context of interactive isosurface ray tracing of large octree volumes [S1, unofficial]. The method was developed to address the performance bottleneck of arbitrary bit shifts during tree traversal, replacing them with precomputed bitmasks and bitwise operations [S1, unofficial]. It is a fundamental component of hierarchical grid traversal in volume rendering, enabling efficient occlusion and empty space skipping.

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.researchgate.net/publication/261082346_Interactive_Isosurface_Ray_Tracing_of_Large_Octree_Volumes, fetched 2026-10-29 (unofficial)
[S2] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
