# Marching Caches

Also called: Explicit Cache Management for Volume Ray-Casting

## Overview
Marching Caches is an explicit data caching strategy for volume ray-casting that exploits inter-ray coherence by maintaining a shared cache around a bundle of rays [S1]. It solves the problem of inefficient memory access patterns and high global memory bandwidth usage inherent in standard GPU volume rendering, where implicit hardware caching often fails to retain data due to low computational intensity per memory access [S1]. Unlike its parent method, Inter-Ray Coherency, which tracks individual ray intersections, Marching Caches maintains an explicit shared cache that "marches" along with the ray front, allowing neighboring rays to share fast memory to fill missing values [S1].

## How it works
**Inputs:** A bundle of rays aligned such that their ray fronts access the same data simultaneously; a volume dataset; a reconstruction/enhancement filter (e.g., Catmul-Rom, B-spline, trilinear, bilateral, gradient, ambient occlusion projection) [S1].
**Outputs:** Rendered image with reduced global memory bandwidth and improved performance [S1].

**Data Structures:**
1.  **Marching Cache:** A shared memory buffer containing voxel data required by the current ray bundle [S1].
2.  **Ray Front State:** Tracks the current position of the synchronized ray front across the bundle [S1].

**Formulas:**
*   Ray position: $P(t) = O + t \cdot D$, where $O$ is the ray origin, $D$ is the direction vector, and $t$ is the step distance [G].
*   Voxel index: $I = \lfloor P(t) / V_s \rfloor$, where $V_s$ is the voxel size [G].
*   Cache bounds: $B_{min} = \min(P_{front}) - K$, $B_{max} = \max(P_{front}) + K$, where $K$ is the filter kernel radius [G].
*   Alpha compositing: $C_{out} = C_{in} + (1 - \alpha_{in}) \cdot S \cdot K$, where $S$ is the sample value and $K$ is the filter weight [G].

**Algorithm Steps:**
The method is enabled in three steps, occurring before and during ray traversal [S1]:
1.  **Ray Front Synchronization:** Align the bundle of rays such that the ray front accesses the same data simultaneously [S1].
2.  **Cache Invalidation and Update Calculation:** Determine which part of the cache each ray needs to update when the cache becomes invalid [S1].
3.  **Incremental Cache Update and Rendering:** Incrementally update the cache in the direction of the rays as they traverse the volume [S1]. Neighboring rays share fast memory to fill missing values [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_caches_render(ray_bundle, volume, filter_kernel, step_size, voxel_size):
    # Initialize cache bounds based on ray front and filter support
    cache_min = min(ray.origin for ray in ray_bundle) - filter_kernel.radius
    cache_max = max(ray.origin for ray in ray_bundle) + filter_kernel.radius
    cache = {}

    # Step 1: Synchronize ray front
    # Align rays so the ray front accesses the same data simultaneously [S1].
    for ray in ray_bundle:
        ray.t = 0.0

    # Step 2: Initial cache population
    for x in range(int(cache_min.x // voxel_size), int(cache_max.x // voxel_size) + 1):
        for y in range(int(cache_min.y // voxel_size), int(cache_max.y // voxel_size) + 1):
            for z in range(int(cache_min.z // voxel_size), int(cache_max.z // voxel_size) + 1):
                cache[(x, y, z)] = volume.read((x, y, z))

    # Step 3: Incremental update and rendering loop
    for ray in ray_bundle:
        color = [0.0, 0.0, 0.0, 0.0]
        alpha = 0.0
        t = 0.0

        while t < ray.max_distance:
            pos = ray.origin + t * ray.direction
            idx = (int(pos.x // voxel_size), int(pos.y // voxel_size), int(pos.z // voxel_size))

            if idx not in cache:
                cache[idx] = volume.read(idx)

                # Incrementally update cache as ray front advances [S1]
                new_front = ray.origin + ray.t * ray.direction
                cache_min = new_front - filter_kernel.radius
                cache_max = new_front + filter_kernel.radius
                for x in range(int(cache_min.x // voxel_size), int(cache_max.x // voxel_size) + 1):
                    for y in range(int(cache_min.y // voxel_size), int(cache_max.y // voxel_size) + 1):
                        for z in range(int(cache_min.z // voxel_size), int(cache_max.z // voxel_size) + 1):
                            if (x, y, z) not in cache:
                                cache[(x, y, z)] = volume.read((x, y, z))

            sample = cache[idx]
            weight = filter_kernel.sample(pos)
            color = [c + (1.0 - alpha) * s * weight for c, s in zip(color, sample)]
            alpha = min(1.0, alpha + (1.0 - alpha) * weight)

            t += step_size
            if alpha >= 1.0:
                break

        ray.color = color
```

**Parameters:**
*   **Filters:** Catmul-Rom, B-spline, ambient occlusion projection, trilinear, bilateral, gradient, and filtered gradient [S1].
*   **Cache Size:** Computed for the ray bundle extent [S1].

**Edge Cases and Pitfalls:**
*   **Shared Memory Limits:** The cache must fit in shared memory; if the bundle width or filter kernel exceeds available memory, thrashing occurs [G].
*   **Boundary Handling:** Rays exiting the volume or hitting boundaries must be clamped to prevent out-of-bounds voxel reads [G].

## Subtopics

### [Tile-Based Volume Rendering](17931-tile-based-volume-rendering.md)
Tile-Based Volume Rendering is a rendering technique that splits the image plane into tiles of closely related pixels to limit data access and improve cache locality. It addresses the problem of inefficient memory access patterns in stochastic volume rendering by ensuring that voxels and data accessed during the tracing of one ray are reused for other rays close to the first one. Unlike its parent method, Marching Caches, which maintains an explicit shared cache around a ray bundle, Tile-Based Volume Rendering relies on spatial locality within fixed image tiles and sequential processing of frame estimates to prevent cache replacement.

## Operating characteristics
*   **Performance:** Achieved a speedup of four times compared to using implicit hardware caching [S1].
*   **Memory Bandwidth:** Reduced memory bandwidth to global memory by orders of magnitude [S1].
*   **Strengths:** Allows the programmer to reorder computations to make maximum use of available shared memory; enables the use of higher-order reconstruction and enhancement filters without prohibitive performance costs [S1].
*   **Limitations:** Requires explicit management of cache state and synchronization; performance depends heavily on the coherence of the ray bundle [S1].
*   **When to Use:** When high-fidelity rendering with complex filters is required and inter-ray coherence can be exploited [S1].
*   **When Not to Use:** When ray coherence is low or when the cache size required for the filter exceeds available shared memory [S1].

## Background
Marching Caches was introduced in the paper "Explicit Cache Management for Volume Ray-Casting on Parallel Architectures" by the authors of [S1]. The method addresses the limitations of modern GPUs, which are complex parallel devices allowing SIMD processing but providing no flexibility in data caching provided by the graphics architecture [S1]. The paper focuses on volume ray-casting and shows the benefits of algorithm-aware data caching [S1]. The method was tested with seven different filters, demonstrating significant performance improvements and bandwidth reduction [S1].

## Sources
[S1] Explicit Cache Management for Volume Ray-Casting on Parallel ..., https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2012/marchingcaches_final.pdf, fetched 2026-10-29
