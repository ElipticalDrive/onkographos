# Ray-AABB Intersection (Slab/Min-Max)

Also called: CUDA Octree Box Intersection, Min-Max Method, Slab Test, Slab method, VoxelGridBase::intersectionWithBox

## Overview
The Slab/Min-Max method determines the intersection interval $[t_{min}, t_{max}]$ of a ray with an Axis-Aligned Bounding Box (AABB) by computing the entry and exit distances along each axis independently [S1, S3, S4, unofficial]. It solves the problem of efficiently culling rays that miss a volume primitive or defining the valid traversal range for ray marching [S1, S5, unofficial]. It differs from [Ray-box intersection](8356-ray-box-intersection.md) by providing the explicit parametric interval required for downstream sampling, and from [Ray-Cube Intersection](2466-ray-cube-intersection.md) by supporting arbitrary box dimensions rather than assuming unit cubes [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
- Ray origin $\mathbf{o} = (o_x, o_y, o_z)$ [S1, S3, S4, unofficial].
- Ray direction $\mathbf{d} = (d_x, d_y, d_z)$ [S1, S3, S4, unofficial].
- AABB defined by minimum corner $\mathbf{b}_{min} = (b_{min,x}, b_{min,y}, b_{min,z})$ and maximum corner $\mathbf{b}_{max} = (b_{max,x}, b_{max,y}, b_{max,z})$ [S1, S3, S4, unofficial].

**Outputs:**
- Boolean hit status [S3, unofficial].
- Intersection interval $[t_{near}, t_{far}]$ [S4, unofficial].
- Intersection point $\mathbf{p} = \mathbf{o} + t_{near}\mathbf{d}$ [S3, unofficial].
- (Optional) Normal vector at intersection [S1, unofficial].

**Data Structures:**
- Vectors are 3-component floating-point arrays.
- The AABB is stored as two 3D points [S3, S4, unofficial].

**Steps:**

1. **Compute Slab Intersections:** For each axis $i \in \{x, y, z\}$, calculate the parameter $t$ where the ray intersects the min and max planes of the box [S1, S3, S4, unofficial]:
   $$
   t_{min,i} = \frac{b_{min,i} - o_i}{d_i}
   $$
   $$
   t_{max,i} = \frac{b_{max,i} - o_i}{d_i}
   $$
   [S1, S3, S4, unofficial]

2. **Determine Local Interval:** For each axis, the ray is inside the slab between the smaller and larger of the two computed $t$ values [S3, S4, unofficial]:
   $$
   t_{1,i} = \min(t_{min,i}, t_{max,i})
   $$
   $$
   t_{2,i} = \max(t_{min,i}, t_{max,i})
   $$
   [S4, unofficial]

3. **Compute Global Interval:** The ray is inside the box only when it is inside all slabs simultaneously. The entry distance $t_{near}$ is the maximum of the local entry distances, and the exit distance $t_{far}$ is the minimum of the local exit distances [S3, S4, unofficial]:
   $$
   t_{near} = \max(t_{1,x}, t_{1,y}, t_{1,z})
   $$
   $$
   t_{far} = \min(t_{2,x}, t_{2,y}, t_{2,z})
   $$
   [S3, S4, unofficial]

   Alternatively, using component-wise operations on vectors $\mathbf{t}_{min}$ and $\mathbf{t}_{max}$ from Step 1 [S3, unofficial]:
   $$
   t_{near} = \max(\min(\mathbf{t}_{min}, \mathbf{t}_{max}))
   $$
   $$
   t_{far} = \min(\max(\mathbf{t}_{min}, \mathbf{t}_{max}))
   $$
   [S3, unofficial]

4. **Check Intersection:** An intersection exists if the interval is valid and in front of the ray origin [S3, unofficial]:
   $$
   t_{near} \le t_{far} \quad \text{and} \quad t_{near} \ge 0
   $$
   [S3, unofficial]

5. **Calculate Intersection Point (Optional):** If an intersection exists, the entry point is:
   $$
   \mathbf{p} = \mathbf{o} + t_{near} \mathbf{d}
   $$
   [S3, unofficial]

**Parameters:**
- `voxelsize`: The size of the voxels (AABBs) if the box is derived from a voxel grid [S2, unofficial].
- `n_max`: The maximum number of intersections to store per ray if batching multiple boxes [S2, unofficial].

**Edge Cases and Pitfalls:**
- **Zero Direction Component:** If $d_i = 0$, the division in Step 1 produces infinity or NaN. If the ray origin is outside the slab ($o_i < b_{min,i}$ or $o_i > b_{max,i}$), the ray misses the box. If the origin is inside, the ray is parallel to the slab and never exits/enters along that axis, effectively ignoring the constraint [S2, S3, unofficial]. Standard implementations must handle division by zero, often by checking for zero direction or using precomputed inverse directions [S2, S3, unofficial].
- **NaN Propagation:** If not handled, NaNs from zero-direction divisions can propagate through min/max operations, causing false negatives [S2, unofficial].
- **Grazing Rays:** Rays parallel to a face or lying exactly on a boundary may suffer from precision issues, potentially missing the box due to floating-point errors [S2, unofficial].
- **Hollow Objects:** When ray marching inside hollow objects, the gradient at the start of the ray may be zero if the ray starts "inside the material" because gradient calculation uses data before the ray start [S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_aabb_slab_intersection(ray_origin, ray_direction, box_min, box_max):
    """
    Computes the intersection of a ray with an AABB using the slab method.
    
    Args:
        ray_origin: tuple (ox, oy, oz)
        ray_direction: tuple (dx, dy, dz)
        box_min: tuple (bmin_x, bmin_y, bmin_z)
        box_max: tuple (bmax_x, bmax_y, bmax_z)
        
    Returns:
        tuple (hit: bool, t_near: float, t_far: float, point: tuple or None)
    """
    ox, oy, oz = ray_origin
    dx, dy, dz = ray_direction
    bmin_x, bmin_y, bmin_z = box_min
    bmax_x, bmax_y, bmax_z = box_max

    # Handle division by zero for direction components
    # If dx is 0, t_min_x and t_max_x become +/- infinity.
    # We use a large number or handle explicitly. 
    # Standard approach: compute inverse direction.
    
    inv_dx = 1.0 / dx if dx != 0.0 else float('inf')
    inv_dy = 1.0 / dy if dy != 0.0 else float('inf')
    inv_dz = 1.0 / dz if dz != 0.0 else float('inf')

    # Compute t values for min and max planes
    t_min_x = (bmin_x - ox) * inv_dx
    t_max_x = (bmax_x - ox) * inv_dx
    
    t_min_y = (bmin_y - oy) * inv_dy
    t_max_y = (bmax_y - oy) * inv_dy
    
    t_min_z = (bmin_z - oz) * inv_dz
    t_max_z = (bmax_z - oz) * inv_dz

    # Determine local min/max for each axis
    t1_x = min(t_min_x, t_max_x)
    t2_x = max(t_min_x, t_max_x)
    
    t1_y = min(t_min_y, t_max_y)
    t2_y = max(t_min_y, t_max_y)
    
    t1_z = min(t_min_z, t_max_z)
    t2_z = max(t_min_z, t_max_z)

    # Compute global near and far
    t_near = max(t1_x, t1_y, t1_z)
    t_far = min(t2_x, t2_y, t2_z)

    # Check for intersection
    # t_near <= t_far ensures the intervals overlap
    # t_near >= 0 ensures the intersection is in front of the ray
    if t_near <= t_far and t_near >= 0:
        hit = True
        # Calculate intersection point
        px = ox + t_near * dx
        py = oy + t_near * dy
        pz = oz + t_near * dz
        point = (px, py, pz)
    else:
        hit = False
        t_near = float('inf')
        t_far = float('-inf')
        point = None

    return hit, t_near, t_far, point
```

## Subtopics

### [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md)
This method computes the intersection interval $[t_{min}, t_{max}]$ of camera rays with a scene bounding box (AABB), clamps these intervals to near/far clipping planes, and generates a set of query points along each ray for volume rendering. It solves the problem of determining valid spatial extents for ray marching and preparing structured sample coordinates (including ray IDs and step indices) for downstream neural radiance field evaluation. It differs from its sibling [Ray-Cube Intersection](2466-ray-cube-intersection.md) by supporting arbitrary axis-aligned bounding boxes rather than just cubes, and from [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md) by explicitly handling the generation of sample points and metadata (ray/step IDs) rather than just returning boolean intersection results.

### [Ray-Cube Intersection](2466-ray-cube-intersection.md)
This method calculates the precise entry ($t_0$) and exit ($t_1$) intersection points of a ray with an axis-aligned cube defined by lower and upper bounds. It solves the problem of determining the exact boundary hit for ray-casting or ray-marching within voxel grids, providing the entry position and the exit face index required for subsequent traversal or shading. Unlike [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md), which computes the interval $[t_{min}, t_{max}]$ for traversal logic, this method explicitly returns the 3D intersection coordinates and the specific face index exited, which is critical for hardware-accelerated voxel rendering where the first solid voxel must be identified.

### [Branchless Ray/AABB Intersection with NaN Handling](3297-branchless-ray-aabb-intersection-with-nan-handling.md)
This method computes the intersection of a ray with an Axis-Aligned Bounding Box (AABB) using a branchless algorithm that correctly handles IEEE 754 floating-point edge cases, specifically NaNs generated when a ray direction component is zero. It builds on [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md) by replacing conditional checks for zero direction with precomputed inverse directions and nested min/max operations that suppress NaN propagation. Unlike the standard branchless slab method, this variant guarantees robustness for degenerate cases where the ray origin lies exactly on a box boundary or the ray is parallel to an axis, preventing false positives caused by NaN propagation.

### [Ray-OBB Intersection Test](6603-ray-obb-intersection-test.md)
The Ray-OBB Intersection Test determines if a ray intersects an Oriented Bounding Box (OBB) and computes the distance to the nearest intersection point. It extends the slab method by projecting the ray onto the OBB's local axes to compute intersection intervals (slabs) along each axis. Unlike [Ray-box intersection](776-ray-box-intersection.md) which assumes axis alignment, this method handles arbitrary orientations by testing intersection with pairs of parallel planes perpendicular to the OBB's local axes.

### [AABB Ray Intersection](11086-aabb-ray-intersection.md)
The AABB Ray Intersection method computes the entry ($t_{min}$) and exit ($t_{max}$) distances of a ray against an Axis-Aligned Bounding Box (AABB), returning a boolean hit status. It solves the fundamental problem of determining whether a ray intersects a volume primitive and, if so, the interval along the ray parameter $t$ where the intersection occurs. It is distinguished from its siblings by its reliance on the "slab" method (computing intersections with min/max planes per axis) and its common implementation as a batched kernel for GPU acceleration, whereas [Ray-Bounding Box Intersection and Sampling Preparation](2364-ray-bounding-box-intersection-and-sampling-preparation.md) focuses on sampling preparation and [Ray-Cube Intersection](2466-ray-cube-intersection.md) assumes unit cubes.

### [AABB Intersection via Face Traversal](13498-aabb-intersection-via-face-traversal.md)
AABB Intersection via Face Traversal is a method to determine the entry (front) and exit (back) points of a ray intersecting an Axis-Aligned Bounding Box (AABB) by explicitly testing the ray against each of the six faces of the box. Unlike the standard slab method which computes intersection intervals along axes, this approach decomposes the AABB into six planar faces and performs individual ray-face intersection tests. It is distinguished from its siblings by its explicit geometric decomposition into faces and triangles, making it conceptually simpler but computationally more expensive than the slab method.

### [Segment-Box Collision Detection with Radius](17033-segment-box-collision-detection-with-radius.md)
Determines if a line segment defined by endpoints $a$ and $b$ intersects an axis-aligned box defined by lower corner $lo$ and upper corner $hi$, considering a spherical radius $r$ around the segment. It sets itself apart from standard ray-box intersection by treating the segment as a capsule (cylinder with hemispherical caps) rather than an infinitely thin line, allowing for collision detection with thickness.

### [Watertight BVH Box Traversal Fix](17518-watertight-bvh-box-traversal-fix.md)
This method fixes a robustness bug in shared slab tests where a ray with a zero direction component lying exactly in the plane of a box face misses the box on one side. It ensures that grazing rays, such as orthographic camera rays landing exactly on a bound, correctly intersect triangles whose edges they touch, by treating BVH boxes as closed sets. Unlike standard slab tests which may compute an empty slab for $t > 0$ due to zero-inversion artifacts, this fix guarantees consistent intersection decisions for axis-parallel rays on face planes.

### Ray-Bounding Box Intersection
A bounding box is used to determine ray entry and exit from the volumetric dataset, accelerating calculation by skipping rays that miss the volume and enabling 'cutting' the volume to look inside [S5, unofficial]. When looking inside a hollow object (like the Stanford Bunny), the gradient at the start of the ray may be 0 if the ray starts 'inside the material' because gradient calculation uses data before the ray start [S5, unofficial].

### Ray-AABB Intersection and Grid Traversal Setup
Compute ray-AABB intersections for all grids to get entry ($t_{mins}$) and exit ($t_{maxs}$) distances, and a hit flag ($hits$) [S6, unofficial]. Clamp $t_{mins}$ and $t_{maxs}$ to the near and far planes [S6, unofficial]. Set $t_{mins}$ and $t_{maxs}$ to $miss\_value$ if there is no intersection ($hits$ is false) [S6, unofficial]. Sort the concatenated $t_{mins}$ and $t_{maxs}$ values for each ray to prepare for traversal [S6, unofficial]. The parameter $miss\_value$ is the value assigned to $t_{min}/t_{max}$ when a ray misses an AABB (typically infinity or a value outside the valid range) [S6, unofficial].

### Slab-based Bounding Volume
A bounding volume defined as the intersection of slabs, where slabs are formed by the intersection of two half-spaces extending in opposite directions [S7]. Utilizing slabs significantly speeds up the evaluation of the signed distance function $h_e$ by dividing the number of scalar products by two [S7].

## Operating characteristics
- **Time Complexity:** $O(1)$ per ray-box intersection, involving a constant number of arithmetic operations (subtractions, divisions, min/max comparisons) [S1, S3, S4, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary space [S1, S3, S4, unofficial].
- **Performance:** The slab method is generally faster than face-traversal methods because it avoids explicit plane equations and normal calculations for each face, relying instead on simple component-wise operations [S1, S2, unofficial][S7]. It is highly parallelizable, making it suitable for GPU acceleration [S2, unofficial].
- **Strengths:** Simplicity, speed, and robustness (when NaNs are handled) [S1, S2, unofficial][S7].
- **Limitations:** Standard implementations may fail or produce incorrect results for rays with zero direction components if not carefully coded (NaN propagation) [S2, unofficial]. It does not provide face normals or specific exit faces without additional logic [S1, unofficial].
- **When to use:** When bounding volume hierarchy (BVH) traversal or simple ray-marching culling is required [S1, S5, unofficial].
- **When not to use:** When oriented bounding boxes (OBBs) are needed, or when specific face intersection details (like exit face index) are required without post-processing [S1, unofficial].

## Background
The slab method is a fundamental technique in ray tracing and volume rendering, widely used for ray-primitive intersection tests [S1, S3, S4, unofficial]. It is based on the geometric principle that an AABB is the intersection of three slabs (pairs of parallel planes) [S7]. The method is commonly implemented in graphics libraries and custom volume rendering engines [S1, S2, S3, S4, S5, S6, unofficial]. Variants and optimizations for handling floating-point edge cases and GPU acceleration are actively researched and implemented in open-source projects [S1, S2, S3, S6, unofficial].

## Sources
[S1] Mike-Leo-Smith/SolidVoxelization, https://github.com/Mike-Leo-Smith/SolidVoxelization, fetched 2026-10-27 (unofficial)
[S2] facebookresearch/NSVF, https://github.com/facebookresearch/NSVF, fetched 2026-10-27 (unofficial)
[S3] whoiszzj/FDS-GS, https://github.com/whoiszzj/FDS-GS, fetched 2026-10-27 (unofficial)
[S4] GitHub - mlavik1/UnityVolumeRendering: Volume rendering, implemented in ..., https://github.com/mlavik1/UnityVolumeRendering, fetched 2026-10-28 (unofficial)
[S5] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S6] taylor2nerf/taylor-nerf, https://github.com/taylor2nerf/taylor-nerf, fetched 2026-10-26 (unofficial)
[S7] Sphere Carving: Bounding Volumes for Signed Distance ..., https://research.adobe.com/publication/sphere-carving-bounding-volumes-for-signed-distance-fields/, fetched 2026-11-05
