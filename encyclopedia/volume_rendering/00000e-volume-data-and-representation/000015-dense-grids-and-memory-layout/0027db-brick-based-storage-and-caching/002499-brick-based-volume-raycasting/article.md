# Brick-based Volume Raycasting

Also called: Brick-wise raycasting, Bricked volume layout

## Overview
Brick-based volume raycasting decomposes a dense volume dataset into fixed-size, power-of-two cubic blocks (bricks) to maximize CPU cache coherency during ray traversal, solving the view-dependent performance degradation inherent in linear memory layouts [S1, unofficial]. Unlike spread memory layouts that increase memory usage by a factor of four, this method maintains low memory overhead while providing nearly constant rendering performance regardless of viewing direction [S1, unofficial]. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by defining the specific ray-marching algorithm and addressing scheme required to traverse this one-level subdivision hierarchy.

## How it works
**Inputs:**
*   Volume data $V$ of logical dimensions $D_x \times D_y \times D_z$.
*   Brick dimensions $B_x, B_y, B_z$ (must be powers of two).
*   Transfer function $T(\cdot)$ mapping scalar values to color/opacity.
*   Camera parameters (origin $\mathbf{o}$, direction $\mathbf{d}$, near/far planes).

**Outputs:**
*   Rendered image buffer.

**Data Structures:**
1.  **Brick Array:** A linear array of bricks. The index of brick $(i, j, k)$ is:
    $$ Index(i, j, k) = i + j \cdot N_x + k \cdot N_x \cdot N_y $$
    where $N_x = \lceil D_x / B_x \rceil$, $N_y = \lceil D_y / B_y \rceil$, $N_z = \lceil D_z / B_z \rceil$ [S1, unofficial].
2.  **Voxel Storage:** Each brick stores $(B_x+1) \times (B_y+1) \times (B_z+1)$ voxels to support trilinear interpolation at boundaries without accessing adjacent bricks [S1, unofficial].
3.  **Ray Structure:** For each pixel, a ray segment is defined by its entry and exit points within the current brick.

**Algorithm Steps:**
1.  **Preprocessing:** Partition $V$ into bricks. Store bricks in linear xyz-order. Compute empty brick detection buffers (histograms) for each brick [S2, unofficial].
2.  **Ray Initialization:** For each pixel, compute the ray $\mathbf{r}(t) = \mathbf{o} + t\mathbf{d}$. Determine the initial intersection with the volume bounding box.
3.  **Brick Traversal:**
    *   Identify the brick containing the current ray position.
    *   Check the empty brick detection buffer. If the brick is empty (no voxels contribute to the transfer function), skip loading the brick and advance the ray to the next brick exit [S2, unofficial].
    *   Load the brick into cache.
4.  **Ray Marching within Brick:**
    *   Compute the entry $t_{in}$ and exit $t_{out}$ of the ray relative to the current brick's boundaries.
    *   Sample voxels along the ray segment $[t_{in}, t_{out}]$ using trilinear interpolation from the padded brick data.
    *   Composite samples using standard volume rendering integral approximation (e.g., front-to-back blending).
5.  **Iteration:** Update ray position. If the ray has exited the volume, terminate. Otherwise, return to Step 3.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def brick_based_raycasting(volume_data, brick_size, transfer_function, camera):
    # Precompute brick grid dimensions
    Dx, Dy, Dz = volume_data.shape
    Bx, By, Bz = brick_size
    Nx = ceil(Dx / Bx)
    Ny = ceil(Dy / By)
    Nz = ceil(Dz / Bz)
    
    # Precompute empty brick flags (from histogram buffers)
    # Not given in the sources: Implementation of histogram computation
    empty_bricks = compute_empty_bricks(volume_data, brick_size, transfer_function)

    for pixel in pixels:
        ray_origin, ray_direction = camera.get_ray(pixel)
        t = ray_origin.near_plane
        
        while t < ray_origin.far_plane:
            # Determine current brick indices
            # Not given in the sources: Exact formula for mapping ray position to brick index
            bx, by, bz = get_brick_indices(ray_origin, ray_direction, t, Bx, By, Bz, Nx, Ny, Nz)
            
            brick_index = bx + by * Nx + bz * Nx * Ny
            
            if empty_bricks[brick_index]:
                # Skip empty brick
                # Not given in the sources: Formula for t_next_brick_exit
                t = t_next_brick_exit(ray_origin, ray_direction, bx, by, bz, Bx, By, Bz)
                continue
            
            # Load brick into cache
            brick_data = load_brick(volume_data, brick_index, Bx, By, Bz)
            
            # Compute ray segment within this brick
            t_in, t_out = get_brick_intersections(ray_origin, ray_direction, bx, by, bz, Bx, By, Bz)
            
            # March through the brick
            sample_t = t_in
            while sample_t < t_out:
                # Interpolate voxel value from padded brick data
                # Not given in the sources: Trilinear interpolation implementation details
                value = trilinear_interpolate(brick_data, ray_origin + sample_t * ray_direction)
                
                # Composite
                color, opacity = transfer_function(value)
                alpha = opacity * step_size
                # Front-to-back blending
                # Not given in the sources: Standard blending formula
                out_color = current_color + alpha * color * (1 - current_opacity)
                current_opacity += alpha * (1 - current_opacity)
                
                sample_t += step_size
            
            # Advance to next brick
            t = t_out
```

**Parameters:**
*   **Brick Size:** Must be a power of two. Optimal size is 64KB ($32 \times 32 \times 32$ voxels for 16-bit data) [S1, unofficial].
*   **Padding:** Each brick is stored with a 1-voxel padding layer on all sides, resulting in $(B+1)^3$ voxels per brick [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Brick Size Selection:** Increasing brick size beyond 64KB leads to performance decreases due to exceeding cache capacity and bricking overhead [S1, unofficial].
*   **View Dependency:** Linear layouts suffer from enormous performance decreases depending on viewing direction; brick-based layouts exhibit nearly no view-dependent performance variation [S1, unofficial].
*   **Boundary Interpolation:** Padding is required to support trilinear interpolation at brick boundaries without accessing adjacent bricks [S1, unofficial].

## Subtopics

### Brick Caching with LRU Replacement
Manages the cache of loaded bricks using Least Recently Used eviction. Choose when memory is constrained and brick access patterns are spatially localized. [Link](879-brick-caching-with-lru-replacement.md)

### N³-tree with Brick Pool and Node Pool
Uses a hierarchical tree structure for adaptive brick management. Choose when data density varies significantly across the volume. [Link](3407-n-tree-with-brick-pool-and-node-pool.md)

### Multi-Level Cache and Prefetching
Optimizes data fetching with multiple cache levels. Choose when memory bandwidth is the bottleneck and access patterns are predictable. [Link](4631-multi-level-cache-and-prefetching.md)

### Bricked Volume Hierarchy
Organizes bricks into a hierarchy for faster traversal. Choose when rendering very large datasets requiring coarse-to-fine culling. [Link](5036-bricked-volume-hierarchy.md)

### Raymarching with LOD-based Brick Caching
Adjusts brick resolution based on distance. Choose when performance varies with camera distance and detail requirements differ. [Link](5776-raymarching-with-lod-based-brick-caching.md)

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of samples, but with a significantly lower constant factor due to cache hits. Performance is nearly constant regardless of viewing direction [S1, unofficial].
*   **Memory Complexity:** $O(V)$ where $V$ is the volume size, with minimal overhead from padding and brick metadata. Unlike spread layouts, it does not increase memory usage by a factor of four [S1, unofficial].
*   **Measured Performance:** On a Pentium 4 Xeon with 512KB L2 cache, brick-based raycasting with a 64KB brick size achieves a speedup of approximately 2.8 compared to common raycasting on a linear volume. A 1KB brick size yields a speedup of about 2.0 [S1, unofficial].
*   **Strengths:** High cache coherency, constant performance across viewing angles, low memory overhead.
*   **Limitations:** Performance degrades if brick size exceeds optimal cache capacity (e.g., >64KB on the tested hardware). Bricking overhead exists for very small bricks [S1, unofficial].
*   **When to Use:** Real-time rendering of large medical datasets on standard PC hardware where cache performance is critical.
*   **When Not to Use:** When memory bandwidth is the primary bottleneck and spread layouts are viable, or when adaptive resolution is required (use LOD-based methods instead).

## Background
Brick-based volume raycasting was developed to address the poor CPU utilization in real-time volume rendering of large medical datasets on standard PC hardware [S1, unofficial]. The method was introduced in a dissertation by Knittel (2005), which proposed a one-level subdivision hierarchy to achieve high cache coherency without the memory overhead of spread layouts [S1, unofficial]. The approach was benchmarked on a Pentium 4 Xeon, demonstrating significant speedups over linear raycasting [S1, unofficial].

## Sources
[S1] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
