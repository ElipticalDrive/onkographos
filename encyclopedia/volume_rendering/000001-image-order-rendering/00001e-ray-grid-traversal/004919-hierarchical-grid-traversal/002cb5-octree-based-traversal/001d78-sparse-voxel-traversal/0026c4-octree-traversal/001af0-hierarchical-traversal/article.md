# Hierarchical Traversal

Also called: SVO Traversal, Sparse Voxel Octree (SVO) Traversal

## Overview
Hierarchical traversal accelerates ray-object intersection and volume rendering by stepping through a grid hierarchy where each higher-level cell covers a $2 \times 2$ area of the level below [S1, unofficial]. It solves the inefficiency of non-hierarchical voxel stepping by allowing rays to take larger steps through empty space at coarser levels, reducing the total step count [S1, unofficial]. Unlike flat grid traversal, it skips entire empty regions represented by parent nodes, and unlike pure ray-triangle BVH traversal, it operates directly on volumetric or implicit spatial subdivisions [S4, unofficial][S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a hierarchical data structure (e.g., Sparse Voxel Octree or BVH) where nodes bound spatial extents [S4, unofficial].
*   **Output:** A sequence of active nodes or leaves intersected by the ray, or a boolean indicating intersection within a distance threshold [S2, unofficial].

**Data Structures:**
*   **Octree/SVO Node:** Contains pointers or indices to children [S1, unofficial]. In SVO implementations, only the highest level is stored initially and nodes are subdivided on demand to represent the grid [S1, unofficial].
*   **BVH Node:** As defined in [S2, unofficial], a `BVHNode` struct contains `center` (`[f32; 4]`), `radius` (`f32`), `left_child` (`u32`), and `right_child` (`u32`), packed in 16-byte alignment [S2, unofficial].
*   **Leaf Identification:** Leaf nodes are identified by setting the high bit (`0x80000000`) of the child indices [S2, unofficial]. If a leaf is the last element and has no sibling, its `right_child` is set to `empty_node_idx` (0) [S2, unofficial].

**Algorithm Steps:**
The traversal generally follows a stack-based or recursive descent, prioritizing closer nodes [S5, unofficial].

1.  **Initialization:**
    *   Initialize a stack with the root node [S2, unofficial].
    *   Compute ray-box intersection parameters ($t_{min}, t_{max}$) for the root bounding volume [S2, unofficial].
2.  **Traversal Loop:**
    While the stack is not empty:
    1.  Pop the top node index from the stack [S2, unofficial].
    2.  Retrieve the node data [S2, unofficial].
    3.  **Intersection Test:** Check if the ray intersects the current node's bounding volume [S2, unofficial].
    4.  **Leaf Check:** If the node is a leaf (identified by the high bit in child indices for BVH, or specific type flag for Octree):
        *   Process the leaf (e.g., check for surface descendant, sample volume, or return intersection) [S3, unofficial][S6, unofficial].
        *   Return true if the minimum intersection distance $t$ is less than a threshold (e.g., 10000.0) [S2, unofficial].
    5.  **Internal Node Descent:**
        *   Identify children (left and right for BVH; 8 for Octree) [S2, unofficial].
        *   If both children need descending (intersect the ray):
            *   Push the child with the smaller intersection distance onto the stack first, then descend into the closer child immediately [S2, unofficial].
            *   Formula for ordering: `if left_t < right_t: push right, descend left` [S2, unofficial].
        *   If only one child intersects, descend into that child [S2, unofficial].
        *   If neither intersects, pop from stack (handled by the loop condition or explicit check) [S2, unofficial].
3.  **Octree-Specific Path Traversal:**
    For SVO/Octree variants, traversal may follow a precomputed path or octant mask [S3, unofficial].
    *   Iterate through path octants [S3, unofficial].
    *   If the current cell is a Branch, retrieve its children index, access the specific child in the children vector, and update the current pointer [S3, unofficial].
    *   If it is not a Branch (Leaf), return the current cell [S3, unofficial].

**Parameters:**
*   **Max Depth:** Limits the recursion depth to prevent infinite loops in degenerate cases [S2, unofficial].
*   **Threshold:** Distance cutoff for early termination (e.g., 10000.0) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Empty Siblings:** In BVH leaf handling, if a leaf is the last element and has no sibling, `right_child` is set to 0 [S2, unofficial].
*   **Zero Direction Components:** Ray direction components must be handled to avoid division by zero during slab intersection tests (e.g., clamping small values to epsilon) [S2, unofficial].
*   **Boundary Voxels:** Special handling is required for rays intersecting boundary voxels where standard step masks might be zero [S2, unofficial].

**Pseudocode (BVH Variant):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_bvh(ray_origin, ray_direction, bvh_nodes, max_depth=100):
    # Stack stores (node_index, t_min, t_max)
    stack = [(0, 0.0, 10000.0)]
    depth = 0

    while stack and depth < max_depth:
        node_idx, t_min, t_max = stack.pop()
        depth += 1

        if node_idx >= len(bvh_nodes):
            continue

        node = bvh_nodes[node_idx]

        # Check if leaf (high bit set in left_child)
        is_leaf = (node.left_child & 0x80000000) != 0

        if is_leaf:
            # Process leaf: check intersection within t_max
            if box_intersects_ray(node.center, node.radius, ray_origin, ray_direction, t_min, t_max):
                return True
            continue

        # Internal node: get children indices
        left_child = node.left_child & 0x7FFFFFFF
        right_child = node.right_child & 0x7FFFFFFF

        # Compute intersection distances for children
        left_hit = box_intersects_ray(node.center, node.radius, ray_origin, ray_direction, t_min, t_max)
        right_hit = box_intersects_ray(node.center, node.radius, ray_origin, ray_direction, t_min, t_max)

        if left_hit is None:
            left_hit = (-float('inf'), -float('inf'))
        if right_hit is None:
            right_hit = (-float('inf'), -float('inf'))

        left_t_min, left_t_max = left_hit
        right_t_min, right_t_max = right_hit

        # Push children in order of distance (closer first)
        # Only push if they intersect the current ray segment [t_min, t_max]
        if left_t_max >= t_min and right_t_max >= t_min:
            if left_t_min < right_t_min:
                if right_t_max >= t_min:
                    stack.append((right_child, right_t_min, right_t_max))
                stack.append((left_child, left_t_min, left_t_max))
            else:
                if left_t_max >= t_min:
                    stack.append((left_child, left_t_min, left_t_max))
                stack.append((right_child, right_t_min, right_t_max))
        elif left_t_max >= t_min:
            stack.append((left_child, left_t_min, left_t_max))
        elif right_t_max >= t_min:
            stack.append((right_child, right_t_min, right_t_max))

    return False

def box_intersects_ray(center, radius, origin, direction, t_min, t_max):
    # Simple AABB intersection returning (t_min_hit, t_max_hit) or None
    t_min_hit = -float('inf')
    t_max_hit = float('inf')
    for i in range(3):
        inv_d = 1.0 / direction[i] if direction[i] != 0 else float('inf')
        t1 = (center[i] - radius - origin[i]) * inv_d
        t2 = (center[i] + radius - origin[i]) * inv_d
        t_min_hit = max(t_min_hit, min(t1, t2))
        t_max_hit = min(t_max_hit, max(t1, t2))
    if t_max_hit < t_min or t_min_hit > t_max:
        return None
    return (max(t_min_hit, t_min), min(t_max_hit, t_max))
```

## Subtopics

### [Leaf Collection](899-leaf-collection.md)
Leaf Collection identifies and gathers all terminal nodes intersected by a ray grid during hierarchical traversal, converting a sparse traversal path into a contiguous list of active volumetric blocks for subsequent ray marching or shading [899-leaf-collection.md]. It separates node identification from radiance computation to enable efficient memory compaction and parallel processing [899-leaf-collection.md]. Choose it when rendering requires batched shading of active blocks rather than immediate per-voxel evaluation.

### [Point Plane Test Traversal](982-point-plane-test-traversal.md)
Point Plane Test Traversal locates the specific child tetrahedron within a TET-AMR refinement tree that contains a given query point [982-point-plane-test-traversal.md]. It solves the in-element lookup problem by traversing from the coarse root to the leaf using geometric half-space tests against planes defined by edge midpoints, avoiding the redundancy of checking all eight children individually [982-point-plane-test-traversal.md]. Choose it for adaptive tetrahedral meshes where explicit geometric plane tests are preferred over incremental rasterization logic.

### [Multi-Resolution Rendering with K-D Tree](1942-multi-resolution-rendering-with-k-d-tree.md)
Multi-resolution rendering with K-D Tree accelerates image-order ray casting by storing approximations of the underlying data within K-D tree nodes [1942-multi-resolution-rendering-with-k-d-tree.md]. It solves the performance bottleneck of rendering large, high-resolution datasets by allowing the renderer to skip detailed traversal when the data within a spatial region can be represented by a simple model within a user-specified error threshold [1942-multi-resolution-rendering-with-k-d-tree.md]. Choose it when fast preview images and adaptive detail are required, as it stores a complete model for each region unlike wavelet-based methods.

### [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md)
Coarse Mesh Traversal accelerates volume rendering by decoupling the global search for active data regions from the local sampling of fine-grained geometry [3062-coarse-mesh-traversal.md]. It solves the problem of inefficiently traversing large, sparse, or multi-resolution datasets by first identifying which coarse elements a ray intersects, and then performing detailed rendering only within those specific regions [3062-coarse-mesh-traversal.md]. Choose it when explicit construction of a coarse mesh (typically a BVH) from the underlying adaptive data structure allows skipping empty volumes before entering the fine-level traversal loop.

### [Node Omission in Ray-Tree Traversal](3305-node-omission-in-ray-tree-traversal.md)
Node omission prevents adding child nodes to the traversal stack if it is known they will not intersect the current ray, reducing the number of intersection tests [3305-node-omission-in-ray-tree-traversal.md]. It sets itself apart by operating on the spatial relationship between the ray's entry/exit parameters and the node's bounding box midpoints, rather than relying on leaf-level data structures or plane-based splitting planes [3305-node-omission-in-ray-tree-traversal.md]. Choose it when minimizing stack operations and intersection tests is critical for performance.

### [Two-Pass Hierarchy Traversal](3339-two-pass-hierarchy-traversal.md)
Two-Pass Hierarchy Traversal reduces computational cost by splitting the descent into a coarse initial pass and a fine subsequent pass [3339-two-pass-hierarchy-traversal.md]. It sets itself apart by explicitly decoupling the identification of relevant subtrees from the detailed node evaluation, allowing for significant reuse across pixel blocks [3339-two-pass-hierarchy-traversal.md]. Choose it when rendering large pixel blocks where subtree identification can be reused across multiple passes.

### [Adaptive Out-of-Core Single-Pass GPU Raycasting](3440-adaptive-out-of-core-single-pass-gpu-raycasting.md)
This method enables interactive rendering of multi-gigabyte scalar volumes on desktop hardware by combining an out-of-core octree data structure with a single-pass, stackless GPU raycasting algorithm [3440-adaptive-out-of-core-single-pass-gpu-raycasting.md]. It solves the memory capacity and bandwidth bottlenecks of in-core rendering by maintaining a view- and transfer-function-dependent working set of volumetric bricks in GPU memory, while the rest of the dataset remains on disk [3440-adaptive-out-of-core-single-pass-gpu-raycasting.md]. Choose it when rendering datasets exceeding GPU memory limits, as it performs the entire volume traversal and compositing in a single fragment shader pass.

### [Bounding Volume Hierarchy (BVH) Traversal](18487-bounding-volume-hierarchy-bvh-traversal.md)
Bounding Volume Hierarchy traversal accelerates ray-scene intersection and distance queries by recursively testing rays against a binary tree of Axis-Aligned Bounding Boxes [18487-bounding-volume-hierarchy-bvh-traversal.md]. It is distinguished from hierarchical grid methods by operating directly on unstructured geometry rather than voxelized volumes, and from software-only BVHs by leveraging hardware acceleration for ray-triangle intersections [18487-bounding-volume-hierarchy-bvh-traversal.md]. Choose it when rendering unstructured triangle meshes or when hardware ray-tracing cores are available.

### Surface Descendant Check
If the cell is a Leaf, return whether it has a sign change [S3, unofficial]. If it is a Branch, recursively check if any of its children have a surface descendant [S3, unofficial]. If it is Empty or Full, return false [S3, unofficial]. Choose it when extracting isosurfaces from implicit scalar fields stored in hierarchical grids.

### Coarse Mesh Traversal via BVH
Uses a Bounding Volume Hierarchy built over the coarse tetrahedral mesh to efficiently locate the coarse element containing a sample point [S7, unofficial]. Build a triangle BVH over the coarse mesh by inserting four triangular faces facing outside for every coarse element [S7, unofficial]. Trace rays into the BVH using OptiX Wrapper Library hit and miss functions [S7, unofficial]. The hit function returns the hit point and the coarse tetrahedron ID [S7, unofficial]. The miss function reports no intersection with BVH faces [S7, unofficial]. Choose it for point location querying in adaptive tetrahedral meshes where element marching is inefficient.

### Warp-Level Intersection Caching
Optimization to delay intersection tests until enough threads in a warp have stored an intersection child node [S7, unofficial]. When an intersection is found, store the current voxel intersection data [S7, unofficial]. At the start of every traversal step, check if enough threads have a stored intersection [S7, unofficial]. If so, proceed by doing an intersection test [S7, unofficial]. Otherwise, continue traversal with the complete warp [S7, unofficial]. If threads find the next intersection test while already having an intersection stored, stop traversing until the next intersection test is performed [S7, unofficial]. Choose it for GPU implementations where warp synchronization reduces redundant intersection calculations.

### BFS Hierarchy Cut Algorithm
An algorithm to cut a binary tree hierarchy along a certain condition using a breadth-first search queue [S8, unofficial]. Choose it when restructuring or pruning hierarchical trees based on spatial or data-driven conditions.

### Packet Ray Traversal / Ray Coherence Optimization
Configure the CUDA grid and block dimensions as 2D [S9, unofficial]. Map threads such that neighboring threads are spatially coherent, causing them to traverse the octree in similar ways, improving cache hit rates [S9, unofficial]. Thread Divergence: In volumetric ray tracing or deep octree subdivisions, neighboring rays may diverge significantly in their traversal paths, causing SIMD stalls [S9, unofficial]. Choose it when rendering coherent ray packets, but avoid it when high-quality volume rendering requires deep subdivisions that exacerbate thread divergence.

### Octree Traversal with Texture Memory Access
Bind the octree node data to a CUDA texture object [S9, unofficial]. Access node data using the `tex1Dfetch()` function, which utilizes the texture cache for random access patterns typical in tree traversal [S9, unofficial]. Texture memory is preferred over constant memory for octree nodes because constant memory is limited in size and optimized for broadcast reads, whereas octree traversal involves random access patterns across different nodes [S9, unofficial]. Choose it for GPU implementations requiring efficient random access to octree node data.

## Operating characteristics
*   **Time Complexity:** $O(\log N)$ for balanced trees in the average case, where $N$ is the number of nodes, compared to $O(N)$ for linear traversal [S1, unofficial]. The number of steps is significantly reduced; for example, in a hierarchical SVO example, traversal took 5 steps compared to 11 steps for non-hierarchical traversal [S1, unofficial].
*   **Memory Complexity:** Depends on the structure [S2, unofficial]. SVO stores only the highest level initially and subdivides on demand, enabling sparse storage [S1, unofficial]. BVH nodes are packed tightly (e.g., 16-byte alignment) [S2, unofficial].
*   **Performance:** Results demonstrate that the technique is several times faster than other published algorithms [S4, unofficial].
*   **Scaling:** Efficiently skips empty space and reduces memory bandwidth usage by accessing only relevant nodes [S5, unofficial]. Supports out-of-core rendering via hierarchical paging [S5, unofficial].
*   **Strengths:** Reduces traversal steps, enables sparse storage, supports out-of-core workflows, and integrates with GPU-specific optimizations like texture memory access and warp-level caching [S5, unofficial][S10, unofficial][S8, unofficial].
*   **Limitations:** Stack depth can be an issue on GPUs [S5, unofficial]. Thread divergence in packet traversal can cause SIMD stalls [S10, unofficial].
*   **When to use:** Sparse volumetric data, large datasets exceeding memory limits, or scenes requiring efficient empty-space skipping.
*   **When not to use:** Dense uniform grids where flat traversal is simpler, or unstructured triangle meshes where BVH traversal is more appropriate.

## Background
Hierarchical traversal originated from the need to accelerate ray tracing and volume rendering for large datasets [S4, unofficial]. Early work by Kay and Kajiya (1986) introduced a new hierarchy traversal algorithm for ray-object intersection, demonstrating it was several times faster than previous methods [S4, unofficial]. The concept of using octrees and SVOs for volume rendering was further developed to handle sparse data, where the tree acts as a 'page table' coupled to the resolution hierarchy [S5, unofficial]. Modern implementations leverage GPU-specific optimizations like texture memory access and warp-level caching [S10, unofficial][S8, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
[S3] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S4] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
[S5] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
[S6] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
[S7] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S8] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
[S9] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
[S10] Best way of traversing an octree in CUDA? - CUDA Programming and Performance - NVIDIA Developer Forums, https://forums.developer.nvidia.com/t/best-way-of-traversing-an-octree-in-cuda/9179, fetched 2026-10-24 (unofficial)
