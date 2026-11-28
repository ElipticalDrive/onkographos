# GPU Octree Voxelization via Triangle-AABB Intersection

Also called: CUDA Octree Voxelization, Parallel Voxelization via Octree Construction

## Overview
This method constructs an octree from triangle meshes by recursively subdividing nodes and querying GPU-accelerated CUDA kernels for geometric intersection and signed distance calculations [S1, unofficial]. It differs from fully GPU-side octree builds by performing the recursive subdivision logic on the CPU while offloading expensive geometric queries (bounding box computation, intersection tests, and signed distance calculations) to parallel CUDA kernels [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A list of triangle meshes, where each triangle is represented as a `glm::fmat3x3` (3x3 matrix of floats, where each row is a vertex `glm::fvec3`) [S1, unofficial].
- **Output:** An octree structure rooted at `m_root`, where each node contains an Axis-Aligned Bounding Box (AABB), a node type, and a signed distance value [S1, unofficial].

**Data Structures:**
1.  **AABB:** Defined as `struct AABB { glm::fmat2x3 m_box; glm::fvec3 m_center; };` [S1, unofficial].
    -   `m_box`: A 2x3 float matrix representing the minimum and maximum coordinates of the box [S1, unofficial].
    -   `m_center`: The center point of the AABB [S1, unofficial].
2.  **Node:** Defined as `struct Node { AABB m_box; float m_singed_dist; int m_node_type; Node* m_parent; Node* m_children[8]; };` [S1, unofficial].
    -   `m_node_type`: Enum values include `NT_LEAF`, `NT_INTERNEL`, `NT_EMPTY` [S1, unofficial].
    -   `m_singed_dist`: Stores the shortest distance from the node center to the surface [S1, unofficial].
3.  **Triangle Mesh:** Stored in device memory as an array of `glm::fmat3x3` [S1, unofficial].

**Parameters:**
-   `voxel_size`: Default value is `1` [S1, unofficial].
-   `gridSize`: CUDA grid dimension, default `dim3(256)` [S1, unofficial].
-   `blockSize`: CUDA block dimension, default `dim3(256)` [S1, unofficial]. The source uses `dim3` which allows 3D dimensions, but 1D is implied by the loop logic [S1, unofficial].

**Algorithm Steps:**
1.  **Memory Allocation:** Allocate device memory for the triangle meshes and copy data from host to device [S1, unofficial].
2.  **Root Initialization:** Create the root node and compute its AABB by calling the parallel bounding box kernel `getBoundingBox` [S1, unofficial].
3.  **Recursive Subdivision:** Call `recSubDivision` on the root node [S1, unofficial].
    -   For each node, check for intersection with triangles using `isBoxleIntersectsTriangle` [S1, unofficial].
    -   **If Intersection is True:**
        -   **If Leaf Condition Met (e.g., `dim == voxel_dim`):**
            -   Set node type to `NT_LEAF` [S1, unofficial].
            -   Calculate signed distance using `calcLeafNodeSignedDistance` kernel [S1, unofficial].
            -   Store result in `m_singed_dist` [S1, unofficial].
        -   **If Internal Node:**
            -   Set node type to `NT_INTERNEL` [S1, unofficial].
            -   Initialize 8 children [S1, unofficial].
            -   Set each child's AABB using `setAABB` [S1, unofficial].
            -   Recursively call `recSubDivision` on each child [S1, unofficial].
    -   **If Intersection is False:**
        -   Set node type to `NT_EMPTY` [S1, unofficial].
        -   Calculate signed distance using `calcLeafNodeSignedDistance` kernel [S1, unofficial].
        -   Store result in `m_singed_dist` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_octree(tri_meshes, voxel_size=1.0):
    # 1. Memory Allocation
    d_tri_meshes = cuda_malloc(len(tri_meshes) * sizeof(glm_fmat3x3))
    cuda_memcpy(d_tri_meshes, tri_meshes, len(tri_meshes) * sizeof(glm_fmat3x3))
    # 2. Root Initialization
    root = Node()
    root.m_box = get_bounding_box(d_tri_meshes, len(tri_meshes))
    root.m_parent = None
    root.m_children = [None] * 8
    # 3. Recursive Subdivision
    rec_sub_division(root, d_tri_meshes, len(tri_meshes), voxel_size)
    return root

def rec_sub_division(node, d_tri_meshes, num_triangles, voxel_size):
    is_intersect = is_box_intersects_triangle(d_tri_meshes, node.m_box.min, node.m_box.max)
    if is_intersect:
        if node.dim == voxel_size:
            node.m_node_type = NT_LEAF
            node.m_singed_dist = calc_leaf_node_signed_distance(d_tri_meshes, node.m_box.center)
        else:
            node.m_node_type = NT_INTERNEL
            node.m_singed_dist = 0.0
            for i in range(8):
                child = Node()
                child.m_parent = node
                child.m_box = set_aabb(node.m_box, i)
                node.m_children[i] = child
                rec_sub_division(child, d_tri_meshes, num_triangles, voxel_size)
    else:
        node.m_node_type = NT_EMPTY
        node.m_singed_dist = calc_leaf_node_signed_distance(d_tri_meshes, node.m_box.center)
```

**Geometric Kernels:**
1.  **Bounding Box Reduction (`findRecudedBoundingBox`):**
    -   Uses shared memory `s_cache` for parallel reduction [S1, unofficial].
    -   Uses a mutex (`atomicCAS`) to handle atomic min/max operations since CUDA lacks native atomic min/max for floats [S1, unofficial].
    -   Formula for min/max update:
        $$ \text{box}_{min} = \min(\text{box}_{min}, \text{cache}_{min}) $$
        $$ \text{box}_{max} = \max(\text{box}_{max}, \text{cache}_{max}) $$
        [S1, unofficial].
2.  **Signed Distance Calculation (`calcLeafNodeSignedDistance`):**
    -   Iterates over all triangles to find the closest point on any triangle to the node center [S1, unofficial].
    -   Uses `findClosestPointOnTriangle` to compute the closest point using barycentric coordinates [S1, unofficial].
    -   Formula for closest point:
        $$ \text{closestPoint} = \text{triangle}[0] + s \cdot \text{edge0} + t \cdot \text{edge1} $$
        where `edge0 = triangle[1] - triangle[0]` and `edge1 = triangle[2] - triangle[0]` [S1, unofficial].
    -   Barycentric coordinates `s` and `t` are computed using dot products:
        $$ a = \text{dot}(\text{edge0}, \text{edge0}) $$
        $$ b = \text{dot}(\text{edge0}, \text{edge1}) $$
        $$ c = \text{dot}(\text{edge1}, \text{edge1}) $$
        $$ \text{det} = a \cdot c - b^2 $$
        [S1, unofficial]. The specific cases for `s` and `t` depend on the position of the projection relative to the triangle edges [S1, unofficial].

**Edge Cases and Pitfalls:**
-   **Intersection Test Implementation:** The `isBoxleIntersectsTriangle` function is declared but has an empty body in the source code, meaning intersection testing is not implemented [S1, unofficial].
-   **Memory Allocation Efficiency:** `calcLeafNodeSignedDistance` allocates memory on host and device for every call, which is inefficient [S1, unofficial].
-   **Mutex Size:** The source uses `cudaMemset(d_mutex, 0, sizeof(float))` instead of `sizeof(int)`, which may cause issues if the mutex is expected to be an integer [S1, unofficial].

## Operating characteristics
Time complexity is not given in the sources. Memory complexity is not given in the sources. Strengths: Leverages GPU parallelism for expensive geometric queries [S1, unofficial]. Limitations: The recursive CPU-side control flow may lead to kernel launch overhead. The source code has incomplete intersection testing [S1, unofficial].

## Background
This method is described in the "ParallelVoxelizer" project by allenkim0129 [S1, unofficial]. It is a GPU-accelerated voxelization technique that constructs an octree from triangle meshes [S1, unofficial].

## Sources
[S1] allenkim0129/ParallelVoxelizer, https://github.com/allenkim0129/ParallelVoxelizer, fetched 2026-10-26 (unofficial)
