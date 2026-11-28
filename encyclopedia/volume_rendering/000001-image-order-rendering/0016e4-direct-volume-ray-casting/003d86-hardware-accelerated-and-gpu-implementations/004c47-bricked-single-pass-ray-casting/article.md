# Bricked Single-Pass Ray-Casting

Also called: 3D Brick Cache Ray Casting, Brick cache raycasting, Brick-based Ray Casting, Bricked Ray-casting, Single-pass raycasting for bricked volumes

## Overview
Bricked single-pass ray-casting is a hardware-accelerated technique for rendering volumetric datasets that exceed the capacity of GPU video memory [S1]. It solves the out-of-core rendering problem by storing a dynamic working set of volume bricks in a large "brick cache" texture and performing ray-marching in a single fragment shader pass [S1]. Unlike multi-pass approaches that render each brick separately, this method translates virtual volume coordinates to physical cache coordinates on-the-fly, enabling efficient rendering of large datasets while maintaining the visual quality of direct ray-casting [S1].

## How it works
**Inputs and Outputs:**
- Input: A large 3D volume dataset partitioned into fixed-size bricks (sub-volumes).
- Input: A camera view (position, orientation, field of view).
- Input: A transfer function mapping scalar values to color and opacity.
- Input: A brick cache texture (3D texture) containing the currently loaded bricks.
- Output: A 2D image (color and opacity per pixel).

**Data Structures:**
1.  **Volume Grid:** The logical volume is divided into a regular grid of bricks. Each brick has a fixed resolution (e.g., $N \times N \times N$ voxels). The total volume dimensions are $V_x \times V_y \times V_z$.
2.  **Brick Cache Texture:** A large 3D texture object in GPU memory. Its dimensions are determined by the maximum number of bricks that can fit in available VRAM, not the total volume size. The texture stores the voxel data of the active bricks.
3.  **Virtual-to-Physical Mapping:** A mechanism to map a voxel coordinate $(x, y, z)$ in the logical volume to a coordinate $(u, v, w)$ in the brick cache texture. This involves:
    -   Calculating the brick ID: $b_x = \lfloor x / N \rfloor$, $b_y = \lfloor y / N \rfloor$, $b_z = \lfloor z / N \rfloor$.
    -   Calculating the local offset within the brick: $l_x = x \mod N$, $l_y = y \mod N$, $l_z = z \mod N$.
    -   Mapping the brick ID to a position in the cache texture. The cache layout must be consistent (e.g., row-major or column-major) and known to the shader.

**Numbered Steps:**
1.  **Ray Setup:** For each pixel, compute the ray origin and direction in object space. Perform object-space culling to determine the entry and exit points of the ray into the volume bounding box [S2, unofficial]. Perform empty space skipping if the volume has an associated empty-space indicator (e.g., a sparse structure or a separate empty-space texture) to skip non-voxel regions [S2, unofficial].
2.  **Ray Marching Loop:** Iterate along the ray from the entry point to the exit point. The step size is typically determined by the voxel size or a multiple thereof to ensure proper sampling.
3.  **Coordinate Translation:** For each sample point $P$ along the ray:
    -   Convert $P$ to integer brick coordinates $(b_x, b_y, b_z)$ and local fractional coordinates $(l_x, l_y, l_z)$.
    -   Determine the physical location of the brick $(b_x, b_y, b_z)$ in the brick cache texture. This requires a dynamic lookup or a precomputed mapping table if the cache layout is complex.
    -   Compute the texture coordinates $(u, v, w)$ in the brick cache texture corresponding to the sample point. This includes the base offset of the brick in the cache plus the local offset.
4.  **Sampling:** Sample the scalar value $S$ at $(u, v, w)$ from the brick cache texture using trilinear interpolation. The hardware texture unit handles the interpolation across the brick boundaries, ensuring correct tri-linear interpolation between adjacent bricks [S2, unofficial].
5.  **Classification:** Map the scalar value $S$ to color $C$ and opacity $\alpha$ using the transfer function.
6.  **Compositing:** Update the accumulated color $C'$ and opacity $\alpha'$ using the front-to-back compositing equation:
    $$
    C'_{i} = C'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i C_i
    $$
    $$
    \alpha'_{i} = \alpha'_{i-1} + (1 - \alpha'_{i-1}) \alpha_i
    $$
7.  **Early Ray Termination (ERT):** If $\alpha'$ exceeds a threshold (e.g., 0.95 or 1.0), terminate the ray marching loop early [G].
8.  **Output:** Write the final $C'$ and $\alpha'$ to the framebuffer.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the fragment shader
# Inputs: ray_origin, ray_direction, transfer_function, brick_cache_texture, brick_size, cache_layout_map
# Outputs: out_color, out_opacity

def fragment_shader(ray_origin, ray_direction):
    # 1. Ray Setup
    entry_t, exit_t = compute_entry_exit(ray_origin, ray_direction, volume_bounds)
    if entry_t > exit_t:
        return background_color, 0.0

    # 2. Initialize compositing
    acc_color = vec3(0.0)
    acc_opacity = 0.0
    t = entry_t

    # 3. Ray Marching Loop
    while t < exit_t and acc_opacity < 0.95: # ERT threshold
        # Sample point in world/object space
        sample_point = ray_origin + ray_direction * t

        # 4. Coordinate Translation
        # Convert to brick coordinates
        brick_coords = floor(sample_point / brick_size)
        local_coords = fract(sample_point / brick_size)

        # Map brick coordinates to cache texture coordinates
        # cache_layout_map is a function or texture that maps (bx, by, bz) to (u, v, w) in cache
        cache_coords = cache_layout_map(brick_coords)

        # Add local offset to cache coordinates
        # Note: local_coords are in [0,1), need to scale by brick_size if cache texture is not normalized
        # Assuming cache texture uses normalized coordinates for the brick
        final_cache_coords = cache_coords + local_coords

        # 5. Sampling
        scalar_value = texture(brick_cache_texture, final_cache_coords).r

        # 6. Classification
        color, opacity = transfer_function(scalar_value)

        # 7. Compositing
        acc_color = acc_color + (1.0 - acc_opacity) * opacity * color
        acc_opacity = acc_opacity + (1.0 - acc_opacity) * opacity

        # 8. Advance ray
        t += step_size # step_size is typically voxel_size or a multiple

    return acc_color, acc_opacity
```

**Parameters:**
-   **Brick Size:** The resolution of each brick (e.g., $32^3$, $64^3$). Larger bricks reduce the number of cache misses but increase memory usage per brick. Not given in the sources.
-   **Cache Size:** The maximum number of bricks that can fit in GPU memory. Determined by available VRAM and brick size. Not given in the sources.
-   **Step Size:** The distance between samples along the ray. Typically equal to the voxel size or a fraction thereof. Not given in the sources.
-   **ERT Threshold:** The opacity threshold for early ray termination. Typically 0.95 or 1.0 [G].

**Edge Cases and Pitfalls:**
-   **Cache Management:** The brick cache must be managed fully dynamically to accommodate changes in the current view and the current transfer function [S1]. If the view changes, the set of visible bricks changes, requiring cache updates. If the transfer function changes, the classification step changes, but the cache data itself may not need to change, though the rendering pipeline must handle this [S1].
-   **Interpolation at Brick Boundaries:** Correct tri-linear interpolation between bricks is essential to avoid visual artifacts. The hardware texture unit handles this if the brick cache texture is constructed such that adjacent bricks are adjacent in texture memory, or if the shader correctly handles the wrapping/offsetting [S2, unofficial].
-   **Address Translation Overhead:** The on-the-fly address translation from virtual to physical coordinates can be computationally expensive. Efficient mapping strategies are required to minimize this overhead [S1].

## Operating characteristics
-   **Time Complexity:** $O(N)$ per pixel, where $N$ is the number of samples along the ray. The address translation adds a constant overhead per sample.
-   **Memory Complexity:** $O(B)$, where $B$ is the number of bricks in the cache. This is much smaller than the total volume size, allowing rendering of volumes larger than GPU memory.
-   **Measured Performance:** Not given in the sources.
-   **Scaling:** Scales well with volume size, limited by GPU memory and cache management efficiency. Performance is sensitive to cache hit rates.
-   **Strengths:** Supports volumes larger than GPU memory; single-pass rendering is efficient; correct interpolation between bricks.
-   **Limitations:** Requires dynamic cache management; address translation overhead; complex implementation.
-   **When to Use:** When rendering large volumetric datasets that do not fit in GPU memory.
-   **When Not to Use:** When the volume fits in GPU memory, simpler 3D texture ray-casting may be more efficient.

## Background
Bricked single-pass ray-casting was introduced by Hadwiger et al. in 2005 and Beyer et al. in 2007 [S2, unofficial]. It was developed to address the limitations of earlier out-of-core rendering techniques, such as texture slicing, which suffered from visual artifacts and limited flexibility. The method builds on the principles of hardware-accelerated ray-casting and extends them to handle large-scale data by leveraging a brick cache architecture [S1]. It is widely used in scientific visualization for large datasets, such as medical imaging and climate data [S2, unofficial].

## Sources
[S1] State-of-the-Art in GPU-Based Large-Scale Volume Visualization, https://vcg.seas.harvard.edu/publications/20150513-stateof-the-art-in-gpu-based-large-scale-volume-visualization/paper, fetched 2026-10-24
[S2] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (unofficial)
