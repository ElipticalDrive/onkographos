# Marching Caches

Also called: Explicit Cache Management for Volume Ray-Casting

## Overview
Marching Caches is an explicit data caching strategy for volume ray-casting that exploits inter-ray coherence by maintaining a shared cache around a bundle of rays [S1]. It solves the problem of inefficient memory access patterns and high global memory bandwidth usage inherent in standard GPU volume rendering, where implicit hardware caching often fails to retain data due to low computational intensity per memory access [S1]. Unlike its parent method, Inter-Ray Coherency, which tracks individual ray intersections, Marching Caches maintains an explicit shared cache that "marches" along with the ray front, allowing neighboring rays to share fast memory to fill missing values [S1].

## How it works
**Inputs:** A bundle of rays aligned such that their ray fronts access the same data simultaneously; a volume dataset; a reconstruction/enhancement filter (e.g., Catmul-Rom, B-spline, trilinear, bilateral, gradient, ambient occlusion projection) [S1].
**Outputs:** Rendered image with reduced global memory bandwidth and improved performance [S1].

**Data Structures:**
1.  **Marching Cache:** A shared memory buffer (typically GPU shared memory) containing voxel data required by the current ray bundle. The cache is structured to cover the spatial extent of the ray front plus a margin for higher-order filters [S1].
2.  **Ray Bundle State:** Tracks the current position of each ray in the bundle and the synchronization status of the ray front [S1].

**Algorithm Steps:**
The method is enabled in three steps, occurring before and during ray traversal [S1]:

1.  **Ray Front Synchronization:** Align the bundle of rays such that the ray front accesses the same data simultaneously. This ensures that threads in a warp or SIMD unit are processing data that can be coalesced or shared [S1].
2.  **Cache Invalidation and Update Calculation:** Determine which part of the cache each ray needs to update when the cache becomes invalid. This involves computing the size of the cache required for the specific filter and determining which voxels need to be fetched from global memory [S1].
3.  **Incremental Cache Update and Rendering:** Incrementally update the cache in the direction of the rays as they traverse the volume. Neighboring rays share fast memory to fill missing values. The cache is synchronized and updated as the ray front advances [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def marching_caches_render(ray_bundle, volume, filter_type):
    # Step 1: Synchronize ray front
    # Align rays so the front accesses the same data simultaneously.
    # This may involve adjusting ray origins or step sizes to ensure coherence.
    synchronize_ray_front(ray_bundle)
    
    # Step 2: Determine cache requirements
    # Calculate the spatial extent of the ray front and the filter kernel.
    cache_extent = compute_cache_extent(ray_bundle, filter_type)
    cache_size = cache_extent.volume * volume.voxel_size
    
    # Initialize cache with data from previous frame or global memory
    # if this is the first frame or cache is invalid.
    if cache_invalid:
        fetch_data_from_global_memory(cache, cache_extent, volume)
        cache_invalid = False
    
    # Step 3: Incremental update and rendering loop
    while not ray_bundle.exited_volume():
        # Determine which voxels are needed for the current step
        needed_voxels = compute_needed_voxels(ray_bundle, cache_extent)
        
        # Update cache: fetch new voxels from global memory if missing
        # Neighboring rays share the cache, so only unique missing voxels are fetched.
        for voxel in needed_voxels:
            if not voxel in cache:
                cache[voxel] = volume.read(voxel)
        
        # Synchronize threads to ensure cache consistency
        sync_threads()
        
        # Render using cached data
        for ray in ray_bundle:
            color = apply_filter(ray, cache, filter_type)
            ray.composite(color)
        
        # Advance ray front
        advance_ray_front(ray_bundle)
        
        # Check if cache needs to be shifted or refilled
        if cache_extent.shifted():
            shift_cache(ray_bundle)
            fetch_missing_data(cache, cache_extent, volume)
```

**Parameters:**
*   **Filters:** Catmul-Rom, B-spline, ambient occlusion projection, trilinear, bilateral, gradient, and filtered gradient [S1].
*   **Cache Size:** Determined by the ray bundle width and the filter kernel support [S1].

**Edge Cases and Pitfalls:**
*   **Cache Thrashing:** If the ray bundle is too wide or the filter kernel too large, the cache may not fit in shared memory, leading to thrashing. [G]
*   **Synchronization Overhead:** Frequent synchronization of the ray front can introduce latency. [S1]
*   **Invalid Cache State:** If the cache becomes invalid (e.g., due to large ray steps or scene changes), a full refetch from global memory is required, which can cause a performance spike. [S1]

## Subtopics

### [Tile-Based Volume Rendering](17931-tile-based-volume-rendering.md)
Tile-Based Volume Rendering is a rendering technique that splits the image plane into tiles of closely related pixels to limit data access and improve cache locality. It addresses the problem of inefficient memory access patterns in stochastic volume rendering by ensuring that voxels and data accessed during the tracing of one ray are reused for other rays close to the first one. Unlike its parent method, Marching Caches, which maintains an explicit shared cache around a ray bundle, Tile-Based Volume Rendering relies on spatial locality within fixed image tiles and sequential processing of frame estimates to prevent cache replacement.

## Operating characteristics
*   **Performance:** Achieved a speedup of four times compared to using implicit hardware caching [S1].
*   **Memory Bandwidth:** Reduced memory bandwidth to global memory by orders of magnitude [S1].
*   **Strengths:** Allows the programmer to reorder computations to make maximum use of available shared memory; enables the use of higher-order reconstruction and enhancement filters (e.g., B-spline, ambient occlusion) without prohibitive performance costs [S1].
*   **Limitations:** Requires explicit management of cache state and synchronization; performance depends heavily on the coherence of the ray bundle [S1].
*   **When to Use:** When high-fidelity rendering with complex filters is required and inter-ray coherence can be exploited [S1].
*   **When Not to Use:** When ray coherence is low or when the cache size required for the filter exceeds available shared memory [S1].

## Background
Marching Caches was introduced in the paper "Explicit Cache Management for Volume Ray-Casting on Parallel Architectures" by the authors of [S1]. The method addresses the limitations of modern GPUs, which are complex parallel devices allowing SIMD processing but providing no flexibility in data caching provided by the graphics architecture [S1]. The paper focuses on volume ray-casting and shows the benefits of algorithm-aware data caching [S1]. The method was tested with seven different filters, demonstrating significant performance improvements and bandwidth reduction [S1].

## Sources
[S1] Explicit Cache Management for Volume Ray-Casting on Parallel ..., https://www.uni-ulm.de/fileadmin/website_uni_ulm/iui.inst.100/institut/Papers/viscom/2012/marchingcaches_final.pdf, fetched 2026-10-29
