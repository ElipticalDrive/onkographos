# Implicit Octree Storage

Also called: Implicit Order Storage

## Overview
Implicit Octree Storage is a memory-efficient serialization technique that represents a sparse voxel octree without explicit pointers by sorting the tree nodes in a deterministic implicit order, such as breadth-first order [S1]. It solves the problem of high memory overhead and pointer-chasing costs associated with traditional pointer-based octrees, making the structure suitable for streaming, persistent storage, or out-of-core management [S1]. Unlike sibling methods like [OpenVDB (VDB)](1508-openvdb-vdb.md) which use linked lists or compressed bitmaps for traversal, or [Sparse Voxel Brick Set](11886-sparse-voxel-brick-set.md) which uses hybrid brick structures, Implicit Octree Storage relies purely on index arithmetic or array position to define topology [S1]. Its primary trade-off is that while it minimizes storage size, the resulting linear array is not suitable for efficient direct traversal because nodes near each other in the list may have widely different spatial locations, preventing spatial locality optimizations [S1][S4, unofficial].

## How it works
The method converts a hierarchical tree structure into a linear sequence of nodes. The core mechanism is the elimination of child/parent pointers by replacing them with implicit relationships derived from the node's position in the array or via arithmetic formulas.

### Inputs and Outputs
- **Input:** A fully constructed sparse voxel octree (or DAG) where each node contains metadata (type, occupancy) and data (voxel values, min/max bounds).
- **Output:** A contiguous linear array (or file stream) of node structures, ordered implicitly (e.g., breadth-first).

### Data Structures
The specific layout depends on the variant, but the general concept stores node attributes in a flat array.

**General Node Layout (Implicit):**
- `index`: Integer index in the linear array (implicit pointer).
- `data`: Voxel value or metadata (e.g., color, density, min/max bounds).
- `type`: Node type (empty, leaf, interior) if not inferred from context.

**Specific Layout (Linear Octree with Parent/Child Index Arithmetic) [S4, unofficial]:**
- `sChunk *buffer`: Pointer to a cached vertex buffer (implementation-specific, not part of the implicit topology itself).
- `u32 data`: Stores voxel color or scalar value.

### Numbered Steps
1. **Traverse the Tree:** Perform a traversal of the octree (typically breadth-first or depth-first) to visit every node.
2. **Serialize Nodes:** For each visited node, extract its data fields (occupancy, value, bounds) and append them to a linear buffer.
3. **Assign Implicit Indices:** Assign each node a sequential integer index based on its position in the buffer (0, 1, 2, ...).
4. **Discard Pointers:** Remove all explicit child and parent pointers from the node structures.
5. **Store/Stream:** Write the linear buffer to disk or memory.

### Formulas
For the variant using **Parent/Child Index Arithmetic** [S4, unofficial]:
- The root node is at index `-1`.
- The parent of a node at index $n$ is calculated as:
  $$ P(n) = \lfloor n / 8 \rfloor - 1 $$
- The $c$-th child (where $c \in [0, 7]$) of a node at index $n$ is calculated as:
  $$ C(n, c) = (n + 1) \times 8 + c $$

### Parameters
- **Ordering:** Breadth-first order is cited as a common implicit order [S1].
- **Compression:** Lossless compression libraries (e.g., LZO) can be applied to the serialized data stream [S2].

### Edge Cases and Pitfalls
- **Spatial Locality:** Nodes that are adjacent in the linear array may correspond to spatially distant regions in the 3D volume. This prevents efficient spatial caching or direct coordinate-to-index mapping [S4, unofficial].
- **Traversal Efficiency:** The structure is not suitable for efficient runtime traversal because the parent-child relationships must be computed via arithmetic or searched for, rather than followed via pointers [S1].
- **Empty Branches:** In variants like the Integral Octree, empty branches must be handled carefully to avoid generating holes; this may require looping through edge voxels to verify emptiness [S3].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Implicit Octree Storage: Breadth-First Serialization
# Note: This pseudocode illustrates the serialization process.
# It does not implement the arithmetic lookup formulas directly,
# as those are used during reconstruction/traversal.

class Node:
    def __init__(self, data, is_leaf):
        self.data = data
        self.is_leaf = is_leaf
        self.children = [] # List of Node objects

def serialize_octree(root: Node) -> list:
    """
    Serializes an octree into a linear list using breadth-first order.
    Returns a list of serialized node data.
    """
    serialized_nodes = []
    queue = [root]
    
    while queue:
        current_node = queue.pop(0)
        
        # Serialize current node
        # In a real implementation, this might include bit-packing
        node_record = {
            'data': current_node.data,
            'is_leaf': current_node.is_leaf
        }
        serialized_nodes.append(node_record)
        
        # Add children to queue for BFS
        if not current_node.is_leaf:
            for child in current_node.children:
                if child is not None: # Skip null children (empty branches)
                    queue.append(child)
                    
    return serialized_nodes

def get_parent_index(node_index: int) -> int:
    """
    Calculates the parent index for a node in a linear octree
    using the arithmetic formula from [S4, unofficial].
    Root is at index -1.
    """
    if node_index == -1:
        return -1
    return (node_index // 8) - 1

def get_child_index(parent_index: int, child_id: int) -> int:
    """
    Calculates the child index for a node in a linear octree
    using the arithmetic formula from [S4, unofficial].
    child_id must be in [0, 7].
    """
    return (parent_index + 1) * 8 + child_id
```

## Subtopics

### Out-of-Core Octree Storage
A variant that stores the volume octree in an out-of-core structure, typically using a database management system like Berkeley DB, to handle datasets larger than available RAM [S2]. It often employs lossless compression libraries (e.g., LZO) to reduce I/O bandwidth [S2].

### Integral Octree
A data structure that attempts to preserve properties from both "Octree Max/Min" and "Integral Volumes" [S3]. It stores the sum of leaf values in interior nodes to allow constant-time volume sum queries [S3].
- **Pitfalls:** It consumes more memory for saving leaves into a 1D-array. If the isolevel is lower than the noise threshold, it is more affected by low-level noise than Octree Max/Min because it checks the sum of leaves rather than the max value. It must loop through all voxels at the edges of an empty branch to avoid generating holes [S3].
- **Usage:** Beneficial for multiresolution direct volumetric rendering and blurring the volume for noise removal [S3].

### Linear Octree with Parent/Child Index Arithmetic
A method to store an octree hierarchy in a linear array without pointers, inferring parent-child relationships via integer arithmetic on node indices [S4, unofficial].
- **Formula:** Parent of node $n$ is $\lfloor n / 8 \rfloor - 1$. Child $c$ of node $n$ is $(n + 1) \times 8 + c$.
- **Pitfalls:** Nodes near each other in the linear list have wildly different spatial locations, making spatial locality poor and preventing direct calculation of index from coordinate [S4, unofficial].
- **Layout:** Node structure contains `sChunk *buffer` (pointer to cached vertex buffer) and `u32 data` (stores voxel color) [S4, unofficial].

## Operating characteristics
- **Memory Footprint:** Significantly reduced compared to pointer-based octrees because explicit pointers are eliminated [S1]. The size is proportional to the number of nodes in the tree.
- **Storage Efficiency:** Suitable for streaming and persistent storage due to the linear, contiguous layout [S1].
- **Traversal Cost:** High. The structure is not suitable for efficient traversal because spatial locality is poor and parent-child relationships must be computed or searched for [S1][S4, unofficial].
- **Construction Cost:** Requires a full traversal and serialization step.
- **When to Use:** When memory bandwidth for storage/streaming is the bottleneck, or when the tree needs to be persisted to disk.
- **When Not to Use:** When real-time traversal performance is critical, or when spatial locality is required for cache efficiency [S1][S4, unofficial].

## Background
The concept of storing octrees without pointers by sorting them in an implicit order was introduced by Schnabel and Klein [2006] in the context of point cloud geometry discretized into an octree [S1]. The goal was to decrease memory consumption for large datasets (e.g., $4K^3$ resolution) [S1]. The approach is related to common subtree merging techniques used in cartographic images [S1].

## Sources
[S1] High resolution sparse voxel DAGs, https://www.researchgate.net/publication/262367808_High_resolution_sparse_voxel_DAGs, fetched 2026-10-27
[S2] Towards Advanced Volumetric Display of the Human Musculoskeletal System, https://doi.org/10.2312/localchapterevents/italchap/italianchapconf2008/021-025, fetched 2026-10-26
[S3] A Comparative Study about Data Structures Used for Efficient Management of Voxelised Full-Waveform Airborne LiDAR Data during 3D Polygonal Model Creation, https://doi.org/10.3390/rs13040559, fetched 2026-10-26
[S4] Weekend Project – Voxel Octree – Bit-Hacks Pixel Dungeon, https://bit-hack.net/2014/05/04/weekend-project-voxel-octree-2/, fetched 2026-10-24 (unofficial)
