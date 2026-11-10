# Morton-coded Sparse Voxel Tree


## Overview
A Morton-coded Sparse Voxel Tree (SVO) is a flat-array sparse voxel tree that uses Morton codes (Z-order curve) for spatial ordering and an 8-bit mask per node to track children [S1, unofficial]. It avoids recursive pointers by storing nodes in contiguous blocks of 8 elements, indexed via `node_format::_block_index` [S1, unofficial]. Unlike its sibling [Linear Octree with Z-order curve (Morton tree)](7054-linear-octree-with-z-order-curve-morton-tree.md) which flattens the tree to store only relevant data trading traversal speed for memory efficiency, this method retains a hierarchical mask structure [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: 3D voxel coordinates (integer or fixed-point) and metadata (state, type, user data).
*   Output: A linear memory pool containing node blocks and voxel data, accessible via Morton codes or block indices.

**Data Structures:**
1.  **`node_format`**: Stores tree topology.
    *   `_block_index`: `uint32_t`. Index into the memory pool's block array.
    *   `_depth`: `uint8_t`. Depth of the node in the tree.
    *   `_mask`: `uint8_t`. 8-bit bitmask where bit $i$ (0-7) indicates the existence of the $i$-th child octant.
    *   `has_children()`: Returns `true` if `_mask & 0xFF` is non-zero [S1, unofficial].
2.  **`mem_pool<T>`**: Manages memory allocation.
    *   `_blocks`: `std::vector<std::array<T, 8>>`. Contiguous storage for nodes. Nodes are stored in groups of 8 to align with the 8 octants of a parent [S1, unofficial].
    *   `_free`: `std::queue<uint32_t>`. Queue of available block indices.
    *   `alloc()`: Pops from `_free` if available; otherwise appends a new `std::array<T, 8>` to `_blocks` and returns the new index [S1, unofficial].
    *   `dealloc()`: Pushes a block index back into `_free` [S1, unofficial].
3.  **`basic_voxel_format`**: Stores leaf data.
    *   `_packed`: `uint64_t`. Bit-packed structure:
        *   Bits 0-15: Voxel Index (Morton code) [S1, unofficial].
        *   Bit 15: State bit [S1, unofficial].
        *   Bits 16-31: Type info [S1, unofficial].
        *   Bits 32-63: User data [S1, unofficial].

**Parameters:**
*   `BIT_WIDTH`: Template parameter determining precision.
    *   `morton_16b`: Axis bit width 5. Max extent $32^3$ [S1, unofficial].
    *   `morton_32b`: Axis bit width 10. Max extent $1024^3$ [S1, unofficial].
*   `SPACE_ABSOLUTE_MAX_DEPTH`: 5 for 16b, 10 for 32b [S1, unofficial].
*   `component_type`: `uint8_t` for 16b, `uint16_t` for 32b [S1, unofficial].

**Algorithm Steps:**
1.  **Position to Morton Code**: Convert 3D coordinates $(x, y, z)$ to a Morton code $M$.
    *   Use `libmorton::m3D_e_sLUT<morton_type>(x, y, z)` [S1, unofficial].
2.  **Morton Code to Position**: Convert Morton code $M$ to 3D coordinates $(x, y, z)$.
    *   Use `libmorton::m3D_d_sLUT<morton_type>(M, x, y, z)` [S1, unofficial].
3.  **Child Node Traversal**: Given a parent node at `node_position` with `node_extent`, determine if a child exists at `voxel_transformed`.
    *   Compute transformed position: `node_transformed = node_position << 1`.
    *   Compute distance vector: `dist_vector = voxel_transformed - node_transformed`.
    *   Compute cell vector (octant index components): `cell_vector = dist_vector / node_extent`.
    *   Compute octant index: `index = (cell_vector[0] << 2) + (cell_vector[1] << 1) + cell_vector[2]`.
    *   Compute child bit: `child_bit = 1 << index`.
    *   Check existence: `exist = (node_->_mask & child_bit) != 0` [S1, unofficial].
    *   Compute next node position: `next_node_position = node_position + (cell_vector * node_extent / 2)` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_SIZE = 8

# Data Structures
class Node:
    block_index: int  # uint32_t
    depth: int        # uint8_t
    mask: int         # uint8_t

class MemPool:
    blocks: list[list[Node]]
    free: list[int]

    def alloc(self) -> int:
        if self.free:
            idx = self.free.pop(0)
            return idx
        else:
            new_block = [Node() for _ in range(BLOCK_SIZE)]
            self.blocks.append(new_block)
            return len(self.blocks) - 1

    def dealloc(self, idx: int):
        self.free.append(idx)

    def get_node(self, idx: int, child_idx: int) -> Node:
        # child_idx is 0-7
        return self.blocks[idx][child_idx]

def pos_to_morton(x: int, y: int, z: int, bit_width: int) -> int:
    # Uses libmorton::m3D_e_sLUT
    # Implementation details depend on libmorton library
    # For 16b: morton_type is uint16_t
    # For 32b: morton_type is uint32_t
    pass

def morton_to_pos(morton: int, bit_width: int) -> tuple:
    # Uses libmorton::m3D_d_sLUT
    pass

def traverse_child(parent_node: Node, parent_pos: int, parent_extent: int, child_pos: int) -> bool:
    # SVO_NODE_RELATION_MATH_IMPL macro logic
    node_transformed = parent_pos << 1
    dist_vector = child_pos - node_transformed
    cell_vector = dist_vector // parent_extent
    
    # cell_vector is a 3D vector (x, y, z)
    index = (cell_vector[0] << 2) + (cell_vector[1] << 1) + cell_vector[2]
    child_bit = 1 << index
    exist = (parent_node.mask & child_bit) != 0
    
    next_node_position = parent_pos + (cell_vector * parent_extent // 2)
    
    return exist, next_node_position, index
```

**Edge Cases and Pitfalls:**
*   **Overhead**: The implementation explicitly avoids relying on compiler inlining for the node relation macro due to zero tolerance for overhead [S1, unofficial]. Use `#define LOOP_UNROLL` for pragma unrolling [S1, unofficial].
*   **Integer Space**: The implementation works in unsigned even integer space to reduce complexity [S1, unofficial].
*   **Alignment**: Memory allocation pads size to minimum alignment [S1, unofficial].

## Subtopics

### [Linear Octree with Z-order curve (Morton tree)](7054-linear-octree-with-z-order-curve-morton-tree.md)
A storage method for sparse octrees that stores nodes in a linear array generated using a Z-order curve, eliminating the storage of inner nodes. It is designed for raycasting and voxel applications where the tree can be precalculated and resources are limited, such as GPU textures. Unlike the parent [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md) which retains a hierarchical mask structure, this method flattens the tree to store only relevant data, trading traversal speed for memory efficiency.

## Operating characteristics
*   **Time Complexity**: Traversal is $O(1)$ per node step due to direct indexing via block index and mask check [S1, unofficial].
*   **Memory Complexity**: $O(N)$ where $N$ is the number of nodes, stored in contiguous blocks of 8 [S1, unofficial].
*   **Performance**: Optimized for low overhead with explicit loop unrolling [S1, unofficial].
*   **Strengths**: No recursive pointers, cache-friendly contiguous memory, fast traversal [S1, unofficial].
*   **Limitations**: Fixed max depth and extent based on bit width [S1, unofficial].

## Background
Originated in the `rapid-svo` library by Severi Suominen [S1, unofficial]. The library is an unofficial implementation of a sparse voxel octree optimized for performance [S1, unofficial].

## Sources
[S1] SeveriSuominen/rapid-svo, https://github.com/SeveriSuominen/rapid-svo, fetched 2026-10-26 (unofficial)
