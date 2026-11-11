# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal is a ray-grid traversal algorithm that accelerates volume rendering by iterating through levels of a grid mipmap (hierarchical grid tree) [S1, unofficial][S2, unofficial]. It solves the problem of expensive fine-grained sampling in large volumes by recursively visiting child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB) [S1, unofficial]. Unlike flat grid traversal methods such as [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) which visit every cell along the ray path, this method uses lookup tables to map coarse grid voxels to their fine grid children, allowing it to skip empty or inactive regions at multiple resolution levels [S1, unofficial][S2, unofficial]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md) by applying the DDA principle recursively across a multi-resolution hierarchy [S2, unofficial].

## How it works
The method traverses a hierarchical grid structure, typically a mipmap, using a precomputed lookup table to determine child nodes. It operates by stepping through the hierarchy from the root level down to a specified maximum level, identifying spans where the ray intersects active nodes [S2, unofficial].

### Inputs and Outputs
*   **Inputs:**
    *   Ray: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$, with a valid interval $[t_0, t_1]$ [S2, unofficial].
    *   Hierarchy: A set of lookup tables (`tables_list`) where each level maps parent voxel indices to child voxel indices [S1, unofficial].
    *   Parameters: `max_level` (the depth of the mipmap hierarchy) and `cube_size` (physical size of a voxel at the current level) [S1, unofficial].
*   **Outputs:**
    *   A list of voxel positions (or indices) at the maximum level that the ray intersects [S1, unofficial].
    *   Alternatively, a sequence of spans $(t_{entry}, t_{exit})$ associated with active nodes during traversal [S2, unofficial].

### Data Structures
The core data structure is a lookup table for each level of the hierarchy [S1, unofficial].

**Table Layout:**
Each row in the table corresponds to a coarse voxel at the current level [S1, unofficial].
*   **Columns 0–7:** Store the indices of the 8 child voxels in the next finer level. If a child voxel is empty or inactive, the value is $-1$ [S1, unofficial].
*   **Columns -3 to -1 (or 8–10):** Store the 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].

**Class State (e.g., `TreeMarcher`):**
*   `mAcc`: Reference to the acceleration structure containing the hierarchy [S2, unofficial].
*   `mRay`: The ray being traversed [S2, unofficial].
*   `mHdda`: A Hierarchical DDA instance used to compute intersection times and update indices [S2, unofficial].
*   `mT0`: The current entry time into the active node [S2, unofficial].
*   `mTmax`: The maximum exit time for the current traversal segment [S2, unofficial].

### Pseudocode
The following pseudocode combines the table-based iteration from [S1, unofficial] with the stateful stepping from [S2, unofficial]. It assumes a helper function `intersect_ray_aabb` exists.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, t_min, t_max, tables_list, max_level, voxel_size_base):
    """
    Traverses a hierarchical grid mipmap to find intersected voxels at the finest level.
    
    Args:
        ray_origin: (x, y, z) ray origin.
        ray_direction: (x, y, z) ray direction.
        t_min: Start time of the ray segment.
        t_max: End time of the ray segment.
        tables_list: List of lookup tables, one per level. Each table is a 2D array [num_voxels, 11].
        max_level: The deepest level to traverse to.
        voxel_size_base: The physical size of a voxel at level 0.
        
    Returns:
        List of (x, y, z) coordinates of intersected voxels at max_level.
    """
    # 1. Initialize ray and bounds
    # Clip ray to the root bounding box if necessary (handled by caller or init)
    # Add epsilon to avoid precision issues at boundaries
    eps = 1e-6
    t0 = t_min + eps
    t1 = t_max - eps
    
    if t0 > t1:
        return []

    # 2. Initialize traversal state
    # Start at the root (level 0). The root voxel index is typically 0 or determined by grid bounds.
    # For a full grid, we start with the root node index [0].
    current_voxel_indices = [0] 
    final_hits = []

    # 3. Iterate through hierarchy levels
    for level in range(max_level + 1):
        table = tables_list[level]
        new_voxel_indices = []
        
        # Get physical size for current level
        # Assuming binary subdivision: size halves each level
        current_voxel_size = voxel_size_base / (2 ** level)
        
        # 4. Process each voxel from the previous level
        for idx in current_voxel_indices:
            # Retrieve voxel data from table
            # Columns -3 to -1 are coordinates, 0-7 are children
            row = table[idx]
            coords = row[-3:] # (x, y, z) grid coordinates
            children = row[:8] # Indices of 8 children
            
            # Calculate AABB for the current voxel
            # Convert grid coordinates to world space
            cube_origin = coords * current_voxel_size
            cube_end = cube_origin + current_voxel_size
            
            # 5. Ray-AABB Intersection
            # Compute ray direction inverse for efficiency
            ray_inv_dir = 1.0 / ray_direction
            
            # Compute intersection times (tn, tf)
            # This is a standard AABB intersection test
            t_near, t_far = intersect_ray_aabb(ray_origin, ray_inv_dir, cube_origin, cube_end)
            
            # Check if ray intersects this voxel
            if t_near <= t_far:
                # If we are at the maximum level, record the hit
                if level == max_level:
                    final_hits.append(coords)
                else:
                    # Otherwise, add valid children to the next level's list
                    # Filter out invalid children (marked as -1)
                    valid_children = [c for c in children if c != -1]
                    new_voxel_indices.extend(valid_children)
        
        # Update current indices for the next level
        current_voxel_indices = new_voxel_indices
        
        # Optimization: If no voxels are touched at this level, stop early
        if not current_voxel_indices:
            break

    return final_hits

def intersect_ray_aabb(ray_origin, ray_inv_dir, box_min, box_max):
    """
    Standard ray-AABB intersection returning t_near and t_far.
    Returns (inf, -inf) if no intersection.
    """
    t_min = (box_min - ray_origin) * ray_inv_dir
    t_max = (box_max - ray_origin) * ray_inv_dir
    
    # Swap if necessary to ensure t_min <= t_max
    t1 = jnp.minimum(t_min, t_max)
    t2 = jnp.maximum(t_min, t_max)
    
    # Intersection is the max of mins and min of maxs
    t_near = jnp.max(t1)
    t_far = jnp.min(t2)
    
    return t_near, t_far
```

*Note: The pseudocode above implements the "for-loop" variant described in [S1, unofficial]. The "vectorized" variant [S1, unofficial] processes all rays in `cube_touched_idx` simultaneously using `np.zeros_like` for `tn` and `tf` and masks out non-intersecting rays. The "stateful" variant [S2, unofficial] uses a `TreeMarcher` class with an internal `HDDA` (Hierarchical DDA) to step through nodes, updating `mT0` and `mTmax` based on `mHdda.step()` and `mHdda.time()`. In the stateful variant, `Delta = 0.01f` is used to predict the next voxel index for dimension lookup, and `Eps = 0.000001f` is added to `t0` to clip tiny corner intersections.*

### Parameters
*   **`max_level`**: The maximum depth of the mipmap hierarchy. The loop runs from level 0 to `max_level` inclusive [S1, unofficial].
*   **`cube_size`**: The physical size of a voxel at the current level. For a base grid resolution of 256 units with binary subdivision, `cube_size = 256 / (2 ** level)` [S1, unofficial].
*   **`Eps`**: A small epsilon value (e.g., $10^{-6}$) added to the ray start time $t_0$ to avoid precision errors at boundaries [S2, unofficial].
*   **`Delta`**: A small time increment (e.g., $0.01$) used to estimate the next voxel index for dimension lookup in the stateful variant [S2, unofficial].

### Edge Cases and Pitfalls
*   **Precision Clipping**: Using `Eps` to clip the ray start may clip tiny corner intersections [S2, unofficial].
*   **Delta Sensitivity**: In the stateful variant, if `Delta` is too large, it clips corners of nodes visibly; if too small, it fails due to floating-point rounding errors when the ray is far from the grid [S2, unofficial].
*   **Empty Voxels**: Child indices marked as $-1$ must be filtered out to prevent traversal into non-existent nodes [S1, unofficial].
*   **Zero Direction Components**: If a ray direction component is zero, the corresponding `t_delta` should be handled as infinity or zero appropriately to avoid division by zero, consistent with the parent DDA method [S2, unofficial][S5].

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

### [Multi-Grid Ray Traversal](4142-multi-grid-ray-traversal.md)
Multi-Grid Ray Traversal traverses rays through multiple binary occupancy grids to generate ray intervals, sample positions, and termination planes. It solves the problem of efficiently sampling dense volumetric data by pre-computing ray-AABB intersections to determine which grids are hit, sorting these entry/exit points, and then performing voxel traversal only within the valid intervals. Unlike single-grid traversal, it handles multiple distinct spatial partitions (e.g., different resolution levels or separate scene objects) in a unified pass, merging their intersection intervals. It differs from [Hierarchical Grid Traversal](2560-hierarchical-grid-traversal.md) by operating on a flat list of grids with pre-computed intersections rather than recursively descending a mipmap hierarchy using lookup tables.

### [First Active Voxel Detection](4196-first-active-voxel-detection.md)
First Active Voxel Detection determines if a ray intersects an active value at any level of a hierarchical grid and identifies the first such voxel or tile. It solves the problem of efficiently skipping empty space in sparse volume data structures by performing empty-space ray-marching at all tree levels using a Hierarchical DDA (HDDA). Unlike standard linear traversal, it leverages the hierarchical structure to jump over inactive regions, setting the intersection time and index upon the first hit.

### [Hierarchical and Block-Based Data Structures](4392-hierarchical-and-block-based-data-structures.md)
This family of methods accelerates volume rendering by organizing voxel data into a spatial subdivision hierarchy (e.g., octrees, bricked grids, BVHs) where internal nodes store aggregate information derived from their children or leaf voxels [4133-hierarchical-data-structures]. This structure enables the renderer to prune empty or inactive regions during ray traversal, significantly reducing the number of samples required to generate an image [4133-hierarchical-data-structures]. Unlike flat grid traversal methods, hierarchical approaches adaptively skip large volumes of empty space, making them particularly effective for sparse or multi-resolution datasets [4133-hierarchical-data-structures].

### [Hierarchical DDA with Super-brick Skipping](5197-hierarchical-dda-with-super-brick-skipping.md)
This method accelerates ray tracing through sparse volumetric data by introducing a two-level occupancy hierarchy (super-bricks and bricks) to skip large empty regions, while maintaining pixel-identical output to the fine per-voxel DDA. It solves the performance bottleneck of traversing empty space by verifying emptiness at a coarser granularity and using an axis-aligned bounding box (AABB) for early termination, ensuring exactness by only skipping verified empty blocks and terminating only when the ray definitively passes the occupied volume.

### [Octree-based traversal](5348-octree-based-traversal.md)
Octree-based traversal accelerates volume rendering and implicit surface rendering by recursively navigating a hierarchical spatial subdivision (typically an octree) to skip empty space or cull nodes that do not intersect the ray. It differs from flat grid traversal by using a tree structure to bypass large volumes of empty space, and from Empty-Space Skipping which uses primitive hulls, by employing hierarchical spatial subdivision to locate non-empty space faster. It sets itself apart from its siblings, such as [Octree Partitioning for Sphere Tracing](1541-octree-partitioning-for-sphere-tracing.md), by using explicit bounds tracking and entry/exit side tracking with a callback for leaf processing rather than fixed-step iteration.

### [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md)
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (transparent or empty) regions of a volume dataset before or during ray sampling. Unlike methods that rely on hardware occlusion culling or ray coherence, this family uses explicit secondary data structures—such as min-max blocks, occupancy grids, BVHs, or hybrid trees—to store metadata about the spatial distribution of active data [G]. This allows rays to determine entry and exit points of active regions analytically or via coarse traversal, skipping large intervals of empty space that would otherwise require expensive texture lookups and compositing operations.

### [Multi-level Grid Structures](6354-multi-level-grid-structures.md)
Multi-level Grid Structures are a family of hierarchical acceleration algorithms that organize spatial data into nested grids of varying resolution to optimize traversal and rendering. They solve the problem of inefficient access in sparse or non-uniform scenes by using a coarse grid to index finer sub-grids, thereby reducing the number of visited cells or primitives. Unlike flat grids, they skip empty regions in a single step, and unlike tree structures, they typically use fixed-size grid arrays at each level to enable faster traversal via integer arithmetic [G].

### [Quadtree Raycasting](8208-quadtree-raycasting.md)
Quadtree Raycasting is a recursive ray-quadtree intersection algorithm that traverses a 2D quadtree to find the first hit node. It solves the problem of efficiently locating the first occupied voxel in a hierarchical 2D grid by handling ray direction flipping via a mask to ensure consistent traversal order regardless of ray direction. It sets itself apart from non-hierarchical methods like DDA by leveraging the spatial coherence of the quadtree structure to skip empty regions, and differs from standard AABB traversal by operating on a tree of variable-sized nodes rather than a uniform grid.

### [Grid Sphere Traversal](9649-grid-sphere-traversal.md)
Grid Sphere Traversal (GST) is a ray-grid traversal method that performs sphere tracing through a dense grid of Signed Distance Function (SDF) values. It solves the problem of efficiently traversing empty regions in SDF-based rendering by using a hierarchy of SDF grids where lower-resolution levels encode larger SDF values to accelerate step sizes. It is distinguished from its siblings by performing the entire traversal in shader code using dense grids, whereas alternatives like [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) or [XBrickMap](3116-xbrickmap.md) often rely on sparse structures or hardware-accelerated BVH traversal.

### [Hierarchical Tree Node Traversal](12213-hierarchical-tree-node-traversal.md)
Hierarchical Tree Node Traversal is a ray-intersection method that accelerates volume rendering by traversing a boolean tree structure representing the active regions of a volume grid. It solves the problem of efficiently skipping empty space and coarse structures during ray marching. Unlike sibling methods that may traverse uniform grids or specific brick layouts, this approach operates on a generic hierarchical tree (typically an octree or similar sparse structure) where nodes represent tiles or voxels, allowing for variable-resolution traversal. It is implemented in OpenVDB as the `VolumeRayIntersector` class.

### KD-Tree Partitioned Volume Rendering
A method to avoid computing all scalar values regardless of visibility by partitioning the volume into smaller sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map corresponds to a region of the frame buffer and stores the minimum opacity of the frame buffer pixels found within that region [S3, unofficial]. It groups scalar values within a certain range together based on the transfer function to define sub-volume properties, uses a kd-tree to render the partitioned volume with correct visibility order (where each node is associated with a sub-volume), and culls/clips each sub-volume against the opacity map [S3, unofficial]. Reported performance was 10 fps for the Engine data set using a 2.5 GHz Pentium-IV and a nVidia GeForce4 graphics adapter [S3, unofficial].

### Mipmap Table Construction
Constructs a lookup table for hierarchical grid traversal, mapping coarse grid voxels to their fine grid children [S1, unofficial]. It identifies occupied voxels in the fine grid, assigns unique indices, and extracts the 8 sub-grid regions corresponding to the 8 children of a coarse voxel by striding the fine grid indices [S1, unofficial]. It flattens these sub-grids, filters them using the coarse grid mask, initializes the table with shape `[num_coarse_voxels, 11]` (filled with -1), populates columns 0-7 with filtered child indices, and columns 8-10 with coarse voxel coordinates [S1, unofficial].

### VIPER Ray-Traversal and Surface Intersection Algorithm
A ray-traversal algorithm where the relative distance is calculated as $(target - value1) / (value2 - value1)$ [S4, unofficial]. Its time complexity is $O(p \cdot r)$, where $p$ is the number of pixels and $r$ is the volume's largest dimension in voxels, with space complexity dependent on the size of the data volume [S4, unofficial].

### Hierarchical Data Structure Traversal
A parallel processing approach where each processor performs cell connectivity pre-steps, excludes cells outside the camera view, and applies ray-casting to its data [S5]. Rays enter from exterior faces, use connectivity information to determine the next cell, and exit upon intersecting a second exterior face [S5]. Image pixels are assigned to different processors, and the Hierarchical Data Structure (HDS) is used to exchange rays, which are then composited to produce the final image [S5].

### World-Space Voxel Raycast with Transform
Performs raycasting in world space by transforming the ray to local voxel space, calling the generic raycast, and transforming results back [S6, unofficial]. It computes the ray end in world space, transforms origin and end to local space, calculates local direction and max distance, and handles zero-length rays by returning an empty result if squared distance $< 10^{-6}$ [S6, unofficial]. On success, it scales the distance back to world space and transforms the normal if enabled [S6, unofficial].

## Operating characteristics
*   **Time Complexity:** The complexity depends on the sparsity of the volume and the depth of the hierarchy. In the worst case (fully occupied volume), it approaches the cost of flat traversal but with overhead for table lookups. In sparse volumes, it significantly reduces the number of visited nodes by skipping empty regions at coarse levels [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** Requires storage for the hierarchy lookup tables. The table size is proportional to the number of voxels at each level. For a binary mipmap, the total number of nodes is roughly $4/3$ of the finest level, but only active nodes are stored in sparse implementations [S1, unofficial].
*   **Strengths:** Efficiently handles sparse and multi-resolution data. Reduces memory bandwidth usage by skipping empty space. Flexible resolution adaptation.
*   **Limitations:** Requires pre-computation of the hierarchy and lookup tables. Precision issues can arise at boundaries if epsilon/delta parameters are not tuned correctly [S2, unofficial].
*   **When to use:** Sparse volumes, multi-resolution datasets, or when memory bandwidth is a bottleneck.
*   **When not to use:** Dense, fully occupied volumes where the overhead of hierarchy traversal outweighs the benefits of skipping.

## Background
Hierarchical Grid Traversal is a method for accelerating volume rendering by utilizing multi-resolution data structures. It builds upon the principles of ray-grid traversal, specifically the Digital Differential Analyzer (DDA) algorithm, by applying them recursively across a hierarchy of grids (mipmaps) [S1, unofficial][S2, unofficial]. The approach allows for efficient traversal of large, sparse volumes by skipping empty regions at coarse resolution levels before descending to finer levels [S1, unofficial]. Variants such as the `TreeMarcher` class in [S2, unofficial] and the table-based approach in [S1, unofficial] demonstrate its implementation in modern software renderers and game engines.

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S4] DTIC ADA202661: Volumetric Rendering Techniques for the Display of Three-Dimensional Aerodynamic Flow Field Data, https://archive.org/details/DTIC_ADA202661, fetched 2026-10-26 (unofficial)
[S5] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
[S6] Zylann/godot_voxel, https://github.com/Zylann/godot_voxel, fetched 2026-10-26 (unofficial)
