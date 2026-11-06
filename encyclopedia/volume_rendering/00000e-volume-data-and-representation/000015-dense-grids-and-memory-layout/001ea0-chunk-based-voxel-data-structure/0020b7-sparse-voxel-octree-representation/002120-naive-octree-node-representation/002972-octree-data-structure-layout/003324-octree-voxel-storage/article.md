# Octree Voxel Storage

Also called: Hierarchical Voxel Storage, Quadtree (2D equivalent)

## Overview
Octree Voxel Storage is a hierarchical data structure that groups adjacent voxels of the same type into larger uniform cells, significantly reducing memory usage for sparse or uniform regions compared to flat arrays [S1, unofficial]. It differs from flat arrays by trading $O(1)$ direct access for logarithmic traversal time to save space, and differs from instancing by allowing the culling of hidden faces and merging of coplanar geometry rather than rendering all cubes regardless of occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid (typically divided into chunks) where each voxel has a type (e.g., air, stone).
*   **Output:** An octree structure where nodes represent either a uniform value (leaf) or a subdivision into 8 children (internal node).

**Data Structures:**
The structure is built recursively. A node contains:
1.  `is_uniform`: Boolean flag indicating if the node represents a single value.
2.  `value`: The voxel type, valid only if `is_uniform` is true.
3.  `children`: An array of 8 child nodes, valid only if `is_uniform` is false.

**Parameters:**
*   **Chunk Size:** The base resolution of a chunk, typically $16 \times 16 \times 16$ voxels [S1, unofficial]. This determines the maximum depth of the octree. For a $16^3$ chunk, the depth is 5 levels (Root $\to$ $8^3$ $\to$ $4^3$ $\to$ $2^3$ $\to$ $1^3$) [S1, unofficial].

**Child Index Calculation:**
Children are addressed in Morton order (Z-order curve). The index $i \in [0, 7]$ is calculated from the query point's relative position to the node's origin:
$$
i = 4 \cdot \mathbb{I}(x > x_{origin}) + 2 \cdot \mathbb{I}(y > y_{origin}) + 1 \cdot \mathbb{I}(z > z_{origin})
$$
Where $\mathbb{I}(\cdot)$ is the indicator function (1 if true, 0 if false) [S1, unofficial]. This maps 3D coordinates to a linear child index based on bit interleaving [S1, unofficial].

**Steps:**

1.  **Initialization (Build):**
    *   Create a root node representing the entire chunk (e.g., $16 \times 16 \times 16$).
    *   Recursively subdivide any node containing mixed voxel types into 8 child octants.
    *   Continue subdivision until reaching leaf nodes representing single voxels ($1 \times 1 \times 1$) or uniform blocks where all contained voxels are of the same type [S1, unofficial].

2.  **Update on Modification:**
    *   When a voxel in a large uniform block is modified, identify the leaf node containing that voxel.
    *   If the node is a leaf representing a uniform block, split it into 8 children.
    *   Update the specific child corresponding to the modified voxel.
    *   Recursively check parent nodes: if a parent's children are now all uniform and identical, merge them back into a single uniform node; otherwise, mark the parent as non-uniform [S1, unofficial].

3.  **Query Logic:**
    *   To find a voxel at a specific position, traverse from the root.
    *   At each node, check if the node represents a uniform value. If yes, return that value.
    *   If no, calculate the child index based on the query point's relative position to the node's origin.
    *   Recurse into the selected child [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_octree_value(node, depth, node_origin, query_point):
    """
    Retrieves the voxel value at query_point from the octree.
    
    Args:
        node: The current OctreeNode.
        depth: Current depth in the tree (0 for root).
        node_origin: int3, the minimum coordinate of the region covered by this node.
        query_point: int3, the absolute coordinate of the voxel to retrieve.
        
    Returns:
        The voxel type/value.
    """
    # Base case: Node is uniform (leaf or merged block)
    if node.is_uniform:
        return node.value

    # Calculate child index using Morton order logic
    # Bit 2 (value 4) for X, Bit 1 (value 2) for Y, Bit 0 (value 1) for Z
    child_index = 0
    if query_point.x > node_origin.x:
        child_index += 4
    if query_point.y > node_origin.y:
        child_index += 2
    if query_point.z > node_origin.z:
        child_index += 1

    # Get the child node
    child = node.children[child_index]

    # Calculate the origin for the child node
    # The child's origin is the node's origin plus the offset of the child within the node
    # Offset calculation depends on chunk size and depth. 
    # For a node of size S at depth d, the child size is S/2.
    # The offset for child_index i is derived from the Morton bit decomposition.
    # Specifically, if child_index has bit k set, add 2^(max_depth - depth - 1 - k) to the respective axis.
    # However, the source code passes `nodeOrigin + childOffset[depth, childIndex]`.
    # We must define childOffset or compute it. 
    # Standard Morton offset: 
    # Let step = 1 << (max_depth - depth - 1). 
    # x_offset = (child_index >> 2) & 1 ? step : 0
    # y_offset = (child_index >> 1) & 1 ? step : 0
    # z_offset = child_index & 1 ? step : 0
    
    step = 1 << (5 - depth - 1) # Assuming max depth 5 for 16^3 chunk, or generally log2(chunk_size)
    
    x_offset = (step if (child_index & 4) else 0)
    y_offset = (step if (child_index & 2) else 0)
    z_offset = (step if (child_index & 1) else 0)
    
    child_origin = (node_origin.x + x_offset, 
                    node_origin.y + y_offset, 
                    node_origin.z + z_offset)

    return get_octree_value(child, depth + 1, child_origin, query_point)
```

**Edge Cases and Pitfalls:**
*   **Fragmentation:** In destructible worlds (e.g., Minecraft-style games) where players build extensively, the octree can become filled with many small leaf nodes (single voxels), negating memory benefits [S1, unofficial]. Solutions involve merging adjacent uniform leaves or limiting minimum cell size [S1, unofficial].
*   **Precision:** The comparison `query_point.x > node_origin.x` assumes integer coordinates. Boundary conditions must be handled consistently with the coordinate system definition [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Lookup is $O(\log N)$ where $N$ is the number of voxels in a chunk (specifically $O(\log_2(\text{chunk\_size}))$ steps) [S1, unofficial]. This is significantly faster than scanning a flat array for sparse data but slower than $O(1)$ direct array access [S1, unofficial].
*   **Memory Complexity:** Reduces memory usage significantly for sparse data. An example showed a 57% decrease in stored values for a $32 \times 32$ quadtree (2D equivalent) and further reduction with deeper grouping [S1, unofficial].
*   **Strengths:** Efficient storage for sparse data; allows culling of hidden faces and merging of coplanar faces during rendering, reducing overdraw compared to instancing [S1, unofficial].
*   **Limitations:** Lookup complexity is higher than flat arrays; fragmentation can occur in highly dynamic environments [S1, unofficial].
*   **When to use:** When voxel data is sparse or contains large uniform regions, and memory is a constraint [S1, unofficial].
*   **When not to use:** When the data is dense and uniform (flat array is simpler and faster), or when the workload involves frequent random access to individual voxels in a dense grid where the overhead of traversal is unjustified [S1, unofficial].

## Background
Octrees were introduced as a solution to store sparse voxel data without the large search times associated with linear scans [S1, unofficial]. The concept is an extension of quadtrees (2D) to 3D [S1, unofficial]. It is widely used in voxel-based games and engines, such as those inspired by Minecraft or Terraria, to manage large worlds efficiently [S1, unofficial]. The specific implementation details regarding Morton order traversal and chunk-based subdivision are commonly discussed in game development communities for optimizing voxel engines [S1, unofficial].

## Sources
[S1] Storing voxels for a voxel Engine in C++, https://gamedev.stackexchange.com/questions/172119/storing-voxels-for-a-voxel-engine-in-c, fetched 2026-10-26 (unofficial)
