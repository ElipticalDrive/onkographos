# Hierarchical Traversal

Also called: SVO Traversal, Sparse Voxel Octree (SVO) Traversal

## Overview
Hierarchical traversal accelerates volume rendering by navigating a multi-resolution spatial hierarchy (such as an Octree or Sparse Voxel Octree) where higher-level cells cover larger areas (e.g., $2^k \times 2^k \times 2^k$) of the level below [S1, unofficial]. This allows the renderer to take larger steps through empty space, significantly reducing the total number of traversal steps compared to non-hierarchical grid walks [S1, unofficial]. It sets itself apart from flat grid traversal by adapting step size to local data density, and from [BVH Traversal](18487-bounding-volume-hierarchy-bvh-traversal.md) by maintaining a strict resolution hierarchy coupled to the data structure rather than arbitrary bounding volumes [S6, unofficial].

## How it works
Hierarchical traversal builds on the stack-based principles of [Stack-based traversal](23146-stack-based-traversal.md), applying them to nodes with variable spatial extents.

### Inputs and Outputs
*   **Input**: A ray defined by origin $O$ and direction $D$; a hierarchical data structure (Octree/SVO) where each node has a bounding box and a child pointer (or index).
*   **Output**: A sequence of leaf nodes (or active voxels) intersected by the ray, processed for shading/compositing.

### Data Structures
The hierarchy is typically stored as a linear array of nodes. A common layout for a Sparse Voxel Octree (SVO) node is [S3, unofficial]:

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
struct SVO_Node:
    # Internal node
    children_start: u32  # Index of first child in the node pool
    children_count: u32  # Number of children (up to 8)
    # Leaf node
    data_offset: u32     # Offset into the leaf data buffer
    data_size: u32       # Size of leaf data
    # Flags
    is_leaf: bool
    is_empty: bool
    is_full: bool
```
*Note: Some implementations pack these into 16-byte aligned structures or use bit-masking for child indices to distinguish leaves [S3, unofficial].*

Cell naming conventions often refer to the area covered at Level 0: Level 0 cells are $1\times1$, Level 1 are $2\times2$, Level 2 are $4\times4$ [S1, unofficial].

### Algorithm Steps
1.  **Initialization**:
    *   Compute the ray's intersection with the root node's bounding box.
    *   If no intersection, terminate.
    *   Push the root node index and intersection interval $[t_{min}, t_{max}]$ onto the traversal stack.

2.  **Traversal Loop**:
    *   While the stack is not empty:
        1.  Pop the top state $(node\_idx, t_{min}, t_{max})$.
        2.  Retrieve the node from the data structure.
        3.  **Leaf Check**: If the node is a leaf:
            *   Perform volume sampling/shading for the ray segment within this leaf.
            *   Update $t_{min}$ to the exit point of this leaf.
            *   If $t_{min} \ge t_{max}$, terminate the ray.
            *   Continue to next stack item.
        4.  **Internal Node Check**:
            *   Identify which children intersect the ray.
            *   Compute intersection intervals $[t_{child\_min}, t_{child\_max}]$ for each child.
            *   Filter children where $t_{child\_max} < t_{min}$ or $t_{child\_min} > t_{max}$.
            *   Sort remaining children by $t_{child\_min}$ (ascending).
            *   Push children onto the stack in **reverse order** (farthest first) so the closest child is popped next.

3.  **Optimization - Surface Descendant Check**:
    *   For isosurface extraction, before traversing children, check if the current node has a "surface descendant" (a sign change in the scalar field) [S4, unofficial].
    *   If the node is a leaf, return whether it has a sign change.
    *   If the node is a branch, recursively check if any child has a surface descendant.
    *   If empty or full (no sign change), prune the subtree [S4, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_traverse(ray_origin, ray_dir, root_node, max_depth):
    # Stack stores tuples: (node_index, t_min, t_max)
    stack = [(root_node.index, ray_origin.t_min, ray_origin.t_max)]
    
    while stack:
        node_idx, t_min, t_max = stack.pop()
        node = get_node(node_idx)
        
        # 1. Leaf Processing
        if node.is_leaf:
            # Sample volume or extract surface
            process_leaf(node, ray_origin, ray_dir, t_min, t_max)
            
            # Update t_min for next potential intersections in this ray
            # Assuming leaf covers a segment, we might need to re-intersect
            # or simply advance t_min past this leaf's exit.
            # For simplicity, if leaf is fully processed, we continue.
            # In many SVO implementations, leaves are small enough that
            # we just march or sample.
            continue
            
        # 2. Internal Node Traversal
        if node.is_empty:
            continue
            
        children = node.get_children()
        child_intersections = []
        
        for child in children:
            if child is None:
                continue
                
            # Compute intersection of ray with child's AABB
            t_child_min, t_child_max = intersect_aabb(ray_origin, ray_dir, child.bbox)
            
            # Check overlap with current ray segment
            if t_child_max > t_min and t_child_min < t_max:
                child_intersections.append((child, t_child_min, t_child_max))
        
        # Sort by entry distance (ascending)
        child_intersections.sort(key=lambda x: x[1])
        
        # Push in reverse order (farthest first)
        for child, t_min_c, t_max_c in reversed(child_intersections):
            stack.append((child.index, t_min_c, t_max_c))
            
    return
```

### Parameters and Edge Cases
*   **Termination**: A termination object (e.g., max depth, opacity threshold) stops traversal early [S2, unofficial]. The termination object must be convex [S2, unofficial].
*   **Precision**: Handling zero direction components in ray-box intersection requires clamping $d_i$ to $\epsilon$ or using infinity for $t$ calculations [S1, unofficial].
*   **Boundary Voxels**: Special care is needed for rays entering/exiting the volume bounds to avoid infinite loops or missed nodes [S3, unofficial].

## Subtopics

### [Leaf Collection](899-leaf-collection.md)
Identifies and gathers all terminal nodes intersected by a ray grid, converting sparse traversal into a contiguous list for parallel processing.

### [Point Plane Test Traversal](982-point-plane-test-traversal.md)
Locates specific child tetrahedra in TET-AMR trees using geometric half-space tests, avoiding redundant checks.

### [Multi-Resolution Rendering with K-D Tree](1942-multi-resolution-rendering-with-k-d-tree.md)
Stores complete models in K-D tree nodes for fast preview and adaptive detail, building on hierarchical traversal.

### [Coarse Mesh Traversal](3062-coarse-mesh-traversal.md)
Decouples global search (coarse BVH) from local sampling, skipping empty regions entirely.

### [Node Omission in Ray-Tree Traversal](3305-node-omission-in-ray-tree-traversal.md)
Optimizes traversal by omitting children from the stack if they are known not to intersect the ray.

### [Adaptive Out-of-Core Single-Pass GPU Raycasting](3440-adaptive-out-of-core-single-pass-gpu-raycasting.md)
Combines out-of-core octrees with stackless GPU raycasting for multi-gigabyte volumes.

### [From-Point Algorithm](18431-from-point-algorithm.md)
Uses octree traversal to generate screen-space occluders for visibility culling.

### [Bounding Volume Hierarchy (BVH) Traversal](18487-bounding-volume-hierarchy-bvh-traversal.md)
Uses a tree of bounding volumes for ray-primitive intersection, adapting to geometry distribution.

### [Hierarchical Longest Common Traversal Sequence](21519-hierarchical-longest-common-traversal-sequence.md)
Matches traversal histories to create common sequences for multiple rays, reusing state.

### [Traversal Order and Control Flow](21568-traversal-order-and-control-flow.md)
Defines algorithmic strategies for navigating hierarchies, optimizing descent/ascent paths.

### Surface Descendant Check
An optimization for isosurface extraction where the traversal checks if a node or its descendants contain a sign change in the scalar field. If a node is a leaf, it returns whether it has a sign change. If it is a branch, it recursively checks its children. If empty or full, it returns false, allowing the traversal to skip the entire subtree [S4, unofficial].

### Warp-Level Intersection Caching
A GPU optimization that delays intersection tests until enough threads in a warp have stored an intersection child node. When an intersection is found, the voxel data is stored. At the start of each step, the warp checks if enough threads have stored intersections; if so, they proceed with the test. Otherwise, the warp continues traversing together. Threads that find a new intersection while already holding one stop until the next test phase [S8, unofficial].

### Packet Ray Traversal / Ray Coherence Optimization
Configures CUDA grids/blocks as 2D to map spatially coherent threads (rays) to neighboring threads. This causes rays with similar directions to traverse the octree similarly, improving cache hit rates. A pitfall is thread divergence in deep subdivisions, causing SIMD stalls [S9, unofficial].

### Octree Traversal with Texture Memory Access
Binds octree node data to CUDA texture objects and accesses them via `tex1Dfetch()`. This utilizes the texture cache for the random access patterns typical in tree traversal, preferred over constant memory due to size limits and broadcast optimization mismatches [S9, unofficial].

### Bookkeeping Method
Inspired by Frisken & Perry (2002), this method avoids starting the search from the root node for every wall passage in simulations. It maintains bookkeeping state to continue traversal from the last known position, reducing overhead compared to top-down searches [S10].

## Operating characteristics
*   **Time Complexity**: $O(\log N)$ for sparse data, where $N$ is the number of voxels, compared to $O(N)$ for dense grids. The number of steps is significantly reduced; e.g., 5 steps for SVO vs. 11 for non-hierarchical in comparable examples [S1, unofficial].
*   **Memory Complexity**: Depends on the structure. SVOs store only occupied nodes, offering significant space savings for sparse data. However, pointer overhead exists.
*   **Performance**: Several times faster than non-hierarchical algorithms [S5, unofficial].
*   **Strengths**: Efficient for sparse volumes; adaptive resolution; reduces overdraw.
*   **Limitations**: Stack depth can be large for deep trees; pointer chasing can cause cache misses on CPU; thread divergence on GPU [S9, unofficial].
*   **When to use**: Sparse volumetric data, multi-resolution datasets, isosurface extraction.
*   **When not to use**: Dense, fully occupied volumes where grid traversal is simpler and faster; hardware with limited stack size (requires stackless variants).

## Background
Hierarchical traversal originated with the development of ray tracing acceleration structures. Kay and Kajiya (1986) introduced a hierarchy traversal algorithm for ray-object intersection, demonstrating significant speedups [S5, unofficial]. In volume rendering, the concept was adapted to Octrees and Sparse Voxel Octrees (SVOs) to handle large, sparse datasets efficiently [S1, unofficial]. The technique is widely used in modern volume renderers, particularly those dealing with scientific visualization and medical imaging, where data sparsity is common [S1, S6, unofficial].

## Sources
[S1] versal algorithm for ray tracing Hur man gör en hierarkisk ..., https://kth.diva-portal.org/smash/get/diva2:2069409/FULLTEXT01.pdf, fetched 2026-10-25 (unofficial)
[S2] LCTS: Ray Shooting using Longest Common Traversal Sequences, https://www.slideserve.com/liam/vlastimil-havran, fetched 2026-10-26 (unofficial)
[S3] mxbrt/svo, https://github.com/mxbrt/svo, fetched 2026-10-26 (unofficial)
[S4] narnia-ai-mason/isomesh, https://github.com/narnia-ai-mason/isomesh, fetched 2026-10-27 (unofficial)
[S5] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
[S6] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
[S7] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
[S8] Level-of-Detail Independent Voxel-Based Surface ..., https://studenttheses.uu.nl/items/a8b8233a-d142-43f1-8f6d-c0b0fe43a767, fetched 2026-10-27 (unofficial)
[S9] Best way of traversing an octree in CUDA? - CUDA Programming and Performance - NVIDIA Developer Forums, https://forums.developer.nvidia.com/t/best-way-of-traversing-an-octree-in-cuda/9179, fetched 2026-10-24 (unofficial)
[S10] Using hierarchical octrees in Monte Carlo radiative transfer simulations, https://doi.org/10.1051/0004-6361/201220854, fetched 2026-10-24
