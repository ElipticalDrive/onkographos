# Ray-box intersection

Also called: Slab Test, Smits' method, Smits' Ray-Box, AABB Slab Method, Slab intersection, AABB Ray Intersection, Ray-AABB Intersection, Slab intersection test, Ray/Box Intersection Test, Slab-based Ray/Box Intersection

## Overview

The Ray-box intersection method (specifically the Slab Test) computes the parametric entry ($t_{near}$) and exit ($t_{far}$) distances of a ray intersecting an Axis-Aligned Bounding Box (AABB) [S2, S3, S14, S16, unofficial]. It solves the problem of determining the valid sampling interval along a ray for volume rendering or ray tracing, enabling early rejection of rays that miss the volume or primitives [S1, S11, S26, unofficial]. It sets itself apart from [Ray-box intersection via OpenGL Geometry Shaders](1768-ray-box-intersection-via-opengl-geometry-shaders.md) by using an analytical arithmetic approach rather than geometric shader primitives, and from [Ray-Volume Intersection (Face Distance Method)](2381-ray-volume-intersection-face-distance-method.md) by operating on the bounding box of the volume rather than the volume's internal structure directly [S1, S11, unofficial].

## How it works

**Inputs and Outputs**
*   **Input**: Ray origin $\mathbf{o} = (o_x, o_y, o_z)$, ray direction $\mathbf{d} = (d_x, d_y, d_z)$, AABB minimum corner $\mathbf{b}_{min} = (b_{min,x}, b_{min,y}, b_{min,z})$, AABB maximum corner $\mathbf{b}_{max} = (b_{max,x}, b_{max,y}, b_{max,z})$ [S10, S46, S48, unofficial].
*   **Output**: A boolean indicating intersection, and if true, the scalar interval $[t_{near}, t_{far}]$ along the ray parameter $t \ge 0$ [S2, S3, S14, unofficial].

**Data Structures**
*   **Ray**: Defined by origin $\mathbf{o}$ and direction $\mathbf{d}$. The direction is typically normalized but does not strictly need to be for the intersection test itself, provided the inverse is handled correctly [S8, S46, unofficial].
*   **AABB**: Defined by two 3D vectors, $\mathbf{b}_{min}$ and $\mathbf{b}_{max}$, representing the minimum and maximum coordinates along each axis [S48, S58, unofficial].

**Precomputation**
To optimize performance, the inverse ray direction is precomputed to replace division with multiplication [S8, S15, S58, unofficial]:
$$ \mathbf{inv\_d} = \left( \frac{1}{d_x}, \frac{1}{d_y}, \frac{1}{d_z} \right) $$
If a direction component $d_i$ is zero, $\frac{1}{d_i}$ is undefined. Robust implementations handle this by either checking for zero explicitly or using a large constant (e.g., $10^{32}$) or clamping the direction to avoid division by zero [S31, S37, S41, S59, unofficial].

**Algorithm Steps**
The method treats the AABB as the intersection of three pairs of parallel planes (slabs), one pair for each axis [S49, unofficial][S50]. For each axis $i \in \{x, y, z\}$, the ray intersects the slab boundaries at parameters $t_{i1}$ and $t_{i2}$ [S46, S52, unofficial]:

$$ t_{i1} = (b_{min,i} - o_i) \cdot inv\_d_i $$
$$ t_{i2} = (b_{max,i} - o_i) \cdot inv\_d_i $$

1.  **Compute Slab Intersections**: Calculate $t_{i1}$ and $t_{i2}$ for all three axes [S14, unofficial][S27][S51, unofficial].
2.  **Order Slab Intervals**: For each axis, ensure $t_{i1} \le t_{i2}$. If $d_i < 0$, the ray enters at $b_{max,i}$ and exits at $b_{min,i}$, so the values are swapped [S4, S10, S46, unofficial]. Alternatively, compute $t_{min\_axis} = \min(t_{i1}, t_{i2})$ and $t_{max\_axis} = \max(t_{i1}, t_{i2})$ [S23, S49, unofficial].
3.  **Intersect Intervals**: The ray intersects the box if the intersection of all three slab intervals is non-empty.
    *   $t_{near} = \max(t_{min\_x}, t_{min\_y}, t_{min\_z})$
    *   $t_{far} = \min(t_{max\_x}, t_{max\_y}, t_{max\_z})$
    [S15, S23, S55, unofficial].
4.  **Check Validity**: The ray intersects the box if $t_{near} \le t_{far}$ and $t_{far} \ge 0$ [S2, S55, unofficial].
    *   If $t_{near} < 0$ and $t_{far} \ge 0$, the ray origin is inside the box [S4, S10, unofficial].
    *   If $t_{far} < 0$, the box is behind the ray origin [S6, unofficial].

**Pseudocode**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_box_intersection(ray_origin, ray_direction, box_min, box_max):
    """
    Computes the intersection interval [t_near, t_far] of a ray with an AABB.
    
    Args:
        ray_origin: tuple or array of 3 floats (ox, oy, oz)
        ray_direction: tuple or array of 3 floats (dx, dy, dz)
        box_min: tuple or array of 3 floats (bmin_x, bmin_y, bmin_z)
        box_max: tuple or array of 3 floats (bmax_x, bmax_y, bmax_z)
        
    Returns:
        (bool, float, float): (intersects, t_near, t_far)
        If intersects is False, t_near and t_far are undefined or set to (inf, -inf).
    """
    # Precompute inverse direction
    # Handle zero direction components to avoid division by zero
    # [G] Using a large constant or clamping is a common robustness technique.
    # Here we use a small epsilon check for clarity, matching [S2, S6, unofficial].
    EPSILON = 1e-6
    inv_dir = []
    for i in range(3):
        if abs(ray_direction[i]) < EPSILON:
            # Ray is parallel to the slab planes for this axis.
            # We set inv_dir to 0 for calculation, but handle the parallel case separately.
            inv_dir.append(0.0)
        else:
            inv_dir.append(1.0 / ray_direction[i])

    # Initialize t_near to -infinity and t_far to +infinity
    t_near = -float('inf')
    t_far = float('inf')

    # Check for parallel rays and compute intervals
    for i in range(3):
        # Calculate intersection distances with min and max bounds
        # t0 = (box_min[i] - ray_origin[i]) * inv_dir[i]
        # t1 = (box_max[i] - ray_origin[i]) * inv_dir[i]
        
        # If direction is near zero, check if origin is within bounds
        if abs(ray_direction[i]) < EPSILON:
            if ray_origin[i] < box_min[i] or ray_origin[i] > box_max[i]:
                # Ray is parallel and outside the box slab
                return (False, float('inf'), -float('inf'))
            # If inside, the slab imposes no constraint on t, so we skip updating t_near/t_far
            # effectively treating the interval as (-inf, inf) for this axis
            continue

        t0 = (box_min[i] - ray_origin[i]) * inv_dir[i]
        t1 = (box_max[i] - ray_origin[i]) * inv_dir[i]

        # Ensure t0 is the entry and t1 is the exit for this slab
        if t0 > t1:
            t0, t1 = t1, t0

        # Update the global intersection interval
        if t0 > t_near:
            t_near = t0
        if t1 < t_far:
            t_far = t1

        # Early exit if interval is invalid
        if t_near > t_far:
            return (False, float('inf'), -float('inf'))

    # Final check: intersection must be in the positive direction of the ray
    if t_far < 0:
        return (False, float('inf'), -float('inf'))
    
    # If t_near is negative, the ray starts inside the box
    # The valid entry point for traversal is usually max(t_near, 0)
    # But the method returns the full interval [t_near, t_far]
    
    return (True, t_near, t_far)
```

**Parameters and Edge Cases**
*   **Epsilon**: A small value (e.g., $1.0 \times 10^{-6}$) is used to detect parallel rays [S2, S6, unofficial].
*   **Parallel Rays**: If $d_i = 0$, the ray is parallel to the planes perpendicular to axis $i$. If the ray origin is outside $[b_{min,i}, b_{max,i}]$, there is no intersection [S2, S6, S51, unofficial]. If inside, the slab imposes no constraint.
*   **Ray Inside Box**: If $t_{near} < 0$ and $t_{far} \ge 0$, the ray originates inside the box [S4, S10, unofficial].
*   **Precision**: Using precomputed inverse direction avoids division by zero issues if handled correctly, but IEEE floating-point properties (like -0.0) must be considered for robustness [S56].

## Subtopics

### [Plane-AABB Overlap Test](1783-plane-aabb-overlap-test.md)
Determines if an axis-aligned bounding box (AABB) overlaps with a plane defined by a normal vector and a point (or signed distance). It solves the problem of culling voxels or primitives that lie entirely on one side of a cutting plane. It differs from [Ray-box intersection](776-ray-box-intersection.md) by testing a static infinite plane against a static box rather than a parametric ray, and from [AABB Collision Detection](2052-aabb-collision-detection.md) by handling a single infinite half-space rather than two moving boxes.

### [AABB Culling](1789-aabb-culling.md)
AABB Culling determines if a box defined by 8 vertices is completely outside a viewing box along any axis, returning false if culled and true otherwise. It serves as a fast rejection test in volume rendering and ray tracing to avoid expensive intersection calculations for primitives that are clearly outside the view frustum or a specific spatial region. Unlike [Ray-box intersection](776-ray-box-intersection.md), which computes precise entry and exit distances for potential hits, this method performs a coarse binary check to discard objects early.

### [AABB Collision Detection](2052-aabb-collision-detection.md)
AABB Collision Detection is a movement resolution method for voxel worlds that prevents entities (players, cameras, NPCs) from passing through solid blocks by separating movement into individual axis checks. It applies displacement along one axis, tests for intersection with the voxel grid, and commits or cancels that movement before proceeding to the next axis, enabling sliding along walls.

### [Raycasting Model Matrix & AABB Planes](2607-raycasting-model-matrix-aabb-planes.md)
This method prepares the geometric state for ray-box intersection tests in volume raycasting by computing a normalized model matrix and the bounding planes of the volume's axis-aligned bounding box (AABB). It solves the problem of mapping world-space rays into a canonical unit cube space where intersection logic is simplified. It differs from general ray-box intersection methods by explicitly handling the scaling of the volume's physical dimensions to a unit extent and computing the specific plane offsets required for the slab test.

### [2D Axis-Aligned Bounding Box (AABB) Intersection](4027-2d-axis-aligned-bounding-box-aabb-intersection.md)
This method determines whether two 2D axis-aligned bounding boxes (AABBs) overlap. It solves the problem of spatial coherence checks in 2D space, such as tile-based culling or sprite collision detection. It differs from ray-box intersection [Ray-box intersection](776-ray-box-intersection.md) by operating on two static volumes rather than a ray and a volume, and from 3D AABB intersection by restricting the test to the X and Y axes only.

### [AABB Representation (Min/Max)](6310-aabb-representation-min-max.md)
The Min/Max AABB representation defines an axis-aligned bounding box by explicitly storing the minimum and maximum coordinate values along each of the three spatial axes. It treats the box as the intersection of three orthogonal "slabs" (infinite regions bounded by parallel planes). This representation is preferred over point-plus-dimensions (centroid + size) because merging two boxes is a trivial component-wise min/max operation, whereas the latter requires extra arithmetic.

### [Ray-AABB Intersection (Slab/Min-Max)](6548-ray-aabb-intersection-slab-min-max.md)
The Slab/Min-Max method determines if a ray intersects an Axis-Aligned Bounding Box (AABB) by computing the entry ($t_{near}$) and exit ($t_{far}$) distances along the ray parameter $t$. It treats the box as the intersection of three pairs of parallel planes (slabs) [Parent]. It sets itself apart from [Ray-box intersection](776-ray-box-intersection.md) by explicitly providing the scalar interval $[t_{min}, t_{max}]$ required for subsequent volume sampling or traversal, rather than just a boolean hit test.

### [Ray-OBB Intersection Test](6603-ray-obb-intersection-test.md)
The Ray-OBB Intersection Test determines if a ray intersects an Oriented Bounding Box (OBB) and computes the distance to the nearest intersection point. It extends the slab method used for Axis-Aligned Bounding Boxes (AABBs) by projecting the ray onto the OBB's local axes, allowing for efficient intersection testing of rotated and translated boxes. Unlike [Ray-box intersection](776-ray-box-intersection.md) which assumes axis alignment, this method handles arbitrary orientations by transforming the problem into the OBB's local coordinate space implicitly through dot products.

### Annulus Box Bounds Conversion
Converts an inner bounding box to a list of bounds relative to an enclosing box. Adds entries for sides where the inner box is strictly contained.
*   **Step**: Initialize `n_bnds = 0`. Loop `i` from 0 to `dim-1`. If `inner_box.lo[i] > bnd_box.lo[i]`, `n_bnds++`. If `inner_box.hi[i] < bnd_box.hi[i]`, `n_bnds++`. Allocate `bnds = new ANNorthHalfSpace[n_bnds]`. Reset `j = 0`. Loop `i` from 0 to `dim-1`. If `inner_box.lo[i] > bnd_box.lo[i]`, set `bnds[j].cd = i`, `bnds[j].cv = inner_box.lo[i]`, `bnds[j].sd = +1`, `j++`. If `inner_box.hi[i] < bnd_box.hi[i]`, set `bnds[j].cd = i`, `bnds[j].cv = inner_box.hi[i]`, `bnds[j].sd = -1`, `j++` [S25, unofficial].
*   **Layout**: `ANNorthHalfSpace` struct: `cd` (dimension index), `cv` (cutting value), `sd` (sign/direction: +1 or -1) [S25, unofficial].

## Operating characteristics

*   **Time Complexity**: $O(1)$ per ray-box pair. The algorithm performs a constant number of arithmetic operations (subtractions, multiplications, comparisons) regardless of the box size or resolution [S8, S49, unofficial].
*   **Memory Complexity**: $O(1)$ auxiliary space. Only a few scalar variables are needed to store the intersection intervals [S49, unofficial].
*   **Performance**: The slab method is one of the fastest ray-box intersection algorithms due to its simplicity and lack of branching in optimized implementations (e.g., SIMD-friendly versions) [S49, S57, S58, unofficial]. Precomputing the inverse direction reduces computational cost by replacing divisions with multiplications [S8, S58, unofficial].
*   **Strengths**: Fast, robust, easy to implement, works in any dimension [S53, unofficial], suitable for parallel execution on GPUs [S19, S20, unofficial].
*   **Limitations**: Only works for axis-aligned boxes. For oriented boxes, transformation to local space is required, adding overhead [S57, S6603-ray-obb-intersection-test.md, unofficial]. Precision issues can arise with rays parallel to axes or starting exactly on box boundaries, requiring careful handling of epsilon or IEEE floating-point properties [S56][S60, unofficial].
*   **When to use**: As a first-level intersection test in ray tracing and volume rendering to cull rays that do not intersect the bounding volume [S1, S26, unofficial].
*   **When not to use**: When dealing with oriented bounding boxes (use Ray-OBB) or when higher-level geometric primitives (triangles, spheres) are needed directly.

## Background

The slab method for ray-AABB intersection is a fundamental technique in computer graphics, particularly in ray tracing and volume rendering. It is based on the geometric definition of an AABB as the intersection of three slabs (pairs of parallel planes) [S49, unofficial][S50]. The method is often attributed to Kay and Kajiya (1986) or Smits (1995) in various implementations [S5, S10, unofficial][S47]. It is widely used in modern rendering engines and volume visualization tools due to its efficiency and simplicity [S1, S8, S49, unofficial].

## Sources

[S1] Volume Rendering Based On 3D Voxel Grids, https://scratchapixel.com/lessons/3d-basic-rendering/volume-rendering-for-developers/volume-rendering-voxel-grids.html, fetched 2026-10-24 (unofficial)
[S2] hectorpiteau/sparse-voxels, https://github.com/hectorpiteau/sparse-voxels, fetched 2026-10-24 (unofficial)
[S3] Noah-Everett/Fast-Voxel-Traversal-Python, https://github.com/Noah-Everett/Fast-Voxel-Traversal-Python, fetched 2026-10-24 (unofficial)
[S4] newDINO/voxel_ray_demo, https://github.com/newDINO/voxel_ray_demo, fetched 2026-10-24 (unofficial)
[S5] Forceflow/cpu_voxel_raycaster, https://github.com/Forceflow/cpu_voxel_raycaster, fetched 2026-10-24 (unofficial)
[S6] RMonica/basic_next_best_view, https://github.com/RMonica/basic_next_best_view, fetched 2026-10-24 (unofficial)
[S7] Nelarius/vxray, https://github.com/Nelarius/vxray, fetched 2026-10-24 (unofficial)
[S8] This Tiny Algorithm Can Render BILLIONS of Voxels in Real Time, https://www.youtube.com/watch?v=ztkh1r1ioZo, fetched 2026-10-25 (unofficial)
[S9] Raycasting-basiertes Voxelrendering, https://serwiss.bib.hs-hannover.de/frontdoor/index/index/docId/2250, fetched 2026-10-25 (unofficial)
[S10] Estudio de la compresión y la eficiencia del Sparse Voxel Octree en el cálculo de la Iluminación Global, https://hdl.handle.net/2117/445971, fetched 2026-10-26 (unofficial)
[S11] Volume Raycasting mit OpenCL, https://arxiv.org/abs/1609.01317v1, fetched 2026-10-26 (unofficial)
[S12] Show HN: Volume rendering 3D data in Three.js and GLSL, https://github.com/SuboptimalEng/volume-rendering, fetched 2026-10-26 (unofficial)
[S13] kkhese/Volume-Rendering, https://github.com/kkhese/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S14] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
[S15] thoniorf/volumeraycasting, https://github.com/thoniorf/volumeraycasting, fetched 2026-10-26 (unofficial)
[S16] ShashwatC/Ray, https://github.com/ShashwatC/Ray, fetched 2026-10-26 (unofficial)
[S17] TakTikFR/Volumetric-Path-Tracing, https://github.com/TakTikFR/Volumetric-Path-Tracing, fetched 2026-10-26 (unofficial)
[S18] hexhowells/GoTracer, https://github.com/hexhowells/GoTracer, fetched 2026-10-26 (unofficial)
[S19] nerfstudio-project/nerfacc, https://github.com/nerfstudio-project/nerfacc, fetched 2026-10-26 (unofficial)
[S20] Fictionarry/ER-NeRF, https://github.com/Fictionarry/ER-NeRF, fetched 2026-10-26 (unofficial)
[S21] indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, https://github.com/indira654/implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting, fetched 2026-10-26 (unofficial)
[S22] vulekhanh/VolumeRendering, https://github.com/vulekhanh/VolumeRendering, fetched 2026-10-26 (unofficial)
[S23] tzhazuma/Fluid_simulation, https://github.com/tzhazuma/Fluid_simulation, fetched 2026-10-26 (unofficial)
[S24] KaanAres10/Volume-Rendering, https://github.com/KaanAres10/Volume-Rendering, fetched 2026-10-26 (unofficial)
[S25] bouguerra-khalil/DualContouring-3DRendring, https://github.com/bouguerra-khalil/DualContouring-3DRendring, fetched 2026-10-26 (unofficial)
[S26] GitHub - narnia-ai-mason/iso-surface-rendering: Robust iso-surface ..., https://github.com/narnia-ai-mason/iso-surface-rendering, fetched 2026-10-26 (unofficial)
[S27] (PDF) A Comparison between GPU-based Volume Ray Casting ..., https://www.researchgate.net/publication/321454253_A_Comparison_between_GPU-based_Volume_Ray_Casting_Implementations_Fragment_Shader_Compute_Shader_OpenCL_and_CUDA, fetched 2026-10-26
[S28] szat/Jax-Volumetric-Renderer, https://github.com/szat/Jax-Volumetric-Renderer, fetched 2026-10-27 (unofficial)
[S29] M4trixStrike/WebGL-Volumetric-clouds, https://github.com/M4trixStrike/WebGL-Volumetric-clouds, fetched 2026-10-27 (unofficial)
[S30] itsyouyi/Voxelizer, https://github.com/itsyouyi/Voxelizer, fetched 2026-10-27 (unofficial)
[S31] an1pse/voxel-traversal, https://github.com/an1pse/voxel-traversal, fetched 2026-10-27 (unofficial)
[S32] hopexn/VolumeRenderingParallel, https://github.com/hopexn/VolumeRenderingParallel, fetched 2026-10-27 (unofficial)
[S33] GitHub - clayjohn/voxel-renderer: A short experiment in high ..., https://github.com/clayjohn/voxel-renderer, fetched 2026-10-27 (unofficial)
[S34] ldarnbr/volumetric-cloud-renderer, https://github.com/ldarnbr/volumetric-cloud-renderer, fetched 2026-10-27 (unofficial)
[S35] Linyou/taichi-ngp-renderer, https://github.com/Linyou/taichi-ngp-renderer, fetched 2026-10-27 (unofficial)
[S36] megvii-research/cv-master-ex, https://github.com/megvii-research/cv-master-ex, fetched 2026-10-27 (unofficial)
[S37] blurgyy/jaxngp, https://github.com/blurgyy/jaxngp, fetched 2026-10-27 (unofficial)
[S38] NeRF-Factory/IDDR-NGP, https://github.com/NeRF-Factory/IDDR-NGP, fetched 2026-10-27 (unofficial)
[S39] InanXR/FLASH, https://github.com/InanXR/FLASH, fetched 2026-10-27 (unofficial)
[S40] HaouchatY/differentiable_xrt, https://github.com/HaouchatY/differentiable_xrt, fetched 2026-10-27 (unofficial)
[S41] AlanHsu314/CoroNeRF_public, https://github.com/AlanHsu314/CoroNeRF_public, fetched 2026-10-27 (unofficial)
[S42] cgibson/Mr-Voxel, https://github.com/cgibson/Mr-Voxel, fetched 2026-10-27 (unofficial)
[S43] Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, https://github.com/Loveof1ife7/cuVRender-A-CUDA-Accelerated-Volume-Rendering-Engine, fetched 2026-10-27 (unofficial)
[S44] SRaimondi/VolumeRender, https://github.com/SRaimondi/VolumeRender, fetched 2026-10-27 (unofficial)
[S45] GitHub - m-pilia/volume-raycasting: Simple visualiser based on GPU ..., https://github.com/m-pilia/volume-raycasting, fetched 2026-10-27 (unofficial)
[S46] Ray‐Box Intersection Using the Slab Method | Every Algorithm, https://every-algorithm.github.io/2024/11/09/slab_method.html, fetched 2026-10-29 (unofficial)
[S47] Ray - Box Intersection - ACM SIGGRAPH, https://education.siggraph.org/static/HyperGraph/raytrace/rtinter3.htm, fetched 2026-10-29
[S48] (PDF) Ray Axis-Aligned Bounding Box Intersection - ResearchGate, https://www.researchgate.net/publication/354064987_Ray_Axis-Aligned_Bounding_Box_Intersection, fetched 2026-10-30 (unofficial)
[S49] Fast, Branchless Ray/Bounding Box Intersections - tavianator.com, https://tavianator.com/2011/ray_box.html, fetched 2026-10-29 (unofficial)
[S50] RAY AXIS-ALIGNED BOUNDING BOX INTERSECTION, https://link.springer.com/content/pdf/10.1007/978-1-4842-7185-8_2.pdf, fetched 2026-10-29
[S51] Optimize STL ray-box intersection, https://github.com/AMReX-Codes/amrex/pull/5609, fetched 2026-10-29 (unofficial)
[S52] A Couple 3D AABB Tricks, https://gpfault.net/posts/aabb-tricks.html, fetched 2026-10-29 (unofficial)
[S53] GitHub - stackgl/ray-aabb-intersection: Determine the point of ..., https://github.com/stackgl/ray-aabb-intersection, fetched 2026-10-29 (unofficial)
[S54] intersect: AABB Ray Intersection Algorithm - glre.dev, https://glre.dev/addons/geometry/aabb/intersect/, fetched 2026-10-29
[S55] Ray-AABB (Axis Aligned Bounding Box) intersection. · GitHub, https://gist.github.com/DomNomNom/46bb1ce47f68d255fd5d, fetched 2026-10-29 (unofficial)
[S56] PDF An Efficient and Robust Ray-Box Intersection Algorithm, https://people.csail.mit.edu/amy/papers/box-jgt.pdf, fetched 2026-10-29
[S57] (PDF) Axis‐Normalized Ray‐Box Intersection - ResearchGate, https://www.researchgate.net/publication/390921688_Axis-Normalized_Ray-Box_Intersection, fetched 2026-10-30 (unofficial)
[S58] (PDF) An Efficient and Robust Ray-Box Intersection Algorithm, https://www.researchgate.net/publication/220494140_An_Efficient_and_Robust_Ray-Box_Intersection_Algorithm, fetched 2026-11-01 (unofficial)
[S59] GitHub - yuehaowang/volume_renderer: CUDA-based interactive volume ..., https://github.com/yuehaowang/volume_renderer, fetched 2026-10-29 (unofficial)
[S60] http://www.flipcode.com/archives/SSE_RayBox_Intersection_Test.shtml, http://www.flipcode.com/archives/SSE_RayBox_Intersection_Test.shtml, fetched 2026-10-29 (unofficial)
[S61] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
[S62] GitHub - saurlax/model_voxelizer: A 3D model voxelization tool that ..., https://github.com/saurlax/model_voxelizer, fetched 2026-10-30 (unofficial)
