# Hierarchical Traversal

Also called: Hierarchical Octree Traversal, Kay & Kajiya Hierarchy Traversal, Kay-Kajiya traversal, Octree Hierarchical Traversal, Octree Navigation, Octree Node Activity Check, SVO Traversal, Sparse Voxel Octree (SVO) Traversal

## Overview
Hierarchical traversal accelerates ray-object intersection and volume rendering by navigating a spatial hierarchy (such as an octree or bounding volume hierarchy) where higher-level nodes cover larger spatial extents than lower-level nodes [S1, unofficial][S4, unofficial]. It solves the performance bottleneck of non-hierarchical traversal by allowing the ray to take larger steps through empty space, significantly reducing the total number of intersection tests and node visits required to reach active data [S1, unofficial]. Unlike flat grid traversal which checks every voxel, hierarchical traversal skips entire subtrees that do not intersect the ray, enabling efficient rendering of sparse or multi-resolution datasets [S1, unofficial].

## How it works
**Inputs:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a hierarchical data structure (e.g., Sparse Voxel Octree or BVH) where each node has a bounding volume and references to children [S1, unofficial][S2, unofficial].
**Outputs:** A sequence of active leaf nodes or intersection points along the ray [S1, unofficial].

**Data Structures:**
The hierarchy is typically stored as an array of nodes [S2, unofficial]. For BVH implementations, a node struct contains `center` (4-component vector), `radius` (scalar), `left_child` (u32 index), and `right_child` (u32 index), packed with 16-byte alignment [S2, unofficial]. Leaf nodes are identified by setting the high bit (0x80000000) of the child indices [S2, unofficial]. If a leaf is the last element and has no sibling, its `right_child` is set to an empty node index (0) [S2, unofficial]. For Octree/SVO implementations, nodes store an `is_active` boolean flag indicating if the node or any descendant contains data, and an array of 8 child indices [S1, unofficial][S5, unofficial]. Cell naming follows the area they cover in level 0: Level 0 cells are 1x1, Level 1 cells are 2x2, and Level 2 cells are 4x4 [S1, unofficial].

**Algorithm Steps:**
1. **Initialization:** Compute ray-box intersection parameters for the root node's bounding volume. Initialize a traversal stack with the root node index and its intersection distance [S2, unofficial].
2. **Traversal Loop:** While the stack is not empty:
   a. Pop the top node index and its associated $t$ value from the stack [S2, unofficial].
   b. Retrieve the node from the data structure.
   c. **Activity Check:** If the node is an internal node, check if it is active. A node is active if `node->is_active` is true, or if any of its 8 children exist and are active [S5, unofficial]. If not active, skip to step 2a.
   d. **Intersection Test:** Test if the ray intersects the current node's bounding volume. If the minimum intersection distance $t$ is greater than the ray's near plane or existing closest hit, skip to step 2a [S2, unofficial].
   e. **Leaf Check:** If the node is a leaf (identified by specific flags or index ranges), process the leaf (e.g., perform ray-marching or surface intersection) and record the hit. Skip to step 2a.
   f. **Descend:** If the node is an internal node and active:
      i. Retrieve the indices of the left and right children (or all 8 children for octrees).
      ii. For each child, compute the intersection distance with its bounding volume.
      iii. Push children that intersect the ray onto the stack, ordered by increasing intersection distance (closer child first) [S2, unofficial].
      iv. If only one child intersects, push only that one. If neither, do not push any.
3. **Termination:** Return when the stack is empty or the closest hit distance exceeds the ray's far plane [S2, unofficial].

**Pseudocode (SVO/BVH-style):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_hierarchy(ray_origin, ray_direction, nodes, stack_max_size):
    # Clamp direction components to avoid division by zero [S2, unofficial]
    EPSILON = 1e-4
    d = [max(abs(c), EPSILON) * (1.0 if c >= 0 else -1.0) for c in ray_direction]
    o = ray_origin

    # Precompute ray-box intersection coefficients [S2, unofficial]
    tx_coef = 1.0 / d[0]
    ty_coef = 1.0 / d[1]
    tz_coef = 1.0 / d[2]
    tx_bias = tx_coef * o[0]
    ty_bias = ty_coef * o[1]
    tz_bias = tz_coef * o[2]

    stack = [None] * stack_max_size
    stack_ptr = 0

    def intersect_box(node_idx):
        node = nodes[node_idx]
        center = node['center']
        radius = node['radius']
        # Compute t intervals for each axis [S2, unofficial]
        t1 = (center[0] - radius[0]) * tx_coef - tx_bias
        t2 = (center[0] + radius[0]) * tx_coef - tx_bias
        t_min = min(t1, t2)
        t_max = max(t1, t2)

        t3 = (center[1] - radius[1]) * ty_coef - ty_bias
        t4 = (center[1] + radius[1]) * ty_coef - ty_bias
        t_min = max(t_min, min(t3, t4))
        t_max = min(t_max, max(t3, t4))

        t5 = (center[2] - radius[2]) * tz_coef - tz_bias
        t6 = (center[2] + radius[2]) * tz_coef - tz_bias
        t_min = max(t_min, min(t5, t6))
        t_max = min(t_max, max(t5, t6))

        # Fix: wrap componentwise comparison in all() to avoid TypeError [G]
        if all([t_min <= t_max, t_max >= 0]):
            return t_min
        return None

    def is_leaf(node):
        return (node['left_child'] & 0x80000000) != 0

    # Push root
    t_hit = intersect_box(0)
    if t_hit is None:
        return []
    stack[stack_ptr] = (0, t_hit)
    stack_ptr += 1

    hits = []
    while stack_ptr > 0:
        stack_ptr -= 1
        node_idx, t_node = stack[stack_ptr]
        node = nodes[node_idx]

        if is_leaf(node):
            hits.append((node_idx, t_node))
            continue

        left_idx = node['left_child'] & 0x7FFFFFFF
        right_idx = node['right_child'] & 0x7FFFFFFF

        t_left = intersect_box(left_idx)
        t_right = intersect_box(right_idx)

        if t_left is not None and t_right is not None:
            if t_left < t_right:
                stack[stack_ptr] = (left_idx, t_left)
                stack_ptr += 1
                stack[stack_ptr] = (right_idx, t_right)
                stack_ptr += 1
            else:
                stack[stack_ptr] = (right_idx, t_right)
                stack_ptr += 1
                stack[stack_ptr] = (left_idx, t_left)
                stack_ptr += 1
        elif t_left is not None:
            stack[stack_ptr] = (left_idx, t_left)
            stack_ptr += 1
        elif t_right is not None:
            stack[stack_ptr] = (right_idx, t_right)
            stack_ptr += 1

    return hits
```

**Pitfalls:**
- **Precision:** Division by zero can occur if ray direction components are zero; clamp small direction components to a small epsilon [S2, unofficial].
- **Boundary Conditions:** Leaf nodes at the end of the array may have invalid sibling indices; handle empty node indices explicitly [S2, unofficial].
- **Stack Overflow:** Ensure stack size is sufficient for the maximum depth of the hierarchy [S2, unofficial].

## Subtopics

### [Leaf Collection](899-leaf-collection.md)
Leaf Collection identifies and gathers all terminal nodes (leaves) intersected by a ray grid during hierarchical traversal, converting a sparse traversal path into a contiguous list of active volumetric blocks for subsequent ray marching or shading. It separates node identification from radiance computation to enable efficient memory compaction and parallel processing. Choose it when decoupling traversal from shading improves GPU occupancy or enables batched memory compaction.

### [Point Plane Test Traversal](982-point-plane-test-traversal.md)
Point Plane Test Traversal is a method for locating the specific child tetrahedron within a TET-AMR refinement tree that contains a given query point. It solves the in-element lookup problem by traversing from the coarse root to the leaf using geometric half-space tests against planes defined by edge midpoints, avoiding the redundancy of checking all eight children individually. It is distinguished from Leaf Collection by its on-the-fly vertex computation and from VolumeHDDA by its reliance on explicit geometric plane tests rather than incremental rasterization logic. Choose it for adaptive tetrahedral meshes where explicit geometric splitting is preferred over connectivity marching.

### [Mesh BVH](1062-mesh-bvh.md)
Mesh BVH is a GPU-accelerated bounding volume hierarchy designed for triangle meshes to support efficient distance queries and ray intersection tests. It provides spatial indexing for unsigned distance calculation and topological signing. Unlike leaf collection or point-plane tests which may handle specific traversal optimizations, Mesh BVH provides the foundational spatial indexing for both unsigned distance calculation and topological signing. Choose it when rendering large-scale triangle meshes or neural rendering pipelines requiring signed distance fields.

### [Multi-Resolution Rendering with K-D Tree](1942-multi-resolution-rendering-with-k-d-tree.md)
Multi-resolution rendering with K-D Tree is a hierarchical volume rendering technique that accelerates image-order ray casting by storing approximations of the underlying data within K-D tree nodes. It solves the performance bottleneck of rendering large, high-resolution datasets by allowing the renderer to skip detailed traversal when the data within a spatial region can be represented by a simple model within a user-specified error threshold. Unlike wavelet-based methods which store only detail coefficients, this method stores a complete model for each region, enabling fast "preview" images and adaptive detail. It builds on Hierarchical Traversal. Choose it when adaptive error thresholds and preview rendering are required.

### [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md)
Coarse Mesh Traversal is a hierarchical rendering technique that accelerates volume rendering by decoupling the global search for active data regions from the local sampling of fine-grained geometry. It solves the problem of inefficiently traversing large, sparse, or multi-resolution datasets by first identifying which coarse elements (macrocells) a ray intersects, and then performing detailed rendering only within those specific regions. What sets it apart from its siblings is the explicit construction of a coarse mesh (typically a BVH) from the underlying adaptive or hierarchical data structure, allowing the renderer to skip empty or irrelevant volumes entirely before entering the fine-level traversal loop. Choose it for multi-resolution datasets where coarse-to-fine decoupling reduces traversal overhead.

### [Node Omission in Ray-Tree Traversal](3305-node-omission-in-ray-tree-traversal.md)
Node omission is an optimization for hierarchical ray-tree traversal that prevents adding child nodes to the traversal stack if it is known they will not intersect the current ray, reducing the number of intersection tests. It sets itself apart from sibling methods like Leaf Collection and Point Plane Test Traversal by operating on the spatial relationship between the ray's entry/exit parameters and the node's bounding box midpoints, rather than relying on leaf-level data structures or plane-based splitting planes. Choose it when stack memory is constrained or when bounding volumes are tightly fitted.

### [Two-Pass Hierarchy Traversal](3339-two-pass-hierarchy-traversal.md)
Two-Pass Hierarchy Traversal is a spatial hierarchy traversal optimization that reduces computational cost by splitting the descent into a coarse initial pass and a fine subsequent pass. It sets itself apart from single-pass methods like Leaf Collection or Point Plane Test Traversal by explicitly decoupling the identification of relevant subtrees from the detailed node evaluation, allowing for significant reuse across pixel blocks. Choose it for tile-based or block-based renderers where spatial coherence across pixels can be exploited.

### [Adaptive Out-of-Core Single-Pass GPU Raycasting](3440-adaptive-out-of-core-single-pass-gpu-raycasting.md)
This method enables interactive rendering of multi-gigabyte scalar volumes on desktop hardware by combining an out-of-core octree data structure with a single-pass, stackless GPU raycasting algorithm. It solves the memory capacity and bandwidth bottlenecks of in-core rendering by maintaining a view- and transfer-function-dependent working set of volumetric bricks in GPU memory, while the rest of the dataset remains on disk. Unlike separate block rendering techniques that suffer from boundary artifacts and cannot support refractive or global illumination models, this approach performs the entire volume traversal and compositing in a single fragment shader pass, adapting sampling density to the local brick resolution. Choose it for desktop hardware rendering of terabyte-scale volumes.

#### Surface Descendant Check
If the cell is a Leaf, return whether it has a sign change. If it is a Branch, recursively check if any of its children have a surface descendant. If it is Empty or Full, return false [S3, unofficial].

#### Coarse Mesh Traversal via BVH
Uses a Bounding Volume Hierarchy (BVH) built over the coarse tetrahedral mesh to efficiently locate the coarse element containing a sample point [S6, unofficial]. Build a triangle BVH over the coarse mesh by inserting four triangular faces facing outside for every coarse element. Trace rays into the BVH using OptiX Wrapper Library (OWL) hit and miss functions. The hit function returns the hit point and the coarse tetrahedron ID. The miss function reports no intersection with BVH faces. Unlike element marching which uses connectivity, point location querying skips coarse elements not being sampled and allows varying refinement levels in the coarse mesh [S6, unofficial].

#### Warp-Level Intersection Caching
Optimization to delay intersection tests until enough threads in a warp have stored an intersection child node, similar to [AL09] [S7, unofficial]. When an intersection is found, store the current voxel intersection data. At the start of every traversal step, check if enough threads have a stored intersection. If so, proceed by doing an intersection test. Otherwise, continue traversal with the complete warp. If threads find the next intersection test while already having an intersection stored, stop traversing until the next intersection test is performed [S7, unofficial].

#### BFS Hierarchy Cut Algorithm
An algorithm to cut a binary tree hierarchy along a certain condition using a breadth-first search (BFS) queue [S8, unofficial].

## Operating characteristics
**Performance:** Hierarchical traversal reduces the number of steps significantly. In a specific SVO example, non-hierarchical traversal took 11 steps to intersect an occupied voxel, while hierarchical SVO traversal took 5 steps [S1, unofficial]. Results demonstrate that the technique is several times faster than other published algorithms [S4, unofficial].
**Strengths:** Efficient skipping of empty space; scalable to large datasets; supports multi-resolution data.
**Limitations:** Overhead of tree traversal and stack management; performance sensitive to tree balance and data distribution.

## Background
Hierarchical traversal was introduced by Tim Kay and James T. Kajiya in 1986 in the paper "Ray Tracing Complex Scenes" [S4, unofficial]. They presented a new algorithm for speeding up ray-object intersection calculations using a hierarchy of bounding extents. The method allows objects to be bounded by extents that fit convex hulls tightly and placed into a hierarchy, enabling efficient querying of objects along the ray in an efficient order [S4, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
[S3] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S4] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/399216888_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
[S5] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
[S6] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
[S7] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
[S8] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
