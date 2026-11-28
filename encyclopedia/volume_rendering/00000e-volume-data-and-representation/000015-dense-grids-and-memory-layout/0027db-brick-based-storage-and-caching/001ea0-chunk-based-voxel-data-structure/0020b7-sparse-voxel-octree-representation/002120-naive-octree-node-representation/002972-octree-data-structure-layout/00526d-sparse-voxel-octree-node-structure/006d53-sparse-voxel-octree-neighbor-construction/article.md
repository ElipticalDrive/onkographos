# Sparse Voxel Octree Neighbor Construction

Also called: GPU Neighbor Linking, Neighbor Construction

## Overview
This method computes the adjacency relationships (neighbors) for nodes in a Sparse Voxel Octree (SVO) on the GPU, enabling efficient traversal and meshing by linking sibling and cousin nodes [S1, unofficial]. It differs from CPU-based neighbor construction by utilizing parallel CUDA kernels and lookup tables to resolve spatial adjacency based on Morton codes, avoiding expensive geometric intersection tests [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Input:** An array of SVO nodes (`d_nodeArray`), the total number of nodes (`nNodes`), the tree depth, and the index of the global top node.
*   **Output:** The `neighbors` array within each `SVONode` struct, populated with indices of adjacent nodes in 27 directions (3x3x3 neighborhood excluding self).

**Data Structures**
The node structure is defined as [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
struct SVONode {
    uint32_t mortonCode;      // 32-bit Morton key
    bool isLeaf;              // Leaf flag
    Eigen::Vector3f origin;   // 3D position
    float width;              // Node size
    unsigned int parent;      // Index of parent node
    unsigned int childs[8];   // Indices of 8 children
    unsigned int neighbors[27]; // Indices of 27 neighbors
};
```
The algorithm relies on two precomputed lookup tables: `neighbor_LUTparent` and `neighbor_LUTchild`. These tables map the local Morton key (lower 3 bits) and a relative direction index (0-26) to the correct child index within a parent or cousin node [S1, unofficial].

**Parameters**
*   Block size: `blockSize.x = 32`, `blockSize.y = 32` [S1, unofficial].
*   Mask: `LOWER_3BIT_MASK` (value `0x7`) used to extract the local Morton key [S1, unofficial].
*   Null value: `UINT32_MAX` indicates no neighbor exists [S1, unofficial].

**Steps**
1.  **Initialize Root:** Set the global top node index in device constant memory using `cudaMemcpyToSymbol(d_topNodeIdx, &idx, sizeof(size_t))` [S1, unofficial].
2.  **Root Neighbor Setup:** Launch the `findNeighbors<true>` kernel with a 1x1 grid and block to set the root node's neighbor reference [S1, unofficial].
3.  **Iterate Levels:** Loop from `treeDepth - 2` down to `0` to process neighbors for each level, excluding the bottom-most leaf level which is handled implicitly or separately [S1, unofficial].
4.  **Configure Kernel:** For the current level, calculate the grid dimensions:
    $$ \text{gridSize}.x = \lceil \frac{nNodes}{\text{blockSize}.x} \rceil $$
    where `blockSize.x = 32` and `blockSize.y = 32` [S1, unofficial].
5.  **Launch Kernel:** Launch `findNeighbors<false>` for the current level nodes [S1, unofficial].
6.  **Thread Execution:** Inside the kernel, each thread performs the following for a specific node and direction:
    a. Compute thread indices: `tid_x = threadIdx.x + blockIdx.x * blockDim.x`, `tid_y = threadIdx.y + blockIdx.y * blockDim.y` [S1, unofficial].
    b. Check bounds: `if (tid_x < nNodes && tid_y < 27)` [S1, unofficial].
    c. Load current node `t` from `d_nodeArray[preESumTreeNodes + tid_x]` and its parent `p` from `d_nodeArray[t.parent]` [S1, unofficial].
    d. Extract local Morton key: `key = t.mortonCode & LOWER_3BIT_MASK` [S1, unofficial].
    e. Look up parent's neighbor index: `p_neighborIdx = p.neighbors[neighbor_LUTparent[key][tid_y]]` [S1, unofficial].
    f. If `p_neighborIdx != UINT32_MAX`, load the neighbor node `h` and set the current node's neighbor: `t.neighbors[tid_y] = h.childs[neighbor_LUTchild[key][tid_y]]` [S1, unofficial].
    g. If `p_neighborIdx == UINT32_MAX`, set `t.neighbors[tid_y] = UINT32_MAX` [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Precomputed lookup tables (conceptual)
# neighbor_LUTparent[local_key][direction] -> index in parent's neighbor array
# neighbor_LUTchild[local_key][direction] -> index in cousin's child array

def initialize_root_neighbor(top_node_idx):
    # Set device symbol for top node index
    cudaMemcpyToSymbol("d_topNodeIdx", &top_node_idx, sizeof(size_t))
    # Launch kernel with 1x1 grid/block to handle root
    findNeighbors_kernel[1, 1](is_root=True)

def build_neighbors_level(nNodes, tree_depth, current_level):
    if current_level < 0:
        return
    
    blockSize = (32, 32)
    gridSize = ((nNodes + blockSize[0] - 1) // blockSize[0], 1)
    
    findNeighbors_kernel[gridSize, blockSize](is_root=False)
    
    # Recursively or iteratively process next level down
    build_neighbors_level(nNodes, tree_depth, current_level - 1)

def findNeighbors_kernel(is_root):
    tid_x = threadIdx.x + blockIdx.x * blockDim.x
    tid_y = threadIdx.y + blockIdx.y * blockDim.y
    
    if tid_x < nNodes and tid_y < 27:
        t = d_nodeArray[preESumTreeNodes + tid_x]
        
        if is_root:
            # Handle root neighbor setup specifically
            # Implementation details for root are implicit in [S1, unofficial]
            pass
        else:
            p = d_nodeArray[t.parent]
            key = t.mortonCode & 0x7 # LOWER_3BIT_MASK
            
            # Lookup parent's neighbor index for the relative direction
            p_neighbor_idx = p.neighbors[neighbor_LUTparent[key][tid_y]]
            
            if p_neighbor_idx != UINT32_MAX:
                h = d_nodeArray[p_neighbor_idx]
                # Set current node's neighbor to the corresponding child of the cousin
                t.neighbors[tid_y] = h.childs[neighbor_LUTchild[key][tid_y]]
            else:
                t.neighbors[tid_y] = UINT32_MAX
```

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of nodes, as each node is processed by a single thread in the parallel kernel [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for storing the neighbor arrays, adding 27 `unsigned int` entries per node [S1, unofficial].
*   **Performance:** The method is optimized for GPU execution using CUDA, leveraging parallelism to construct neighbors for all nodes simultaneously [S1, unofficial].
*   **Strengths:** Efficient on GPU; avoids complex geometric calculations; uses simple table lookups [S1, unofficial].
*   **Limitations:** Requires precomputed lookup tables; relies on Morton code properties; specific implementation details for the root node and leaf levels are not fully detailed in the source [S1, unofficial].

## Background
This method originates from GPU-based Sparse Voxel Octree implementations, specifically those designed for real-time volume rendering and meshing [S1, unofficial]. It addresses the need for fast neighbor access during ray marching or marching cubes algorithms on the GPU, where random access to neighbor nodes is critical for performance [S1, unofficial].

## Sources
[S1] GitHub - Alan-Leo-Wong/GPU-SVO, https://github.com/Alan-Leo-Wong/GPU-SVO, fetched 2026-10-29 (unofficial)
