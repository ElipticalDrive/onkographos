# Multi-Hit BVH Traversal

Also called: Multi-Hit BVH Traversal for Volume Clipping, Multi-hit intersection, Multi-hit query, Ray Traced Volume Clipping Using Multi-Hit BVH Traversal

## Overview
Multi-Hit BVH Traversal is a SIMD-friendly clipping algorithm for ray-traced direct volume rendering (DVR) that identifies visible volume regions by intersecting viewing rays with arbitrary geometric surface primitives (planes, quadrics, triangle meshes) organized in a Bounding Volume Hierarchy (BVH) [S1]. It sets itself apart from rasterization-based depth peeling or texture-based slicing approaches by implementing clipping as a simple two-pass algorithm that avoids multiple render passes, dynamic memory reallocation, and the conversion of parametric surfaces to polygon meshes [S1].

## How it works
The method determines the $N$ closest intersections of a viewing ray with compound clip objects to construct clip intervals for volume rendering. It operates in two passes: first, intersecting primary viewing rays with all opaque geometry to identify visible volume regions; second, casting primary rays through the volume density and considering only voxels within those visible regions [S1].

### Inputs and Outputs
*   **Inputs:**
    *   Primary viewing rays defined by origin $\mathbf{o}$ and direction $\mathbf{d}$ [S1].
    *   Clip geometry primitives (planes, spheres, triangle meshes) stored as Plain Old Data (POD) objects in a C array [S1].
    *   A BVH acceleration structure managing the triangle meshes [S1].
    *   $N$: The maximum number of intersections to retrieve, decided a priori [S1].
*   **Outputs:**
    *   A static array of size $N$ containing the $N$ closest intersection distances (or hit records) [S1].
    *   Clip intervals derived from these intersections, used to mask volume rendering in the second pass [S1].

### Data Structures
*   **Clip Geometry:** Primitives are stored as POD objects in a C array. Triangle meshes are managed by a BVH. Planes and spheres are handled via generic routines [S1].
*   **BVH:** A hierarchical spatial subdivision structure used to reduce ray/primitive interactions. The BVH is rebuilt completely if triangle meshes change, as dynamic updates are not supported in this implementation [S1].
*   **Hit Buffer:** A static memory array of size $N$ preallocated to store the result of the multi-hit query. This avoids the costly dynamic memory reallocation associated with "all-hit" queries [S1].

### Algorithm Steps
1.  **Preprocessing:** Construct the BVH for the triangle mesh clip geometry. If the mesh changes, rebuild the BVH completely [S1].
2.  **Pass 1 (Clipping):** For each primary viewing ray:
    a. Execute a multi-hit query on the BVH to find the $N$ closest intersections with the clip geometry [S1].
    b. Store the results in the preallocated static array of size $N$ [S1].
    c. Process the hit records to construct clip intervals (visible and invisible regions) [S1].
3.  **Pass 2 (Volume Rendering):** For each primary viewing ray:
    a. Cast the ray through the volume density [S1].
    b. Integrate color and opacity only for voxels that fall within the visible regions identified in Pass 1 [S1].
    c. Skip voxels that are clipped (i.e., inside the geometry or in occluded regions) [S1].

### Formulas and Parameters
*   **Ray-Primitive Intersection:** The algorithm uses generic routines that call specific `intersect()` functions for planes, spheres, and triangles. To avoid dynamic branching in the innermost traversal loop, the primitive type is determined without virtual inheritance or vtable lookups [S1].
*   **Parameter $N$:** The maximum number of intersections. This value must be chosen to balance the complexity of the clip geometry and the available register space on the GPU [S1].
*   **Clip Intervals:** Derived from the sorted intersection distances $t_1, t_2, \dots, t_N$. The visible regions are typically the intervals between $t_{2i}$ and $t_{2i+1}$ (assuming the ray starts outside the geometry) [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Preprocessing: Build BVH for triangle meshes
# Note: BVH is rebuilt if triangle meshes change.
bvh = build_bvh(triangle_meshes)

# Pass 1: Determine Clip Intervals
# For each pixel (ray o, d):
#   hits = multi_hit_query(bvh, o, d, max_hits=N)
#   # hits is a static array of size N, sorted by distance t
#   clip_intervals = construct_intervals(hits)

# Pass 2: Volume Rendering
# For each pixel (ray o, d):
#   color = vec3(0.0)
#   alpha = 0.0
#   t = 0.0
#   while t < max_distance:
#       voxel = sample_volume(o + t * d)
#       if voxel in clip_intervals:
#           # Standard volume integration step
#           color, alpha = integrate(voxel, color, alpha)
#       t += step_size
#   display(color, alpha)
```

### Edge Cases and Pitfalls
*   **Dynamic Branching:** Switching over primitive types in the innermost traversal loop causes performance degradation. The implementation avoids this by using generic routines that call specific `intersect()` functions [S1].
*   **Register Pressure:** Storing clip intervals in on-chip GPU memory increases register demand per thread on the shading multi-processor (SM), which may impact kernel performance [S1].
*   **Static Geometry:** The implementation does not support dynamic updates to triangle meshes. Changing the mesh requires a complete BVH rebuild [S1].
*   **Precision:** Sub-voxel accurate clipping is achieved, avoiding artifacts associated with rasterization-based clipping of irregular concavities [S1].

## Subtopics

### [Multi-hit Collection via OptiX anyhit](19781-multi-hit-collection-via-optix-anyhit.md)
A technique to process all intersections along a ray for volume rendering on a tetrahedral mesh by modifying a payload in the anyhit shader and collecting results in the raygen program, avoiding the overhead of multiple `optixTrace` calls. It sets itself apart from multiple closehit calls by shaving off per-ray-cast overhead at the cost of memory bandwidth for storing hits and explicit sorting.

## Operating characteristics
*   **Time Complexity:** The time complexity of ray tracing grows with the product of the number of primitives and the number of image pixels. The use of BVH reduces the asymptotic behavior to average logarithmic in the number of geometric primitives [S1].
*   **Memory Complexity:** Uses a static array of size $N$ for hit storage, avoiding dynamic memory allocation [S1].
*   **Performance:** Evaluated on a 1024x1024 resolution MRI dataset (MNI152, 0.5mm³ voxel size, 364x436x364 voxels) clipped with Utah teapot models [S1].
    *   Level 1: 2,464 triangles, 1,559 BVH nodes [S1].
    *   Level 2: 14,784 triangles, 9,771 BVH nodes [S1].
    *   Level 3: 59,136 triangles, 39,063 BVH nodes [S1].
    *   Level 4: 236,544 triangles, 151,637 BVH nodes [S1].
*   **Strengths:** Sub-voxel accurate clipping for arbitrary geometry; no multiple render passes; no conversion of parametric surfaces to polygons; real-time capable on GPU [S1].
*   **Limitations:** Static triangle meshes only (BVH rebuild required for changes); increased register pressure on GPU; requires a priori knowledge of maximum intersections $N$ [S1].
*   **When to Use:** Neurosurgical visualization, tumor border inspection, vessel detection, and intersection analysis with surgical instrumentation or fiber tracks [S1].
*   **When Not to Use:** When dynamic mesh deformation is required; when memory bandwidth is severely constrained and $N$ is large; when sub-voxel accuracy is not necessary and rasterization suffices [S1].

## Background
The method was presented in the paper "Ray Traced Volume Clipping Using Multi-Hit BVH Traversal" [S1]. It addresses the limitations of traditional rasterization pipelines, which require multiple render passes to peel away depth layers for translucent surfaces or complex clipping, and texture-based approaches that require multiple rendering passes or storing clip intervals in GPU DDR3 memory [S1]. The implementation uses Visionaray's multi-hit implementation with a template parameter for the maximum size of the static array [S1].

## Sources
[S1] Ray Traced Volume Clipping Using Multi-Hit BVH Traversal, https://doi.org/10.2352/issn.2470-1173.2017.1.vda-392, fetched 2026-10-24
