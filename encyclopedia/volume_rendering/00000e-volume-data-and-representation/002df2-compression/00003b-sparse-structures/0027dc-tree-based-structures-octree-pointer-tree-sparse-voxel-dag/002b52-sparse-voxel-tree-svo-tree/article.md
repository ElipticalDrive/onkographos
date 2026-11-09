# Sparse Voxel Tree (SVO Tree)

Also called: ESVO, Hierarchical Voxel Storage, Sparse voxel octrees, Voxel Tree

## Overview
A Sparse Voxel Tree (SVO) is a hierarchical data structure for sparse voxel grids where the 3D domain is recursively subdivided, and every internal node has up to 8 children (one for each octant) [S3, unofficial]. It solves the problem of storing and querying large, sparse 3D volumes by allocating memory only for non-empty regions, providing compression proportional to sparsity [S2]. It sets itself apart from [Sparse Voxel DAGs](2791-sparse-voxel-dags.md) by allowing multiple paths to the same voxel (no sharing), and from [Compact pointer trees](844-compact-pointer-tree.md) by using a fixed branching factor of 8 rather than variable $N$ [S3, unofficial].

## How it works
**Inputs:** A 3D voxel position (integer coordinates) and voxel data (e.g., color, density).
**Outputs:** A pointer to the voxel data or a status indicating success/failure.

**Data Structures:**
The structure is defined by `tree<FORMAT_T, BIT_WIDTH, DETAILS>` [S1, unofficial].
1.  **`node_format`**: Contains:
    *   `_depth`: `uint32_t`, current depth in the tree [S1, unofficial].
    *   `_mask`: Bitmask indicating which of the 8 children exist [S1, unofficial].
    *   `_block_index`: Index into the memory pool's block array [S1, unofficial].
2.  **`mem_pool<T>`**: Manages fixed-size blocks of memory.
    *   `_blocks`: Array of `T[]` arrays. Each block holds exactly 8 elements [S1, unofficial].
    *   `_free`: Free list for reuse [S1, unofficial].
3.  **`_root_node`**: The root `node_format` [S1, unofficial].
4.  **`_node_pool`**: `mem_pool<node_format>` storing internal nodes [S1, unofficial].
5.  **`_voxel_pool`**: `mem_pool<FORMAT_T>` storing leaf voxels [S1, unofficial].

**Parameters:**
*   `BIT_WIDTH`: Template parameter for coordinate precision. Values are 16 or 32 [S1, unofficial].
*   `MAX_DEPTH`: Derived from `BIT_WIDTH`. For 16-bit, `MAX_DEPTH=5` ($32^3$); for 32-bit, `MAX_DEPTH=10` ($1024^3$) [S1, unofficial].
*   `BOUNDS[3]`: Maximum bounds per axis. Defaults to `ABSOLUTE_AXIS_WIDTH` if not specified [S1, unofficial].
*   `_discard_overflow`: Boolean. If `true`, silently drops out-of-bounds voxels; if `false`, triggers an assertion [S1, unofficial].

**Algorithm (Allocation):**
The traversal aligns coordinates to an "even-space" grid by shifting left by 1 bit [S1, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def alloc(voxel_position, voxel_data):
    # 1. Overflow Check
    if voxel_position.x >= BOUNDS[0] or \
       voxel_position.y >= BOUNDS[1] or \
       voxel_position.z >= BOUNDS[2]:
        if _discard_overflow:
            return
        else:
            assert False, "Out of bounds"

    # 2. Coordinate Transform
    # Shift left by 1 bit to align with even-space grid
    pos = voxel_position << 1

    # 3. Root Initialization
    node = _root_node
    node_position = {0, 0, 0}
    
    # 4. Internal Node Traversal
    # Iterate from depth 0 to MAX_DEPTH-2
    for i in range(0, MAX_DEPTH - 2):
        depth = i
        node_extent = AXIS_WIDTH >> depth
        
        # Compute child index, bit, existence, and next position
        # SVO_NODE_RELATION_MATH_IMPL() logic:
        # index = (pos.x - node_position.x) / node_extent
        # child_bit = 1 << index
        # exist = (node._mask & child_bit) != 0
        # next_node_position = node_position + (index % 2) * node_extent, etc.
        
        index, child_bit, exist, next_node_position = compute_svo_relation(pos, node_position, node_extent)
        
        # Ensure mask bit is set if child exists
        node._mask |= child_bit
        
        # Fetch current node block
        node_block = _node_pool._blocks[node._block_index]
        
        if not exist:
            # Allocate new node block
            new_node = node_format()
            new_node._depth = depth + 1
            new_node._mask = 0
            new_node._block_index = _node_pool.acquire_next_index()
            
            # Store new node in parent's block
            node_block[index] = new_node
            
            # Allocate new block for the new node
            _node_pool.alloc()
            
            # Move to new node
            node = node_block[index]
        else:
            # Move to existing child
            node = node_block[index]
            
        node_position = next_node_position

    # 5. Voxel Octant Allocation (Depth MAX_DEPTH-1)
    depth = MAX_DEPTH - 1
    node_extent = 4 # Specific extent for this level
    
    index, child_bit, exist, next_node_position = compute_svo_relation(pos, node_position, node_extent)
    node._mask |= child_bit
    
    if not exist:
        new_node = node_format()
        new_node._depth = MAX_DEPTH - 1
        new_node._mask = 0
        # Acquire from voxel pool for the block index? 
        # Note: Source says acquire voxel block index, but stores in node.
        new_node._block_index = _voxel_pool.acquire_next_index()
        
        # Fetch parent block and store new node
        parent_block = _node_pool._blocks[node._block_index] # Wait, node is parent here?
        # Correction: 'node' is the parent of the voxel. 
        # The source code snippet shows: node_block_[index] = new_node_
        # But 'node' was updated in the loop. 
        # Let's follow the source logic strictly:
        # The loop ends at MAX_DEPTH-2. 'node' is at MAX_DEPTH-2.
        # We are allocating the child at MAX_DEPTH-1.
        
        # Re-evaluating source [S1, unofficial] snippet:
        # "new_node_._block_index = _voxel_pool.acquire_next_index()"
        # "node_block_[index] = new_node_"
        # This implies the node at MAX_DEPTH-1 points to a voxel block.
        
        # Let's assume 'node' is the parent (MAX_DEPTH-2).
        # We need to fetch its block.
        parent_block = _node_pool._blocks[node._block_index]
        parent_block[index] = new_node
        
        # Alloc voxel block
        _voxel_pool.alloc()
        
        # Move to new node (which is the parent of the actual voxel)
        node = parent_block[index]
    else:
        node = _node_pool._blocks[node._block_index][index]

    # 6. Voxel Allocation (Depth MAX_DEPTH)
    node_extent = 2
    index, child_bit, exist, next_node_position = compute_svo_relation(pos, node_position, node_extent)
    
    # Store voxel in the voxel block
    voxel_block = _voxel_pool._blocks[node._block_index]
    voxel_block[index] = voxel_data
```

**Algorithm (Deallocation):**
1.  **Trace:** Call `get_traced(voxel_position)` to retrieve the voxel pointer and fill `path_` (nodes) and `child_bits` arrays up to the reached depth [S1, unofficial]. Return false if not found.
2.  **Voxel Block Deallocation:**
    *   Clear the child bit from `path_[depth]._mask`.
    *   If `path_[depth]._mask != 0`, return true (node still has children).
    *   Else, dealloc voxel block via `_voxel_pool.dealloc(path_[depth]._block_index)`.
3.  **Node Block Deallocation:**
    *   Loop `depth` from current down to 1.
    *   If `path_[depth]._mask != 0`, break (parent still has other children).
    *   Else, dealloc node block via `_node_pool.dealloc(path_[depth]._block_index)`.
    *   Clear child bit from `path_[depth-1]._mask`.
    *   Decrement depth [S1, unofficial].

**Pitfalls:**
*   **Compiler Performance:** The traversal loop causes significant branch/pipeline overhead on MSVC. Applying loop unrolling yields ~2x speedup on Clang/Windows [S1, unofficial].
*   **Overflow:** Coordinates exceeding `BOUNDS` must be handled according to `_discard_overflow` [S1, unofficial].

## Subtopics

### [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md)
A Morton-coded Sparse Voxel Tree is a flat-array sparse voxel tree that uses Morton codes for spatial ordering and an 8-bit mask per node to track children. It avoids recursive pointers by storing nodes in contiguous blocks of 8 elements, indexed via `node_format::_block_index`. This structure builds on [Sparse Voxel Tree (SVO Tree)](4983-sparse-voxel-tree-svo-tree.md) and differs from [Hashed-Octree](7486-hashed-octree.md), [Spatio-Temporal Sparse Voxel Octree (STSVO)](7493-spatio-temporal-sparse-voxel-octree-stsvo.md), and [n-Dimensional Sparse Voxel Tree (nD SVT)](7999-n-dimensional-sparse-voxel-tree-nd-svt.md) by using contiguous blocks of 8 indexed via `_block_index` instead of hash tables or spatio-temporal tracking.

### [Hashed-Octree](7486-hashed-octree.md)
The Hashed-Octree is a sparse voxel representation that combines octrees with hash tables to provide constant-time data access complexity for huge 3D environments, addressing the $O(d)$ access time of standard Sparse Voxel Octrees (SVOs). It eliminates the need for explicit pointers by storing octants in a hash map using a unique 53-bit locational code key, allowing intermediate octants to be omitted. Unlike its siblings which may rely on Morton codes or pointer trees, this method uses a direct hash of coordinate-aligned root keys for top-level access, reducing data access complexity from $O(d)$ to $O(d_b)$ where $d_b$ is the depth of internal octrees (typically $d=1$).

### [Spatio-Temporal Sparse Voxel Octree (STSVO)](7493-spatio-temporal-sparse-voxel-octree-stsvo.md)
The Spatio-Temporal Sparse Voxel Octree (STSVO) is a hierarchical data structure for representing the geometry of time-variable 3D scenes by treating the scene as a 4D static space (3 spatial dimensions + 1 time dimension). It uses a single SVO-like structure instead of a set of separate SVOs for each time frame. Compared to storing each time frame as a separate SVO (SVO set), STSVO provides a more compact binary representation, achieving 1.73 to 2.08 times more compactness.

### [n-Dimensional Sparse Voxel Tree (nD SVT)](7999-n-dimensional-sparse-voxel-tree-nd-svt.md)
The n-Dimensional Sparse Voxel Tree (nD SVT) is a hierarchical data structure that generalizes Sparse Voxel Octrees (SVOs) to $n$-dimensional grids, where $n > 3$, to achieve a more compact binary representation of voxelized scenes. It solves the problem of storage efficiency in 3D volume rendering by upscaling the dimensionality of the scene from 3D to $m$D ($m > 3$), reducing the total number of nodes and pointers required to represent the geometry. Unlike its sibling [Morton-coded Sparse Voxel Tree](5212-morton-coded-sparse-voxel-tree.md) which remains strictly 3D, or [Hashed-Octree](7486-hashed-octree.md) which uses hashing for flat storage, the nD SVT trades increased per-node complexity (larger child masks) for a significantly reduced tree depth and node count.

### [Least Squares Surface Simplification](11029-least-squares-surface-simplification.md)
Least Squares Surface Simplification is a method for reducing the resolution of a Sparse Voxel Octree (SVO) by finding the best-fit scalar field values for a coarser voxel level. It solves the problem of data reduction by minimizing the energy difference between the prolonged coarse field and the original fine field, formulated as a least squares problem. It sets itself apart from sibling methods like Morton-coded SVOs by explicitly optimizing field values to minimize approximation error rather than relying solely on structural sparsity or hashing.

## Operating characteristics
*   **Time Complexity:** $O(\text{MAX\_DEPTH})$ for allocation, retrieval, and deallocation. Since `MAX_DEPTH` is constant (3 to 10), operations are effectively $O(1)$ with a small constant factor [S1, unofficial].
*   **Memory Complexity:** $O(N)$ where $N$ is the number of allocated voxels. Each block holds exactly 8 elements (nodes or voxels). Overhead includes block metadata and free lists [S1, unofficial].
*   **Measured Performance:** Requires about 5 bytes per voxel with color, normal, and contour, or 6.67 bytes per finest-level sample once the mip hierarchy is counted [S2]. If only surfaces are encoded, memory usage drops to $O(N^2)$ [S2].
*   **Strengths:** Efficient for storage and rendering of sparse data. Dynamic updates are facilitated by block-based storage with relative pointers [S2].
*   **Limitations:** Traversal is worse than hierarchical-grid methods by as much as 60% [S2]. Some experts consider SVOs "good only for storage and rendering, and not even great at those" [S2]. Bulk allocation estimation is difficult; pre-allocation is avoided to prevent performance degradation from overestimation [S1, unofficial].

## Background
The Sparse Voxel Octree (SVO) was popularized for real-time rendering by Laine and Karras (2010) [S7, unofficial]. It is used in techniques such as Interactive Indirect Illumination Using Voxel Cone Tracing, where it stores pre-filtered hierarchical voxel representations of scene geometry on the GPU to estimate visibility and integrate indirect energy [S6]. The structure is also foundational in Neural Sparse Voxel Fields, which learn sparse voxel grids progressively [S3, unofficial], and in PlenOctrees for NeRF-SH conversion [S4, unofficial].

## Sources
[S1] SeveriSuominen/rapid-svo, https://github.com/SeveriSuominen/rapid-svo, fetched 2026-10-26 (unofficial)
[S2] https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, https://github.com/theHimanshuShekhar/bhayanakvoxel/blob/research/voxel-data-structures/docs/research/voxel-data-structures.md, fetched 2026-10-28
[S3] Faster Neural Radiance Fields Inference — Chris Choy, https://chrischoy.org/posts/nerfs/, fetched 2026-10-29 (unofficial)
[S4] marceline-cramer/svo-cpu, https://github.com/marceline-cramer/svo-cpu, fetched 2026-10-27 (unofficial)
[S5] Voxel Compression - GitHub Pages, https://eisenwave.github.io/voxel-compression-docs/introduction.html, fetched 2026-10-27
[S6] Interactive Indirect Illumination Using Voxel Cone Tracing, https://doi.org/10.1111/j.1467-8659.2011.02063.x, fetched 2026-10-26
[S7] PDF Real-time Rendering and Construction of Signed Distance Fields, https://jambuttenshaw.github.io/devportfolio/Dissertation.pdf, fetched 2026-10-28 (unofficial)
