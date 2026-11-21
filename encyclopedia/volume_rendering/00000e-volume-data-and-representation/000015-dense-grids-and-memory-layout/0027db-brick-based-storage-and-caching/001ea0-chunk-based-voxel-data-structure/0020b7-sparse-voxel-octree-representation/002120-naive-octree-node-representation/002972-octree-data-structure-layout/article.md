# Octree Data Structure Layout

Also called: Octree Class Layout

## Overview
This article defines the memory layout and class structure for a naive octree representation, serving as the foundational implementation for sparse voxel octrees. It solves the problem of organizing hierarchical volumetric data in contiguous memory while maintaining access to bounding box metadata and voxel attributes. It differs from its parent [Naive Octree Node Representation](2445-naive-octree-node-representation.md) by explicitly defining the container class fields (`min`, `max`, `size`, `gridlength`) and the storage mechanism (`std::vector<Node>`), and it differs from its sibling [Bitmask Leaf Compression](3603-bitmask-leaf-compression.md) by storing full node structures rather than compressing leaf status into bitmasks.

## How it works
### Inputs and Outputs
*   **Input**: None (structural definition).
*   **Output**: An in-memory representation of the octree hierarchy and associated voxel data.

### Data Structures
The `Octree` class maintains the following fields [S1, unofficial]:

1.  **Bounding Box and Grid Metadata**:
    *   `vec3 min`: The minimum coordinate of the root bounding box.
    *   `vec3 max`: The maximum coordinate of the root bounding box.
    *   `vec3 size`: The dimensions of the root bounding box (`max - min`).
    *   `size_t gridlength`: The resolution of the finest level (number of voxels per side). Default is 128.
2.  **Statistics**:
    *   `size_t n_nodes`: Count of allocated nodes.
    *   `size_t n_data`: Count of allocated voxel data elements.
3.  **Data Storage**:
    *   `VoxelData* data`: A pointer to a contiguous array storing voxel attributes (e.g., color, density).
    *   `std::vector<Node> nodes`: A contiguous vector storing the octree nodes.

**Node Layout**:
The `Node` structure is implied by the traversal logic in [S1, unofficial] to contain:
*   Pointers or indices to children (8 children per internal node).
*   A pointer or index to `VoxelData` (for leaf nodes).
*   State flags (e.g., `nextchild` for traversal state).

### Initialization
The default constructor initializes the root node with the following values [S1, unofficial]:
*   `min` = $(0, 0, 0)$
*   `max` = $(1, 1, 1)$
*   `size` = $(1, 1, 1)$
*   `gridlength` = 128

### Accessing the Root
The root node is stored at the end of the `nodes` vector to facilitate dynamic growth without invalidating pointers [S1, unofficial].
*   `getRootNode()` returns `&nodes.back()`.

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class VoxelData:
    """Placeholder for voxel attributes (e.g., color, density)."""
    pass

class Node:
    """
    Represents a node in the octree.
    Contains children pointers and data pointer.
    """
    def __init__(self):
        self.children = [None] * 8  # 8 children for an octree
        self.data = None            # Pointer to VoxelData if leaf
        self.nextchild = 0          # Traversal state: which child to visit next

class Octree:
    def __init__(self):
        # Bounding box and grid metadata
        self.min = Vec3(0, 0, 0)
        self.max = Vec3(1, 1, 1)
        self.size = Vec3(1, 1, 1)
        self.gridlength = 128
        
        # Statistics
        self.n_nodes = 0
        self.n_data = 0
        
        # Data storage
        self.data = None  # Pointer to voxel data array
        self.nodes = []   # Vector of Node objects
        
        # Initialize root node
        root = Node()
        self.nodes.append(root)
        self.n_nodes = 1

    def get_root_node(self):
        """Returns the root node, stored at the back of the nodes vector."""
        return self.nodes[-1]

    def allocate_node(self):
        """Allocates a new node and returns it."""
        new_node = Node()
        self.nodes.append(new_node)
        self.n_nodes += 1
        return new_node

    def allocate_voxel_data(self):
        """Allocates a new voxel data element."""
        # In a real implementation, this would manage the 'data' array
        new_data = VoxelData()
        self.n_data += 1
        return new_data
```

### Edge Cases and Pitfalls
*   **Memory Contiguity**: Storing nodes in a `std::vector` provides cache locality but requires reallocation if the vector grows. Pointers to nodes must be managed carefully during reallocation (e.g., using indices instead of raw pointers, or ensuring the vector is reserved).
*   **Root Position**: The root is at the end of the vector. This is a specific design choice in [S1, unofficial] that allows efficient appending but requires `back()` access rather than `front()`.
*   **Precision**: When calculating bounding box subdivisions, floating-point precision errors can accumulate, especially at high depths.

## Subtopics

### [Octree Voxel Storage](6922-octree-voxel-storage.md)
Octree Voxel Storage is a hierarchical data structure used to store voxel data efficiently by grouping adjacent voxels of the same type into larger cells, reducing memory usage for sparse or uniform regions compared to flat arrays. It differs from flat arrays by trading $O(1)$ direct access for logarithmic traversal time to save space, and differs from instancing by allowing the culling of hidden faces and merging of coplanar geometry rather than rendering all cubes regardless of occlusion.

### [OctreeVoxelGrid](14633-octreevoxelgrid.md)
OctreeVoxelGrid is a sparse, multi-resolution voxel data structure that implements the `VoxelGrid` protocol, allowing it to replace uniform grids in existing engine code without modification. It solves the memory inefficiency of dense grids for sparse scenes by storing only active regions, achieving $O(\text{active cells} \times \text{layers})$ memory complexity instead of $O(\text{grid}^3)$. It distinguishes itself from sibling structures by supporting dynamic refinement/coarsening policies and inheriting parent values to children upon split to avoid interpolation artifacts at refinement boundaries.

### Octree Memory Size Calculation
Calculates the maximum memory footprint of a 100% dense octree by summing powers of 8 up to the maximum depth and multiplying by octant size [S2, unofficial].
*   **Formula**: $S(n) = \frac{8^{n+1} - 1}{7}$ where $n$ is the highest power of 8 (depth).
*   **Total Bytes**: $S(n) \times \text{octantSize}$.
*   **Parameters**:
    *   $n$: highest power of 8, up to 24 (after which precision errors become a problem) [S2, unofficial].
    *   `octantSize`: size of a single octant in bytes [S2, unofficial].
    *   `depth`: octree depth [S2, unofficial].
*   **Pitfalls**: Precision errors occur for $n > 24$ [S2, unofficial].
*   **Pseudocode**:
    ```python
    def sump8(n):
        return (8**(n+1) - 1) / 7

    def full_octree_size(octant_size, depth):
        return sump8(depth) * octant_size
    ```

### Sparse Voxel Octree Memory Management
A memory management system for the octree using a flat array and free lists for branches and attributes [S3, unofficial].
*   **Allocation**: Check the free structure memory list; if empty, append to the data array, otherwise reuse the last free pointer. Record updates for the branch pointer and the last element of the branch (ptr + 7) [S3, unofficial].
*   **Deallocation**: Free a branch by adding its pointer to the free structure memory list and recursively freeing its children. Free attributes by adding their pointer to the free attribute memory list [S3, unofficial].

### Chunk-Column-Limit Representation
A volumetric terrain representation that divides data into three levels (chunks, columns, limits) to reduce memory usage compared to storing every voxel in a 3D grid, addressing the problem that an $N_t \times M_t \times L_t$ terrain stores values for $N_t \times M_t \times L_t$ voxels which is not optimal for interactive environments [S4].
*   **Layout**:
    1.  **Chunks**: The highest unit of volume, defined as a 3D portion of terrain with size $N \times M \times L$ voxels.
    2.  **Columns**: Vertical structures within chunks.
    3.  **Limits**: Boundaries defining material transitions.
*   **Comparison**: Unlike traditional voxel representation which stores data for every voxel in a 3D regular grid, this method reduces memory size by using stacked material structures and optimized representation designed to accelerate generation and reduce memory footprint [S4].

### Multi-resolution Octree Data Indexing
A data structure layout that allows efficient retrieval of voxels in coarser or finer grids by storing block offsets and using integer coordinates at the finest resolution, scaled by a factor determined by the tree depth and voxel depth [S5].
*   **Cell-Centred Voxels**: Sampling locations are placed at the centre of the enclosing octant, ensuring parent voxels are the linear combination of their children [S5].
*   **Formula**:
    *   Scaling factor: $s = 2^{d-l}$, with $d - \log_2(8) \le l \le d$.
    *   Index coordinates: $x^l_w = s \lfloor x^d_w / s \rfloor$.
*   **Parameters**:
    *   $d$: maximum depth of the octree tree.
    *   $l$: desired voxel depth (scale level).
    *   $x^d_w$: integer coordinates of the voxel at the finest resolution (depth $d$) in the world frame.
    *   $x^l_w$: integer coordinates of the voxel at scale $l$ in the world frame.
    *   `offset(s)`: precomputed function returning the starting position of the local sub-grid for scale $s$.
    *   $v^l$: number of voxels per side at scale $l$.
    *   $x^l_{bx}, x^l_{by}, x^l_{bz}$: block index coordinates components.

## Operating characteristics
*   **Memory Complexity**: The memory usage depends on the sparsity of the data. In the worst case (100% dense), the memory footprint is proportional to the sum of powers of 8 up to the maximum depth, calculated as $S(n) \times \text{octantSize}$ [S2, unofficial]. For sparse data, the memory usage is significantly lower, proportional to the number of active nodes [S1, unofficial].
*   **Access Time**: Accessing a specific voxel requires traversing the octree from the root to the leaf, resulting in $O(\log N)$ time complexity, where $N$ is the number of voxels along one dimension [S1, unofficial].
*   **Strengths**:
    *   Efficient storage for sparse data.
    *   Hierarchical structure allows for efficient culling and level-of-detail rendering.
    *   Contiguous memory layout (vector of nodes) improves cache performance during traversal.
*   **Limitations**:
    *   Traversal overhead compared to flat arrays.
    *   Potential for pointer invalidation if the node vector is reallocated (mitigated by storing root at the end and using indices or careful pointer management).
    *   Precision errors in memory size calculations for very deep trees ($n > 24$) [S2, unofficial].

## Background
The octree data structure layout described here is based on implementations found in open-source projects for CPU-based voxel raycasting and sparse voxel octree generation. The specific layout with the root node at the end of the vector is documented in the `cpu_voxel_raycaster` project [S1, unofficial]. The memory size calculation formula is derived from the `voctopus` project [S2, unofficial]. The memory management techniques using free lists are from `unity-sparse-voxel-octrees` [S3, unofficial]. The chunk-column-limit representation is described in research on procedural volumetric terrain generation [S4]. The multi-resolution indexing scheme is detailed in work on adaptive-resolution octree-based volumetric SLAM [S5].

## Sources
[S1] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S2] nphyx/voctopus, https://github.com/nphyx/voctopus, fetched 2026-10-26 (unofficial)
[S3] xandergos/unity-sparse-voxel-octrees, https://github.com/xandergos/unity-sparse-voxel-octrees, fetched 2026-10-26 (unofficial)
[S4] Procedural approach to volumetric terrain generation, https://zenodo.org/records/18429902, fetched 2026-10-27
[S5] Adaptive-resolution octree-based volumetric SLAM, https://www.doc.ic.ac.uk/~sleutene/publications/Vespa_3DV19.pdf, fetched 2026-10-29
