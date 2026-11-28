# Octree Voxel Storage

Also called: Hierarchical Voxel Storage, Quadtree (2D equivalent)

## Overview
Octree Voxel Storage is a hierarchical data structure used to store voxel data efficiently by grouping adjacent voxels of the same type into larger cells, reducing memory usage for sparse or uniform regions compared to flat arrays [S1, unofficial]. It differs from flat arrays by trading $O(1)$ direct access for logarithmic traversal time to save space, and differs from instancing by allowing the culling of hidden faces and merging of coplanar geometry rather than rendering all cubes regardless of occlusion [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D voxel grid divided into chunks, where each voxel has a type (e.g., air, stone) [S1, unofficial].
*   **Output:** An octree structure where nodes represent either a uniform value (leaf) or a subdivision into 8 children (internal node) [S1, unofficial].

**Data Structures:**
The structure is built recursively. A node contains:
1.  `is_uniform`: Boolean flag indicating if the node represents a single value [S1, unofficial].
2.  `value`: The voxel type, valid only if `is_uniform` is true [S1, unofficial].
3.  `children`: An array of 8 child nodes, valid only if `is_uniform` is false [S1, unofficial].

**Parameters:**
*   **Chunk Size:** The base resolution of a chunk, typically $16 \times 16 \times 16$ voxels [S1, unofficial]. This determines the maximum depth of the octree, resulting in 5 levels for a $16^3$ chunk (root $\to$ $8^3$ $\to$ $4^3$ $\to$ $2^3$ $\to$ $1^3$) [S1, unofficial].

**Child Index Calculation:**
Children are addressed in Morton order (Z-order curve) [S1, unofficial]. The index $i \in [0, 7]$ is calculated from the query point's relative position to the node's origin:
$$
i = 4 \cdot \mathbb{I}(x > x_{origin}) + 2 \cdot \mathbb{I}(y > y_{origin}) + 1 \cdot \mathbb{I}(z > z_{origin})
$$
Where $\mathbb{I}(\cdot)$ is the indicator function (1 if true, 0 if false) [S1, unofficial]. This maps 3D coordinates to a linear child index based on bit interleaving [S1, unofficial].

**Steps:**

1.  **Initialization (Build):**
    *   Create a root node representing the entire chunk (e.g., $16 \times 16 \times 16$) [S1, unofficial].
    *   Recursively subdivide any node containing mixed voxel types into 8 child octants [S1, unofficial].
    *   Continue subdivision until reaching leaf nodes representing single voxels ($1 \times 1 \times 1$) or uniform blocks where all contained voxels are of the same type [S1, unofficial].

2.  **Update on Modification:**
    *   When a voxel in a large uniform block is modified, subdivide that block's cell recursively until the change is isolated in a leaf node or small cell [S1, unofficial].

3.  **Query Logic:**
    *   To find a voxel at a specific position, traverse from the root [S1, unofficial].
    *   At each node, check if the node represents a uniform value; if yes, return that value [S1, unofficial].
    *   If no, calculate the child index based on the query point's relative position to the node's origin and recurse into the selected child [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def get_octree_value(node, depth, node_origin, query_point, chunk_size):
    """
    Retrieves the voxel value at query_point from the octree.
    
    Args:
        node: The current OctreeNode.
        depth: Current depth in the tree (0 for root).
        node_origin: tuple(int, int, int), the minimum coordinate of the region covered by this node.
        query_point: tuple(int, int, int), the absolute coordinate of the voxel to retrieve.
        chunk_size: int, the base resolution of the chunk (e.g., 16).
        
    Returns:
        The voxel type/value.
    """
    # Base case: Node is uniform (leaf or merged block)
    if node.is_uniform:
        return node.value

    # Calculate child index using Morton order logic
    # Bit 2 (value 4) for X, Bit 1 (value 2) for Y, Bit 0 (value 1) for Z
    child_index = 0
    if query_point[0] > node_origin[0]:
        child_index += 4
    if query_point[1] > node_origin[1]:
        child_index += 2
    if query_point[2] > node_origin[2]:
        child_index += 1

    # Get the child node
    child = node.children[child_index]

    # Compute child origin offset based on chunk size and current depth [G]
    # The source assumes a precomputed childOffset table; this computes it inline.
    # Step size halves at each depth level. Leaf nodes are caught by the uniform check above.
    step = chunk_size >> (depth + 1)
    x_offset = step if (child_index & 4) else 0
    y_offset = step if (child_index & 2) else 0
    z_offset = step if (child_index & 1) else 0
    
    child_origin = (node_origin[0] + x_offset, 
                    node_origin[1] + y_offset, 
                    node_origin[2] + z_offset)

    return get_octree_value(child, depth + 1, child_origin, query_point, chunk_size)
```

**Edge Cases and Pitfalls:**
*   **Fragmentation:** In destructible worlds (e.g., Minecraft-style games) where players build extensively, the octree can become filled with many small leaf nodes (single voxels), negating memory benefits [S1, unofficial]. Solutions involve merging adjacent uniform leaves or limiting minimum cell size [S1, unofficial].
*   **Precision:** The comparison `query_point.x > node_origin.x` assumes integer coordinates; boundary conditions must be handled consistently with the coordinate system definition [G].

## Subtopics

### Terrain Storage (5-Layer Octree)
A data structure for voxel terrain that uses a 5-layer octree to compress repetitive block data in system memory and on disk [S2, unofficial]. It divides chunks into $16 \times 256 \times 16$ blocks and includes a separate Position→Instance lookup table for per-position instance data like inventories or fluids [S2, unofficial]. In chunks with complex block arrangements or high block variety, the octree compression may increase memory usage compared to flat storage, but this is accepted because simple chunks vastly outnumber complex ones [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Lookup requires at most 5 steps for a $16^3$ chunk, or $O(\log_2(\text{chunk\_size}))$ generally [S1, unofficial]. This is faster than scanning a flat array for sparse data but slower than $O(1)$ direct access [S1, unofficial].
*   **Memory Complexity:** Reduces memory usage significantly for sparse data; an example showed a 57% decrease in stored values for a $32 \times 32$ quadtree (2D equivalent) compared to storing 1024 separate values [S1, unofficial].
*   **Strengths:** Efficient storage for sparse data; allows culling of hidden faces and merging of coplanar faces during rendering, reducing overdraw compared to instancing [S1, unofficial].
*   **Limitations:** Lookup complexity is higher than flat arrays; fragmentation can occur in highly dynamic environments [S1, unofficial].
*   **When to use:** When voxel data is sparse or contains large uniform regions, and memory is a constraint [S1, unofficial].

## Background
Octrees were introduced as a solution to store sparse voxel data without the large search times associated with linear scans [S1, unofficial]. The concept is an extension of quadtrees (2D) to 3D [S1, unofficial]. It is widely used in voxel-based games and engines, such as those inspired by Minecraft or Terraria, to manage large worlds efficiently [S1, unofficial]. The specific implementation details regarding Morton order traversal and chunk-based subdivision are commonly discussed in game development communities for optimizing voxel engines [S1, unofficial].

## Sources
[S1] Storing voxels for a voxel Engine in C++, https://gamedev.stackexchange.com/questions/172119/storing-voxels-for-a-voxel-engine-in-c, fetched 2026-10-26 (unofficial)
[S2] DigBuild: Sandbox Voxel Game and Engine Using Vulkan, https://gredos.usal.es/handle/10366/149914, fetched 2026-10-31 (unofficial)
