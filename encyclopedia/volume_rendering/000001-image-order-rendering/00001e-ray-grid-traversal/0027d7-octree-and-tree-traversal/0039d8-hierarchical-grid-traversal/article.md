# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal accelerates volume rendering by marching a ray through a multi-resolution grid hierarchy (mipmap) rather than a single flat grid. It recursively visits child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB), using lookup tables to map coarse indices to fine indices [S1, unofficial]. Unlike standard [Ray-grid traversal](17-ray-grid-traversal.md) which steps through every voxel, this method skips empty or homogeneous regions by traversing the tree structure, significantly reducing the number of samples required for sparse or multi-resolution data [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Ray origin $\mathbf{o}$, ray direction $\mathbf{d}$, hierarchical grid data structure (accessor `AccT`), maximum traversal level `max_level`.
**Outputs:** List of voxel positions (or indices) at the finest level (`max_level`) intersected by the ray.

**Data Structures:**
1.  **Lookup Table (`tables_list`):** A list of arrays, one per level. Each row corresponds to a coarse voxel at that level.
    *   Columns 0-7: Indices of the 8 child voxels in the next finer level. Value `-1` indicates an empty child [S1, unofficial].
    *   Columns -3 to -1 (or 8-10 in 0-indexed layout): The 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].
2.  **Hierarchical DDA (HDDA):** A helper structure (e.g., `nanovdb::math::HDDA`) that manages the ray's position and time within the current node, handling boundary crossings and time spans [S2, unofficial].
3.  **TreeMarcher Class:** Maintains state including `mAcc` (grid accessor), `mRay` (clipped ray), `mHdda` (HDDA instance), `mT0` (current entry time), and `mTmax` (exit time) [S2, unofficial].

**Algorithm Steps:**

1.  **Initialization:**
    *   Clip the input ray to the root bounding box of the hierarchy. If no intersection, terminate [S2, unofficial].
    *   Set epsilon `Eps = 0.000001f` to avoid precision issues at boundaries [S2, unofficial].
    *   Compute initial entry time $t_0 = \max(\text{ray.t0()} + \text{Eps}, \text{grid\_min\_t})$ and exit time $t_1 = \min(\text{ray.t1()} - \text{Eps}, \text{grid\_max\_t})$. If $t_0 > t_1$, terminate [S2, unofficial].
    *   Determine the initial voxel index `ijk` at $t_0$ using `RoundDown<Coord>(mRay(t0))` [S2, unofficial].
    *   Initialize the HDDA with the ray, $t_0$, $t_1$, and the dimension of the current node [S2, unofficial].
    *   Set `cube_touched_idx` to `[0]` (representing the root voxel index) [S1, unofficial].

2.  **Hierarchical Iteration:**
    *   Iterate through levels $L$ from $0$ to `max_level`:
        *   Retrieve the lookup table for level $L$: `table = tables_list[L]` [S1, unofficial].
        *   Initialize `new_list` as empty [S1, unofficial].
        *   For each voxel index `i` in `cube_touched_idx`:
            *   Retrieve the voxel's physical bounds:
                *   Origin: $\mathbf{p}_{ori} = \text{table}[i, -3:] \times \text{cube\_size}$
                *   End: $\mathbf{p}_{end} = \mathbf{p}_{ori} + \text{cube\_size}$
                *   Where $\text{cube\_size} = 256 / 2^L$ (assuming base resolution 256) [S1, unofficial].
            *   Perform Ray-AABB intersection: `intersect_ray_aabb(ray, p_ori, p_end)` returning near time $t_n$ and far time $t_f$ [S1, unofficial].
            *   If $t_n \le t_f$ (intersection exists):
                *   If $L == \text{max\_level}$:
                    *   Append the voxel's grid coordinates $\text{table}[i, -3:]$ to the final output list `cube_ori_list` [S1, unofficial].
                *   Else:
                    *   Extract child indices from `table[i, 0:8]` [S1, unofficial].
                    *   Filter valid children (where index $\neq -1$) using a mask [S1, unofficial].
                    *   Append valid child indices to `new_list` [S1, unofficial].
        *   Update `cube_touched_idx = new_list` [S1, unofficial].

3.  **Vectorized Variant Optimization:**
    *   In the vectorized implementation (`ray_mipmap_intersect_vectorized`), the AABB intersection is computed for all rays in `cube_touched_idx` simultaneously using batched operations [S1, unofficial].
    *   A mask `mask = tn <= tf` filters out non-intersecting rays before extracting children [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, tables_list, max_level, base_size=256):
    """
    Traverses a hierarchical grid mipmap to find intersected voxels at the finest level.
    
    Args:
        ray_origin: (3,) array, ray origin.
        ray_direction: (3,) array, ray direction.
        tables_list: List of arrays. tables_list[L] has shape (N_L, 11).
                     Columns 0-7: child indices (-1 if empty).
                     Columns 8-10: voxel coordinates (x, y, z).
        max_level: int, the deepest level to traverse.
        base_size: float, physical size of the root grid (default 256).
        
    Returns:
        List of (x, y, z) coordinates of intersected voxels at max_level.
    """
    # 1. Initialization
    # Clip ray to root bounds (simplified: assume root covers [0, base_size]^3)
    # In practice, use a robust AABB intersection test against the root bounding box.
    # If no intersection, return empty list.
    
    # Initial voxel index is the root (index 0)
    cube_touched_idx = [0]
    final_voxels = []
    
    # 2. Iterate through levels
    for level in range(max_level + 1):
        table = tables_list[level]
        new_list = []
        
        # Calculate voxel size at this level
        # Assuming power-of-2 reduction from base_size
        voxel_size = base_size / (2 ** level)
        
        # Process each touched voxel from the previous level
        for idx in cube_touched_idx:
            # Get voxel coordinates and bounds
            # Columns -3 to -1 are x, y, z
            coords = table[idx, -3:]
            p_ori = coords * voxel_size
            p_end = p_ori + voxel_size
            
            # Ray-AABB Intersection
            # Compute inverse direction to handle axis-aligned checks
            inv_dir = 1.0 / ray_direction
            inv_dir = np.where(ray_direction == 0, float('inf'), inv_dir)
            
            # Calculate t intervals for each axis
            t1 = (p_ori - ray_origin) * inv_dir
            t2 = (p_end - ray_origin) * inv_dir
            
            # t_min and t_max for the box
            t_min = np.max(np.minimum(t1, t2))
            t_max = np.min(np.maximum(t1, t2))
            
            # Check intersection
            # t_min > 0 ensures we are looking forward (ray origin is outside or at start)
            if t_max >= 0 and t_min <= t_max:
                if level == max_level:
                    # Reached finest level, record voxel
                    final_voxels.append(coords)
                else:
                    # Not at finest level, find children
                    # Columns 0-7 are child indices
                    children = table[idx, :8]
                    
                    # Filter out invalid children (-1)
                    valid_mask = children != -1
                    valid_children = children[valid_mask]
                    
                    # Add valid children to the next level's list
                    new_list.extend(valid_children.tolist())
        
        # Update touched voxels for next level
        cube_touched_idx = new_list
        
        # Optimization: If no voxels are touched, break early
        if not cube_touched_idx:
            break
            
    return final_voxels
```

**Parameters:**
*   `max_level`: The depth of the mipmap hierarchy. The loop runs from $0$ to `max_level` inclusive [S1, unofficial].
*   `cube_size`: Physical size of a voxel at level $L$, calculated as $256 / 2^L$ (assuming a base resolution of 256 units) [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Precision Clipping:** Using a small epsilon (`Eps = 0.000001f`) to clip the ray near the start can cause tiny corner intersections to be missed [S2, unofficial].
*   **Delta Step Size:** In the HDDA update step, a fixed delta (`Delta = 0.01f`) is used to estimate the next voxel position. If `Delta` is too large, it may clip corners of nodes visibly. If too small, it may fail due to floating-point rounding errors when the ray is far from the grid origin [S2, unofficial].
*   **Empty Regions:** The table stores `-1` for empty children. Failing to mask these out will result in invalid memory access or incorrect traversal [S1, unofficial].

## Subtopics

### [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md)
Accelerates volume rendering by traversing a multi-dimensional hierarchy (mdh) to adaptively select resolution levels based on user-defined error tolerance and data importance weighting. Unlike standard ray-grid traversal, it approximates data regions using least-squares trilinear coefficients to decide whether to refine traversal or evaluate data directly, generalizing homogeneity checks to continuous basis functions.

### [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md)
A recursive raycasting algorithm for nested voxel structures (e.g., chunks with subvoxels). It transforms rays into local subvoxel space when entering detail voxels, allowing efficient handling of nested data. Unlike [First Active Voxel Detection](4196-first-active-voxel-detection.md) which stops at the first hit, this method fully traverses the hierarchy to compute precise intersections.

### VolumeHDDA
A specialized Hierarchical Digital Differential Analyzer for traversing volume trees, templated on tree type, ray type, and starting node level [S1, unofficial][S2, unofficial]. It builds on Hierarchical Grid Traversal by implementing recursive ray-marching that enters child nodes when present and processes active/inactive tiles at leaf levels, focusing on discrete tile intersection and time-span management rather than continuous sampling [S1, unofficial][S2, unofficial].

### [XBrickMap](3116-xbrickmap.md)
A space-partitioning ray traversal algorithm for 3-level hierarchical voxel grids using $4^3$ occupancy bitmasks to skip empty regions via bitwise tests. It improves upon simpler skipping by using tiled bitmasks for hierarchical skips of $4^3, 2^3, 1^3$ cuboids and left-packs empty bricks to save memory. Distinguished by compacted bitmasks for random access and support for arbitrary in-place edits.

### [Spherical Volume Voxel Traversal](3877-spherical-volume-voxel-traversal.md)
A ray-marching algorithm for voxel grids structured in spherical coordinates (radial, polar, azimuthal). It solves distortion issues in Cartesian grids for spherical domains (e.g., planetary atmospheres) by computing intersections with spherical shells, polar cones, and azimuthal wedges directly, handling the origin singularity by starting at the first non-zero radial shell.

### [First Active Voxel Detection](4196-first-active-voxel-detection.md)
Determines if a ray intersects an active value at any level of a hierarchical grid and identifies the first such voxel. It solves the problem of skipping empty space in sparse volumes by performing empty-space ray-marching at all tree levels using a Hierarchical DDA (HDDA), setting the intersection time and index upon the first hit.

### [Hierarchical and Block-Based Data Structures](4392-hierarchical-and-block-based-data-structures.md)
A family of methods organizing voxel data into spatial subdivision hierarchies (octrees, bricked grids, BVHs) where internal nodes store aggregate information [4133-hierarchical-data-structures]. This enables pruning of empty/inactive regions, reducing samples for sparse or multi-resolution datasets [4133-hierarchical-data-structures].

### [Hierarchical DDA with Super-brick Skipping](5197-hierarchical-dda-with-super-brick-skipping.md)
Accelerates ray tracing through sparse data using a two-level occupancy hierarchy (super-bricks and bricks) to skip large empty regions. It maintains pixel-identical output to fine per-voxel DDA by verifying emptiness at coarser granularity and using AABBs for early termination, ensuring exactness by only skipping verified empty blocks.

### [Octree-based traversal](5348-octree-based-traversal.md)
Recursively navigates a hierarchical spatial subdivision (typically an octree) to skip empty space or cull nodes. It differs from flat grid traversal by using a tree structure to bypass empty space and from Empty-Space Skipping by employing hierarchical subdivision. It sets itself apart by using explicit bounds tracking, entry/exit side tracking, and callbacks for leaf processing rather than fixed-step iteration.

### [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md)
Reduces computational cost by identifying and bypassing inactive regions using explicit, precomputed or dynamically maintained data structures (min-max blocks, occupancy grids, BVHs, hash maps). Unlike [Space Leaping](5863-space-leaping.md) (which relies on ray coherence) or [Early Shader Termination](969-early-shader-termination.md) (dynamic compositing stop), this family uses spatial bounds defined by scalar block statistics or hierarchical trees.

### [Multi-level Grid Structures](6354-multi-level-grid-structures.md)
Recursively subdivides a spatial domain into nested grids of varying resolution to optimize ray traversal. They solve inefficiency in non-uniform scenes by replacing pointer-chasing trees with contiguous array-based grids, improving memory locality and reducing ray-primitive counts via integer arithmetic.

### [Grid Sphere Traversal](9649-grid-sphere-traversal.md)
Performs sphere tracing through a dense grid of Signed Distance Function (SDF) values. It solves empty region traversal in SDF rendering by using a hierarchy of SDF grids where lower-resolution levels encode larger SDF values to accelerate step sizes. Distinguished by performing traversal entirely in shader code using dense grids, unlike sparse structures or BVHs.

### [Hierarchical Tree Node Traversal](12213-hierarchical-tree-node-traversal.md)
Accelerates volume rendering by traversing a boolean tree structure representing active regions. It solves efficient skipping of empty space and coarse structures. Implemented in OpenVDB as `VolumeRayIntersector`, it operates on generic hierarchical trees (octrees/sparse structures) allowing variable-resolution traversal.

#### KD-Tree Partitioned Volume Rendering
Avoids computing all scalar values by partitioning the volume into sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map stores the minimum opacity of frame buffer pixels within a region. It groups scalar values, uses a kd-tree for visibility-ordered rendering, and culls/clips sub-volumes against the opacity map. Reported 10 fps for the Engine dataset on a 2.5 GHz Pentium-IV and nVidia GeForce4 [S3, unofficial].

#### Mipmap Table Construction
Constructs the lookup table for Hierarchical Grid Traversal by mapping coarse grid voxels to fine grid children [S1, unofficial]. It identifies occupied voxels in the fine grid, assigns unique indices, extracts 8 sub-grid regions for each coarse voxel's children, filters them using a coarse grid mask, and populates the table with child indices (cols 0-7) and coarse coordinates (cols 8-10) [S1, unofficial].

#### Hierarchical Data Structure Traversal
A distributed rendering approach where each processor performs cell connectivity pre-steps, excludes cells outside the camera view, and applies ray-casting [S4]. Rays enter from exterior faces and use connectivity info to determine the next cell, exiting at the second exterior face. Image pixels are assigned to processors, and HDS is used to exchange rays for compositing [S4].

## Operating characteristics
*   **Time Complexity:** $O(N_{visited})$ where $N_{visited}$ is the number of active voxels/tiles intersected by the ray. In sparse volumes, $N_{visited} \ll N_{total}$, offering significant speedup over flat DDA.
*   **Memory Complexity:** $O(N_{nodes})$ for the hierarchical structure and lookup tables. The lookup tables add overhead compared to flat grids but enable skipping.
*   **Performance:** Depends heavily on the sparsity and hierarchy depth. For dense, fully occupied volumes, the overhead of tree traversal and table lookups may make it slower than flat DDA. For sparse volumes, it is substantially faster.
*   **Strengths:** Efficient handling of sparse data, multi-resolution support, reduced memory bandwidth for empty regions.
*   **Limitations:** Preprocessing cost to build the hierarchy and lookup tables. Precision issues with ray-box intersections at boundaries. Not suitable for dense, uniformly sampled data where flat traversal is optimal.
*   **When to Use:** Sparse volumes, multi-resolution datasets, scenes with large empty regions.
*   **When Not to Use:** Dense, fully occupied volumes; real-time applications with strict preprocessing constraints; hardware without efficient random access for table lookups.

## Background
Hierarchical Grid Traversal is a method for accelerating volume rendering by leveraging multi-resolution data structures. It builds on the principles of [Ray-grid traversal](17-ray-grid-traversal.md) but extends them to hierarchical spaces. The method is implemented in various forms, such as the `TreeMarcher` class in HNanoSolver [S2, unofficial] and lookup-table-based traversal in Jax-Volumetric-Renderer [S1, unofficial]. It addresses the computational bottleneck of traversing large, sparse volumes by skipping empty space at multiple resolution levels.

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S4] A Scalable Hybrid Scheme for Ray-Casting of Unstructured Volume Data, https://doi.org/10.1109/tvcg.2018.2833113, fetched 2026-10-24
