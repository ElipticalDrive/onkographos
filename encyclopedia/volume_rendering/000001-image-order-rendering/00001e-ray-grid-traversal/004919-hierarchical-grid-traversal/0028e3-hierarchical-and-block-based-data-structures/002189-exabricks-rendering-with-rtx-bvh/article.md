# ExaBricks Rendering with RTX BVH

Also called: Ray tracing structured AMR data using ExaBricks

## Overview
ExaBricks Rendering with RTX BVH is a method for rendering Adaptive Block-Refined (ABR) volume data that utilizes hardware-accelerated Bounding Volume Hierarchy (BVH) traversal to iterate through data regions, performing volume integration within each region while skipping empty space [S1]. It is distinguished from sibling methods by its direct reliance on RTX hardware acceleration for traversal, avoiding non-trivial control flow in shaders [S1]. Unlike [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) which may use software-based logic for region boundaries, or [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md) which focuses on sparse data structures, this method explicitly leverages RTX ray tracing cores for efficient space skipping [S1].

## How it works
**Inputs:**
*   A set of Adaptive Block-Refined (ABR) regions derived from structured AMR data [S1].
*   A transfer function mapping scalar values to color and opacity [S1].
*   A ray origin $\mathbf{o}$ and direction $\mathbf{d}$ for each pixel/sample [G].
*   A maximum traversal distance $t_{max}$ [S1].

**Outputs:**
*   A final color and opacity value for each ray [G].

**Data Structures:**
*   **ABR Region:** A spatial region containing a block of volume data [S1]. Each region has a precomputed scalar range $[min, max]$ and a bounding box [S1].
*   **RTX BVH:** A Bounding Volume Hierarchy constructed from the ABR regions [S1]. Each leaf node corresponds to one ABR region [S1]. The BVH is built using NVIDIA RTX hardware acceleration primitives [S1].

**Steps:**

1.  **BVH Construction (Preprocessing):**
    *   For each ABR region, compute the maximum opacity $\alpha_{max}$ achievable by the transfer function over the region's scalar range $[min, max]$ [S1].
    *   If $\alpha_{max} = 0$ (fully transparent), discard the region from the BVH by returning an empty bounding box or excluding it from the primitive list [S1].
    *   If $\alpha_{max} > 0$, include the region's bounding box in the BVH construction [S1].
    *   Build the BVH using the RTX acceleration structure builder [S1].

2.  **Ray Initialization:**
    *   For each ray, initialize the search interval $[0, t_{max})$ [S1].

3.  **Hardware Traversal Loop:**
    *   Trace the ray through the RTX BVH in hardware to find the first intersected region [S1].
    *   If no region is intersected, the ray terminates [S1].
    *   If a region is intersected, compute the overlap interval $[t_{in}, t_{out}]$ between the ray and the region's bounding box [S1].

4.  **Volume Integration:**
    *   Perform volume integration over the interval $[t_{in}, t_{out}]$ using the samples within the intersected ABR region [S1].

5.  **Early Termination Check:**
    *   If the accumulated opacity reaches 1.0 (fully opaque), terminate the ray [S1].

6.  **Next Region Search:**
    *   If the ray is not opaque, trace a new ray starting from $t_{out} + \epsilon$ (where $\epsilon$ is a small bias to prevent self-intersection) to find the next ABR region [S1].
    *   Repeat steps 3-6 until no subsequent region is found or the ray becomes opaque [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_exabricks_ray(ray_origin, ray_direction, t_max, bvh, transfer_function, region_data):
    """
    Renders a single ray using ExaBricks with RTX BVH.
    
    Args:
        ray_origin: vec3, origin of the ray.
        ray_direction: vec3, normalized direction of the ray.
        t_max: float, maximum distance to trace.
        bvh: RTXAccelerationStructure, the BVH of ABR regions.
        transfer_function: callable, maps scalar value to (color, opacity).
        region_data: dict, maps region_id to region metadata (bbox, sampling routine).
        
    Returns:
        (color, opacity): The final accumulated color and opacity.
    """
    accumulated_color = (0.0, 0.0, 0.0)
    accumulated_opacity = 0.0
    t_current = 0.0
    
    while t_current < t_max and accumulated_opacity < 1.0:
        # Trace ray through RTX BVH starting from t_current
        hit = bvh.intersect(ray_origin, ray_direction, t_current, t_max)
        
        if hit is None:
            break
            
        t_hit, region_id = hit
        
        # Compute overlap interval [t_in, t_out] for the region's bounding box
        region_bbox = region_data[region_id].bbox
        t_min, t_max_bbox = intersect_aabb(ray_origin, ray_direction, region_bbox)
        
        # Skip if ray misses the bounding box behind the camera or in negative direction
        if t_max_bbox < 0:
            t_current = t_hit + 1e-6
            continue
            
        t_in = max(t_min, t_current)
        t_out = t_max_bbox
        
        # Volume integrate over [t_in, t_out]
        color_contribution, opacity_contribution = volume_integrate(
            ray_origin, ray_direction, t_in, t_out, region_id, transfer_function
        )
        
        # Update accumulated color and opacity using standard discrete compositing [G]
        accumulated_color = tuple(c1 + c2 * (1.0 - accumulated_opacity) for c1, c2 in zip(accumulated_color, color_contribution))
        accumulated_opacity += opacity_contribution * (1.0 - accumulated_opacity)
        
        # Advance ray start point for next iteration
        t_current = t_out + 1e-6
        
    return accumulated_color, accumulated_opacity

def intersect_aabb(origin, direction, bbox):
    """
    Computes entry (t_min) and exit (t_max) distances of a ray 
    against an axis-aligned bounding box using the slab method.
    """
    t_min = -float('inf')
    t_max = float('inf')
    
    for i in range(3):
        if direction[i] == 0.0:
            if origin[i] < bbox.min[i] or origin[i] > bbox.max[i]:
                return -1.0, -1.0
        else:
            t1 = (bbox.min[i] - origin[i]) / direction[i]
            t2 = (bbox.max[i] - origin[i]) / direction[i]
            t_min = max(t_min, min(t1, t2))
            t_max = min(t_max, max(t1, t2))
            
    return t_min, t_max

def volume_integrate(origin, direction, t_in, t_out, region_id, transfer_function):
    """
    Performs volume integration over the interval [t_in, t_out] for a specific region.
    Uses standard discrete ray marching compositing [G].
    """
    color = (0.0, 0.0, 0.0)
    opacity = 0.0
    
    # Sample the region's scalar field along the ray segment [t_in, t_out]
    samples = region_data[region_id].sample_ray(origin, direction, t_in, t_out)
    
    for scalar in samples:
        c, a = transfer_function(scalar)
        color = tuple(c1 + c2 * a * (1.0 - opacity) for c1, c2 in zip(color, c))
        opacity += a * (1.0 - opacity)
        if opacity >= 1.0:
            break
            
    return color, opacity
```

**Parameters:**
*   $t_{max}$: Maximum ray traversal distance. Typical value: scene extent [S1].
*   $\epsilon$: Small bias for next ray start. Typical value: $10^{-6}$ [S1].

**Edge Cases and Pitfalls:**
*   **BVH Rebuild Cost:** The BVH must be rebuilt when the transfer function or iso-value changes, as regions are pruned based on maximum opacity [S1]. On the Exajet dataset, a full rebuild takes approximately 300ms on a Titan RTX or RTX 8000 GPU [S1].
*   **Transfer Function Changes:** Any change in the transfer function that affects the opacity of any region requires a BVH rebuild or refit [S1].
*   **Self-Intersection:** A small epsilon must be added to $t_{out}$ to prevent the next ray from intersecting the same region again [S1].

## Subtopics

### [RTX BVH Ray Tracing](2957-rtx-bvh-ray-tracing.md)
RTX BVH Ray Tracing is a method for volume rendering that utilizes hardware-accelerated Bounding Volume Hierarchy (BVH) traversal to iterate through data regions, performing volume integration within each region while skipping empty space. It is distinguished from sibling methods by its reliance on NVIDIA RTX hardware acceleration for both traversal and shading, enabling interactive transfer function and iso-surface editing alongside adaptive sampling. Unlike [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) which may use software-based logic for region boundaries, or [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md) which focuses on sparse data structures, this method explicitly leverages the RTX ray tracing cores for efficient space skipping and adaptive sample density control.

### [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md)
Adaptive Sampling with Region Delimiters is a sampling strategy for Adaptive Block-Refined (ABR) volume data that adapts the sampling rate to the finest cell size within each region, using sample positions as delimiters to handle boundary artifacts. Unlike Ljung [23], which defines a global set of finest level samples skipped by coarseness, this method defines independent sampling intervals for each region. It solves the problem of boundary artifacts caused by successive regions sampled at different rates, where the distance between the last sample of one region and the first of the next can be smaller or larger than either region's sampling rate.

### [BVH Pruning for Space Skipping](4772-bvh-pruning-for-space-skipping.md)
BVH Pruning for Space Skipping is a technique that accelerates volume rendering by excluding fully transparent regions from the Bounding Volume Hierarchy (BVH) during construction, thereby allowing hardware ray traversal to skip empty space and non-visible data without runtime checks. It differs from sibling methods like [Adaptive Sampling with Region Delimiters](3421-adaptive-sampling-with-region-delimiters.md) by integrating visibility culling directly into the BVH structure rather than managing traversal state, and from [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md) by relying on transfer-function-aware pruning rather than static sparsity masks.

### [Sparse Brick Set Rendering](9909-sparse-brick-set-rendering.md)
Sparse Brick Set Rendering is a method for rendering Signed Distance Fields (SDFs) using hardware-accelerated raytracing against a Bounding Volume Hierarchy (BVH) of leaf bricks, where each brick contains SDF data and is intersected via software shaders and sphere-tracing. It differs from its parent [ExaBricks Rendering with RTX BVH](2577-exabricks-rendering-with-rtx-bvh.md) by specifically utilizing SDF sphere-tracing within leaf nodes rather than general volume integration, and from [RTX BVH Ray Tracing](2957-rtx-bvh-ray-tracing.md) by relying on software intersection shaders for the SDF geometry rather than hardware-native triangle or primitive intersection.

## Operating characteristics
*   **Time Complexity:** The traversal cost is proportional to the number of intersected regions [S1]. Each region requires a separate hardware ray traversal, which is amortized over multiple samples [S1]. The BVH construction cost is non-trivial and must be paid when the transfer function changes [S1].
*   **Memory Complexity:** The BVH stores one primitive per ABR region [S1]. The memory overhead is low compared to the volume data itself [S1].
*   **Measured Performance:** On the Exajet dataset (656M cells, 55.8M regions), a full BVH rebuild takes approximately 300ms on a Titan RTX or RTX 8000 GPU [S1].
*   **Strengths:** Hardware-accelerated traversal avoids non-trivial control flow in shaders, which was a limitation of previous software k-d tree approaches [S1]. Space skipping is efficient as transparent regions are excluded from the BVH entirely [S1].
*   **Limitations:** The BVH must be rebuilt when the transfer function or iso-value changes, which can be costly for interactive applications [S1].
*   **When to Use:** Suitable for rendering large-scale structured AMR data where hardware ray tracing is available and interactive transfer function editing is required [S1].
*   **When Not to Use:** Not suitable for scenarios where the transfer function changes frequently and BVH rebuild cost is prohibitive, or on hardware without RTX support [S1].

## Background
The method was introduced in the paper "Ray Tracing Structured AMR Data Using ExaBricks" [S1]. It addresses the limitations of previous software-based k-d tree traversals for AMR data, such as those by Kähler et al. [17], which required per-thread stacks and complex control flow in shaders [S1]. By leveraging NVIDIA RTX hardware, the method simplifies the shader logic and improves performance [S1]. The approach is part of the ExaBricks framework for rendering adaptive block-refined data [S1].

## Sources
[S1] PDF Ray Tracing Structured AMR Data Using ExaBricks, https://www.sci.utah.edu/~wald/Publications/2020/exabrick/exabrick.pdf, fetched 2026-10-30
