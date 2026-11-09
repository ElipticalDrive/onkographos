# Morton-coded Sparse Voxel Tree

## Overview
A Morton-coded Sparse Voxel Tree is a flat-array sparse voxel tree that uses Morton codes for spatial ordering and an 8-bit mask per node to track children [S1, unofficial]. It avoids recursive pointers by storing nodes in contiguous blocks of 8 elements, indexed via `node_format::_block_index` [S1, unofficial]. This structure builds on [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md) and differs from [Hashed-Octree](7486-hashed-octree.md), [Spatio-Temporal Sparse Voxel Octree (STSVO)](7493-spatio-temporal-sparse-voxel-octree-stsvo.md), and [n-Dimensional Sparse Voxel Tree (nD SVT)](7999-n-dimensional-sparse-voxel-tree-nd-svt.md) by using contiguous blocks of 8 indexed via `_block_index` instead of hash tables or spatio-temporal tracking [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: A 3D coordinate space (typically unsigned even integers) and a target voxel position [S1, unofficial].
*   Output: A traversal path to the leaf node or a determination of child existence at a given node [S1, unofficial].

**Data Structures:**
1.  **`node_format`**: Stores metadata for each tree node [S1, unofficial].
    *   `_block_index`: `uint32_t`. Index into the memory pool's block array [S1, unofficial].
    *   `_depth`: `uint8_t`. Depth of the node in the tree [S1, unofficial].
    *   `_mask`: `uint8_t`. Bitmask representing the existence of 8 children (one bit per octant) [S1, unofficial]. Bit $i$ is set if child $i$ exists [S1, unofficial].
    *   `has_children()`: Returns `true` if `_mask & 0xFF` is non-zero [S1, unofficial].
2.  **`mem_pool<T>`**: Manages contiguous memory allocation [S1, unofficial].
    *   `_blocks`: `std::vector<std::array<T, 8>>`. Stores nodes in fixed-size blocks of 8 [S1, unofficial].
    *   `_free`: `std::queue<uint32_t>`. Queue of available block indices [S1, unofficial].
    *   `alloc()`: Pops an index from `_free`. If empty, appends a new block to `_blocks` and returns the new index [S1, unofficial].
    *   `dealloc()`: Pushes a block index onto `_free` [S1, unofficial].
3.  **`morton_util<BIT_WIDTH>`**: Handles coordinate conversions [S1, unofficial].
    *   `BIT_WIDTH`: Template parameter [S1, unofficial].
        *   `morton_16b`: Axis bit width 5, max extent $32^3$. `morton_type` is `uint16_t`, `component_type` is `uint8_t` [S1, unofficial].
        *   `morton_32b`: Axis bit width 10, max extent $1024^3$. `morton_type` is `uint32_t`, `component_type` is `uint16_t` [S1, unofficial].
    *   `AXIS_MAX`: `32` for 16b, `1024` for 32b [S1, unofficial].
    *   `pos_to_morton()`: Converts 3D position to Morton code using `libmorton::m3D_e_sLUT` [S1, unofficial].
    *   `morton_to_pos()`: Converts Morton code to 3D position using `libmorton::m3D_d_sLUT` [S1, unofficial].

**Numbered Steps for Child Traversal:**
Given a current node `node_` at `node_position` with `node_extent`, and a target `voxel_transformed`:
1.  Transform the node position: `node_transformed = node_position << 1` [S1, unofficial].
2.  Compute the distance vector: `dist_vector = voxel_transformed - node_transformed` [S1, unofficial].
3.  Compute the cell vector (octant index components): `cell_vector = dist_vector / node_extent` [S1, unofficial].
4.  Compute the linear octant index: `index = (cell_vector[0] << 2) + (cell_vector[1] << 1) + cell_vector[2]` [S1, unofficial].
5.  Compute the child bit mask: `child_bit = 1 << index` [S1, unofficial].
6.  Check existence: `exist = (node_->_mask & child_bit) != 0` [S1, unofficial].
7.  Compute next node position (if child exists): `next_node_position = node_position + (cell_vector * node_extent / 2)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_child_relation(node_position, node_extent, voxel_transformed, node_mask):
    """
    Computes the octant index and checks child existence.
    node_position: tuple (x, y, z)
    node_extent: int
    voxel_transformed: tuple (x, y, z)
    node_mask: int (8-bit)
    Returns: (index, exists, next_node_position)
    """
    # (1) Transform node position
    node_transformed = (node_position[0] << 1, 
                        node_position[1] << 1, 
                        node_position[2] << 1)
    
    # (2) Distance vector
    dist_vector = (voxel_transformed[0] - node_transformed[0],
                   voxel_transformed[1] - node_transformed[1],
                   voxel_transformed[2] - node_transformed[2])
    
    # (3) Cell vector (integer division)
    cell_vector = (dist_vector[0] // node_extent,
                   dist_vector[1] // node_extent,
                   dist_vector[2] // node_extent)
    
    # (4) Octant index
    index = (cell_vector[0] << 2) + (cell_vector[1] << 1) + cell_vector[2]
    
    # (5) Child bit
    child_bit = 1 << index
    
    # (6) Existence check
    exists = (node_mask & child_bit) != 0
    
    # (7) Next node position
    next_node_position = (
        node_position[0] + (cell_vector[0] * node_extent) // 2,
        node_position[1] + (cell_vector[1] * node_extent) // 2,
        node_position[2] + (cell_vector[2] * node_extent) // 2
    )
    
    return index, exists, next_node_position

def pos_to_morton_16b(x, y, z):
    """Converts 3D position to 16-bit Morton code using standard bit interleaving [G]."""
    x &= 0x1F; y &= 0x1F; z &= 0x1F
    m = 0
    for i in range(5):
        m |= (x & 1) << (3 * i)
        m |= (y & 1) << (3 * i + 1)
        m |= (z & 1) << (3 * i + 2)
        x >>= 1; y >>= 1; z >>= 1
    return m

def morton_to_pos_16b(morton):
    """Converts 16-bit Morton code to 3D position using standard bit de-interleaving [G]."""
    x = y = z = 0
    for i in range(5):
        x |= (morton & 1) << i
        y |= (morton & 2) << (i - 1)
        z |= (morton & 4) << (i - 2)
        morton >>= 3
    return (x, y, z)
```

**Parameters:**
*   `BIT_WIDTH`: `morton_16b` or `morton_32b` [S1, unofficial].
*   `SPACE_ABSOLUTE_MAX_DEPTH`: 5 for 16b, 10 for 32b [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Overhead:** The implementation explicitly avoids relying on compiler inlining for the node relation macro due to zero tolerance for overhead [S1, unofficial]. It uses `#define LOOP_UNROLL` for pragma unrolling [S1, unofficial].
*   **Coordinate Space:** Works in unsigned even integer space to reduce complexity [S1, unofficial].
*   **Precision:** Integer division is used for cell vector calculation. Ensure inputs are aligned to the grid [G].

## Subtopics

### Linear Octree with Z-order curve (Morton tree)
A storage method for sparse octrees that stores nodes in a linear array generated using a Z-order curve, eliminating the storage of inner nodes [S2, unofficial]. It stores the octree nodes in a linear array ordered by their Z-order (Morton) code [S2, unofficial]. Because inner nodes are eliminated from the primary storage, accessing voxel data requires cross-referencing the linear tree array with a second data texture [S2, unofficial]. It is suggested for raycasting/voxel applications where the tree can be precalculated and resources are limited (e.g., GPU textures) [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Traversal is $O(1)$ per level due to direct index calculation and bitmask check [S1, unofficial]. No pointer chasing is required [G].
*   **Memory Complexity:** Nodes are stored in contiguous blocks of 8 [S1, unofficial]. Each node uses 6 bytes (`uint32_t` + `uint8_t` + `uint8_t`) [S1, unofficial]. Memory pool overhead is minimal [G].
*   **Performance:** The structure is designed for minimal overhead [S1, unofficial]. No recursive calls or pointer indirection are used [S1, unofficial].
*   **Strengths:** Cache-friendly due to contiguous memory layout [G]. Fast traversal is achieved [S1, unofficial].
*   **Limitations:** Fixed block size of 8 is enforced [S1, unofficial]. The structure is limited by Morton code bit width (max extent $32^3$ or $1024^3$) [S1, unofficial].
*   **When to use:** High-performance volume rendering where traversal speed is critical [G].
*   **When not to use:** Scenes requiring extents larger than $1024^3$ without 32b Morton code support [S1, unofficial].

## Background
Originated in the `rapid-svo` library by Severi Suominen [S1, unofficial]. The library emphasizes zero-overhead traversal and efficient memory management for sparse voxel octrees [S1, unofficial].

## Sources
[S1] SeveriSuominen/rapid-svo, https://github.com/SeveriSuominen/rapid-svo, fetched 2026-10-26 (unofficial)
[S2] Efficient storage for a sparse octree?, https://stackoverflow.com/questions/6085209/efficient-storage-for-a-sparse-octree, fetched 2026-10-26 (unofficial)
