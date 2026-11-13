# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal is a ray-grid traversal algorithm that accelerates volume rendering by iterating through levels of a grid mipmap (hierarchical grid tree) [S1, unofficial][S2, unofficial]. It solves the problem of expensive fine-grained sampling in large volumes by recursively visiting child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB) [S1, unofficial]. Unlike flat grid traversal methods such as [DDA Traversal](3061-digital-differential-analyzer-dda-traversal.md) which step through every voxel, this method skips empty or inactive regions at coarse levels, descending into finer grids only where necessary [S1, unofficial]. It sets itself apart from [Octree and Tree Traversal](4134-octree-and-tree-traversal.md) by using a fixed-depth, array-based mipmap structure with lookup tables rather than pointer-based nodes, optimizing for memory locality and batch processing [S1, unofficial][S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$; a hierarchical grid structure (mipmap) with $L$ levels; a precomputed lookup table (`tables_list`) mapping coarse voxel indices to child indices and coordinates; `max_level` (the depth of the hierarchy); `cube_size` (physical size of a voxel at the current level).
*   **Output:** A list of voxel positions (or indices) at the maximum level (`max_level`) that are intersected by the ray.

**Data Structures:**
*   **Lookup Table (`tables_list`):** A list of arrays, one per level. Each row corresponds to a coarse voxel.
    *   Columns 0–7: Indices of the 8 child voxels (or -1 if the child is empty/invalid) [S1, unofficial].
    *   Columns -3 to -1 (or 8–10): The 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].
*   **Ray:** Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [G].
*   **State:** `cube_touched_idx`: A list of indices of voxels at the current level that the ray intersects.

**Parameters:**
*   `max_level`: The maximum depth of the mipmap hierarchy. The loop runs from level 0 to `max_level` inclusive [S1, unofficial].
*   `cube_size`: The physical size of a voxel at the current level. For a base grid resolution of 256 units, `cube_size` at level $l$ is calculated as $256 / 2^l$ [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:**
    *   Set `cube_touched_idx` to `[0]`, representing the root voxel index [S1, unofficial].
    *   Initialize `new_list` as an empty list [S1, unofficial].
    *   Initialize `cube_ori_list` as an empty list to store final hits [S1, unofficial].

2.  **Iterate through Levels:**
    For each level $l$ from 0 to `max_level`:
    *   Retrieve the lookup table for the current level: `table = tables_list[l]` [S1, unofficial].
    *   For each voxel index $i$ in `cube_touched_idx`:
        1.  **Retrieve Voxel Bounds:**
            *   Get the voxel's 3D grid coordinates from the table: `grid_coords = table[i, -3:]` [S1, unofficial].
            *   Calculate the physical origin: `cube_ori = grid_coords * cube_size` [S1, unofficial].
            *   Calculate the physical end: `cube_end = cube_ori + cube_size` [S1, unofficial].
        2.  **Ray-AABB Intersection:**
            *   Compute the intersection of the ray with the AABB defined by `cube_ori` and `cube_end` [S1, unofficial].
            *   This yields a near distance $t_n$ and a far distance $t_f$ [S1, unofficial].
            *   *Note:* If the ray does not intersect the AABB, $t_n > t_f$ or the intersection is behind the ray origin [S1, unofficial].
        3.  **Check Intersection:**
            *   If $t_n \le t_f$ (intersection exists and is in front of the ray):
                *   **If $l == max_level$:**
                    *   Append the voxel's grid position `table[i, -3:]` to `cube_ori_list` [S1, unofficial].
                *   **If $l < max_level$:**
                    *   Extract the 8 child indices from the table: `children = table[i, :8]` [S1, unofficial].
                    *   Filter out invalid children (marked as -1): `mask = children != -1` [S1, unofficial].
                    *   Convert valid indices to integers and add them to `new_list` [S1, unofficial].
    *   **Update State:**
        *   Set `cube_touched_idx` to `new_list` [S1, unofficial].
        *   Reset `new_list` to an empty list for the next level [S1, unofficial].
        *   Update `cube_size` for the next level: `cube_size = cube_size / 2` [S1, unofficial].

3.  **Return Result:**
    *   Return `cube_ori_list` containing the positions of voxels touched at the maximum level [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, tables_list, max_level):
    """
    Traverses a hierarchical grid mipmap to find intersected voxels at the finest level.
    
    Args:
        ray_origin: (3,) array, ray origin in world space.
        ray_direction: (3,) array, ray direction in world space.
        tables_list: List of arrays, where tables_list[l] is the lookup table for level l.
                     Each row has shape (11,): [child0, ..., child7, x, y, z].
        max_level: int, the maximum depth of the hierarchy.
        
    Returns:
        cube_ori_list: List of (3,) arrays, grid coordinates of intersected voxels at max_level.
    """
    cube_touched_idx = [0]  # Start with root voxel index
    cube_ori_list = []
    cube_size = 256.0  # Base voxel size, assuming 256^3 root grid
    
    for level in range(max_level + 1):
        table = tables_list[level]
        new_list = []
        
        # Update cube_size for the current level if needed, 
        # though typically cube_size is derived from level: 256 / (2 ** level)
        current_cube_size = 256.0 / (2 ** level)
        
        for i in cube_touched_idx:
            # Retrieve voxel coordinates and bounds
            # Columns -3 to -1 store x, y, z
            grid_coords = table[i, -3:]
            cube_ori = grid_coords * current_cube_size
            cube_end = cube_ori + current_cube_size
            
            # Compute ray-AABB intersection
            tn, tf = intersect_ray_aabb(ray_origin, ray_direction, cube_ori, cube_end)
            
            # Check if intersection exists
            if tn <= tf:
                if level == max_level:
                    # At the finest level, record the voxel position
                    cube_ori_list.append(grid_coords)
                else:
                    # Not at the finest level, collect child indices
                    # Columns 0-7 store child indices
                    children = table[i, :8]
                    # Filter out invalid children (-1)
                    mask = children != -1
                    valid_children = children[mask]
                    new_list.extend(valid_children)
        
        cube_touched_idx = new_list
        if not cube_touched_idx:
            break
            
    return cube_ori_list

def intersect_ray_aabb(ray_origin, ray_direction, aabb_min, aabb_max):
    """
    Computes the intersection of a ray with an AABB.
    Returns (tn, tf) where tn is the near intersection distance and tf is the far intersection distance.
    If no intersection, tn > tf.
    """
    # Compute inverse direction
    inv_dir = 1.0 / ray_direction
    
    # Compute t values for each axis
    t1 = (aabb_min[0] - ray_origin[0]) * inv_dir[0]
    t2 = (aabb_max[0] - ray_origin[0]) * inv_dir[0]
    
    t3 = (aabb_min[1] - ray_origin[1]) * inv_dir[1]
    t4 = (aabb_max[1] - ray_origin[1]) * inv_dir[1]
    
    t5 = (aabb_min[2] - ray_origin[2]) * inv_dir[2]
    t6 = (aabb_max[2] - ray_origin[2]) * inv_dir[2]
    
    # Find the largest of the min t's and smallest of the max t's
    tn = max(max(t1, t2), max(t3, t4), max(t5, t6))
    tf = min(min(t1, t2), min(t3, t4), min(t5, t6))
    
    return tn, tf
```

**Edge Cases and Pitfalls:**
*   **Zero-Length Ray:** If the ray direction is zero or the ray has no length, the intersection test should return no intersection [S5, unofficial].
*   **Precision Issues:** When the ray is far from the grid, floating-point precision errors in the AABB intersection test may cause incorrect culling. A small epsilon may be needed for robustness [S2, unofficial].
*   **Corner Intersections:** Using a small epsilon to clip ray spans may clip tiny corner intersections [S2, unofficial].
*   **Empty Voxels:** The lookup table must correctly mark empty children with -1 to avoid traversing non-existent nodes [S1, unofficial].

## Subtopics

### [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md)
Hierarchical Trilinear Ray Casting is a volume rendering technique that accelerates ray casting by traversing a multi-dimensional hierarchy (mdh) to adaptively choose the resolution level for evaluating volume data. It solves the problem of expensive ray traversal in large volumes by approximating homogeneous regions with least-squares trilinear coefficients, allowing the renderer to skip fine-grained sampling in unimportant or smooth areas. Unlike sibling methods that rely on simple opacity thresholds or constant-value homogeneity, this method uses a user-defined error tolerance and data importance weighting to control refinement, providing a balance between image quality and performance.

### [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md)
Hierarchical Voxel Traversal is a recursive raycasting algorithm designed to traverse nested voxel structures, such as chunks containing subvoxels, to find intersections. It distinguishes itself from flat traversal methods by transforming rays into local subvoxel space when entering a detail voxel, allowing for efficient handling of nested voxel data. Unlike [First Active Voxel Detection](4196-first-active-voxel-detection.md) which focuses on finding the first non-empty voxel, this method fully traverses the hierarchy to compute precise intersections within detail voxels.

### [VolumeHDDA](2353-volumehdda.md)
VolumeHDDA is a specialized Hierarchical Digital Differential Analyzer for traversing volume trees, templated on the tree type, ray type, and the starting node level for traversal. It builds on [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by implementing a recursive ray-marching approach that enters child nodes when present and processes active or inactive tiles at leaf levels. It sets itself apart from siblings like [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md) by focusing on discrete tile/voxel intersection detection and time-span management rather than continuous sampling, and from [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) by explicitly handling the hierarchical node structure via template recursion.

### [XBrickMap](3116-xbrickmap.md)
XBrickMap is a space-partitioning ray traversal algorithm for 3-level hierarchical voxel grids that uses $4^3$ occupancy bitmasks to skip empty regions via bitwise tests. It improves upon simpler skipping methods by using tiled bitmasks to perform hierarchical skips of $4^3$, $2^3$, and $1^3$ cuboids, while left-packing empty bricks within sectors to save memory. It is distinguished from its sibling [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by its specific use of compacted bitmasks for random access and its support for arbitrary in-place edits within bricks.

### [Spherical Volume Voxel Traversal](3877-spherical-volume-voxel-traversal.md)
Spherical Volume Voxel Traversal is a ray-marching algorithm that traverses a ray through a voxel grid structured in spherical coordinates (radial, polar, azimuthal), returning the sequence of voxels intersected by the ray. It solves the problem of efficiently sampling volumetric data defined on a spherical domain, such as planetary atmospheres or star models, where Cartesian grids introduce significant distortion or wasted space near the center. Unlike Cartesian hierarchical traversal which relies on axis-aligned bounding boxes, this method computes intersections with spherical shells, polar cones, and azimuthal wedges directly, handling the singularity at the origin by starting traversal at the first non-zero radial shell. It builds on [Hierarchical Grid Traversal](8384-hierarchical-grid-traversal.md) by specializing the traversal logic for spherical topology rather than octree or uniform grid structures.

### [First Active Voxel Detection](4196-first-active-voxel-detection.md)
First Active Voxel Detection determines if a ray intersects an active value at any level of a hierarchical grid and identifies the first such voxel or tile. It solves the problem of efficiently skipping empty space in sparse volume data structures by performing empty-space ray-marching at all tree levels using a Hierarchical DDA (HDDA). Unlike standard linear traversal, it leverages the hierarchical structure to jump over inactive regions, setting the intersection time and index upon the first hit.

### [Hierarchical and Block-Based Data Structures](4392-hierarchical-and-block-based-data-structures.md)
This family of methods accelerates volume rendering by organizing voxel data into a spatial subdivision hierarchy (e.g., octrees, bricked grids, BVHs) where internal nodes store aggregate information derived from their children or leaf voxels [Hierarchical Data Structures](4133-hierarchical-data-structures.md). This structure enables the renderer to prune empty or inactive regions during ray traversal, significantly reducing the number of samples required to generate an image [Hierarchical Data Structures](4133-hierarchical-data-structures.md). Unlike flat grid traversal methods, hierarchical approaches adaptively skip large volumes of empty space, making them particularly effective for sparse or multi-resolution datasets [Hierarchical Data Structures](4133-hierarchical-data-structures.md).

### [Hierarchical DDA with Super-brick Skipping](5197-hierarchical-dda-with-super-brick-skipping.md)
This method accelerates ray tracing through sparse volumetric data by introducing a two-level occupancy hierarchy (super-bricks and bricks) to skip large empty regions, while maintaining pixel-identical output to the fine per-voxel DDA. It solves the performance bottleneck of traversing empty space by verifying emptiness at a coarser granularity and using an axis-aligned bounding box (AABB) for early termination, ensuring exactness by only skipping verified empty blocks and terminating only when the ray definitively passes the occupied volume.

### [Octree-based traversal](5348-octree-based-traversal.md)
Octree-based traversal accelerates volume rendering and implicit surface rendering by recursively navigating a hierarchical spatial subdivision (typically an octree) to skip empty space or cull nodes that do not intersect the ray. It differs from flat grid traversal by using a tree structure to bypass large volumes of empty space, and from Empty-Space Skipping which uses primitive hulls, by employing hierarchical spatial subdivision to locate non-empty space faster. It sets itself apart from its siblings, such as [Octree Partitioning for Sphere Tracing](1541-octree-partitioning-for-sphere-tracing.md), by using explicit bounds tracking and entry/exit side tracking with a callback for leaf processing rather than fixed-step iteration.

### [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md)
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (transparent or empty) regions of a volume dataset before or during ray sampling. Unlike methods that rely on hardware occlusion culling or ray coherence, this family uses explicit secondary data structures—such as min-max blocks, occupancy grids, BVHs, or hybrid trees—to store metadata about the spatial distribution of active data [G]. This allows rays to determine entry and exit points of active regions analytically or via coarse traversal, skipping large intervals of empty space that would otherwise require expensive texture lookups and compositing operations.

### [Multi-level Grid Structures](6354-multi-level-grid-structures.md)
Multi-level Grid Structures are a family of hierarchical acceleration techniques for ray tracing and volume rendering that organize spatial data into nested grids of varying resolution. They solve the problem of inefficient traversal in sparse or non-uniform scenes by allowing rays to skip large empty regions at coarse levels before descending into finer grids only where necessary. Unlike flat grids, they adapt resolution based on object density, and unlike pointer-based trees (e.g., [Octree-based traversal](5348-octree-based-traversal.md)), they often use contiguous array-based grids or fixed-depth hierarchies to optimize for memory locality and reduced ray-primitive intersection counts.

### [Quadtree Raycasting](8208-quadtree-raycasting.md)
Quadtree Raycasting is a recursive ray-quadtree intersection algorithm that traverses a 2D quadtree to find the first hit node. It solves the problem of efficiently locating the first occupied voxel in a hierarchical 2D grid by handling ray direction flipping via a mask to ensure consistent traversal order regardless of ray direction. It sets itself apart from non-hierarchical methods like DDA by leveraging the spatial coherence of the quadtree structure to skip empty regions, and differs from standard AABB traversal by operating on a tree of variable-sized nodes rather than a uniform grid.

### [Grid Sphere Traversal](9649-grid-sphere-traversal.md)
Grid Sphere Traversal (GST) is a ray-grid traversal method that performs sphere tracing through a dense grid of Signed Distance Function (SDF) values. It solves the problem of efficiently traversing empty regions in SDF-based rendering by using a hierarchy of SDF grids where lower-resolution levels encode larger SDF values to accelerate step sizes. It is distinguished from its siblings by performing the entire traversal in shader code using dense grids, whereas alternatives like [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) or [XBrickMap](3116-xbrickmap.md) often rely on sparse structures or hardware-accelerated BVH traversal.

### [Ray Grid Traversal with Adaptive Step Sizing](12497-ray-grid-traversal-with-adaptive-step-sizing.md)
A CUDA kernel that marches rays through one or more axis-aligned voxel grids using precomputed, sorted entry/exit intersections to generate sample points and interval boundaries. It handles empty cells by skipping them and fills continuous segments, supporting both fixed step sizes and cone-angle-aware adaptive stepping. It sets itself apart from standard DDA by decoupling the grid boundary traversal from the sample generation, allowing for variable step sizes that scale with distance from the camera origin.

### KD-Tree Partitioned Volume Rendering
A method to avoid computing all scalar values regardless of visibility by partitioning the volume into smaller sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map corresponds to a region of the frame buffer and stores the minimum opacity of the frame buffer pixels found within that region [S3, unofficial]. It groups scalar values within a certain range together based on the transfer function to define sub-volume properties [S3, unofficial]. It uses a kd-tree to render the partitioned volume with correct visibility order, where each node is associated with a sub-volume [S3, unofficial]. It culls and clips each sub-volume against an opacity map [S3, unofficial]. It reported 10 fps for the Engine data set using a 2.5 GHz Pentium-IV and a nVidia GeForce4 graphics adapter [S3, unofficial].

### Mipmap Table Construction
Constructs a lookup table for hierarchical grid traversal. It maps coarse grid voxels to their fine grid children. The table is built from a fine grid and a coarse grid mask [S1, unofficial]. It identifies occupied voxels in the fine grid and assigns a unique index to each [S1, unofficial]. It extracts the 8 sub-grid regions corresponding to the 8 children of a coarse voxel by striding the fine grid indices [S1, unofficial]. It flattens these sub-grids and filters them using the coarse grid mask to select only the children that belong to occupied coarse voxels [S1, unofficial]. It initializes the table with shape `[num_coarse_voxels, 11]` (8 child indices + 3 coordinates), filled with -1 [S1, unofficial]. It populates columns 0-7 with the filtered child indices and columns 8-10 with the coarse voxel coordinates [S1, unofficial].

### VIPER Ray-Traversal and Surface Intersection Algorithm
A ray-traversal algorithm for volumetric rendering. Its time complexity is $O(p \cdot r)$, where $p$ is the number of pixels and $r$ is the volume's largest dimension in voxels [S4, unofficial]. Its space complexity is dependent on the size of the data volume [S4, unofficial]. It uses a relative distance formula: $\text{relative distance} = (\text{target} - \text{value1}) / (\text{value2} - \text{value1})$ [S4, unofficial].

### World-Space Voxel Raycast with Transform
Performs raycasting in world space by transforming the ray to local voxel space, calling the generic raycast, and transforming results back [S5, unofficial]. It computes the ray end in world space, transforms the ray origin and end to local space, and calculates the local direction and max distance [S5, unofficial]. It handles zero-length rays by returning an empty result when the squared distance is less than $0.000001$ [S5, unofficial]. On success, it scales the distance back to world space and transforms the normal if enabled [S5, unofficial].

## Operating characteristics
*   **Time Complexity:** The time complexity depends on the number of voxels visited. In the worst case (ray passes through all voxels), it is proportional to the number of voxels in the finest level. However, in sparse volumes, it is significantly faster than flat traversal due to early culling at coarse levels [S1, unofficial][S4, unofficial].
*   **Memory Complexity:** The memory complexity is dominated by the storage of the hierarchical grid and the lookup tables. The lookup table size is proportional to the number of coarse voxels times the number of levels [S1, unofficial][S4, unofficial].
*   **Performance:** Performance is highly dependent on the sparsity of the volume and the efficiency of the AABB intersection test. For dense volumes, the overhead of the hierarchy may not be justified [S1, unofficial][S4, unofficial].
*   **Strengths:** Efficiently skips empty space; suitable for sparse and multi-resolution data; good memory locality due to array-based structure [S1, unofficial][S2, unofficial].
*   **Limitations:** Requires precomputation of the lookup table; less flexible than pointer-based trees for dynamic data; potential precision issues with far-away rays [S1, unofficial][S2, unofficial].
*   **When to Use:** Sparse volumes, multi-resolution data, static scenes where precomputation is feasible [S1, unofficial][S2, unofficial].
*   **When Not to Use:** Dense volumes with little empty space, highly dynamic data where the hierarchy needs frequent updates [S1, unofficial][S2, unofficial].

## Background
Hierarchical Grid Traversal, also known as Ray-Mipmap Traversal, is a technique used in volume rendering to accelerate ray casting through hierarchical grid structures [S1, unofficial][S2, unofficial]. It builds on the principles of ray-grid traversal, specifically the DDA algorithm, by adding a hierarchical layer to skip empty regions [S1, unofficial][S2, unofficial]. The method is often implemented using lookup tables to map coarse voxels to fine voxels, allowing for efficient batch processing [S1, unofficial]. It is used in various volume rendering applications, including scientific visualization and medical imaging [S1, unofficial][S2, unofficial].

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S4] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S5] Zylann/godot_voxel, https://github.com/Zylann/godot_voxel, fetched 2026-10-26 (unofficial)
