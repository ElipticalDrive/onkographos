# Brick-based Volume Raycasting

Also called: Brick-wise raycasting, Bricked volume layout

## Overview
Brick-based volume raycasting decomposes a dense volume dataset into fixed-size cubic blocks (bricks) to achieve high cache coherency on CPU architectures without the memory overhead associated with spread memory layouts [S1, unofficial]. Unlike linear raycasting, which suffers from severe view-dependent performance drops due to cache misses when traversing large datasets, this method provides nearly constant rendering performance regardless of viewing direction [S1, unofficial]. It builds on [Brick-based storage and caching](4138-brick-based-storage-and-caching.md) by implementing a one-level subdivision hierarchy optimized for real-time brute-force raycasting.

## How it works
**Inputs and Outputs:**
*   **Input:** A dense volume dataset of dimensions $D_x \times D_y \times D_z$ with voxel data type (e.g., 16-bit unsigned integer).
*   **Input:** Brick dimensions $B_x, B_y, B_z$ (must be powers of two).
*   **Input:** Camera parameters (origin, direction, field of view).
*   **Output:** A rendered 2D image.

**Data Structures:**
The volume is partitioned into a grid of bricks.
1.  **Logical Grid:** The number of bricks in each dimension is:
    $$ N_x = \lceil D_x / B_x \rceil, \quad N_y = \lceil D_y / B_y \rceil, \quad N_z = \lceil D_z / B_z \rceil $$
    Total bricks: $N_{total} = N_x \cdot N_y \cdot N_z$ [S1, unofficial].
2.  **Linear Storage:** Bricks are stored in a linear array in xyz-order. The index of brick $(i, j, k)$ is:
    $$ Index(i, j, k) = i + j \cdot N_x + k \cdot N_x \cdot N_y $$
    where $0 \le i < N_x$, $0 \le j < N_y$, $0 \le k < N_z$ [S1, unofficial].
3.  **Voxel Layout:** Each logical brick of size $(B_x, B_y, B_z)$ is stored with a 1-voxel padding layer on all sides to support trilinear interpolation at boundaries without accessing adjacent bricks. The stored voxel count per brick is:
    $$ V_{brick} = (B_x+1)(B_y+1)(B_z+1) $$
    [S1, unofficial].

**Algorithm Steps:**
1.  **Initialization:** Load the volume data into the linear brick array. Ensure brick dimensions are powers of two.
2.  **Ray Generation:** For each pixel in the output image, generate a primary ray.
3.  **Brick Intersection:** Determine which bricks the ray intersects. This involves calculating the entry and exit points of the ray within the global volume and mapping these coordinates to the corresponding brick indices.
4.  **Ray List Construction:** Maintain a list of active rays for each intersected brick. This avoids per-ray overhead for bricks that are not intersected by any ray (brick skipping) [S1, unofficial].
5.  **Raymarching:** For each brick in the list:
    a. Load the brick data (including padding) into cache/memory.
    b. Iterate through the rays assigned to this brick.
    c. For each ray, sample voxels within the brick's bounds using trilinear interpolation.
    d. Accumulate color and opacity using standard volume integration (e.g., front-to-back compositing).
6.  **Compositing:** Write the accumulated color to the image buffer.

**Parameters:**
*   **Brick Size:** Must be a power of two. The optimal size identified is 64KB, corresponding to $32 \times 32 \times 32$ voxels for 16-bit data [S1, unofficial].
*   **Data Type:** 16-bit per voxel is typical for medical datasets [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Brick Size Selection:** Increasing brick size beyond the optimal point (64KB) leads to performance decreases due to exceeding cache capacity and increased bricking overhead [S1, unofficial].
*   **Volume-Sized Bricks:** If the brick size equals the volume size, the overhead of brick management disappears, reverting to the performance characteristics of a common linear raycaster [S1, unofficial].
*   **View Dependency:** Unlike linear layouts, small bricks eliminate view-dependent performance variation [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def brick_based_raycasting(volume_data, brick_dims, camera):
    """
    volume_data: 3D array of shape (Dx, Dy, Dz)
    brick_dims: tuple (Bx, By, Bz), must be powers of two
    camera: object with origin, direction, and image dimensions
    """
    Dx, Dy, Dz = volume_data.shape
    Bx, By, Bz = brick_dims
    
    # Calculate number of bricks
    Nx = (Dx + Bx - 1) // Bx
    Ny = (Dy + By - 1) // By
    Nz = (Dz + Bz - 1) // Bz
    
    # Create padded bricks
    # Each brick is stored with 1-voxel padding
    padded_brick_dims = (Bx + 2, By + 2, Bz + 2)
    bricks = []
    
    for k in range(Nz):
        for j in range(Ny):
            for i in range(Nx):
                # Extract logical brick region
                x_start = i * Bx
                y_start = j * By
                z_start = k * Bz
                
                x_end = min(x_start + Bx, Dx)
                y_end = min(y_start + By, Dy)
                z_end = min(z_start + Bz, Dz)
                
                # Extract data and pad
                # Note: In a real implementation, this extraction is optimized
                # to avoid copying if possible, but for clarity:
                raw_data = volume_data[x_start:x_end, y_start:y_end, z_start:z_end]
                
                # Create padded brick
                padded = np.zeros(padded_brick_dims, dtype=volume_data.dtype)
                # Map raw data to center of padded brick
                padded[1:x_end-x_start+1, 1:y_end-y_start+1, 1:z_end-z_start+1] = raw_data
                
                bricks.append(padded)
    
    # Initialize image buffer
    image = np.zeros((camera.height, camera.width, 3))
    
    # Generate rays and assign to bricks
    # RayList[brick_index] = list of (ray_origin, ray_direction, pixel_coords)
    ray_lists = [[] for _ in range(Nx * Ny * Nz)]
    
    for y in range(camera.height):
        for x in range(camera.width):
            ray = camera.get_ray(x, y)
            
            # Determine intersected bricks
            # This requires mapping ray bounds to brick indices
            intersected_bricks = get_intersected_bricks(ray, Dx, Dy, Dz, Bx, By, Bz, Nx, Ny, Nz)
            
            for brick_idx in intersected_bricks:
                ray_lists[brick_idx].append((ray, x, y))
    
    # Process each brick
    for brick_idx, rays in enumerate(ray_lists):
        if not rays:
            continue
            
        brick_data = bricks[brick_idx]
        
        for ray, px, py in rays:
            # Raymarch within this brick
            # Calculate entry and exit points relative to brick
            # Sample voxels using trilinear interpolation on brick_data
            # Accumulate color and opacity
            
            color = 0.0
            opacity = 0.0
            
            # Simplified raymarch loop
            t = ray.entry_t
            while t < ray.exit_t:
                # Get voxel position in brick
                voxel_pos = ray.origin + ray.direction * t
                local_pos = voxel_pos - brick_origin(brick_idx)
                
                # Sample with trilinear interpolation
                val = trilinear_sample(brick_data, local_pos)
                
                # Transfer function to color and opacity
                c, a = transfer_function(val)
                
                # Front-to-back compositing
                alpha = a * opacity_factor
                color += c * alpha * (1.0 - opacity)
                opacity += alpha * (1.0 - opacity)
                
                t += ray.step_size
            
            image[py, px] = color
            
    return image
```

## Subtopics

### Brick Caching with LRU Replacement
Manages brick residency in GPU VRAM or CPU cache using Least Recently Used eviction. Choose when memory bandwidth is limited and bricks are larger than cache lines. [Link](879-brick-caching-with-lru-replacement.md)

### N³-tree with Brick Pool and Node Pool
Uses a hierarchical tree structure for adaptive brick resolution. Choose when data density varies significantly across the volume. [Link](3407-n-tree-with-brick-pool-and-node-pool.md)

### Multi-Level Cache and Prefetching
Implements hierarchical caching with prefetching to hide memory latency. Choose when CPU cache misses are the primary bottleneck. [Link](4631-multi-level-cache-and-prefetching.md)

### Bricked Volume Hierarchy
Organizes bricks into a multi-level hierarchy for efficient traversal. Choose when rendering very large datasets with complex occlusion. [Link](5036-bricked-volume-hierarchy.md)

### Raymarching with LOD-based Brick Caching
Adjusts brick resolution based on distance from camera. Choose when performance varies with view distance and detail is not uniform. [Link](5776-raymarching-with-lod-based-brick-caching.md)

## Operating characteristics
*   **Time Complexity:** $O(N_{rays} \cdot N_{samples})$ per frame, where $N_{rays}$ is the number of pixels and $N_{samples}$ is the number of samples per ray. The constant factor is significantly reduced compared to linear raycasting due to cache coherency.
*   **Memory Complexity:** $O(V_{total} \cdot (1 + \epsilon))$, where $\epsilon$ is the overhead from padding. The padding overhead is small compared to spread memory layouts which use 4x memory [S1, unofficial].
*   **Measured Performance:** On a Pentium 4 Xeon with 512KB L2 cache, brick-based raycasting with a 64KB brick size ($32 \times 32 \times 32$ voxels) achieves a speedup of approximately 2.8 compared to common raycasting on a linear volume [S1, unofficial]. A 1KB brick size yields a speedup of about 2.0 [S1, unofficial].
*   **Scaling:** Performance is nearly constant regardless of viewing direction, unlike linear layouts which show enormous performance decreases depending on viewing direction [S1, unofficial].
*   **Strengths:** High cache coherency, constant performance behavior, low memory overhead compared to spread layouts.
*   **Limitations:** Performance decreases if brick size exceeds optimal cache capacity (e.g., >64KB on the tested hardware). Bricking overhead exists for small bricks.
*   **When to Use:** Real-time rendering on CPU hardware with limited cache, where view-dependent performance is unacceptable.
*   **When Not to Use:** When GPU acceleration is available (GPU has its own caching mechanisms), or when data density is highly non-uniform (use N³-tree).

## Background
Brick-based volume raycasting was introduced in the context of real-time mono- and multi-volume rendering of large medical datasets on standard PC hardware [S1, unofficial]. The method addresses the limitations of linear volume layouts, which suffer from poor CPU cache utilization due to non-coherent memory access patterns during ray traversal. The approach was developed to support semitransparent views with high real-time performance without dramatically increasing memory usage [S1, unofficial]. The optimal brick size of 64KB was determined empirically for Pentium 4 Xeon processors with 512KB L2 cache [S1, unofficial].

## Sources
[S1] Real-Time Mono- and Multi-Volume Rendering of Large Medical Datasets on Standard PC Hardware, https://doi.org/10.2312/diss.20058170, fetched 2026-10-26 (unofficial)
