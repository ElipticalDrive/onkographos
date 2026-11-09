# Ray-Driven Splatting

Also called: Hybrid Splatting/Ray Casting, Muller Yagel hybrid, Muller and Yagel hybrid, Ray-driven volume splatting

## Overview
Ray-driven splatting is a hybrid volume rendering technique that combines the object-order footprint calculation of splatting with the ray-order accumulation of ray casting [S1, unofficial][S2, unofficial]. It solves the performance bottleneck of perspective projection in traditional voxel-driven splatting, where incremental mapping of footprints becomes complex and slow [S2, unofficial]. Unlike standard splatting, which projects voxels to the screen and composites them, ray-driven splatting spawns rays from the camera and intersects them with splats suspended in object space [S2, unofficial]. This approach is inherently insensitive to the complexity of the perspective viewing transform and enables the use of ray-tracing optimizations such as early ray termination and spatial acceleration structures [S1, unofficial][S2, unofficial].

## How it works
**Inputs:**
*   Volumetric dataset (voxels with scalar values).
*   Camera parameters (position, orientation, projection type).
*   Rendering resolution (viewport dimensions).

**Outputs:**
*   Final image (color and opacity per pixel).

**Data Structures:**
*   **Splat Grid:** A 3D grid where each voxel contains a precomputed "splat" primitive. The splat represents the spatial footprint (footprint function) of the voxel's contribution, typically defined by a reconstruction kernel [S1, unofficial][S2, unofficial].
*   **Ray Buffer:** An array of rays, one per pixel, defined by origin and direction.

**Method Steps:**
1.  **Splat Suspension:** Imagine splats as being suspended in object space, with one splat at every voxel location [S2, unofficial]. Each splat is defined by its center (voxel position) and its spatial extent (footprint) [S1, unofficial].
2.  **Ray Spawning:** For each pixel in the viewport, spawn a ray from the camera origin through the pixel center [S2, unofficial].
3.  **Ray-Splat Intersection:** Traverse the space along the ray to find intersections with the suspended splats [S2, unofficial]. This step requires an efficient and accurate method to intersect and address the splats [S2, unofficial].
    *   *Note:* The sources describe the concept of intersecting rays with splats but do not provide a specific mathematical formula for the intersection test (e.g., sphere, ellipsoid, or box intersection). [G] In practice, this often involves bounding volume hierarchies (BVH) or grid traversal to find candidate splats, followed by a precise intersection test against the splat's footprint function.
4.  **Accumulation:** Accumulate the contributions of the intersected splats along the ray to determine the final pixel color and opacity [S1, unofficial].

**Parameters:**
*   **Reconstruction Kernel:** Defines the shape and size of the splat footprint. The sources mention that kernels must overlap to avoid rendering artifacts [S1, unofficial].
*   **Viewport Size:** Example: $260^2$ pixels [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Perspective Complexity:** Traditional splatting struggles with perspective projection due to the non-linear mapping of footprints. Ray-driven splatting avoids this by handling perspective in the ray definition [S2, unofficial].
*   **Aliasing:** Incorrect visibility determination during back-to-front compositing can cause aliasing if reconstruction kernels are assumed to be non-overlapping. Adaptations like Heckbert's EWA resampling filter can reduce these effects [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ray_driven_splatting(volumetric_data, camera, viewport):
    # 1. Initialize splat grid
    # Each voxel in volumetric_data is associated with a splat primitive
    # defined by its center position and footprint function.
    splat_grid = create_splat_grid(volumetric_data)
    
    # 2. Initialize output image
    image = zeros(viewport.width, viewport.height, 4) # RGBA
    
    # 3. Spawn rays and accumulate
    for y in range(viewport.height):
        for x in range(viewport.width):
            # Create ray for pixel (x, y)
            ray = create_ray(camera, x, y)
            
            # Find all splats intersected by the ray
            # This step requires spatial acceleration structures for efficiency
            intersected_splats = find_intersected_splats(ray, splat_grid)
            
            # Sort splats by distance from camera (back-to-front or front-to-back)
            intersected_splats.sort_by_distance(ray.origin)
            
            # Accumulate contributions
            color = (0.0, 0.0, 0.0, 0.0)
            for splat in intersected_splats:
                # Evaluate splat contribution at the intersection point
                # The exact formula for contribution evaluation is not provided in sources
                contribution = evaluate_splat_contribution(splat, ray)
                
                # Composite contribution into color
                # Standard volume rendering compositing equation
                alpha = contribution.alpha
                rgb = contribution.rgb
                color = composite(color, rgb, alpha)
                
                # Early ray termination optimization
                if color.alpha > 0.99:
                    break
            
            image[y, x] = color
            
    return image
```

## Subtopics

### [Splat Rasterization via Raycasting](1437-splat-rasterization-via-raycasting.md)
Implements splat rasterization on GPUs by drawing point primitives as screen-space squares and casting a ray for each fragment to test inclusion in the splat. It solves the problem of accurately projecting 3D splats onto the screen without underestimating extents, which causes rendering artifacts with previous approximations. Unlike Botsch et al.'s approximation, this raycasting approach provides a correct upper bound to extents and works for large splats near the near plane.

## Operating characteristics
*   **Performance:** In a test case, rendering the MRbrain dataset with a $260^2$ viewport took 30 seconds on an SGI Indigo with a 200 MHz processor [S1, unofficial].
*   **Scaling:** Offers competitive or superior performance for parallel projection, and superior performance for perspective projection compared to traditional splatting [S2, unofficial].
*   **Strengths:**
    *   Inherently insensitive to the complexity of the perspective viewing transform [S2, unofficial].
    *   Supports acceleration methods such as early ray termination and bounding volumes, which traditional voxel-driven splatting cannot benefit from [S2, unofficial].
    *   Enables the use of other ray tracing optimization techniques such as space leaping, adaptive screen sampling, or spatial acceleration structures [S1, unofficial].
*   **Limitations:**
    *   Requires efficient intersection and addressing of splats, which can be computationally intensive if not optimized [S2, unofficial].
    *   Potential for aliasing effects if reconstruction kernels are not properly handled (e.g., if assumed non-overlapping) [S1, unofficial].

## Background
Ray-driven splatting was proposed by Müller and Yagel [S1, unofficial]. It was developed to address the limitations of traditional voxel-driven splatting, particularly regarding perspective projection and the lack of support for ray-tracing optimizations. The method merges the technique of splatting with principles of raycasting [S2, unofficial].

## Sources
[S1] Interactive volume ray tracing, https://doi.org/10.22028/d291-25931, fetched 2026-10-26 (unofficial)
[S2] (PDF) EWA volume splatting - ResearchGate, https://www.researchgate.net/publication/48546966_EWA_volume_splatting, fetched 2026-10-28 (unofficial)
