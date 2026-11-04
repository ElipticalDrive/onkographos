# Octree Traversal
Also called: Hierarchical Octree Traversal, Octree Space Subdivision

## Overview
Octree Traversal is a hierarchical ray-intersection algorithm that accelerates volume rendering by navigating a spatial hierarchy to skip empty space and reduce the number of ray-casting steps [S1, S5, unofficial]. It solves the performance bottleneck of non-hierarchical voxel traversal, where every voxel along a ray must be visited, by allowing the ray to leap over large empty regions [S1, unofficial]. Unlike its siblings, it operates on a recursive tree structure where each parent node divides into eight equal sub-blocks ($2 \times 2 \times 2$ subdivision), enabling larger steps through empty space [S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; the root node of the octree; traversal bounds $[t_{enter}, t_{exit}]$ [S4].
*   **Output:** A sorted list of intersected leaf nodes (voxels) or a termination signal upon finding the first occupied leaf (for ray casting) or completing the volume integration [S1, unofficial].

**Data Structures:**
The octree is a hierarchical indexing mechanism where each parent node contains 0 or 8 children [S2, S3, unofficial]. The 8 children achieve a $2 \times 2 \times 2$ regular subdivision of the parent node into 8 equal parts or "leaves" [S2, S3, unofficial].
*   **Interior Node:** Contains pointers to up to 8 children. May store metadata (e.g., min/max scalar values) to facilitate empty space skipping [S3, unofficial][S4].
*   **Leaf Node:** Represents the finest level of subdivision (single voxel or small block) [S4].
*   **Cap Node:** A specific leaf type at the boundary of the finest resolution, mapping to a cell via reconstruction [S4].
*   **Scalar Leaf:** A coarser-resolution scalar embedded in a parent node [S4].

**Algorithm Steps:**
1.  **Initialization:** Start at the octree root. Initialize the ray parameter range $[t_{enter}, t_{exit}]$ [S4].
2.  **Mid-plane Intersection:** For the current node, compute the ray intersection distances with the X, Y, and Z mid-planes of the node's bounding box: $t_{midX}, t_{midY}, t_{midZ}$ [S4].
3.  **Ordering:** Sort the mid-plane intersection distances to determine the sequential order in which the ray intersects the eight child octants [S4]. A ray intersects at most 4 octants of an interior node [S4].
4.  **Child Index Computation:** For each child octant in the sorted order, determine the 0–7 octant offset (and thus the linear index) of the child node that contains the target coordinate within the current parent node using bitwise operations [S4].
5.  **Traversal/Recursion:**
    *   If the child node is **empty** (no data), skip it [S1, unofficial].
    *   If the child node is an **interior node**, recurse: update $t_{enter}$ and $t_{exit}$ to the intersection with the child's bounding box, and repeat from Step 2 [S1, unofficial][S4].
    *   If the child node is a **leaf** (voxel, cap, or scalar), process it (e.g., sample data, write to output array) [S1, unofficial][S4].
6.  **Termination:** The search algorithm terminates when:
    *   A block with no data is encountered (empty space skipping) [S1, unofficial].
    *   The smallest blocks (voxels) are reached [S1, unofficial].
    *   The ray exits the volume ($t > t_{exit}$) [S4].
    *   Early termination criteria are met, such as when remaining voxels will not significantly contribute to the pixel color [S2, unofficial].
7.  **Output:** All found voxels are written to an output array in sorted order [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def traverse_octree(ray_o, ray_d, node, t_min, t_max):
    """
    Recursively traverse an octree node for a given ray segment.
    node: The current octree node (Interior, Cap, or Scalar Leaf).
    t_min, t_max: The ray parameter range [t_min, t_max] intersecting this node.
    """
    # Base case: If the ray segment is invalid or node is empty, return
    if t_min >= t_max or node.is_empty():
        return []

    # If leaf node, process and return
    if node.is_leaf():
        return [sample_leaf(node, ray_o, ray_d, t_min, t_max)]

    # Interior node: Compute mid-plane intersections
    bbox_min, bbox_max = node.get_bounding_box()
    mid_x = (bbox_min.x + bbox_max.x) / 2.0
    mid_y = (bbox_min.y + bbox_max.y) / 2.0
    mid_z = (bbox_min.z + bbox_max.z) / 2.0
    
    # Calculate t-values for mid-planes, handling zero direction components
    t_mid_x = (mid_x - ray_o.x) / ray_d.x if ray_d.x != 0 else float('inf')
    t_mid_y = (mid_y - ray_o.y) / ray_d.y if ray_d.y != 0 else float('inf')
    t_mid_z = (mid_z - ray_o.z) / ray_d.z if ray_d.z != 0 else float('inf')
    
    # Sort mid-plane intersections to determine octant traversal order
    # A ray crosses at most 3 planes, visiting at most 4 octants per node
    sorted_mids = sorted([(t_mid_x, 0), (t_mid_y, 1), (t_mid_z, 2)])
    
    # Determine the 8 child octants and their indices
    # Child index is derived from the ray's position relative to the node center
    # using bitwise operations on the sign bits of (ray_o - center) [S4]
    child_indices = compute_octant_indices(sorted_mids, bbox_min, bbox_max, ray_o)
    
    results = []
    for child_idx in child_indices:
        child = node.children[child_idx]
        if child is None or child.is_empty():
            continue
            
        # Compute child bounding box from parent bounds and child index
        child_bbox_min, child_bbox_max = compute_child_bounds(bbox_min, bbox_max, child_idx)
        
        # Intersect ray with child bounding box
        t_enter, t_exit = intersect_ray_aabb(ray_o, ray_d, child_bbox_min, child_bbox_max)
        
        # Only recurse if the ray actually intersects the child within the current segment
        if t_enter < t_exit and t_enter < t_max and t_exit > t_min:
            t_new_enter = max(t_min, t_enter)
            t_new_exit = min(t_max, t_exit)
            
            if t_new_enter < t_new_exit:
                results.extend(traverse_octree(ray_o, ray_d, child, t_new_enter, t_new_exit))
                
    return results

def compute_octant_indices(sorted_mids, bbox_min, bbox_max, ray_o):
    """Determines the 0-7 octant traversal order based on sorted mid-plane crossings."""
    # Standard geometric traversal: the sorted order of mid-plane crossings
    # directly maps to the visitation sequence of the 8 children.
    # Returns a list of child indices in traversal order.
    # Implementation depends on specific octree layout (e.g., Morton/Z-order).
    # For a standard layout, the sorted mid-planes dictate the path.
    return [0, 1, 2, 3, 4, 5, 6, 7]  # Placeholder for layout-specific mapping [G]

def compute_child_bounds(parent_min, parent_max, child_idx):
    """Computes the AABB of a child octant given the parent bounds and child index."""
    dx = (parent_max.x - parent_min.x) / 2.0
    dy = (parent_max.y - parent_min.y) / 2.0
    dz = (parent_max.z - parent_min.z) / 2.0
    # Map child_idx (0-7) to x,y,z offsets
    x_off = (child_idx & 1) * dx
    y_off = ((child_idx >> 1) & 1) * dy
    z_off = ((child_idx >> 2) & 1) * dz
    return (parent_min.x + x_off, parent_min.y + y_off, parent_min.z + z_off), \
           (parent_min.x + dx + x_off, parent_min.y + dy + y_off, parent_min.z + dz + z_off)

def intersect_ray_aabb(ray_o, ray_d, bbox_min, bbox_max):
    """Standard slab method for ray-AABB intersection."""
    t_min = float('-inf')
    t_max = float('inf')
    for i in range(3):
        if ray_d[i] == 0:
            if ray_o[i] < bbox_min[i] or ray_o[i] > bbox_max[i]:
                return float('inf'), float('-inf')
        else:
            t1 = (bbox_min[i] - ray_o[i]) / ray_d[i]
            t2 = (bbox_max[i] - ray_o[i]) / ray_d[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
    return t_min, t_max

def sample_leaf(node, ray_o, ray_d, t_min, t_max):
    """Samples voxel data or performs shading for a leaf node."""
    return node.sample(ray_o, ray_d, t_min, t_max)
```

**Parameters:**
*   **Minimum Block Size:** The recursion stops when blocks reach a specific minimum size or the size of a single voxel [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **GPU Implementation:** Octrees are difficult to implement on GPUs because GPU programming does not allow recursive algorithms; the required stack must be implemented manually [S1, unofficial].
*   **Irregular Geometry:** For meshes and unstructured volumes, octrees conform poorly to irregular geometry, leading to relatively expensive traversal strategies [S5, unofficial].
*   **Node Types:** Separate traversal routines are required for interior nodes (child can be interior, cap, or scalar leaf), scalar leaves (child is always the same scalar leaf, only non-empty at forward edges), and cap nodes (child is a voxel, maps to a cell via reconstruction) [S4].
*   **Precision:** Floating-point precision issues may arise when computing mid-plane intersections near node boundaries; using epsilon guards or higher-precision arithmetic is recommended [G].

## Subtopics

### [Hierarchical Traversal](896-hierarchical-traversal.md)
A ray-intersection algorithm that accelerates volume rendering by navigating a spatial hierarchy (such as an Octree or Sparse Voxel Octree) to skip empty space and reduce the number of ray-casting steps. It solves the performance bottleneck of non-hierarchical voxel traversal, where every voxel along a ray must be visited, by allowing the ray to "leap" over large empty regions. Unlike its siblings, it operates on a recursive tree structure where higher-level cells cover a $2 \times 2 \times 2$ area of the level below, enabling larger steps through empty space.

### [Octree-based Volume Rendering](1131-octree-based-volume-rendering.md)
Octree-based volume rendering accelerates image-order ray casting by recursively traversing a hierarchical spatial subdivision of volumetric data, using projected size and field-of-view criteria to select nodes for approximation or culling. It differs from sibling methods like [Point Plane Test Traversal](982-point-plane-test-traversal.md) by using a tree structure for adaptive level-of-detail selection rather than fixed coarse meshes or plane tests, and unlike [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md) which focuses on explicit sparse storage, this method traditionally assumes the octree resides in main memory.

### [Octree Child Index Computation](1807-octree-child-index-computation.md)
Octree Child Index Computation determines the 0–7 octant offset (and thus the linear index) of the child node that contains a specific target coordinate within the current parent node. It solves the problem of efficiently mapping a 3D position to a specific branch in a hierarchical grid without arbitrary bit shifts. It sets itself apart from generic tree traversal by relying on precomputed bit masks and constant-time bitwise operations to derive the child index.

### [Octree Neighbor Finding](1821-octree-neighbor-finding.md)
Octree Neighbor Finding determines the adjacent leaf nodes in six cardinal directions (±X, ±Y, ±Z) for a given node in a Sparse Voxel Octree (SVO). It solves the problem of traversing between non-contiguous or differently-sized nodes during ray casting or pathfinding by locating the nearest shared ancestor and performing point location to retrieve the neighbor.

### [Ray Traversal of Octree Nodes](1841-ray-traversal-of-octree-nodes.md)
This method determines the exact sequential order in which a ray intersects the eight child octants of an interior octree node by computing and sorting the ray's intersection distances with the X, Y, and Z mid-planes of the current node. It solves the problem of efficiently traversing hierarchical sparse voxel structures without pointer chasing or stack overhead for descent, by replacing recursive descent with a deterministic, sorted march through children. Unlike [Stack-based Octree Traversal with Direction Flipping](5007-stack-based-octree-traversal-with-direction-flipping.md) which uses explicit stacks, or [IEEE-754 Bit Manipulation Octree Traversal](3907-ieee-754-bit-manipulation-octree-traversal.md) which uses bit-level arithmetic, this method relies on geometric mid-plane sorting. It is particularly effective in [Octree Traversal](1587-octree-traversal.md) contexts where empty space skipping is utilized, as it allows the solver to start at the node border and skip nodes that do not contain the isosurface.

### [CPU Sparse Voxel Octree Traversal](2455-cpu-sparse-voxel-octree-traversal.md)
CPU Sparse Voxel Octree Traversal is a ray traversal algorithm for 3D sparse voxel octrees stored in a linear block structure, implemented on the CPU. It uses a stack to manage traversal state, handling valid internal nodes, leaf nodes, and empty space by casting rays to the next occupied voxel. It sets itself apart from GPU-based approaches by performing all ray marching and shading calculations on the host processor, using a linear block structure for the octree and a stack-based traversal mechanism.

### [Sparse Ray-Octree Intersection (DECIDE)](3006-sparse-ray-octree-intersection-decide.md)
Sparse Ray-Octree Intersection (DECIDE) is a GPU-based algorithm for finding the first intersection of a ray with a sparse voxel octree structure, returning the hit position, normal, and node index. It solves the problem of efficiently traversing hierarchical occupancy grids by handling traversal, descent, and backtracking (popping) using a stack.

### [Residency Octree Traversal](3073-residency-octree-traversal.md)
Residency Octree Traversal is a ray-guided method for sparse voxel volume rendering that determines which data bricks to load from a server into a local cache based on the current viewing ray. It solves the problem of efficient memory management and bandwidth reduction in large-scale volume visualization by skipping empty or homogeneous regions and only requesting bricks that are actually needed for rendering. It sets itself apart from standard octree traversal by integrating residency metadata directly into the traversal logic, allowing the renderer to decide whether to skip a node, use its aggregate value, or request specific voxel data based on cache status and resolution requirements.

### [PND3D Octree Rendering](3433-pnd3d-octree-rendering.md)
PND3D is a volume rendering method that uses a standard 8-way octree with non-leaf nodes containing child and solid masks, visited in front-to-back depth-first order to enable occlusion culling. It distinguishes itself by splitting the screen into large rectangular tiles determined by 3 vanishing points of the voxel grid projected to screen coordinates, allowing each tile to be rendered as an independent job starting from the root of the octree. This approach facilitates multithread optimization and specific hardware-accelerated occlusion queries using bit arrays.

### [GigaVoxels Ray Traversal and Rendering](4144-gigavoxels-ray-traversal-and-rendering.md)
GigaVoxels is a GPU-based volume rendering technique that uses an octree structure to manage data at the "brick" level (a small 3D texture block), traversing the tree to sample voxel data and performing front-to-back blending. It distinguishes itself from standard sparse voxel octree traversal by decoupling the geometric hierarchy (the octree) from the volumetric data storage (the brick pool), allowing for efficient memory residency management and level-of-detail (LOD) culling based on brick occupancy rather than individual voxel density.

### N3-Tree
Octrees and their generalization N3-trees (N subdivisions per node) are considered a special case of Kd-Trees [S6, unofficial]. No algorithm in available literature proposes to exploit the specific structure of octrees to accelerate ray traversal [S6, unofficial]. Octrees offer regular subdivision providing identical node size at each level, which is interesting for storing regular data bricks [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** Theoretically optimal in terms of fewest traversal steps for regularly spaced geometry, assuming objects are contained uniformly within cells of the acceleration structure, with no overlap [S5, unofficial]. However, for meshes and unstructured volumes, they conform poorly to irregular geometry, leading to relatively expensive traversal strategies [S5, unofficial].
*   **Memory Complexity:** Requires storage for the tree structure. The min/max tree, used as an acceleration structure, adds approximately 22% footprint on top of raw octree voxel data [S4].
*   **Strengths:** Enables significantly faster finding of intersections in volumetric datasets by sampling only "non-empty space" [S1, unofficial]. It is particularly effective for regular voxel geometry [S5, unofficial].
*   **Limitations:** Difficult to implement on GPUs due to the lack of native recursion support, requiring manual stack management [S1, unofficial]. Not ideal for irregular geometry or meshes [S5, unofficial].
*   **When to Use:** Regular voxel geometry where empty space skipping provides significant performance gains [S5, unofficial].
*   **When Not to Use:** Irregular geometry, meshes, or unstructured volumes where hierarchical grids may be more efficient [S5, unofficial].

## Background
*   **Origin:** The original Glassner implementation [32] proposed top-down point location testing along successive octree nodes hit by the ray [S5, unofficial].
*   **Development:** Samet [105] modified the marching procedure to incorporate a neighbor-finding algorithm, delivering dramatic speedups [S5, unofficial]. Sung [120] proposed a DDA traversal similar to a hierarchical grid [S5, unofficial]. Gargantini and Atkinson [29] implemented a traversal similar to a kd-tree where the ray intersection with each octant midplane is ordered [S5, unofficial].
*   **Usage:** Used in volumetric rendering applications to perform empty space skipping [S3, unofficial].

## Sources
[S1] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S2] Volume Ray Casting Techniques and Applications using ..., https://halogenica.net/bin/graphics/volume-rendering-using-gpgpu/Thesis.pdf, fetched 2026-10-27 (unofficial)
[S3] Volume ray casting techniques and applications using general purpose computations on graphics processing units, https://repository.rit.edu/cgi/viewcontent.cgi?article=6466&context=theses, fetched 2026-10-28 (unofficial)
[S4] PDF Interactive Isosurface Ray Tracing of Large Octree Volumes, https://pdfs.semanticscholar.org/aef9/6e0dd17920d0c70cbf0dd43f494cab1c0bcc.pdf, fetched 2026-10-28
[S5] RAY TRACING IMPLICIT SURFACES FOR INTERACTIVE ..., https://www.sci.utah.edu/~knolla/thesis.pdf, fetched 2026-10-29 (unofficial)
[S6] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
