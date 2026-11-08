# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal is a ray-grid traversal algorithm that iterates through levels of a grid mipmap to efficiently visit child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB) [S1, unofficial]. It solves the problem of redundant sampling in sparse or multi-resolution volumetric data by skipping empty regions at coarse levels before descending into fine details [S1, unofficial]. Unlike flat grid traversal methods, it transforms rays into local subvoxel space when entering detail voxels, allowing for efficient handling of nested voxel data [S1, unofficial]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md) by using a hierarchical structure to accelerate the search for active voxels.

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, a hierarchical grid structure (mipmap) with $L$ levels, and a lookup table (`tables_list`) mapping parent voxel indices to child voxel indices [S1, unofficial].
*   **Output:** A list of voxel positions (or indices) at the maximum level ($L-1$) that are intersected by the ray [S1, unofficial].

**Data Structures:**
*   **Lookup Table (`tables_list`):** A list of arrays, one per level. Each row corresponds to a coarse voxel.
    *   Columns 0–7: Indices of the 8 child voxels (or -1 if empty/invalid) [S1, unofficial].
    *   Columns -3 to -1 (or 8–10): The 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].
*   **TreeMarcher State (C++ implementation):**
    *   `mAcc`: Reference to the acceleration structure [S2, unofficial].
    *   `mRay`: The ray being traversed [S2, unofficial].
    *   `mHdda`: Hierarchical Digital Differential Analyzer instance [S2, unofficial].
    *   `mT0`: Current traversal start time [S2, unofficial].
    *   `mTmax`: Maximum traversal time (ray far clip) [S2, unofficial].

**Parameters:**
*   `max_level`: The maximum depth of the mipmap hierarchy. The loop runs from level 0 to `max_level` inclusive [S1, unofficial].
*   `cube_size`: The physical size of a voxel at the current level. For a base grid resolution of 256 units, `cube_size = 256 / (2 ** level)` [S1, unofficial].
*   `Eps`: Intersection tolerance, typically $10^{-6}$ [S2, unofficial].
*   `Delta`: Small offset for next-step estimation, typically $0.01$ [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Set `cube_touched_idx` to `[0]` (the root voxel index) [S1, unofficial].
    *   Initialize `new_list` as empty [S1, unofficial].
    *   Clip the ray to the root bounding box. If no intersection, return empty [S2, unofficial].
    *   Set $t_0 = \mathbf{o}.t + \text{Eps}$ and $t_1 = \mathbf{o}.t_{\text{far}} - \text{Eps}$ [S2, unofficial]. If $t_0 > t_1$, return false [S2, unofficial].
    *   Compute initial voxel index $\text{ijk} = \lfloor \mathbf{o}(t_0) / \text{voxel\_size} \rfloor$ [S2, unofficial].
    *   Initialize Hierarchical DDA (`mHdda`) with the ray, $t_0$, $t_1$, and the dimension of the current node [S2, unofficial].
    *   Set `mT0` based on whether the current dimension is finer than the child dimension [S2, unofficial].

2.  **Hierarchical Iteration:**
    Iterate through each level $l$ from 0 to `max_level`:
    *   Retrieve the lookup table for level $l$: `table = tables_list[l]` [S1, unofficial].
    *   For each voxel index $i$ in `cube_touched_idx`:
        *   Retrieve the voxel's origin $\mathbf{c}_{\text{ori}}$ and end $\mathbf{c}_{\text{end}}$ from the table coordinates [S1, unofficial].
        *   Compute ray-AABB intersection for the voxel's bounding box, returning near time $t_n$ and far time $t_f$ [S1, unofficial].
        *   **Intersection Check:** If $t_n \le t_f$:
            *   If $l == \text{max\_level}$: Append the voxel's position to the final hit list `cube_ori_list` [S1, unofficial].
            *   Otherwise: Extract child indices from columns 0–7 of the table row. Filter out invalid children (marked as -1) using a mask. Convert valid indices to integers and add them to `new_list` [S1, unofficial].

3.  **Progression:**
    *   Update `cube_touched_idx` to `new_list`.
    *   Reset `new_list` to empty for the next level [S1, unofficial].
    *   *Note on C++ Implementation:* The `TreeMarcher` class uses `mHdda.step()` to advance the ray. It estimates the next voxel index using a small offset $\Delta$ to avoid precision issues, updates the HDDA state, and determines if the current node is valid for processing based on time bounds and node existence [S2, unofficial].

4.  **Termination:**
    *   Return `cube_ori_list` containing the positions of voxels touched at the maximum level [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, tables_list, max_level, cube_size_func):
    """
    Traverses a hierarchical grid mipmap to find intersected leaf voxels.
    
    Args:
        ray_origin: (3,) array, ray origin.
        ray_direction: (3,) array, ray direction.
        tables_list: List of arrays, where tables_list[l] is the lookup table for level l.
                     Each row has shape (11,): [child_0, ..., child_7, x, y, z].
        max_level: int, the maximum level index (leaf level).
        cube_size_func: function(level) -> float, returns voxel size at given level.
        
    Returns:
        List of (x, y, z) coordinates of intersected leaf voxels.
    """
    # Initialize with root voxel index (assumed to be 0)
    cube_touched_idx = [0]
    final_hits = []
    
    # Precompute ray inverse for AABB intersection
    ray_inv_dir = 1.0 / ray_direction
    ray_inv_dir = np.where(ray_direction == 0, np.inf, ray_inv_dir)
    
    for level in range(max_level + 1):
        table = tables_list[level]
        new_list = []
        
        # Get voxel size for current level
        current_voxel_size = cube_size_func(level)
        
        for idx in cube_touched_idx:
            # Retrieve voxel data from table
            # Columns 0-7: children, Columns 8-10: coordinates
            row = table[idx]
            children = row[:8]
            coords = row[8:11] # x, y, z
            
            # Compute AABB bounds
            # Assuming coordinates are grid indices, convert to world space
            # Origin is at grid index * voxel_size
            cube_ori = coords * current_voxel_size
            cube_end = cube_ori + current_voxel_size
            
            # Ray-AABB Intersection
            # Compute t_min and t_max for each axis
            t_min = (cube_ori - ray_origin) * ray_inv_dir
            t_max = (cube_end - ray_origin) * ray_inv_dir
            
            # Swap if necessary so t_min < t_max
            t_min = np.minimum(t_min, t_max)
            t_max = np.maximum(t_min, t_max)
            
            # Find the largest t_min and smallest t_max
            tn = np.max(t_min)
            tf = np.min(t_max)
            
            # Check intersection
            if tn <= tf:
                if level == max_level:
                    # Leaf level: record hit
                    final_hits.append(coords)
                else:
                    # Internal level: collect valid children
                    # Mask for valid children (not -1)
                    mask = children != -1
                    valid_children = children[mask]
                    new_list.extend(valid_children)
        
        cube_touched_idx = new_list
        if not cube_touched_idx:
            break
            
    return final_hits
```

**Edge Cases and Pitfalls:**
*   **Floating-Point Precision:** If the offset `Delta` used to estimate the next voxel is too large, it may clip corners of nodes visibly. If too small, it may fail due to FP32 rounding errors when far from the grid. A value of $0.01$ is recommended [S2, unofficial].
*   **Tiny Intersections:** Tweaking the intersection span with `Eps` ($10^{-6}$) may clip tiny corner intersections [S2, unofficial].
*   **Vectorization:** In vectorized implementations (e.g., `ray_mipmap_intersect_vectorized`), rays are processed simultaneously. Non-intersecting rays are masked out using `tn <= tf` [S1, unofficial].

## Subtopics

### [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md)
Hierarchical Voxel Traversal is a recursive raycasting algorithm designed to traverse nested voxel structures, such as chunks containing subvoxels, to find intersections. It distinguishes itself from flat traversal methods by transforming rays into local subvoxel space when entering a detail voxel, allowing for efficient handling of nested voxel data. Unlike [First Active Voxel Detection](4196-first-active-voxel-detection.md) which focuses on finding the first non-empty voxel, this method fully traverses the hierarchy to compute precise intersections within detail voxels.

### [XBrickMap](3116-xbrickmap.md)
XBrickMap is a space-skipping ray traversal algorithm for 3-level hierarchical voxel grids that uses $4^3$ occupancy bitmasks to perform hierarchical skips ($4^3, 2^3, 1^3$ cuboids) via bitwise tests. It improves upon standard skipping by using tiled bitmasks to identify empty voxel sections and left-packs these bitmasks to omit empty bricks within sectors, saving memory. It is distinguished from siblings like [MultiDDA](2204-hierarchical-voxel-traversal.md) by its specific 3-level sector/brick/voxel hierarchy and bitmask-based compression, and from [Tree64](6354-multi-level-grid-structures.md) by its fixed grid topology and lack of dynamic tree rebalancing.

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

### [Grid Sphere Traversal](9649-grid-sphere-traversal.md)
Grid Sphere Traversal (GST) is a ray-grid traversal method that performs sphere tracing through a dense grid of Signed Distance Function (SDF) values. It solves the problem of efficiently traversing empty regions in SDF-based rendering by using a hierarchy of SDF grids where lower-resolution levels encode larger SDF values to accelerate step sizes. It is distinguished from its siblings by performing the entire traversal in shader code using dense grids, whereas alternatives like [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) or [XBrickMap](3116-xbrickmap.md) often rely on sparse structures or hardware-accelerated BVH traversal.

### KD-Tree Partitioned Volume Rendering
A method to avoid computing all scalar values regardless of visibility by partitioning the volume into smaller sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map corresponds to a region of the frame buffer and stores the minimum opacity of the frame buffer pixels found within that region [S3, unofficial]. It groups scalar values within a certain range together based on the transfer function to define sub-volume properties, uses a kd-tree to render the partitioned volume with correct visibility order (where each node is associated with a sub-volume), and culls/clips each sub-volume against the opacity map [S3, unofficial]. Reported performance was 10 fps for the Engine data set using a 2.5 GHz Pentium-IV and a nVidia GeForce4 graphics adapter [S3, unofficial].

### Mipmap Table Construction
Constructs a lookup table for hierarchical grid traversal. It maps coarse grid voxels to their fine grid children. The table is built from a fine grid and a coarse grid mask [S1, unofficial]. It identifies occupied voxels in the fine grid, assigns unique indices, extracts the 8 sub-grid regions corresponding to the 8 children of a coarse voxel (by striding), flattens and filters them using the coarse grid mask, initializes the table with shape `[num_coarse_voxels, 11]` (filled with -1), populates columns 0–7 with filtered child indices, and columns 8–10 with coarse voxel coordinates [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the sparsity of the volume and the depth of the hierarchy. In the worst case (fully dense), it approaches the complexity of flat traversal but with higher overhead due to table lookups. In sparse cases, it skips large empty regions, significantly reducing the number of visited voxels.
*   **Memory Complexity:** Requires storage for the lookup tables (`tables_list`). The size of the table at level $l$ is proportional to the number of voxels at that level. The total memory is the sum of table sizes across all levels.
*   **Performance:** Efficient for sparse or multi-resolution data. The C++ implementation (`TreeMarcher`) uses optimized HDDA steps and small offsets to handle precision issues [S2, unofficial].
*   **Strengths:** Adaptive resolution, efficient empty space skipping, exact intersection results (no approximation errors in traversal path).
*   **Limitations:** Requires pre-computation of lookup tables. Precision issues can arise if offsets (`Delta`, `Eps`) are not carefully chosen [S2, unofficial].

## Background
Hierarchical Grid Traversal is a method for accelerating volume rendering by traversing a mipmap-like structure of voxel grids. It is implemented in projects such as `Jax-Volumetric-Renderer` [S1, unofficial] and `HNanoSolver` [S2, unofficial]. The approach leverages the spatial coherence of rays and the hierarchical nature of the data to skip empty or uniform regions.

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
