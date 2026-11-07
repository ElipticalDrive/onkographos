# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal is a ray-grid traversal method that accelerates volume rendering by stepping through a multi-resolution grid hierarchy (mipmap) rather than a single flat grid [S1, unofficial][S2, unofficial]. It solves the problem of excessive sampling in sparse or multi-resolution volumes by skipping large empty regions at coarse levels and refining only where the ray intersects active data [S1, unofficial]. It sets itself apart from flat [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) by using a lookup table to jump directly to child voxels, and from [Octree and Tree Traversal](4134-octree-and-tree-traversal.md) by relying on fixed-grid mipmap structures and array-based lookups rather than pointer-chasing tree nodes [S1, unofficial][S2, unofficial].

## How it works
The method iterates through levels of a grid mipmap, starting at the root (level 0). It recursively visits child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB) [S1, unofficial]. The traversal relies on precomputed lookup tables that map parent voxel indices to their 8 children [S1, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Ray: Origin $\mathbf{o}$, direction $\mathbf{d}$, and interval $[t_0, t_1]$ [S2, unofficial].
    *   Hierarchy: A list of lookup tables (`tables_list`), one per level, and the maximum depth `max_level` [S1, unofficial].
    *   Grid parameters: Base resolution (e.g., 256 units) and `cube_size` which scales with level [S1, unofficial].
*   **Outputs:**
    *   A list of voxel positions (or indices) at the maximum level (`cube_ori_list`) that the ray intersects [S1, unofficial].

### Data Structures
**Lookup Table (`tables_list`)**
Each table corresponds to a specific hierarchy level. A row represents a coarse voxel.
*   **Columns 0–7:** Indices of the 8 child voxels. If a child is empty or invalid, the value is $-1$ [S1, unofficial].
*   **Columns 8–10 (or -3 to -1):** The 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].
*   **Layout:** Shape $[N_{coarse}, 11]$ where $N_{coarse}$ is the number of voxels at that level [S1, unofficial].

**Class State (e.g., `TreeMarcher`)**
*   `mAcc`: Reference to the acceleration structure [S2, unofficial].
*   `mRay`: The ray being traversed [S2, unofficial].
*   `mHdda`: A Hierarchical DDA instance for precise boundary crossing [S2, unofficial].
*   `mT0`: Current entry time into the current node [S2, unofficial].
*   `mTmax`: The exit time of the ray from the volume [S2, unofficial].

### Numbered Steps

1.  **Initialization:**
    *   Set `cube_touched_idx` to `[0]` (the root voxel index) [S1, unofficial].
    *   Initialize `new_list` as empty [S1, unofficial].
    *   Clip `mRay` to the root bounding box. If no intersection, return false [S2, unofficial].
    *   Set epsilon $Eps = 10^{-6}$ [S2, unofficial].
    *   Set $t_0 = mRay.t_0() + Eps$ and $t_1 = mRay.t_1() - Eps$. If $t_0 > t_1$, return false [S2, unofficial].
    *   Compute initial voxel index $ijk = \lfloor mRay(t_0) \rfloor$ [S2, unofficial].
    *   Initialize `mHdda` with `mRay`, $t_0$, $t_1$, and the dimension of the current node [S2, unofficial].
    *   Set `mT0` based on whether the current dimension is finer than the child dimension [S2, unofficial].

2.  **Level Iteration:**
    *   Iterate `level` from 0 to `max_level` inclusive [S1, unofficial].
    *   Retrieve the lookup table for the current level: `table = tables_list[level]` [S1, unofficial].

3.  **Ray-AABB Intersection:**
    *   For each voxel index `i` in `cube_touched_idx`:
        *   Retrieve voxel origin: `cube_ori = table[i, -3:] * cube_size` [S1, unofficial].
        *   Retrieve voxel end: `cube_end = cube_ori + cube_size` [S1, unofficial].
        *   Compute ray-AABB intersection, returning near time $t_n$ and far time $t_f$ [S1, unofficial].
        *   Create a mask `mask = (t_n \le t_f)` [S1, unofficial].

4.  **Leaf Detection or Child Extraction:**
    *   If `level == max_level`:
        *   For voxels where `mask` is true, append `table[i, -3:]` to `cube_ori_list` [S1, unofficial].
    *   If `level < max_level`:
        *   Extract child indices from `table[i, 0:8]` [S1, unofficial].
        *   Filter out invalid children (value $-1$) using `mask_children = table[i, :8] != -1` [S1, unofficial].
        *   Convert valid indices to integers and add them to `new_list` [S1, unofficial].

5.  **Update State:**
    *   Set `cube_touched_idx = new_list` [S1, unofficial].
    *   Reset `new_list` to empty [S1, unofficial].
    *   Update `cube_size` for the next level (e.g., `cube_size / 2`) [S1, unofficial].

6.  **Termination:**
    *   Return `cube_ori_list` containing the positions of voxels touched at the maximum level [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, ray_t0, ray_t1, tables_list, max_level, base_resolution=256):
    """
    Traverses a hierarchical grid mipmap to find leaf voxels intersected by a ray.
    
    Args:
        ray_origin: (3,) array, ray origin.
        ray_direction: (3,) array, ray direction.
        ray_t0: float, near clip distance.
        ray_t1: float, far clip distance.
        tables_list: list of arrays, lookup tables for each level.
                     Each table has shape [N, 11]. Cols 0-7: child indices. Cols 8-10: coords.
        max_level: int, maximum depth of the hierarchy.
        base_resolution: int, base grid size (default 256).
        
    Returns:
        list of (3,) arrays, coordinates of leaf voxels intersected.
    """
    # Helper: Ray-AABB Intersection
    def intersect_ray_aabb(origin, direction, box_min, box_max):
        # Compute inverse direction
        inv_dir = 1.0 / direction
        inv_dir = np.where(direction == 0, np.inf, inv_dir)
        
        # Compute t for each slab
        t1 = (box_min - origin) * inv_dir
        t2 = (box_max - origin) * inv_dir
        
        # Near and far times
        tn = np.maximum(t1, t2)
        tf = np.minimum(t1, t2)
        
        # Global near/far
        t_near = np.max(tn)
        t_far = np.min(tf)
        
        return t_near, t_far

    cube_touched_idx = [0] # Start with root
    cube_ori_list = []
    
    # Initial cube size calculation
    # Assuming level 0 covers the full base_resolution
    current_cube_size = base_resolution 
    
    for level in range(max_level + 1):
        table = tables_list[level]
        new_list = []
        
        # Calculate physical size of voxels at this level
        # In a standard mipmap, size halves each level. 
        # Source implies cube_size is derived from level.
        # If base is 256, level 0 size is 256? Or is base_resolution the total extent?
        # Source [S1, unofficial] says: cube_size = 256 / (2 ** level). 
        # This implies the root voxel (level 0) is 256 units wide? 
        # Or is it that the grid is 256^3 and level 0 voxels are 1 unit?
        # Let's stick to the source formula:
        cube_size = base_resolution / (2 ** level)
        
        for i in cube_touched_idx:
            # Get voxel coordinates from table (cols 8-10 or -3:-1)
            voxel_coords = table[i, 8:11] # Using 0-based indexing for cols 8,9,10
            
            # Compute physical bounds
            # Note: voxel_coords are grid indices. 
            # Source: cube_ori = table[i, -3:] * cube_size
            cube_ori = voxel_coords * cube_size
            cube_end = cube_ori + cube_size
            
            # Ray-AABB Intersection
            tn, tf = intersect_ray_aabb(ray_origin, ray_direction, cube_ori, cube_end)
            
            # Check intersection
            if tn <= tf:
                if level == max_level:
                    # Leaf level: add to result
                    cube_ori_list.append(voxel_coords)
                else:
                    # Internal level: extract children
                    # Cols 0-7 are child indices
                    child_indices = table[i, 0:8]
                    
                    # Filter valid children (not -1)
                    valid_mask = child_indices != -1
                    valid_children = child_indices[valid_mask]
                    
                    # Add to next level list
                    new_list.extend(valid_children.tolist())
        
        cube_touched_idx = new_list
        if not cube_touched_idx:
            break
            
    return cube_ori_list
```

### Parameters
*   **`max_level`**: The maximum depth of the mipmap hierarchy. The loop runs from 0 to `max_level` inclusive [S1, unofficial].
*   **`cube_size`**: The physical size of a voxel at the current level. Calculated as `base_resolution / (2 ** level)`. This implies a base grid resolution of 256 units if `base_resolution` is 256 [S1, unofficial].
*   **`Eps`**: Epsilon for clipping ray start/end. Typically $10^{-6}$ [S2, unofficial].
*   **`Delta`**: Small offset for HDDA updates. Typically $0.01$ [S2, unofficial].

### Edge Cases and Pitfalls
*   **Floating-Point Drift:** If `Delta` is too large, it clips corners of nodes visibly. If too small, it fails due to FP32 rounding when far from the grid. A value of $0.01$ is recommended [S2, unofficial].
*   **Tiny Intersections:** Tweaking the intersection span with `Eps` ($10^{-6}$) may clip tiny corner intersections [S2, unofficial].
*   **Vectorization:** The vectorized version `ray_mipmap_intersect_vectorized` processes all rays in `cube_touched_idx` simultaneously. It uses `np.zeros_like` for `tn` and `tf` and iterates over `len(cube_touched_idx)` to call `intersect_ray_aabb` for each ray individually, then masks out non-intersecting rays: `mask = tn <= tf` [S1, unofficial].
*   **Empty Voxels:** Child indices marked as $-1$ in the lookup table must be filtered out to avoid traversing empty space [S1, unofficial].

## Subtopics

### [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md)
Hierarchical Voxel Traversal is a recursive raycasting algorithm designed to traverse nested voxel structures, such as chunks containing subvoxels, to find intersections. It distinguishes itself from flat traversal methods by transforming rays into local subvoxel space when entering a detail voxel, allowing for efficient handling of nested voxel data. Unlike [First Active Voxel Detection](4196-first-active-voxel-detection.md) which focuses on finding the first non-empty voxel, this method fully traverses the hierarchy to compute precise intersections within detail voxels.

### [XBrickMap](3116-xbrickmap.md)
XBrickMap is a space-skipping ray traversal algorithm for 3-level hierarchical voxel grids that uses $4^3$ occupancy bitmasks to perform hierarchical skips ($4^3$, $2^3$, $1^3$ cuboids) via bitwise tests. It improves upon standard skipping by using tiled bitmasks to identify empty voxel sections and left-packs these bitmasks to omit empty bricks within sectors, saving memory. It is distinguished from siblings like [MultiDDA](2204-hierarchical-voxel-traversal.md) by its specific 3-level sector/brick/voxel hierarchy and bitmask-based compression, and from [Tree64](6354-multi-level-grid-structures.md) by its fixed grid topology and lack of dynamic tree rebalancing.

### [Spherical Volume Voxel Traversal](3877-spherical-volume-voxel-traversal.md)
Spherical Volume Voxel Traversal is a ray-marching algorithm that traverses a ray through a voxel grid structured in spherical coordinates (radial, polar, azimuthal), returning the sequence of voxels intersected by the ray. It solves the problem of efficiently sampling volumetric data defined on a spherical domain, such as planetary atmospheres or star models, where Cartesian grids introduce significant distortion or wasted space near the center. Unlike Cartesian hierarchical traversal which relies on axis-aligned bounding boxes, this method computes intersections with spherical shells, polar cones, and azimuthal wedges directly, handling the singularity at the origin by starting traversal at the first non-zero radial shell. It builds on [Hierarchical Grid Traversal](8384-hierarchical-grid-traversal.md) by specializing the traversal logic for spherical topology rather than octree or uniform grid structures.

### [First Active Voxel Detection](4196-first-active-voxel-detection.md)
First Active Voxel Detection determines if a ray intersects an active value at any level of a hierarchical grid and identifies the first such voxel or tile. It solves the problem of efficiently skipping empty space in sparse volume data structures by performing empty-space ray-marching at all tree levels using a Hierarchical DDA (HDDA). Unlike standard linear traversal, it leverages the hierarchical structure to jump over inactive regions, setting the intersection time and index upon the first hit.

### [Hierarchical and Block-Based Data Structures](4392-hierarchical-and-block-based-data-structures.md)
This family of methods accelerates volume rendering by organizing voxel data into a spatial subdivision hierarchy (e.g., octrees, bricked grids, BVHs) where internal nodes store aggregate information derived from their children or leaf voxels [4133-hierarchical-data-structures]. This structure enables the renderer to prune empty or inactive regions during ray traversal, significantly reducing the number of samples required to generate an image [4133-hierarchical-data-structures]. Unlike flat grid traversal methods, hierarchical approaches adaptively skip large volumes of empty space, making them particularly effective for sparse or multi-resolution datasets [4133-hierarchical-data-structures].

### [Hierarchical DDA with Super-brick Skipping](5197-hierarchical-dda-with-super-brick-skipping.md)
This method accelerates ray tracing through sparse volumetric data by introducing a two-level occupancy hierarchy (super-bricks and bricks) to skip large empty regions, while maintaining pixel-identical output to the fine per-voxel DDA. It solves the performance bottleneck of traversing empty space by verifying emptiness at a coarser granularity and using an axis-aligned bounding box (AABB) for early termination, ensuring exactness by only skipping verified empty blocks and terminating only when the ray definitively passes the occupied volume.

### [Octree-based traversal](5348-octree-based-traversal.md)
Octree-based traversal accelerates volume rendering and implicit surface rendering by recursively traversing a hierarchical spatial subdivision (typically an octree) to skip empty space and large uniform regions. It differs from flat grid traversal by using tree structure to bypass empty volumes, and from Empty-Space Skipping which uses primitive hulls, by employing hierarchical spatial subdivision to locate non-empty space faster. It sets itself apart from its siblings, such as [Octree Partitioning for Sphere Tracing](1541-octree-partitioning-for-sphere-tracing.md), by using explicit bounds tracking and entry/exit side tracking with a callback for leaf processing rather than fixed-step iteration.

### [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md)
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (empty) regions of the volume data. It solves the problem of wasting samples on voxels that do not contribute to the final image (e.g., zero density or below an isovalue threshold). Unlike image-order methods like [Space Leaping](5863-space-leaping.md) which rely on ray coherence, or [Distance Map Acceleration](972-distance-map-acceleration.md) which uses metric distances, this family relies on explicit secondary data structures (grids, trees, or maps) to define the spatial extent of active data. The specific implementation varies significantly: some methods use object-order geometry culling [865-object-order-empty-space-skipping.md], others use hardware depth tests [1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md], and others use hierarchical tree traversal [1911-semi-transparent-empty-space-skipping.md].

### [Multi-level Grid Structures](6354-multi-level-grid-structures.md)
Multi-level Grid Structures are a family of hierarchical acceleration algorithms that recursively subdivide a spatial domain into nested grids of varying resolution to optimize ray traversal. They solve the problem of inefficient traversal in non-uniform scenes by replacing pointer-chasing tree structures with contiguous array-based grids, thereby improving memory locality and reducing ray-primitive intersection counts. Unlike flat grids, they adapt resolution based on object density, and unlike tree structures, they use fixed-size grid arrays at each level, allowing for faster traversal via integer arithmetic.

### KD-Tree Partitioned Volume Rendering
A method to avoid computing all scalar values regardless of visibility by partitioning the volume into smaller sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map corresponds to a region of the frame buffer and stores the minimum opacity of the frame buffer pixels found within that region [S3, unofficial]. It groups scalar values within a certain range together based on the transfer function to define sub-volume properties [S3, unofficial]. It uses a kd-tree to render the partitioned volume with correct visibility order, where each node is associated with a sub-volume [S3, unofficial]. It culls and clips each sub-volume against an opacity map [S3, unofficial]. 10 fps was reported for the Engine data set using a 2.5 GHz Pentium-IV and a nVidia GeForce4 graphics adapter [S3, unofficial].

### Mipmap Table Construction
Constructs a lookup table for hierarchical grid traversal. It maps coarse grid voxels to their fine grid children. The table is built from a fine grid and a coarse grid mask [S1, unofficial]. It identifies occupied voxels in the fine grid: $x, y, z = \text{np.where}(\text{grid\_fine})$ [S1, unofficial]. It assigns a unique index to each occupied voxel: `tmp[grid] = idx` where `idx = jnp.arange(0, grid.sum())` [S1, unofficial]. It extracts the 8 sub-grid regions corresponding to the 8 children of a coarse voxel. These are obtained by striding the fine grid indices: `grid_000 = grid[0::2, 0::2, 0::2]`, etc. [S1, unofficial]. It flattens these sub-grids and filters them using the coarse grid mask: `f_000 = f_000[mask_coarse]`. This selects only the children that belong to occupied coarse voxels [S1, unofficial]. It initializes the table with shape `[num_coarse_voxels, 11]` (8 child indices + 3 coordinates), filled with $-1$ [S1, unofficial]. It populates columns 0-7 with the filtered child indices (`f_000` to `f_111`) [S1, unofficial]. It populates columns 8-10 with the coarse voxel coordinates (x, y, z) [S1, unofficial]. Table layout: Rows correspond to coarse voxels. Columns 0-7 are child voxel indices (or -1). Columns 8-10 are coarse voxel coordinates (x, y, z) [S1, unofficial].

### Grid Sphere Traversal
Sphere tracing through a dense grid of SDF values, using a hierarchy of SDF grids to accelerate traversal of empty regions [S4, unofficial]. It uses a hierarchy of SDF grids where lower-resolution levels encode larger SDF values to accelerate traversal of empty regions [S4, unofficial]. It adaptively selects a grid resolution to safely maximize the step size taken by the sphere tracing traversal [S4, unofficial]. It performs traversal entirely in shader code [S4, unofficial]. SDF values are formatted and clamped to $[-1, 1]$, where a value of 1 encodes 2.5 interior voxel diagonals, allowing storage as 8-bit snorm integers [S4, unofficial]. Similar to the technique used in Claybook [Aaltonen 2018] [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the volume and the depth of the hierarchy. In the worst case (dense volume), it approaches the complexity of flat DDA but with overhead for table lookups. In sparse volumes, it skips large regions, significantly reducing the number of visited voxels.
*   **Memory Complexity:** Requires storage for the lookup tables (`tables_list`). The size of the tables depends on the number of voxels at each level. For a grid of size $N^3$ and depth $L$, the total table size is proportional to $\sum_{i=0}^{L} N_i^3 \times 11$ integers, where $N_i$ is the number of voxels at level $i$.
*   **Measured Performance:** Specific performance metrics for the general method are not given in the sources. However, related methods report 10 fps for the Engine data set on a 2.5 GHz Pentium-IV and nVidia GeForce4 [S3, unofficial].
*   **Strengths:** Efficiently handles sparse and multi-resolution data. Avoids pointer chasing by using array-based lookups. Simple to implement with precomputed tables.
*   **Limitations:** Requires precomputation of lookup tables. Less flexible than tree-based structures for dynamic data. Performance degrades if the hierarchy is not well-balanced or if the volume is dense.
*   **When to use:** Sparse volumes, multi-resolution data, static scenes where precomputation is feasible.
*   **When not to use:** Highly dynamic volumes, dense volumes where the overhead of table lookups outweighs the benefits of skipping, or when memory is constrained.

## Background
Hierarchical Grid Traversal is a method for accelerating volume rendering by leveraging multi-resolution grid structures. It builds on the principles of ray-grid traversal [17-ray-grid-traversal.md]. The method is implemented in various open-source projects, such as Jax-Volumetric-Renderer [S1, unofficial] and HNanoSolver [S2, unofficial]. It is also related to techniques used in Grid Sphere Traversal [S4, unofficial] and KD-Tree Partitioned Volume Rendering [S3, unofficial].

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S4] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
