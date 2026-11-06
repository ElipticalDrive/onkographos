# Hierarchical Traversal

Also called: Hierarchical Octree Traversal, Kay & Kajiya Hierarchy Traversal, Kay-Kajiya traversal, Octree Hierarchical Traversal, Octree Navigation, Octree Node Activity Check, SVO Traversal, Sparse Voxel Octree (SVO) Traversal

## Overview
Hierarchical traversal is a method for accelerating ray-object intersection by navigating a spatial hierarchy of grids, such as an Octree or Sparse Voxel Octree (SVO), where each higher-level cell covers a $2 \times 2$ area of the level below [S1, unofficial]. It solves the problem of excessive traversal steps in dense or large datasets by allowing the ray to take larger steps through empty space at coarser levels, reducing the total number of intersection tests compared to non-hierarchical traversal [S1, unofficial]. Unlike flat grid traversal which checks every voxel, hierarchical methods skip entire subtrees when a node is determined to be inactive or empty, a principle first formalized for general ray-object intersection by Kay and Kajiya [S4, unofficial].

## How it works
**Inputs:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, and a hierarchical data structure (Octree or SVO) where nodes represent bounding volumes.
**Outputs:** A list of intersected leaf nodes or a boolean indicating if any active descendant exists within a distance threshold.

**Data Structures:**
1.  **Hierarchical Grid:** Cells are named by the area they cover in Level 0. Level 0 cells are $1 \times 1$, Level 1 cells are $2 \times 2$, Level 2 cells are $4 \times 4$ [S1, unofficial].
2.  **Node Representation:**
    *   **Branch Node:** Contains indices to 8 children. In SVO implementations, children are stored in a vector or array. A node is a Branch if it has children; otherwise, it is a Leaf or Empty [S1, unofficial].
    *   **Leaf Node:** Represents a voxel or a block of voxels. In BVH-based variants, leaf nodes may be identified by setting the high bit ($0x80000000$) of the child indices [S2, unofficial].
    *   **Activity Flag:** Nodes track if they or any descendant are "active" (occupied) [S5, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Start at the root node. If the traversal path is empty, return the root cell [S3, unofficial].
2.  **Activity Check:** Determine if the current node or any of its descendants are active.
    *   If the node is null, return false [S5, unofficial].
    *   If the node is marked active, return true [S5, unofficial].
    *   If not marked, iterate through all 8 children. If a child exists and `isNodeActive(child)` returns true, return true [S5, unofficial].
    *   If no active child is found, return false [S5, unofficial].
3.  **Traversal Loop:**
    *   If the current cell is a Branch, retrieve its children indices.
    *   Determine which children intersect the ray. This is typically done using ray-box intersection tests against the bounding volumes of the children [S4, unofficial].
    *   **Stack-based Descent (BVH-style):**
        *   Maintain a stack of nodes to visit.
        *   While the stack is not empty:
            *   Pop the top node.
            *   If the node is a Leaf, process it (e.g., check for sign change or sample data) [S3, unofficial].
            *   If the node is a Branch:
                *   Identify intersecting children.
                *   If both left and right children need descending, push the child with the smaller intersection distance ($t_{left} < t_{right}$) onto the stack, then descend into the closer child [S2, unofficial].
                *   If only one child intersects, descend into that child.
                *   If neither intersects, pop from the stack (backtrack) [S2, unofficial].
    *   **Path-based Descent (SVO-style):**
        *   Iterate through the path octants.
        *   For each octant, if the current cell is a Branch, access the specific child in the children vector and update the current pointer to that child [S3, unofficial].
        *   If it is not a Branch (Leaf/Empty), return the current cell [S3, unofficial].
4.  **Termination:** Return true if the minimum intersection distance $t$ is less than a threshold (e.g., 10000.0), otherwise return false [S2, unofficial].

**Pseudocode (SVO Activity Check & Descent):**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_hierarchical(ray_origin, ray_direction, root_node, max_depth):
    """
    Traverses a Sparse Voxel Octree to find active leaves intersected by the ray.
    Assumes a standard SVO structure where nodes are indexed in a pool.
    """
    # Stack stores (node_index, t_min, t_max)
    # t_min/t_max are the entry/exit distances of the ray into the node's bounding box
    stack = [(0, 0.0, float('inf'))]  # Start with root at index 0
    
    active_leaves = []
    
    while stack:
        node_idx, t_entry, t_exit = stack.pop()
        
        # Safety check for depth or invalid index
        if node_idx >= len(node_pool) or node_idx == 0: # Assuming 0 is root or invalid
            continue
            
        node = node_pool[node_idx]
        
        # Check if node is active
        if not is_node_active(node):
            continue
            
        # Check if node is a leaf
        if is_leaf(node):
            # Process leaf (e.g., check for surface, sample volume)
            # For activity check, just recording it is sufficient
            active_leaves.append(node_idx)
            continue
            
        # If branch, find intersecting children
        # In SVO, children are stored in a specific order (0-7)
        # We need to compute which of the 8 children the ray intersects
        children_indices = get_children_indices(node)
        
        # Compute bounding boxes for children and test intersection
        # This part depends on the specific coordinate system and child layout
        # Assuming a standard octree layout where children are 2x2x2 subdivisions
        
        intersecting_children = []
        for i in range(8):
            child_idx = children_indices[i]
            if child_idx is None:
                continue
                
            # Get child bounding box
            child_bbox = get_child_bbox(node, i)
            
            # Ray-box intersection
            t_min, t_max = ray_box_intersect(ray_origin, ray_direction, child_bbox)
            
            # If intersects and within current node's range
            if t_min < t_exit and t_max > t_entry:
                intersecting_children.append((child_idx, t_min, t_max))
        
        # Sort children by t_min to process closest first (optional but efficient)
        intersecting_children.sort(key=lambda x: x[1])
        
        # Push to stack
        for child_idx, t_min, t_max in intersecting_children:
            stack.append((child_idx, t_min, t_max))
            
    return active_leaves

def is_node_active(node):
    """
    Recursively checks if a node or any of its descendants are active.
    Optimized by caching the active flag in the node if possible.
    """
    if node.is_active_cached:
        return node.is_active_cached
        
    if node.is_leaf:
        # Leaf is active if it contains data
        active = node.has_data
    else:
        # Branch is active if any child is active
        active = False
        for child_idx in node.children:
            if child_idx is not None:
                child = node_pool[child_idx]
                if is_node_active(child):
                    active = True
                    break
                    
    node.is_active_cached = active
    return active
```

**Parameters:**
*   **Max Depth:** Limits the recursion depth to prevent infinite loops or excessive memory usage. Typical values depend on the resolution of the underlying grid [S1, unofficial].
*   **Threshold Distance:** The maximum $t$ value for which a hit is considered valid (e.g., 10000.0) [S2, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision:** Small ray direction components can cause division by zero in ray-box intersection tests. Guard against this by clamping small components to a small epsilon value (e.g., $10^{-4}$) [S2, unofficial].
*   **Boundary Voxels:** Special handling may be required for voxels at the boundary of the grid, where step masks might be zero [S2, unofficial].
*   **Leaf Identification:** In BVH implementations, leaf nodes are often identified by setting the high bit of child indices ($0x80000000$). If a leaf is the last element and has no sibling, its right child is set to an empty node index (0) [S2, unofficial].

## Subtopics

### [Leaf Collection](899-leaf-collection.md)
Leaf Collection is the post-traversal phase of hierarchical ray-grid traversal that identifies and gathers all terminal nodes (leaves) intersected by the ray grid. It solves the problem of converting a sparse, hierarchical traversal path into a contiguous list of active volumetric blocks required for subsequent ray marching or shading. Unlike sibling methods that may integrate traversal and shading or use different culling strategies, Leaf Collection strictly separates the identification of relevant geometry from the computation of radiance, enabling efficient memory compaction and parallel processing.

### [Point Plane Test Traversal](982-point-plane-test-traversal.md)
Point Plane Test Traversal is a method for locating the specific child tetrahedron within a TET-AMR (Tetrahedral Adaptive Mesh Refinement) refinement tree that contains a given query point. It solves the in-element lookup problem by traversing from the coarse root to the leaf (or max depth) using geometric half-space tests against planes defined by edge midpoints, avoiding the redundancy of checking all eight children individually. It is distinguished from [Leaf Collection](899-leaf-collection.md) by its on-the-fly vertex computation and from [VolumeHDDA](2353-volumehdda.md) by its reliance on explicit geometric plane tests rather than incremental rasterization logic.

### [Mesh BVH](1062-mesh-bvh.md)
Mesh BVH is a GPU-accelerated bounding volume hierarchy designed for efficient distance queries and sign evaluation on triangle meshes. It solves the performance bottleneck of testing all triangles in large meshes by enabling hierarchical pruning during ray-mesh intersection or point-to-mesh distance calculations. It is distinguished from its siblings by its specific integration into the `isoext.sdf.TriangleMeshSDF` pipeline for signed distance field computation, prioritizing raw speed for refractive path searches over guaranteed convergence properties found in interval-based refinement methods.

### [Multi-Resolution Rendering with K-D Tree](1942-multi-resolution-rendering-with-k-d-tree.md)
Multi-resolution rendering with K-D Tree is a hierarchical volume rendering technique that accelerates image-order ray casting by storing approximations of the underlying data within K-D tree nodes. It solves the performance bottleneck of rendering large, high-resolution datasets by allowing the renderer to skip detailed traversal when the data within a spatial region can be represented by a simple model within a user-specified error threshold. Unlike wavelet-based methods which store only detail coefficients, this method stores a complete model for each region, enabling fast "preview" images and adaptive detail. It builds on [Hierarchical Traversal](896-hierarchical-traversal.md).

### VolumeHDDA
VolumeHDDA is a specialized Hierarchical Digital Differential Analyzer for traversing sparse voxel trees (such as OpenVDB grids) to find ray intersections with active values [S1, S2, unofficial]. It solves the problem of efficiently identifying contiguous active segments along a ray in a hierarchical data structure, skipping inactive regions and coarse tiles when necessary [S1, S2, unofficial]. It differs from its siblings by explicitly templating on a `ChildNodeLevel` to define the entry point for hierarchical recursion, ensuring the traversal terminates at the leaf node level rather than the voxel level [S1, S2, unofficial].

### [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md)
Coarse Mesh Traversal accelerates volume rendering by decoupling global traversal from local sampling [G]. It constructs a simplified coarse mesh from the underlying dataset (typically a refinement tree or AMR structure) to perform fast global culling and bounding volume hierarchy (BVH) traversal, while deferring detailed sampling to a finer level [G]. It sets itself apart from sibling methods like [Leaf Collection](899-leaf-collection.md) and [Point Plane Test Traversal](982-point-plane-test-traversal.md) by explicitly managing the granularity of the coarse mesh to balance BVH efficiency against the overhead of the underlying refinement tree traversal [G].

### [Node Omission in Ray-Tree Traversal](3305-node-omission-in-ray-tree-traversal.md)
Node omission is an optimization for hierarchical ray-tree traversal that prevents adding child nodes to the traversal stack if it is known they will not intersect the current ray, reducing the number of intersection tests. It sets itself apart from sibling methods like [Leaf Collection](899-leaf-collection.md) and [Point Plane Test Traversal](982-point-plane-test-traversal.md) by operating on the spatial relationship between the ray's entry/exit parameters and the node's bounding box midpoints, rather than relying on leaf-level data structures or plane-based splitting planes.

### [Two-Pass Hierarchy Traversal](3339-two-pass-hierarchy-traversal.md)
Two-Pass Hierarchy Traversal is a spatial hierarchy traversal optimization that reduces computational cost by splitting the descent into a coarse initial pass and a fine subsequent pass. It sets itself apart from single-pass methods like [Leaf Collection](899-leaf-collection.md) or [Point Plane Test Traversal](982-point-plane-test-traversal.md) by explicitly decoupling the identification of relevant subtrees from the detailed node evaluation, allowing for significant reuse across pixel blocks.

### [Adaptive Out-of-Core Single-Pass GPU Raycasting](3440-adaptive-out-of-core-single-pass-gpu-raycasting.md)
This method enables interactive rendering of multi-gigabyte scalar volumes on desktop hardware by combining an out-of-core octree data structure with a single-pass, stackless GPU raycasting algorithm. It solves the memory capacity and bandwidth bottlenecks of in-core rendering by maintaining a view- and transfer-function-dependent working set of volumetric bricks in GPU memory, while the rest of the dataset remains on disk. Unlike separate block rendering techniques that suffer from boundary artifacts and cannot support refractive or global illumination models, this approach performs the entire volume traversal and compositing in a single fragment shader pass, adapting sampling density to the local brick resolution.

### Surface Descendant Check
If the cell is a Leaf, return whether it has a sign change. If it is a Branch, recursively check if any of its children have a surface descendant. If it is Empty or Full, return false [S3, unofficial].

### Coarse Mesh Traversal via BVH
Uses a Bounding Volume Hierarchy (BVH) built over the coarse tetrahedral mesh to efficiently locate the coarse element containing a sample point [S6, unofficial]. Build a triangle BVH over the coarse mesh by inserting four triangular faces facing outside for every coarse element. Trace rays into the BVH using OptiX Wrapper Library (OWL) hit and miss functions. The hit function returns the hit point and the coarse tetrahedron ID. The miss function reports no intersection with BVH faces. Unlike element marching which uses connectivity, point location querying skips coarse elements not being sampled and allows varying refinement levels in the coarse mesh [S6, unofficial].

## Operating characteristics
**Time Complexity:** $O(\log N)$ for balanced trees, where $N$ is the number of voxels, assuming uniform distribution. In sparse scenes, it approaches $O(1)$ for empty regions. Non-hierarchical traversal takes 11 steps in the example, while hierarchical SVO traversal takes 5 steps, demonstrating a significant reduction in steps [S1, unofficial].
**Memory Complexity:** $O(N)$ for the tree structure, but significantly less than a dense grid for sparse data due to the SVO's on-demand subdivision [S1, unofficial].
**Performance:** The technique is several times faster than other published algorithms for ray-triangle intersection and hierarchical traversal [S4, unofficial].
**Strengths:** Efficient skipping of empty space; reduced number of intersection tests; suitable for sparse data.
**Limitations:** Overhead of tree traversal; potential for stack overflow in deep trees; complexity of implementation compared to flat grid traversal.

## Background
Hierarchical traversal was introduced by Tim Kay and James T. Kajiya in 1986 as a new algorithm for speeding up ray-object intersection calculations [S4, unofficial]. They proposed a new type of extent that can fit convex hulls arbitrarily tightly and placed objects into a hierarchy. The algorithm queries objects along the ray in an efficient order, demonstrating that it is possible to ray trace scenes containing hundreds of thousands of objects [S4, unofficial]. The concept has since been adapted for volume rendering using Octrees and Sparse Voxel Octrees (SVOs), where the hierarchy allows for larger steps through empty space [S1, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
[S3] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S4] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
[S5] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
[S6] Direct volume rendering of tree-based tetrahedral adaptive mesh refinement data, https://repository.bilkent.edu.tr/items/fcbc592a-2646-49af-8b84-67308b6fd3e9, fetched 2026-10-27 (unofficial)
