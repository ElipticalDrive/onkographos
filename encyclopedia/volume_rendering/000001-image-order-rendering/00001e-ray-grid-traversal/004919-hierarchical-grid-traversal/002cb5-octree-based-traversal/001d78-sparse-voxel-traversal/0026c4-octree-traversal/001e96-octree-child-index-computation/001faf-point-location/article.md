# Point Location

Also called: Coordinate-based hashing, Frisken and Perry hashing, Integer coordinate point location, Octree Hashing and Point Location, Octree Point Location, Point Location via Bitwise Masking

## Overview
Point Location is an accelerated hashing scheme that maps a 3D integer coordinate to a specific leaf node in a canonical sparse voxel octree defined on the interval $[0, 2^{d_{max}}]$ [S1, S2][S5, unofficial]. It solves the problem of finding the octree cell containing a given point without traversing parent pointers or performing expensive arbitrary bit shifts [S1, S2]. Unlike sibling methods that may rely on Morton codes or spatial partitioning bounds, Point Location uses precomputed depth masks to determine the child octant index (0–7) directly from the destination coordinates using constant bitwise operations [S1, S2]. It is distinct from [Octree Child Index Computation](1807-octree-child-index-computation.md) in that it performs the full top-down traversal to the leaf, whereas the sibling method yields only the offset of a child relative to a current node [S1, S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A destination coordinate vector `dest` (integer components $x, y, z$), the current `depth` (starting at 0 for root), and the current `index` (starting at 0 for root). The coordinates are constrained to the range $[0, 2^{d_{max}}]$ [S1][S4, S5, unofficial].
*   **Output:** The scalar value stored in the leaf node containing the point, or the leaf node itself [S1][S3, unofficial].

**Data Structures:**
The method relies on a global or context-specific `OctreeData` structure containing:
1.  `nodes`: A 2D array `nodes[depth][index]` storing `OctNode` objects.
2.  `caps`: An array of `OctCap` objects, used for leaf-level storage at the maximum depth [S1, S2].
3.  `child_bit_depth`: A precomputed 1D array where `child_bit_depth[d] = 1 << (max_depth - d - 1)` [S1, S2][S3, unofficial].

**OctNode Layout:**
*   `child_scalars[8]`: Scalar values for 8 potential scalar leaves [S1, S2].
*   `child_start`: An unsigned integer base pointer/index to the children array [S1, S2].
*   `child_offset[8]`: A signed char array. If `child_offset[child] == -1`, the child is a leaf (scalar or cap) [S1, S2]. Otherwise, it is an interior node, and the actual index is `child_start + child_offset[child]` [S1, S2].

**OctCap Layout:**
*   `scalars[8]`: Scalar values for the 8 voxels at the maximum depth [S1, S2].

**Algorithm Steps:**
1.  **Precomputation:** Calculate `child_bit_depth[d] = 1 << (max_depth - d - 1)` for all depths $d \in [0, max\_depth]$ [S1, S2][S3, unofficial]. This avoids arbitrary left shifts during traversal [S1].
2.  **Initialization:** Set `current_depth = depth` and `current_index = index` [S1, S2].
3.  **Traversal Loop:** Iterate indefinitely:
    a. Retrieve the current node: `node = nodes[current_depth][current_index]` [S1, S2].
    b. Retrieve the mask: `child_bit = child_bit_depth[current_depth]` [S1, S2].
    c. **Compute Child Octant:** Determine the 0–7 octant index `child` by testing the bits of `dest` against `child_bit` [S1, S2][S3, unofficial]:
       $$ child = ((dest.x \ \& \ child\_bit) \neq 0) \ll 2 \ | \ ((dest.y \ \& \ child\_bit) \neq 0) \ll 1 \ | \ ((dest.z \ \& \ child\_bit) \neq 0) $$
       This uses constant shifts (0, 1, 2) instead of arbitrary shifts [S1, S2].
    d. **Check Leaf Status:**
       i. If `node.child_offset[child] == -1`:
          - If `current_depth == max_depth - 2`: The child is a cap node. Calculate the final cap index: `cap_index = node.child_start + node.child_offset[child]`. Return `caps[cap_index].scalars[child]` [S1, S2][S3, unofficial].
          - Otherwise: The child is a scalar leaf. Return `node.child_scalars[child]` [S1, S2][S3, unofficial].
       ii. If `node.child_offset[child] != -1`: The child is an interior node. Update `current_index = node.child_start + node.child_offset[child]` and `current_depth++`. Continue loop [S1, S2][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_locate(dest_x, dest_y, dest_z, current_depth, current_index, max_depth, nodes, caps, child_bit_depth):
    """
    Locates the leaf node containing the integer coordinate (dest_x, dest_y, dest_z).
    Returns the scalar value stored in that leaf.
    """
    depth = current_depth
    index = current_index
    
    while True:
        # Retrieve current node
        node = nodes[depth][index]
        
        # Retrieve precomputed bit mask for current depth
        child_bit = child_bit_depth[depth]
        
        # Compute target child octant (0-7) using bitwise AND and constant shifts
        # (val & mask) != 0 evaluates to 1 if bit is set, 0 otherwise
        x_bit = 1 if (dest_x & child_bit) else 0
        y_bit = 1 if (dest_y & child_bit) else 0
        z_bit = 1 if (dest_z & child_bit) else 0
        
        child = (x_bit << 2) | (y_bit << 1) | z_bit
        
        # Check if child is a leaf
        if node.child_offset[child] == -1:
            # Leaf detected
            if depth == max_depth - 2:
                # Cap node: calculate index into caps array
                cap_index = node.child_start + node.child_offset[child]
                return caps[cap_index].scalars[child]
            else:
                # Scalar leaf in interior node
                return node.child_scalars[child]
        
        # Child is an interior node: traverse down
        index = node.child_start + node.child_offset[child]
        depth += 1
```

**Edge Cases and Pitfalls:**
*   **Coordinate Range:** Coordinates must be within $[0, 2^{d_{max}}]$. Out-of-bounds coordinates may access invalid memory or return incorrect nodes [S1][S4, S5, unofficial].
*   **Cap Nodes:** The octree typically uses "cap nodes" at depth $max\_depth - 2$ to store 8 voxels directly, reducing pointer overhead. The algorithm must explicitly handle this depth level by switching from `nodes` to `caps` and using the lowest bit (`& 1`) for the final octant selection if the mask logic differs, though the standard mask `child_bit` at $max\_depth - 2$ is `1 << 1 = 2`, which correctly isolates the second-to-last bit. However, sources specify that at $max\_depth - 2$, the final child index uses `dest & 1` [S1, S2][S3, unofficial]. This implies the cap nodes store the final 2 levels of resolution implicitly or the mask logic adapts. The pseudocode above reflects the source's explicit check for `max_depth - 2` and use of `& 1` for the final octant calculation in caps [S1, S2][S3, unofficial].
*   **Parent Pointers:** Parent pointers are omitted to save memory. Point location alone does not require them, but neighbor finding does [S1, S2].

## Subtopics

### Recursive Octree Insertion with Bitstream
An algorithm to traverse and insert a voxel into an Octree using a pre-computed bitstream to determine child indices [S6, unofficial]. It extracts the child index from the bitstream `b` using `group(b, d, 3)` at each depth `d` [S6, unofficial]. If the child node is null, it creates a new empty node, then recursively inserts the voxel into the child with depth `d-1` [S6, unofficial]. This variant is used for construction rather than lookup [S6, unofficial].

### Sphere Marching
A ray traversal strategy for the outer band that searches for the octree node containing the current point and computes step length by evaluating the corresponding polynomial [S7]. It uses variable error extension with a delta ($\delta$) of 0.02 to reduce octree memory size [S7]. Compared to standard Octree Traversal, it can start at any position on the ray inside the node, generally requiring fewer iterations on average but having a longer tail in iteration count distribution [S7].

### Octant Identity Calculation
Computes the cardinal identity (0-7) of an octant based on absolute voxel coordinates and octree depth levels [S8, unofficial]. It calculates a shift amount $d = d_m - 1 - d_c$ (where $d_m$ is max depth and $d_c$ is current depth) and extracts the bit at position $d$ from the z, y, and x coordinates using unsigned right shift and bitwise AND [S8, unofficial]. The result is combined into a 3-bit integer: `((z_bit << 2) | (y_bit << 1) | x_bit)` [S8, unofficial]. This is a direct calculation of the child index without traversal [S8, unofficial].

## Operating characteristics
**Time Complexity:** $O(d_{max})$ per point location, as it traverses from root to leaf [S1, S2]. Each step involves constant-time bitwise operations [S1, S2].
**Memory Complexity:** $O(1)$ auxiliary space for the traversal, excluding the octree storage itself [S1, S2]. The `child_bit_depth` array requires $O(d_{max})$ space [S1, S2].
**Performance:** The use of precomputed masks avoids expensive arbitrary left-shift instructions, providing noticeably better performance than naive bitwise shifting [S1, S2][S3, unofficial].
**Strengths:** Fast lookup without parent pointers; efficient for sparse data due to octree structure; simple bitwise logic [S1, S2][S4, unofficial].
**Limitations:** Requires integer coordinates; not suitable for floating-point points without prior quantization; cap node handling adds a branch [S1, S2][S4, unofficial].

## Background
The method is based on the octree hashing scheme proposed by Frisken and Perry [S1, S2][S4, S5, unofficial]. They observed that octree locational codes could be represented directly by integer coordinates, allowing hashing from integer coordinates to the correct child node [S4, unofficial]. The specific optimization of caching depth masks to avoid arbitrary shifts was introduced in the implementation of Interactive Isosurface Ray Tracing of Large Octree Volumes [S1, S2][S3, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S4] PDF A Survey of Octree Volume Rendering Methods, https://www.sci.utah.edu/~knolla/octsurvey.pdf, fetched 2026-10-29 (unofficial)
[S5] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S6] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S7] Rendering piecewise approximations of SDFs through analytic intersections, https://doi.org/10.1016/j.cag.2024.103981, fetched 2026-10-27
[S8] nphyx/voctopus, https://github.com/nphyx/voctopus, fetched 2026-10-26 (unofficial)
