# Point Location
Also called: Octree Point Location, Integer coordinate point location

## Overview
Point location maps a continuous 3D coordinate vector to a specific leaf node or voxel within a canonical octree [S1, S2][S3, unofficial]. It solves the problem of efficiently retrieving data associated with a spatial position by traversing the tree from a given starting node to the target leaf [S1, S2][S3, unofficial]. What sets this method apart from generic tree traversal is its reliance on precomputed bit masks to determine child indices using constant-time bitwise operations, avoiding the computational cost of arbitrary bit shifts [S1, S2][S3, unofficial]. It builds on [Octree Child Index Computation](1807-octree-child-index-computation.md) to derive the 0-7 octant offset [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** `dest`: A 3D integer coordinate vector representing the target location [S3, unofficial]. `depth`: The current depth level in the octree, starting from the root or an intermediate node [S3, unofficial]. `index`: The index of the current node in the flat node array [S3, unofficial]. `max_depth`: The maximum depth of the octree [S3, unofficial].
*   **Output:** The scalar value stored in the leaf node containing `dest` [S1, S2][S3, unofficial].

**Data Structures:**
The method assumes the following data structures, typically stored in an `OctreeData` container:
*   `nodes`: A 2D array `nodes[depth][index]` of type `OctNode` [S1, S2].
*   `caps`: An array `caps[index]` of type `OctCap`, used for leaf voxels at the deepest level [S1, S2].
*   `child_bit_depth`: A precomputed 1D array `child_bit_depth[max_depth]` [S1, S2][S3, unofficial].
*   `OctNode` layout:
    *   `child_scalars[8]`: Array of scalar values for leaf children [S1, S2].
    *   `child_start`: Unsigned integer base pointer or index to the children of this node [S1, S2].
    *   `child_offset[8]`: Array of 8 signed chars representing the offset from `child_start` to the child node. A value of `-1` indicates the child is a leaf (scalar) [S1, S2][S3, unofficial].
*   `OctCap` layout:
    *   `child_scalars[8]`: Array of scalar values for the 8 voxels in the cap [S1, S2].

**Precomputation:**
Before traversal, the `child_bit_depth` array is computed for each depth level $d$ from $0$ to `max_depth`:
$$ \text{child\_bit\_depth}[d] = 1 \ll (\text{max\_depth} - d - 1) $$
This precomputation avoids expensive arbitrary left-shift operations during traversal [S1, S2][S3, unofficial].

**Algorithm Steps:**
1.  **Cast Coordinates:** Convert continuous input coordinates to integers and ensure they fall within the canonical interval $[0, 2^{\text{max\_depth}})$ [S1, S2][S3, unofficial].
2.  **Initialize:** Start with the given `depth` and `index` [S3, unofficial].
3.  **Retrieve Node:** Access the current node: `node = nodes[depth][index]` [S3, unofficial].
4.  **Get Mask:** Retrieve the precomputed bit mask for the current depth: `child_bit = child_bit_depth[depth]` [S1, S2][S3, unofficial].
5.  **Compute Child Index:** Determine the 0-7 octant index of the child containing `dest` by testing bits against `child_bit`, shifting results, and combining them:
    $$ \text{child} = ((\text{dest}.x \ \& \ \text{child\_bit}) \neq 0) \ll 2 \ | \ ((\text{dest}.y \ \& \ \text{child\_bit}) \neq 0) \ll 1 \ | \ ((\text{dest}.z \ \& \ \text{child\_bit}) \neq 0) $$
    This yields the 0-7 octant offset of the child, and hence its index [S2][S3, unofficial].
6.  **Check for Leaf in Interior Node:** If `node.child_offset[child] == -1`, the child is a scalar leaf [S3, unofficial]. Return `node.child_scalars[child]` [S1, S2][S3, unofficial].
7.  **Check for Cap Node (Deepest Level):** If `depth == max_depth - 2`, the node is a cap node containing voxels [S2][S3, unofficial].
    *   Calculate the final child index using the lowest bit (bit 0) of the coordinates:
        $$ \text{child} = (\text{dest}.x \ \& \ 1) \ll 2 \ | \ (\text{dest}.y \ \& \ 1) \ll 1 \ | \ (\text{dest}.z \ \& \ 1) $$
    *   Update `index = node.child_start + node.child_offset[child]` and return `caps[index].child_scalars[child]` [S2][S3, unofficial].
8.  **Traverse Down:** If not a leaf and not a cap:
    *   Update `index` to point to the child's start location: `index = node.child_start + node.child_offset[child]` [S2][S3, unofficial].
    *   Increment `depth`: `depth += 1` [S2][S3, unofficial].
    *   Repeat from Step 3.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def point_locate(dest: Vec3i, depth: int, index: int, max_depth: int, 
                 nodes: List[List[OctNode]], caps: List[OctCap], 
                 child_bit_depth: List[int]) -> T:
    while True:
        node = nodes[depth][index]
        child_bit = child_bit_depth[depth]
        
        # Compute child index (0-7) based on bits in dest coordinates
        child = ((dest.x & child_bit) != 0) << 2 | \
                ((dest.y & child_bit) != 0) << 1 | \
                ((dest.z & child_bit) != 0)
        
        # Check if child is a scalar leaf in an interior node
        if node.child_offset[child] == -1:
            return node.child_scalars[child]
        
        # Check if we are at the cap level (max_depth - 2)
        elif depth == max_depth - 2:
            # Use the lowest bit for cap nodes
            index = node.child_start + node.child_offset[child]
            child = (dest.x & 1) << 2 | (dest.y & 1) << 1 | (dest.z & 1)
            return caps[index].child_scalars[child]
        
        # Traverse to the child node
        index = node.child_start + node.child_offset[child]
        depth += 1
        
    return 0  # Fallback return, though the loop always returns for valid inputs
```

**Edge Cases and Pitfalls:**
*   **Coordinate Range:** Coordinates must be within the canonical interval $[0, 2^{\text{max\_depth}}]$ and cast to integers [S1, S2][S3, unofficial].
*   **Cap Nodes:** The algorithm explicitly handles the deepest level (`max_depth - 2`) as a special case using the lowest bit, implying the octree structure uses "caps" for the finest resolution [S2][S3, unofficial].
*   **Parent Pointers:** Parent pointers are omitted to save memory [S2]. Neighbor finding (which may call point location) relies on a separate `parent_trace` array provided by the ray traversal algorithm [S2][S3, unofficial].

## Subtopics

### Recursive Octree Insertion with Bitstream
An algorithm to traverse and insert a voxel into an Octree using a pre-computed bitstream to determine child indices [S4, unofficial].
*   **Definition:** It uses a bitstream `b` encoding the voxel's position to guide insertion [S4, unofficial].
*   **Steps:**
    1.  If `depth d` is 0, return a new node containing the voxel `v` [S4, unofficial].
    2.  Extract the child index from the bitstream `b` using `group(b, d, 3)` and get the corresponding child `n'` from node `n` [S4, unofficial].
    3.  If child `n'` is null, create a new empty node [S4, unofficial].
    4.  Recursively insert the voxel into the child node with `depth d-1`, and set the result as the child at the index derived from the bitstream [S4, unofficial].
*   **Parameters:** `b`: The bitstream encoding the voxel's position [S4, unofficial].
*   **When to choose:** Used for dynamic insertion into an octree where a bitstream representation of coordinates is available [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(d)$, where $d$ is the depth of the octree, as each iteration performs constant-time bitwise operations and array lookups [G].
*   **Memory Complexity:** $O(\text{max\_depth})$ for the precomputed `child_bit_depth` array [S2]. The node structure itself is compact, using `child_offset` as signed chars to save space [S2][S3, unofficial].
*   **Performance:** Precomputing bit masks avoids expensive arbitrary left-shift operations, providing noticeably better performance than the original Frisken and Perry method [S1, S2][S3, unofficial].
*   **Strengths:** Fast lookup, low memory overhead (no parent pointers), simple bitwise logic [S2][S3, unofficial].
*   **Limitations:** Requires canonical coordinate system $[0, 2^{\text{max\_depth}}]$. Neighbor finding requires additional infrastructure (`parent_trace`) [S2][S3, unofficial].

## Background
Originating from the work of Frisken and Perry, this method was adapted for interactive isosurface ray tracing of large octree volumes [S1, S2][S3, unofficial]. The original proposal involved creating a single-bit mask corresponding to the current depth and applying it to coordinates [S1, S2][S3, unofficial]. The optimized version, caching `child_bit_depth` to avoid arbitrary shifts, was introduced in subsequent implementations for performance [S1, S2][S3, unofficial].

## Sources
[S1] Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~wald/Publications/2006/OctIso/octiso.pdf, fetched 2026-10-27
[S2] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://www.sci.utah.edu/~knolla/octiso-rt06.pdf, fetched 2026-10-28
[S3] Interactive isosurface ray tracing of large octree volumes, https://www.researchgate.net/publication/310054812_Interactive_isosurface_ray_tracing_of_large_octree_volumes, fetched 2026-10-29 (unofficial)
[S4] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
