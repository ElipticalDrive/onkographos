# Octree Data Structure Layout
Also called: Octree Class Layout

## Overview
This article defines the memory layout and class structure for a naive octree representation used in CPU-based voxel rendering. It solves the problem of organizing hierarchical voxel data in contiguous memory to facilitate traversal. It differs from its parent [Naive Octree Node Representation](2445-naive-octree-node-representation.md) by encapsulating the node structure within a higher-level `Octree` class that manages global state, bounds, and the node vector, rather than just defining the node itself. It differs from [Bitmask Leaf Compression](3603-bitmask-leaf-compression.md) by storing all nodes explicitly in a vector without compressing leaf status into bitmasks.

## How it works
### Inputs and Outputs
- **Input**: Configuration parameters for the octree bounds and resolution; schema definition for voxel data size.
- **Output**: An `Octree` object containing a `std::vector<Node>` and a `VoxelData` array, ready for traversal.

### Data Structures
The `Octree` class aggregates the following fields [S1, unofficial]:
- `vec3 min`: The minimum corner of the root node's bounding box.
- `vec3 max`: The maximum corner of the root node's bounding box.
- `vec3 size`: The dimensions of the root node (`max - min`).
- `size_t gridlength`: The resolution (number of voxels) along one axis of the root node.
- `size_t n_nodes`: The count of nodes currently in the tree.
- `size_t n_data`: The count of voxel data entries.
- `VoxelData* data`: A pointer to a contiguous array storing the actual voxel attributes (e.g., color, density).
- `std::vector<Node> nodes`: A dynamic array storing all octree nodes. The root node is stored at the end of this vector (`nodes.back()`).

The `Node` structure (inherited from the parent article) contains pointers to children and leaf status flags.

### Initialization
The default constructor initializes the `Octree` with the following values [S1, unofficial]:
- `min` = $(0, 0, 0)$
- `max` = $(1, 1, 1)$
- `size` = $(1, 1, 1)$
- `gridlength` = $128$

### Accessing the Root
The root node is accessed via `getRootNode()`, which returns a pointer to the last element of the `nodes` vector [S1, unofficial]:
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```cpp
Node* getRootNode() {
    return &nodes.back();
}
```

### Memory Footprint Calculation
For a 100% dense octree of depth $n$, the total number of nodes is given by the geometric series sum $S(n)$. The total memory footprint is calculated by multiplying this sum by the size of a single octant (node) in bytes [S2, unofficial].

**Formula**:
$$ S(n) = \frac{8^{n+1} - 1}{7} $$

Where:
- $n$ is the maximum depth of the tree.
- The total bytes = $S(n) \times \text{octantSize}$.

**Parameters**:
- $n$: Maximum depth. Precision errors in floating-point calculations occur for $n > 24$ [S2, unofficial].
- `octantSize`: Size of a single node in bytes.

### Pseudocode
The following pseudocode demonstrates the initialization and memory calculation logic derived from the sources [S1, unofficial][S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Octree:
    def __init__(self):
        # Default initialization [S1, unofficial]
        self.min = (0.0, 0.0, 0.0)
        self.max = (1.0, 1.0, 1.0)
        self.size = (1.0, 1.0, 1.0)
        self.gridlength = 128
        
        self.n_nodes = 0
        self.n_data = 0
        self.data = None  # VoxelData*
        self.nodes = []   # std::vector<Node>

    def get_root_node(self):
        # Root is at the end of the vector [S1, unofficial]
        if not self.nodes:
            return None
        return self.nodes[-1]

def calculate_dense_octree_memory(depth, octant_size_bytes):
    """
    Calculate memory for a 100% dense octree [S2, unofficial].
    """
    if depth > 24:
        raise ValueError("Precision errors occur for depth > 24")
    
    # Geometric series sum: (8^(n+1) - 1) / 7
    # Using integer arithmetic or high-precision float to avoid errors
    num_nodes = (8**(depth + 1) - 1) // 7
    
    total_bytes = num_nodes * octant_size_bytes
    return total_bytes
```

## Subtopics

### [Octree Voxel Storage](6922-octree-voxel-storage.md)
Octree Voxel Storage is a hierarchical data structure that groups adjacent voxels of the same type into larger cells to reduce memory usage for sparse or uniform regions. It differs from flat arrays by trading $O(1)$ direct access for logarithmic traversal time to save space, and differs from instancing by allowing culling of hidden faces and merging of coplanar geometry.

### Octree Memory Size Calculation
This variant focuses on the analytical calculation of the maximum memory footprint for a fully dense octree. It uses the closed-form geometric series sum $S(n) = \frac{8^{n+1}-1}{7}$ to determine the number of nodes, then multiplies by the octant size [S2, unofficial]. It is useful for pre-allocating buffers or estimating memory limits, with a practical depth limit of 24 due to floating-point precision constraints [S2, unofficial].

## Operating characteristics
- **Time Complexity**: Accessing the root is $O(1)$. Traversal depends on the tree depth and sparsity.
- **Memory Complexity**: For a dense tree, memory scales as $O(8^d)$ where $d$ is depth. For sparse trees, it scales with the number of non-empty nodes.
- **Strengths**: Simple implementation; contiguous memory layout for nodes (vector) can improve cache locality during traversal compared to pointer-chasing linked lists.
- **Limitations**: High memory overhead for sparse data compared to compressed variants like bitmask compression. Precision errors in memory calculation for very deep trees ($n > 24$) [S2, unofficial].
- **When to use**: When implementation simplicity is prioritized and data is moderately dense, or as a baseline for comparison.

## Background
The `Octree` class layout described here originates from the `cpu_voxel_raycaster` project by Forceflow [S1, unofficial]. It serves as a foundational data structure for CPU-based ray marching through voxel grids. The memory calculation method is derived from the `voctopus` library by nphyx [S2, unofficial], which was designed for WebGL texture-based octree storage.

## Sources
[S1] cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] voctopus, https://github.com/nphyx/voctopus, fetched 2026-10-26 (unofficial)
