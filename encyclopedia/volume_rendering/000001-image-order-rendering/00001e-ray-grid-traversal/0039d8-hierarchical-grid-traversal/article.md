# Hierarchical Grid Traversal

Also called: Hierarchical Grid Intersection, Ray-Mipmap Traversal

## Overview
Hierarchical Grid Traversal is a ray-grid traversal method that accelerates volume rendering by stepping through a multi-resolution grid hierarchy (mipmap) rather than a single flat grid [S1, unofficial][S2, unofficial]. It solves the problem of excessive traversal overhead in sparse or large-volume datasets by skipping empty or uniform regions at coarse levels and refining only where the ray intersects active data [S1, unofficial]. Unlike flat [Digital Differential Analyzer (DDA) Traversal](3061-digital-differential-analyzer-dda-traversal.md) which visits every voxel boundary, this method uses lookup tables to jump directly from a parent voxel to its children, reducing the number of intersection tests and memory accesses [S1, unofficial].

## How it works
The method iterates through levels of a grid mipmap, starting from the root (level 0) and recursively visiting child voxels only if the ray intersects the parent voxel's axis-aligned bounding box (AABB) [S1, unofficial]. It relies on precomputed lookup tables that map parent voxel indices to their children's indices and coordinates [S1, unofficial].

### Inputs and Outputs
- **Inputs**:
  - Ray: Origin $\mathbf{o}$, direction $\mathbf{d}$, and interval $[t_0, t_1]$ [S2, unofficial].
  - Hierarchy: A list of lookup tables (`tables_list`), one per level, and the maximum level `max_level` [S1, unofficial].
  - Grid parameters: `cube_size` at the current level, typically derived from a base resolution (e.g., $256 / 2^{\text{level}}$) [S1, unofficial].
- **Outputs**:
  - A list of voxel positions (or indices) at the maximum level that the ray intersects [S1, unofficial].

### Data Structures
The core data structure is a lookup table for each hierarchy level [S1, unofficial].

**Table Layout**:
Each row corresponds to a coarse voxel at the current level. The table has 11 columns:
- Columns 0–7: Indices of the 8 child voxels in the next finer level. If a child is empty or invalid, the value is $-1$ [S1, unofficial].
- Columns 8–10 (or -3 to -1): The 3D grid coordinates $(x, y, z)$ of the coarse voxel [S1, unofficial].

**Class State (Implementation Specific)**:
Some implementations use a class `TreeMarcher` with private members:
- `mAcc`: Reference to the acceleration structure [S2, unofficial].
- `mRay`: The ray being traversed [S2, unofficial].
- `mHdda`: A Hierarchical DDA instance for precise boundary crossing [S2, unofficial].
- `mT0`: Current entry time into the current node [S2, unofficial].
- `mTmax`: The exit time of the ray from the entire volume [S2, unofficial].

### Algorithm Steps
1. **Initialization**:
   - Start with `cube_touched_idx = [0]` (the root voxel index) and an empty `new_list` [S1, unofficial].
   - Clip the ray to the root bounding box. If no intersection, terminate [S2, unofficial].
   - Set epsilon values for precision: `Eps = 0.000001f` and `Delta = 0.01f` [S2, unofficial].
   - Compute initial entry time $t_0 = \text{ray}.t_0() + \text{Eps}$ and exit time $t_1 = \text{ray}.t_1() - \text{Eps}$ [S2, unofficial]. If $t_0 > t_1$, terminate [S2, unofficial].
   - Determine the initial voxel index `ijk` by rounding down the ray position at $t_0$ [S2, unofficial].
   - Initialize the Hierarchical DDA (`mHdda`) with the ray, $t_0$, $t_1$, and the dimension of the current node [S2, unofficial].

2. **Level Iteration**:
   - Iterate through each level $L$ from 0 to `max_level` [S1, unofficial].
   - Retrieve the lookup table for level $L$: `table = tables_list[L]` [S1, unofficial].

3. **Ray-AABB Intersection**:
   - For each voxel index `i` in `cube_touched_idx`:
     - Retrieve the voxel's origin and end coordinates from the table:
       $$ \mathbf{cube\_ori} = \text{table}[i, -3:] \times \text{cube\_size} $$
       $$ \mathbf{cube\_end} = \mathbf{cube\_ori} + \text{cube\_size} $$
     - Compute the ray-AABB intersection, returning near time $t_n$ and far time $t_f$ [S1, unofficial].
     - If $t_n \le t_f$, the ray intersects this voxel [S1, unofficial].

4. **Refinement or Termination**:
   - If $L == \text{max\_level}$:
     - Append the voxel's grid coordinates (from the table) to the final hit list `cube_ori_list` [S1, unofficial].
   - If $L < \text{max\_level}$:
     - Extract the 8 child indices from columns 0–7 of the table row [S1, unofficial].
     - Filter out invalid children (where index $= -1$) using a mask: `mask = table[i, :8] != -1` [S1, unofficial].
     - Convert valid indices to integers and add them to `new_list` [S1, unofficial].

5. **Update State**:
   - Set `cube_touched_idx = new_list` and reset `new_list` to empty for the next level [S1, unofficial].
   - (Optional/HDDA-based) Update traversal state using the Hierarchical DDA:
     - Get current node from acceleration structure [S2, unofficial].
     - Step the HDDA: `hddaIsValid = mHdda.step()` [S2, unofficial].
     - Compute next voxel index and dimension [S2, unofficial].
     - Update `mT0` based on whether the next dimension is finer than the child level [S2, unofficial].
     - If `mT0 >= 0`, set the span $[t_0, t_1]$ for the current node and return the node pointer [S2, unofficial].

6. **Final Output**:
   - Return `cube_ori_list` containing the positions of voxels touched at the maximum level [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def hierarchical_grid_traversal(ray_origin, ray_direction, ray_t0, ray_t1, tables_list, max_level, base_resolution=256):
    """
    Traverses a hierarchical grid mipmap to find intersected voxels at the finest level.
    
    Parameters:
    - ray_origin: (3,) array, ray origin
    - ray_direction: (3,) array, ray direction
    - ray_t0: float, near clip distance
    - ray_t1: float, far clip distance
    - tables_list: list of arrays, lookup tables for each level. 
                   Each table has shape (num_voxels, 11). 
                   Cols 0-7: child indices (-1 if empty). Cols 8-10: voxel coords.
    - max_level: int, maximum depth of the hierarchy
    - base_resolution: int, base grid size (e.g., 256)
    
    Returns:
    - list of (3,) arrays, coordinates of intersected voxels at max_level
    """
    
    # Helper: Ray-AABB intersection
    def intersect_ray_aabb(origin, direction, box_min, box_max):
        # Compute inverse direction
        inv_dir = 1.0 / direction
        # Compute t for each face
        t1 = (box_min - origin) * inv_dir
        t2 = (box_max - origin) * inv_dir
        # Near and far times
        tn = max(min(t1), min(t2))
        tf = min(max(t1), max(t2))
        return tn, tf

    # Initialize with root voxel index (0)
    cube_touched_idx = [0]
    cube_ori_list = []

    for level in range(max_level + 1):
        table = tables_list[level]
        # Calculate voxel size at this level
        # Assuming base_resolution is the size at level 0
        cube_size = base_resolution / (2 ** level)
        
        new_list = []
        
        for i in cube_touched_idx:
            # Get voxel coordinates and size
            # Cols 8-10 are x, y, z
            voxel_coords = table[i, 8:11]
            box_min = voxel_coords * cube_size
            box_max = box_min + cube_size
            
            # Check intersection
            tn, tf = intersect_ray_aabb(ray_origin, ray_direction, box_min, box_max)
            
            if tn <= tf:
                if level == max_level:
                    # Reached finest level, add to result
                    cube_ori_list.append(voxel_coords)
                else:
                    # Get child indices (Cols 0-7)
                    children = table[i, 0:8]
                    # Filter out invalid children (-1)
                    valid_mask = children != -1
                    valid_children = children[valid_mask]
                    
                    # Add valid children to next level list
                    new_list.extend(valid_children)
        
        cube_touched_idx = new_list
        if not cube_touched_idx:
            break
            
    return cube_ori_list
```

### Parameters
- **max_level**: The maximum depth of the mipmap hierarchy. The loop runs from 0 to `max_level` inclusive [S1, unofficial].
- **cube_size**: The physical size of a voxel at the current level. For a base resolution of 256, `cube_size = 256 / (2 ** level)` [S1, unofficial].
- **Eps**: Small epsilon for clipping ray bounds to avoid self-intersection or precision issues. Typical value: $0.000001f$ [S2, unofficial].
- **Delta**: Small offset for HDDA step updates. Typical value: $0.01f$ [S2, unofficial].

### Edge Cases and Pitfalls
- **Floating-Point Precision**: If `Delta` is too large, it may clip corners of nodes visibly. If too small, it may fail due to floating-point rounding errors when the ray is far from the grid origin [S2, unofficial].
- **Tiny Intersections**: Tweaking the intersection span with `Eps` may clip tiny corner intersections [S2, unofficial].
- **Empty Voxels**: Children marked as $-1$ in the lookup table must be filtered out to avoid processing non-existent voxels [S1, unofficial].
- **Vectorization**: In vectorized implementations (e.g., processing multiple rays), `np.zeros_like` is used for `tn` and `tf`, and rays are masked out if `tn > tf` [S1, unofficial].

## Subtopics

### [Hierarchical Trilinear Ray Casting](851-hierarchical-trilinear-ray-casting.md)
Hierarchical Trilinear Ray Casting is a volume rendering method that accelerates ray casting by traversing a multi-dimensional hierarchy (mdh) and adaptively selecting the resolution level for data evaluation based on a user-defined error tolerance and data importance weighting. It differs from sibling methods like [Hierarchical Digital Differential Analyzer (HDDA)](1122-hierarchical-digital-differential-analyzer-hdda.md) or [Hierarchical Voxel Traversal](2204-hierarchical-voxel-traversal.md) by using least-squares trilinear basis functions to model data regions, allowing the algorithm to decide whether to refine traversal or evaluate the region directly based on the approximation error. Unlike methods that rely on simple homogeneity (constant values), this approach generalizes homogeneity to least-squares fits, providing a more robust metric for heterogeneous but smooth regions.

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
Octree-based traversal accelerates volume rendering and implicit surface rendering by recursively traversing a hierarchical spatial subdivision (typically an octree) to skip empty space and large uniform regions. It differs from flat grid traversal by using tree structure to bypass empty volumes, and from Empty-Space Skipping which uses primitive hulls, by employing hierarchical spatial subdivision to locate non-empty space faster. It sets itself apart from its siblings, such as [Octree Partitioning for Sphere Tracing](1541-octree-partitioning-for-sphere-tracing.md), by using explicit bounds tracking and entry/exit side tracking with a callback for leaf processing rather than fixed-step iteration.

### [Empty Space Skipping via Data Structures](5619-empty-space-skipping-via-data-structures.md)
Empty Space Skipping via Data Structures is a family of volume rendering optimization techniques that reduce computational cost by identifying and bypassing inactive (empty) regions of the volume data. It solves the problem of wasting samples on voxels that do not contribute to the final image (e.g., zero density or below an isovalue threshold). Unlike image-order methods like [Space Leaping](5863-space-leaping.md) which rely on ray coherence, or [Distance Map Acceleration](972-distance-map-acceleration.md) which uses metric distances, this family relies on explicit secondary data structures (grids, trees, or maps) to define the spatial extent of active data. The specific implementation varies significantly: some methods use object-order geometry culling [865-object-order-empty-space-skipping.md], others use hardware depth tests [1537-ray-casting-with-depth-alpha-test-for-empty-space-skipping.md], and others use hierarchical tree traversal [1911-semi-transparent-empty-space-skipping.md].

### [Multi-level Grid Structures](6354-multi-level-grid-structures.md)
Multi-level Grid Structures are a family of hierarchical acceleration algorithms that recursively subdivide a spatial domain into nested grids of varying resolution to optimize ray traversal. They solve the problem of inefficient traversal in non-uniform scenes by replacing pointer-chasing tree structures with contiguous array-based grids, thereby improving memory locality and reducing ray-primitive intersection counts. Unlike flat grids, they adapt resolution based on object density, and unlike tree structures, they use fixed-size grid arrays at each level, allowing for faster traversal via integer arithmetic.

### KD-Tree Partitioned Volume Rendering
A method to avoid computing all scalar values regardless of visibility by partitioning the volume into smaller sub-volumes with similar properties based on the transfer function [S3, unofficial]. The opacity map corresponds to a region of the frame buffer and stores the minimum opacity of the frame buffer pixels found within that region [S3, unofficial]. Scalar values within a certain range are grouped together to define sub-volume properties [S3, unofficial]. A kd-tree is used to render the partitioned volume with correct visibility order, where each node is associated with a sub-volume [S3, unofficial]. Each sub-volume is culled and clipped against an opacity map [S3, unofficial]. Reported performance: 10 fps for the Engine data set using a 2.5 GHz Pentium-IV and a nVidia GeForce4 graphics adapter [S3, unofficial].

### Mipmap Table Construction
Constructs a lookup table for hierarchical grid traversal. It maps coarse grid voxels to their fine grid children. The table is built from a fine grid and a coarse grid mask [S1, unofficial]. Occupied voxels in the fine grid are identified and assigned unique indices [S1, unofficial]. The 8 sub-grid regions corresponding to the 8 children of a coarse voxel are extracted by striding the fine grid indices [S1, unofficial]. These sub-grids are flattened and filtered using the coarse grid mask to select only children belonging to occupied coarse voxels [S1, unofficial]. The table is initialized with shape `[num_coarse_voxels, 11]` (8 child indices + 3 coordinates), filled with -1 [S1, unofficial]. Columns 0–7 are populated with filtered child indices, and columns 8–10 with coarse voxel coordinates [S1, unofficial].

### Grid Sphere Traversal
Sphere tracing through a dense grid of SDF values, using a hierarchy of SDF grids to accelerate traversal of empty regions [S4, unofficial]. Lower-resolution levels encode larger SDF values to accelerate traversal of empty regions [S4, unofficial]. The algorithm adaptively selects a grid resolution to safely maximize the step size taken by the sphere tracing traversal [S4, unofficial]. Traversal is performed entirely in shader code [S4, unofficial]. SDF values are formatted and clamped to $[-1, 1]$, where a value of 1 encodes 2.5 interior voxel diagonals, allowing storage as 8-bit snorm integers [S4, unofficial]. Similar to the technique used in Claybook [Aaltonen 2018] [S4, unofficial].

## Operating characteristics
- **Time Complexity**: Depends on the sparsity of the volume and the depth of the hierarchy. In the worst case (dense volume), it approaches the complexity of flat DDA but with higher per-voxel overhead due to table lookups. In sparse volumes, it can be significantly faster by skipping large empty regions at coarse levels.
- **Memory Complexity**: Requires storage for the lookup tables. The total size is proportional to the sum of the number of voxels at each level, but only for occupied regions (if sparse). The table size is $11 \times N_{\text{coarse}}$ per level [S1, unofficial].
- **Performance**: Reported performance for related hierarchical methods (e.g., KD-Tree partitioned) is 10 fps for the Engine data set on a 2.5 GHz Pentium-IV and nVidia GeForce4 [S3, unofficial].
- **Strengths**: Efficient for sparse volumes; reduces memory bandwidth by skipping empty space; adaptable resolution.
- **Limitations**: Requires precomputation of lookup tables; overhead for dense volumes; precision issues with small epsilon values [S2, unofficial].
- **When to use**: Sparse volumetric data, large datasets where empty space dominates.
- **When not to use**: Dense volumes where the overhead of table lookups outweighs the benefit of skipping; dynamic data where table construction cost is prohibitive.

## Background
Hierarchical Grid Traversal is a method for accelerating ray traversal in volume rendering by using a multi-resolution grid hierarchy (mipmap) [S1, unofficial][S2, unofficial]. It is closely related to Hierarchical Digital Differential Analyzer (HDDA) and other hierarchical traversal techniques. The method is implemented in projects such as Jax-Volumetric-Renderer [S1, unofficial] and HNanoSolver [S2, unofficial]. Variants include Grid Sphere Traversal for SDF grids [S4, unofficial] and KD-Tree Partitioned Volume Rendering for transfer-function-based partitioning [S3, unofficial].

## Sources
[S1] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S2] ZephirFXEC/HNanoSolver, https://github.com/ZephirFXEC/HNanoSolver, fetched 2026-10-27 (unofficial)
[S3] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S4] Ray tracing of signed distance function grids, https://www.jcgt.org/published/0011/03/06/paper-lowres.pdf, fetched 2026-10-25 (unofficial)
