# Ray-AABB Intersection (Slab/Min-Max)

Also called: CUDA Octree Box Intersection, Min-Max Method, Slab Test, Slab method

## Overview
The Slab/Min-Max method determines if a ray intersects an Axis-Aligned Bounding Box (AABB) by computing the entry ($t_{near}$) and exit ($t_{far}$) distances along the ray parameter $t$. It solves the problem of efficiently culling rays that miss a volume or voxel grid, serving as the foundational primitive for ray-marching and traversal in volume rendering [S1, S3, S4, unofficial]. It differs from its sibling [Ray-Cube Intersection](2466-ray-cube-intersection.md) by supporting arbitrary rectangular bounding boxes rather than just cubes, and from [Ray-OBB Intersection Test](6603-ray-obb-intersection-test.md) by leveraging axis-alignment to avoid rotation matrices, making it computationally cheaper for axis-aligned structures [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o} = (o_x, o_y, o_z)$
- Ray direction $\mathbf{d} = (d_x, d_y, d_z)$
- AABB defined by minimum corner $\mathbf{b}_{min} = (b_{min,x}, b_{min,y}, b_{min,z})$ and maximum corner $\mathbf{b}_{max} = (b_{max,x}, b_{max,y}, b_{max,z})$ [S1, S3, unofficial].

**Outputs:**
- Intersection interval $[t_{near}, t_{far}]$ [S3, unofficial].
- Boolean hit status (intersection exists within valid range) [S1, unofficial].

**Data Structures:**
- The AABB is typically stored as two 3D vectors: `min` and `max` [S1, S3, unofficial].
- The ray is stored as two 3D vectors: `origin` and `direction` [S1, S3, unofficial].
- Intermediate values are scalar floats for each axis.

**Steps:**

1.  **Compute Inverse Directions:**
    To avoid division by zero and handle parallel rays, compute the inverse direction components:
    $$ i_x = \frac{1}{d_x}, \quad i_y = \frac{1}{d_y}, \quad i_z = \frac{1}{d_z} $$
    [S2, S3, unofficial]. If a direction component is zero, the inverse is typically set to $\pm \infty$ or handled via branchless NaN suppression techniques to ensure robustness [S2, unofficial].

2.  **Compute Slab Intersections:**
    For each axis $k \in \{x, y, z\}$, calculate the intersection distances with the two planes defining the slab (min and max bounds):
    $$ t_{k1} = (b_{min,k} - o_k) \cdot i_k $$
    $$ t_{k2} = (b_{max,k} - o_k) \cdot i_k $$
    [S1, S3, unofficial].

3.  **Sort Slab Intervals:**
    For each axis, ensure $t_{k1} \le t_{k2}$. If $t_{k1} > t_{k2}$, swap them. This effectively sorts the entry and exit points for that specific axis:
    $$ t_{min,k} = \min(t_{k1}, t_{k2}) $$
    $$ t_{max,k} = \max(t_{k1}, t_{k2}) $$
    [S3, unofficial].

4.  **Compute Global Entry and Exit:**
    The ray enters the box at the maximum of the local entry points and exits at the minimum of the local exit points:
    $$ t_{near} = \max(t_{min,x}, t_{min,y}, t_{min,z}) $$
    $$ t_{far} = \min(t_{max,x}, t_{max,y}, t_{max,z}) $$
    [S3, unofficial].

5.  **Determine Intersection:**
    An intersection exists if $t_{near} \le t_{far}$ and the interval overlaps with the ray's valid range $[t_{start}, t_{end}]$ (e.g., near/far clipping planes) [S1, S4, unofficial].
    - If $t_{far} < t_{start}$ or $t_{near} > t_{end}$, there is no hit.
    - Otherwise, the valid intersection interval is $[\max(t_{near}, t_{start}), \min(t_{far}, t_{end})]$ [S1, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_aabb_slab_intersection(ray_origin, ray_direction, box_min, box_max):
    """
    Computes the intersection interval [t_near, t_far] of a ray with an AABB.
    
    Args:
        ray_origin: tuple (ox, oy, oz)
        ray_direction: tuple (dx, dy, dz)
        box_min: tuple (bmin_x, bmin_y, bmin_z)
        box_max: tuple (bmax_x, bmax_y, bmax_z)
        
    Returns:
        tuple (t_near, t_far, hit) where hit is a boolean.
    """
    ox, oy, oz = ray_origin
    dx, dy, dz = ray_direction
    bmin_x, bmin_y, bmin_z = box_min
    bmax_x, bmax_y, bmax_z = box_max

    # Step 1: Compute inverse directions
    # Handle zero direction components to avoid division by zero.
    # [G] Standard practice sets 1/0 to +/- infinity. 
    # For robustness against NaNs in parallel rays, see [S2, unofficial].
    if dx == 0.0:
        ix = float('inf') if ox < bmin_x else float('-inf') # Guard for parallel ray
    else:
        ix = 1.0 / dx
        
    if dy == 0.0:
        iy = float('inf') if oy < bmin_y else float('-inf')
    else:
        iy = 1.0 / dy
        
    if dz == 0.0:
        iz = float('inf') if oz < bmin_z else float('-inf')
    else:
        iz = 1.0 / dz

    # Step 2: Compute slab intersections
    t1_x = (bmin_x - ox) * ix
    t2_x = (bmax_x - ox) * ix
    
    t1_y = (bmin_y - oy) * iy
    t2_y = (bmax_y - oy) * iy
    
    t1_z = (bmin_z - oz) * iz
    t2_z = (bmax_z - oz) * iz

    # Step 3: Sort slab intervals (min/max per axis)
    t_min_x = min(t1_x, t2_x)
    t_max_x = max(t1_x, t2_x)
    
    t_min_y = min(t1_y, t2_y)
    t_max_y = max(t1_y, t2_y)
    
    t_min_z = min(t1_z, t2_z)
    t_max_z = max(t1_z, t2_z)

    # Step 4: Compute global entry and exit
    t_near = max(t_min_x, t_min_y, t_min_z)
    t_far = min(t_max_x, t_max_y, t_max_z)

    # Step 5: Determine intersection
    # Hit if the interval is valid (entry before exit)
    hit = t_near <= t_far
    
    return t_near, t_far, hit
```

**Parameters:**
- `voxelsize`: Used in some implementations to derive `box_min` and `box_max` from voxel indices [S2, unofficial].
- `n_max`: Maximum number of intersections to store per ray in batched implementations [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Parallel Rays:** If $d_k = 0$, the ray is parallel to the planes perpendicular to axis $k$. The inverse direction becomes infinite. If the ray origin is outside the slab, the intersection distance is $\pm \infty$, which correctly fails the $\max/\min$ logic if implemented with standard float comparisons. However, NaN propagation can occur if not handled [S2, unofficial].
- **Ray Origin Inside Box:** If the ray origin is inside the box, $t_{near}$ may be negative. The intersection interval should be clamped to $[0, \infty)$ if only forward rays are considered [S1, unofficial].
- **Degenerate Boxes:** If $b_{min,k} = b_{max,k}$, the box is degenerate in that dimension. The method still works, returning a zero-width interval if the ray hits the plane exactly [S1, unofficial].

## Subtopics

### [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md)
This method computes the intersection interval $[t_{min}, t_{max}]$ of camera rays with a scene bounding box (AABB), clamps these intervals to near/far clipping planes, and generates a set of query points along each ray for volume rendering. It solves the problem of determining valid spatial extents for ray marching and preparing structured sample coordinates (including ray IDs and step indices) for downstream neural radiance field evaluation. It differs from its sibling [Ray-Cube Intersection](2466-ray-cube-intersection.md) by supporting arbitrary axis-aligned bounding boxes rather than just cubes, and from [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md) by explicitly handling the generation of sample points and metadata (ray/step IDs) rather than just returning boolean intersection results.

### [Ray-Cube Intersection](2466-ray-cube-intersection.md)
This method calculates the precise entry ($t_0$) and exit ($t_1$) intersection points of a ray with an axis-aligned cube defined by lower and upper bounds. It solves the problem of determining the exact boundary hit for ray-casting or ray-marching within voxel grids, providing the entry position and the exit face normal required for subsequent traversal or shading. Unlike [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md), which computes the interval $[t_{min}, t_{max}]$ for traversal logic, this method explicitly returns the 3D intersection coordinates and the specific face index exited, which is critical for hardware-accelerated voxel rendering where the first solid voxel must be identified.

### [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md)
This method computes the intersection of a ray with an Axis-Aligned Bounding Box (AABB) using a branchless algorithm that correctly handles IEEE 754 floating-point edge cases, specifically NaNs generated when a ray direction component is zero. It builds on [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md) by replacing conditional checks for zero direction with precomputed inverse directions and nested min/max operations that suppress NaN propagation. Unlike the standard branchless slab method, this variant guarantees robustness for degenerate cases where the ray origin lies exactly on a box boundary or the ray is parallel to an axis, preventing false positives caused by NaN propagation.

### [AABB Ray Intersection](11086-aabb-ray-intersection.md)
This method performs parallel ray-axis-aligned bounding box (AABB) intersection tests for volume rendering, specifically designed to handle large-scale ray-voxel queries on GPUs without exceeding memory limits. It differs from standard single-ray implementations by batching rays into chunks and processing them via a CUDA kernel that outputs intersection depths ($t_{min}, t_{max}$) and hit status for multiple rays and AABBs simultaneously. It is chosen when the number of rays and voxels is too large for a single contiguous memory allocation, requiring explicit chunking to stay within hardware constraints (e.g., <2GB VRAM).

### [AABB Intersection via Face Traversal](13498-aabb-intersection-via-face-traversal.md)
A method to find the entry (front) and exit (back) points of a ray intersecting an Axis-Aligned Bounding Box (AABB) by explicitly testing the ray against each of the six faces of the AABB. It is less efficient than the standard slab method (Ray-AABB Intersection (Slab/Min-Max)) because it decomposes each quad face into two triangles and performs ray-triangle intersection tests, but it is explicitly noted as a current limitation in implementations that require ray-to-axis-aligned rectangle intersection rather than full ray-AABB intersection.

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray-box test. It involves a constant number of arithmetic operations (subtractions, multiplications, comparisons) regardless of box size or resolution [S1, S3, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space. Only a few scalar variables are needed per ray [S1, S3, unofficial].
- **Measured Performance:** Highly efficient on CPUs and GPUs due to its branch-predictor-friendly nature (especially when branchless variants are used) and low register pressure [S2, unofficial].
- **Strengths:** Simplicity, speed, and robustness for axis-aligned structures. It is the standard method for initial culling in ray tracers and volume renderers [S1, S4, unofficial].
- **Limitations:** Does not provide the intersection normal or precise hit point without additional computation. It is not suitable for oriented bounding boxes (OBBs) without transformation [S6603, S1].
- **When to use:** For initial broad-phase culling in volume rendering, voxel traversal, and any scenario involving axis-aligned primitives [S1, S4, unofficial].
- **When not to use:** For oriented boxes (use [Ray-OBB Intersection Test](6603-ray-obb-intersection-test.md)) or when precise hit normals are required immediately without post-processing [S1, unofficial].

## Background
The slab method is a classic technique in computer graphics for ray-box intersection. It is widely used in volume rendering to determine the entry and exit points of rays into volumetric datasets, accelerating the rendering process by skipping rays that miss the volume [S4, unofficial]. The method is implemented in various open-source volume rendering projects, including SolidVoxelization [S1, unofficial], NSVF [S2, unofficial], and UnityVolumeRendering [S3, unofficial]. The approach is fundamental to efficient ray-marching algorithms, where the bounding box defines the limits of the sampling domain [S4, unofficial].

## Sources
[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S2] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S3] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S4] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
