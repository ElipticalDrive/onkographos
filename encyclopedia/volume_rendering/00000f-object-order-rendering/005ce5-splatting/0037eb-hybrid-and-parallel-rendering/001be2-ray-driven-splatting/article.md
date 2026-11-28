# Ray-Driven Splatting

Also called: Hybrid Splatting/Ray Casting, Muller Yagel hybrid, Muller and Yagel hybrid, Ray-driven volume splatting

## Overview
Ray-driven splatting is a hybrid volume rendering technique that combines object-order footprint precomputation with ray-order accumulation [S1, unofficial][S2, unofficial]. It solves the performance degradation of traditional voxel-driven splatting under perspective projection by suspending splats in object space and intersecting them via spawned rays [S2, unofficial]. Unlike traditional splatting, which struggles with perspective complexity and lacks standard ray-tracing optimizations, this method enables space leaping, early ray termination, and adaptive sampling [S1, unofficial][S2, unofficial].

## How it works
**Inputs:** Volumetric dataset (voxels with scalar values); camera parameters (viewing transform, projection type); splat kernel definition (footprint shape) [S1, unofficial][S2, unofficial].
**Outputs:** Image plane pixels (color and opacity) [S1, unofficial][S2, unofficial].

**Data Structures:**
*   **Voxel Grid:** 3D array of scalar values [S1, unofficial].
*   **Splat Representation:** Each voxel is associated with a spatial footprint (splat) suspended in object space [S2, unofficial].

**Steps:**
1.  **Precompute Splat Extents:** For each voxel, determine the spatial extent of its contribution (footprint) in object space [S1, unofficial].
2.  **Suspend Splats:** Imagine splats as being suspended in object space, with one splat at every voxel [S2, unofficial].
3.  **Spawn Rays:** For each pixel in the image plane, spawn a viewing ray to traverse the space [S2, unofficial].
4.  **Intersect and Address:** Intersect and address the splats using an efficient and accurate method [S2, unofficial].
5.  **Accumulate:** Accumulate pixels by shooting rays intersecting the splats in space, similar to ray casting [S1, unofficial].

**Formulas:** Not given in the sources. [G] Standard ray-splat intersection uses the parametric ray equation $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$ and a splat implicit surface $f(\mathbf{p}) \leq 0$.

**Parameters:** Splat kernel shape and falloff (e.g., Gaussian, box) [S1, unofficial]. Ray traversal increment or step size [S2, unofficial].

**Edge Cases and Pitfalls:** Traditional splatting requires complex screen-space transformations for perspective views, but ray-driven splatting avoids this by working in object space, making it inherently insensitive to perspective transform complexity [S2, unofficial]. Intersection accuracy is critical, as the method relies on an "efficient and accurate way of intersecting and addressing the splats" [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_driven_splatting(volume, camera, kernel):
    image = zeros((camera.height, camera.width, 3))
    splats = [create_splat(voxel_pos, kernel) for voxel_pos in volume.grid]
    for y in range(camera.height):
        for x in range(camera.width):
            ray_o, ray_d = camera.get_ray(x, y)
            acc_color = 0.0
            acc_opacity = 0.0
            for splat in splats:
                if ray_intersects_splat(ray_o, ray_d, splat):
                    contrib = evaluate_kernel(splat, ray_o, ray_d)
                    acc_color += contrib
                    acc_opacity += 1.0
            image[y, x] = acc_color
    return image
```

## Subtopics

### [Splat Rasterization via Raycasting](1437-splat-rasterization-via-raycasting.md)
This method rasterizes Ellipsoid-Wise Adaptive (EWA) splats on general-purpose graphics hardware by treating point primitives as screen-space squares and casting a viewing ray for each fragment to test inclusion within the splat's volume. It distinguishes itself from polygon-based bounding approximations by providing a correct upper bound to splat extents that remains stable for large splats near the near plane, while avoiding numerical instabilities of explicit conic centralization or matrix inversions. Choose it when hardware-accelerated fragment testing is preferred over CPU-side ray traversal.

## Operating characteristics
*   **Time Complexity:** Not explicitly given in the sources. Measured performance was 30 seconds for the MRbrain dataset rendered with a 260^2 viewport on an SGI Indigo with 200 MHz [S1, unofficial].
*   **Memory Complexity:** Not explicitly given in the sources. Requires storage for the voxel grid and splat representations [S1, unofficial][S2, unofficial].
*   **Strengths:** Inherently insensitive to the complexity of the perspective viewing transform [S2, unofficial]. Supports acceleration methods such as early ray termination and bounding volumes, which traditional voxel-driven splatting cannot benefit from [S2, unofficial]. Enables the use of other ray tracing optimization techniques such as space leaping, adaptive screen sampling, or spatial acceleration structures [S1, unofficial]. Offers competitive or superior performance for parallel projection, and superior performance for perspective projection compared to traditional splatting [S2, unofficial].
*   **Limitations:** Requires efficient ray-splat intersection logic [S2, unofficial]. Performance on older hardware was slow (30 seconds for a small viewport) [S1, unofficial].
*   **When to use:** When perspective projection is required and traditional splatting performance is insufficient, or when ray-tracing optimizations are desired [S1, unofficial][S2, unofficial].
*   **When not to use:** When hardware acceleration for ray-splat intersection is unavailable and performance is critical, or when scene complexity is low enough that traditional splatting suffices [S2, unofficial].

## Background
Ray-driven splatting was proposed by Müller and Yagel [S1, unofficial]. It was introduced as a hybrid method to address the limitations of traditional voxel-driven splatting, particularly under perspective projection [S1, unofficial]. The method merges the technique of splatting with principles of raycasting [S2, unofficial]. It is related to earlier work on footprint evaluation for volume rendering by Westover [S2, unofficial], which established the forward mapping approach of distributing energy from input samples into space.

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) EWA volume splatting - ResearchGate, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
