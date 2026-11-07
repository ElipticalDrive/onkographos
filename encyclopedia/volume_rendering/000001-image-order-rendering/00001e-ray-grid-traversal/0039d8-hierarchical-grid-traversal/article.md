# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal is a ray-marching algorithm that accelerates volume rendering by traversing a multi-resolution grid mipmap structure [S1, S2, unofficial]. It solves the performance bottleneck of visiting empty or uniform regions in sparse volume data by stepping through coarse nodes first and only descending into child voxels when the ray intersects the parent's axis-aligned bounding box (AABB) [S1, unofficial]. Unlike flat traversal methods like [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) which visit every voxel, this method leverages lookup tables to map parent indices to children, significantly reducing the number of intersection tests and memory accesses for sparse datasets [S1, unofficial]. It is distinct from [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) in its specific implementation using pre-computed mipmap tables rather than compile-time template meta-programming for VDB structures [S1, S5, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical grid mipmap structure with $L$ levels; a set of lookup tables `tables_list` where each level $l$ contains a table mapping coarse voxel indices to their children and coordinates.
*   **Output:** A list of voxel positions (or indices) at the maximum resolution level (leaf level) that are intersected by the ray.

**Data Structures:**
*   **Lookup Table (`table`):** A 2D array where each row corresponds to a coarse voxel at level $l$.
    *   Columns 0–7: Store the indices of the 8 child voxels at level $l+1$. If a child is empty or invalid, the value is $-1$ [S1, unofficial].
    *   Columns -3 to -1 (or 8–10 in 0-indexed notation): Store the 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].
*   **State Variables:**
    *   `cube_touched_idx`: A list of indices of voxels at the current level that the ray intersects [S1, unofficial].
    *   `cube_ori_list`: The final list of leaf voxel positions [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Set `cube_touched_idx` to `[0]`, representing the root voxel index [S1, unofficial].
    *   Initialize `new_list` as an empty list [S1, unofficial].
    *   Determine the maximum level `max_level` of the mipmap hierarchy [S1, unofficial].

2.  **Iterative Traversal (Level $l$ from 0 to `max_level`):**
    *   Retrieve the lookup table for the current level: `table = tables_list[l]` [S1, unofficial].
    *   For each voxel index `i` in `cube_touched_idx`:
        *   Retrieve the voxel's 3D grid coordinates from the table: `coords = table[i, -3:]` [S1, unofficial].
        *   Compute the physical origin and end of the voxel's AABB. Let `cube_size` be the physical size of a voxel at level $l$. The sources imply a base resolution scaling, e.g., `cube_size = 256 / (2 ** l)` for a base grid of 256 units [S1, unofficial].
            *   `cube_ori = coords * cube_size`
            *   `cube_end = cube_ori + cube_size`
        *   Compute the ray-AABB intersection using the ray origin $\mathbf{o}$, inverse direction $\mathbf{d}^{-1}$, and the AABB bounds [S1, unofficial].
            *   Calculate near intersection distance $t_n$ and far intersection distance $t_f$.
        *   **Intersection Check:**
            *   If $t_n \le t_f$ (the ray intersects the AABB):
                *   If $l == \text{max\_level}$:
                    *   Append the voxel's grid coordinates `table[i, -3:]` to `cube_ori_list` [S1, unofficial].
                *   Else ($l < \text{max\_level}$):
                    *   Extract the 8 child indices from `table[i, :8]` [S1, unofficial].
                    *   Create a mask for valid children: `mask = table[i, :8] != -1` [S1, unofficial].
                    *   Filter the child indices using the mask and convert to integers [S1, unofficial].
                    *   Append valid child indices to `new_list` [S1, unofficial].
            *   Else (no intersection):
                *   Do not add any children to `new_list` for this parent [S1, unofficial].
    *   Update `cube_touched_idx` to `new_list` [S1, unofficial].
    *   Reset `new_list` to empty for the next level [S1, unofficial].

3.  **Return:**
    *   Return `cube_ori_list` containing the positions of the leaf voxels touched by the ray [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, tables_list, max_level):
    # Initialize state
    cube_touched_idx = [0]  # Start with root voxel index
    cube_ori_list = []
    
    # Precompute inverse direction for AABB intersection
    inv_dir = 1.0 / ray_direction
    
    for level in range(max_level + 1):
        table = tables_list[level]
        new_list = []
        
        # Determine voxel size for current level
        # Assuming base resolution of 256 units as per source [S1, unofficial]
        cube_size = 256.0 / (2 ** level)
        
        for i in cube_touched_idx:
            # Get voxel coordinates from table (columns -3 to -1)
            coords = table[i, -3:]
            
            # Compute AABB bounds
            cube_ori = coords * cube_size
            cube_end = cube_ori + cube_size
            
            # Ray-AABB Intersection
            # Using standard slab method
            t_min = (cube_ori - ray_origin) * inv_dir
            t_max = (cube_end - ray_origin) * inv_dir
            
            # Swap if necessary
            t_min, t_max = (t_min, t_max) if t_min[0] <= t_max[0] else (t_max, t_min)
            # Note: The source uses a generic intersect_ray_aabb function. 
            # The logic below implements the standard check tn <= tf.
            
            # Compute near (tn) and far (tf) intersection distances
            # tn is the maximum of the minimums, tf is the minimum of the maximums
            tn = max(t_min[0], t_min[1], t_min[2])
            tf = min(t_max[0], t_max[1], t_max[2])
            
            if tn <= tf:
                if level == max_level:
                    # Leaf level: add to final list
                    cube_ori_list.append(coords)
                else:
                    # Internal level: find valid children
                    # Columns 0-7 store child indices
                    child_indices = table[i, :8]
                    
                    # Mask out invalid children (-1)
                    mask = child_indices != -1
                    valid_children = child_indices[mask]
                    
                    # Add valid children to the next level's list
                    new_list.extend(valid_children)
        
        # Prepare for next level
        cube_touched_idx = new_list
        if not cube_touched_idx:
            break
            
    return cube_ori_list
```

**Parameters:**
*   `max_level`: The maximum depth of the mipmap hierarchy. The loop runs from level 0 to `max_level` inclusive [S1, unofficial].
*   `cube_size`: The physical size of a voxel at the current level. In the cited implementation, it is calculated as $256 / 2^l$, implying a base grid resolution of 256 units [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Floating-Point Precision:** When using the hierarchical traversal in conjunction with [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) for precise span calculation, a small epsilon `Delta` (e.g., $0.01$) is added to the time to find the next voxel index. If `Delta` is too large, it may clip corners of nodes visibly; if too small, it may fail due to floating-point rounding errors when far from the grid origin [S2, unofficial].
*   **Epsilon Clipping:** An epsilon `Eps` (e.g., $0.000001$) is often added to the ray's near clip plane ($t_0$) to avoid self-intersection or degenerate cases. Tweaking this value may clip tiny corner intersections [S2, unofficial].
*   **Empty Voxels:** The lookup table must correctly mark empty children with $-1$. If the table construction fails to filter out empty regions, the traversal will attempt to process non-existent children, leading to errors or incorrect results [S1, unofficial].

## Subtopics

### [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md)
The Hierarchical Digital Differential Analyzer (HDDA) is a ray traversal algorithm that performs multi-resolution leapfrogging through sparse volume data structures, specifically OpenVDB and NanoVDB grids. It adapts the Digital Differential Analyzer (DDA) to step through nodes or voxels of varying power-of-two dimensions, skipping empty space by jumping between grid nodes of different resolutions. Unlike standard voxel DDA which checks every voxel, HDDA leverages the hierarchical tree structure (B+tree) to traverse coarse tiles first and only descend into occupied children, significantly accelerating ray-marching for sparse volumes. It is distinct from [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) in its specific implementation for VDB structures using compile-time template meta-programming to inline the hierarchy, and from [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by focusing on traversal efficiency rather than shading interpolation.

### [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md)
Hierarchical Voxel Traversal is a recursive raycasting algorithm designed to traverse nested voxel structures, such as chunks containing subvoxels, to find intersections. It distinguishes itself from flat traversal methods by transforming rays into local subvoxel space when entering a detail voxel, allowing for efficient handling of nested voxel data. Unlike [First Active Voxel Detection](4196-first-active-voxel-detection.md) which focuses on finding the first non-empty voxel, this method fully traverses the hierarchy to compute precise intersections within detail voxels.

### [XBrickMap](3116-xbrickmap.md)
XBrickMap is a space-skipping ray traversal algorithm for 3-level hierarchical voxel grids that uses $4^3$ occupancy bitmasks to perform hierarchical skips ($4^3, 2^3, 1^3$ cuboids) via bitwise tests. It improves upon standard skipping by using tiled bitmasks to identify empty voxel sections and left-packs these bitmasks to omit empty bricks within sectors, saving memory. It is distinguished from siblings like [MultiDDA](2204-hierarchical-voxel-traversal.md) by its specific 3-level sector/brick/voxel hierarchy and bitmask-based compression, and from [Tree64](6354-multi-level-grid-structures.md) by its fixed grid topology and lack of dynamic tree rebalancing.

### [Spherical Volume Voxel Traversal](3877-spherical-volume-voxel-traversal.md)
Spherical Volume Voxel Traversal is a ray-marching algorithm that traverses a ray through a voxel grid structured in spherical coordinates (radial, polar, azimuthal), returning the sequence of voxels intersected by the ray. It solves the problem of efficiently sampling volumetric data defined on a spherical domain, such as planetary atmospheres or star models, where Cartesian grids introduce significant distortion or wasted space near the center. Unlike Cartesian hierarchical traversal which relies on axis-aligned bounding boxes, this method computes intersections with spherical shells, polar cones, and azimuthal wedges directly, handling the singularity at the origin by starting traversal at the first non-zero radial shell. It builds on [Hierarchical Grid Traversal](8384-hierarchical-grid-traversal.md) by specializing the traversal logic for spherical topology rather than octree or uniform grid structures.

### [First Active Voxel Detection](4196-first-active-voxel-detection.md)
First Active Voxel Detection determines if a ray intersects an active value at any level of a hierarchical grid and identifies the first such voxel or tile. It solves the problem of efficiently skipping empty space in sparse volume data structures by performing empty-space ray-marching at all tree levels using a Hierarchical DDA (HDDA). Unlike standard linear traversal, it leverages the hierarchical structure to jump over inactive regions, setting the intersection time and index upon the first hit.

### [Hierarchical and Block-Based Data Structures](4392-hierarchical-and-block-based-data-structures.md)
Hierarchical and Block-Based Data Structures provide a spatial subdivision framework for volume rendering, enabling efficient traversal and culling of empty or inactive regions during ray marching. Unlike flat voxel grids, these structures organize data into a tree where internal nodes store aggregate properties (e.g., occupancy, scalar ranges) derived from their children, allowing rays to skip large inactive subtrees [4133-hierarchical-data-structures]. This family encompasses methods that vary in their spatial representation (octrees, pyramids, image-space mipmaps) and traversal order (image-order, object-order, or hybrid). It sets itself apart from sibling methods by focusing on the *data organization* strategy (hierarchical blocks/trees) rather than the specific traversal algorithm (e.g., HDDA) or hardware acceleration mechanism (e.g., RTX BVH) used to navigate it.

### [Hierarchical DDA with Super-brick Skipping](5197-hierarchical-dda-with-super-brick-skipping.md)
This method accelerates ray tracing through sparse volumetric data by introducing a two-level occupancy hierarchy (super-bricks and bricks) to skip large empty regions, while maintaining pixel-identical output to the fine per-voxel DDA. It solves the performance bottleneck of traversing empty space by verifying emptiness at a coarser granularity and using an axis-aligned bounding box (AABB) for early termination, ensuring exactness by only skipping verified empty blocks and terminating only when the ray definitively passes the occupied volume.

### [Octree-based traversal](5348-octree-based-traversal.md)
Octree-based traversal accelerates volume rendering and ray tracing by navigating a hierarchical spatial decomposition (typically an octree) to skip empty space and reduce the number of ray-casting or distance-evaluation steps. It solves the performance bottleneck of non-hierarchical traversal, where every voxel along a ray must be visited, by allowing the ray to leap over large empty regions. Unlike sibling methods that may use fixed-resolution grids or brick maps, this family operates on a recursive tree structure where each parent node divides into eight equal sub-blocks ($2 \times 2 \times 2$ subdivision), enabling larger steps through empty space.

### [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md)
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (empty) regions of the volume data. It solves the problem of wasting samples on voxels that do not contribute to the final image (e.g., zero density or below an isovalue threshold). Unlike image-order methods like [Space Leaping](5863-space-leaping.md) which rely on ray coherence, or [Distance Map Acceleration](972-distance-map-acceleration.md) which uses metric distances, this family relies on explicit secondary data structures (grids, trees, or maps) to define the spatial extent of active data. The specific implementation varies significantly: some methods use object-order geometry culling [865-object-order-empty-space-skipping.md], others use hardware depth tests [1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md], and others use hierarchical tree traversal [1911-semi-transparent-empty-space-skipping.md].

### [Multi-level Grid Structures](6354-multi-level-grid-structures.md)
Multi-level Grid Structures are a family of hierarchical acceleration algorithms that recursively subdivide a spatial domain into nested grids of varying resolution to optimize ray traversal. They solve the problem of inefficient traversal in non-uniform scenes by replacing pointer-chasing tree structures with contiguous array-based grids, thereby improving memory locality and reducing ray-primitive intersection counts. Unlike flat grids, they adapt resolution based on object density, and unlike tree structures, they use fixed-size grid arrays at each level, allowing for faster traversal via integer arithmetic.

### KD-Tree Partitioned Volume Rendering
A method to avoid computing all scalar values regardless of visibility by partitioning the volume into smaller sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map corresponds to a region of the frame buffer and stores the minimum opacity of the frame buffer pixels found within that region [S3, unofficial]. It groups scalar values within a certain range together based on the transfer function to define sub-volume properties [S3, unofficial]. A kd-tree is used to render the partitioned volume with correct visibility order, where each node is associated with a sub-volume [S3, unofficial]. Each sub-volume is culled and clipped against an opacity map [S3, unofficial]. Reported performance was 10 fps for the Engine data set using a 2.5 GHz Pentium-IV and a nVidia GeForce4 graphics adapter [S3, unofficial].

### Mipmap Table Construction
Constructs a lookup table for hierarchical grid traversal. It maps coarse grid voxels to their fine grid children [S1, unofficial]. It identifies occupied voxels in the fine grid and assigns a unique index to each [S1, unofficial]. It extracts the 8 sub-grid regions corresponding to the 8 children of a coarse voxel by striding the fine grid indices [S1, unofficial]. It flattens these sub-grids and filters them using the coarse grid mask to select only the children that belong to occupied coarse voxels [S1, unofficial]. It initializes the table with shape `[num_coarse_voxels, 11]` (8 child indices + 3 coordinates), filled with -1 [S1, unofficial]. It populates columns 0-7 with the filtered child indices and columns 8-10 with the coarse voxel coordinates [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The traversal time depends on the sparsity of the volume and the depth of the hierarchy. In the worst case (fully dense volume), it visits all leaf voxels, similar to flat DDA, but with the overhead of table lookups. In sparse volumes, it skips large empty regions, reducing the number of visited nodes significantly. The cost per level is proportional to the number of intersected voxels at that level.
*   **Memory Complexity:** Requires storage for the mipmap levels and the lookup tables. The lookup table size is proportional to the number of non-empty coarse voxels at each level. For a base resolution of $N$, the table size is manageable if the volume is sparse.
*   **Strengths:** Efficient for sparse volumes; reduces memory bandwidth by skipping empty space; simple implementation using lookup tables; compatible with GPU vectorized execution (as seen in `ray_mipmap_intersect_vectorized`) [S1, unofficial].
*   **Limitations:** Requires pre-computation of the mipmap and lookup tables; less flexible for dynamic data compared to tree-based structures like Octrees; precision issues can arise if epsilon parameters are not tuned correctly [S2, unofficial].
*   **When to use:** Sparse volumetric data where empty space skipping is critical; static volumes where pre-computation is feasible.
*   **When not to use:** Dense volumes where the overhead of table lookups outweighs the benefits; highly dynamic data where rebuilding the hierarchy is too expensive.

## Background
Hierarchical Grid Traversal, also known as Ray-Mipmap Traversal, is a technique used in volume rendering to accelerate ray marching through multi-resolution grids. It is closely related to the concept of mipmapping in texture mapping, where lower-resolution versions of the data are used to quickly determine if a ray intersects a region of interest. The method is implemented in various open-source projects, such as the Jax-Volumetric-Renderer and HNanoSolver, which demonstrate its application in modern, differentiable, and high-performance rendering pipelines [S1, S2, unofficial]. The approach is particularly relevant for sparse volume data, where traditional DDA traversal would waste significant time traversing empty space.

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
