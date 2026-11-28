# Sparse Voxel Octree (SVO)


## Overview
A Sparse Voxel Octree (SVO) is a hierarchical spatial subdivision data structure that represents a scene as a sparse octree to enable efficient ray traversal, frustum culling, and visual level-of-detail (LoD) [S2, unofficial]. Unlike dense grids or KD-trees, the SVO provides tight compression for sparsely populated spaces and uniform subdivision at every level, which is essential for smoothly degrading LoD without the alignment issues of variable-granularity structures [S2, unofficial]. It is distinguished from its parent [Sparse Voxel Octree Representation](2340-sparse-voxel-octree-representation.md) by specific implementation details, such as shader-based traversal and specific SDF storage formats [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D space (typically a bounding box) containing volumetric data (e.g., Signed Distance Fields).
*   **Output:** An octree structure where leaf nodes contain voxel data, and internal nodes define the hierarchy.

**Data Structures:**
The SVO is a tree where each node has up to 8 children.
*   **Node Layout:** Each node represents a cubic region. The subdivision is uniform: every parent splits into 8 children ($2 \times 2 \times 2$) with the same layout [S2, unofficial].
*   **Voxel Storage:** In implementations following Laine and Karras [2010], Signed Distance Field (SDF) values are stored as 8-bit signed normalized (snorm) integers [S1, unofficial].
    *   **Encoding:** A value of 1 encodes 0.5 interior voxel diagonals [S1, unofficial].
    *   **Clamping:** Values are clamped to the range $[-1, 1]$ to fit the snorm format [S1, unofficial].
*   **Voxel Size:** The physical size of a voxel varies by octree level. Lower-resolution levels (higher in the tree) encode larger distances, trading precision for coverage [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Define the root node covering the entire scene bounding box.
2.  **Subdivision:** Recursively subdivide space uniformly. At each level, a node is split into 8 children [S2, unofficial].
3.  **Data Storage:** For leaf nodes, store the voxel data. If using SDFs, convert the continuous distance value $d$ to an 8-bit snorm integer $v$:
    $$ v = \text{clamp}\left( \text{round}\left( \frac{d}{0.5 \cdot \text{voxel\_diagonal}} \right), -128, 127 \right) $$
    *(Note: The exact mapping formula is derived from the description that 1 encodes 0.5 interior voxel diagonals. The snorm range -1 to 1 maps to integer indices -128 to 127.)* [S1, unofficial].
4.  **Traversal:** Traverse the tree from the root. Because the structure is sparse, only existing nodes are visited. Traversal can occur entirely in shader code for real-time rendering [S1, unofficial].
5.  **LoD Selection:** Use the distance from the camera/viewpoint to select the appropriate depth in the octree. Deeper levels provide higher precision for nearby objects; shallower levels cover distant areas [S2, unofficial].

**Parameters:**
*   **Max Depth:** Not given in the sources.
*   **SDF Bit Depth:** 8-bit snorm [S1, unofficial].
*   **Children per Node:** 8 ($2 \times 2 \times 2$) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Cache Friendliness:** Octrees are harder to make cache-friendly compared to flat structures due to pointer chasing and non-linear memory access [S2, unofficial].
*   **Memory Usage:** A complete (dense) octree of a certain depth consumes significant memory. The "sparse" aspect is critical to avoid this overhead [S2, unofficial].
*   **Precision Loss:** Lower-resolution levels encode larger distances at the expense of precision. This must be managed by the LoD system to avoid visual artifacts at distance [S1, unofficial].

## Subtopics

### Sparse Voxel Octree with Bricks
Groups voxels into larger bricks (e.g., $8 \times 8 \times 8$) to improve memory locality and reduce overhead. [Link](2354-sparse-voxel-with-bricks.md)

### Naive Octree Node Representation
Uses a simple array or pointer-based structure for nodes, potentially less optimized for specific hardware than SVO implementations. [Link](2445-naive-octree-node-representation.md)

### SVRaster
A rasterization-based approach using SVOs, differing from ray-tracing traversal. [Link](2464-svraster.md)

### Dynamic Update Optimization
Techniques for efficiently updating the SVO when the underlying scene changes. [Link](4120-dynamic-update-optimization.md)

## Operating characteristics
*   **Time Complexity:** Traversal time depends on the depth of the tree and the number of empty nodes skipped. It is generally efficient for sparse data but can be slower than flat grids for dense data due to pointer indirection.
*   **Memory Complexity:** Scales with the number of non-empty voxels. Significantly more memory-efficient than dense grids for sparse scenes [S2, unofficial].
*   **Strengths:**
    *   Supports visual level-of-detail (LoD) sensibly [S2, unofficial].
    *   Provides extremely tight compression of sparsely-populated spaces [S2, unofficial].
    *   Uniform subdivision allows for accurate bounds on processing cost and smooth LoD degradation [S2, unofficial].
    *   Accelerates iteration through space for raytracing, AI pathfinding, frustum culling, and collision detection [S2, unofficial].
*   **Limitations:**
    *   Harder to make cache-friendly than flat structures [S2, unofficial].
    *   Higher implementation complexity compared to simple grids [S2, unofficial].
    *   The low division ratio (1:2 per axis) results in a high number of subdivision levels, which can impact traversal performance if not optimized [S2, unofficial].
*   **When to Use:** Ideal for voxel engines, real-time rendering with LoD requirements, and scenes with large empty spaces [S2, unofficial].
*   **When Not to Use:** When cache linearization is critical and data is dense (consider flattening to $16 \times 16 \times 16$ bricks instead) [S2, unofficial].

## Background
The Sparse Voxel Octree (SVO) was popularized in computer graphics by Crassin et al. [2009] and further detailed by Laine and Karras [2010] for real-time ray tracing. The structure addresses the limitations of dense voxel grids and KD-trees by combining the uniform subdivision of octrees (good for LoD) with sparsity (good for memory). It is widely used in voxel engines, raytracing accelerators, and AI pathfinding systems [S2, unofficial].

## Sources
[S1] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
[S2] When would you use an octree in a voxel Engine?, https://gamedev.stackexchange.com/questions/70831/when-would-you-use-an-octree-in-a-voxel-engine, fetched 2026-10-26 (unofficial)
