# UltraVis Cache Optimization

Also called: Knittel's Ray Casting, Optimized CPU Ray-Casting, Pentium III Optimized Ray Casting, ULTRAVIS Cache-Based Optimization, UltraVis, UltraVis Cache Management, UltraVis System

## Overview
UltraVis is a software-based volume rendering system designed to maximize cache performance on Intel Pentium III processors by addressing cache thrashing via memory layout and interleaved addressing [S1][S2, unofficial][S3, unofficial]. It sets itself apart from general ray-casting methods by enforcing a rigid cache partitioning scheme that guarantees volume data remains in a private cache block, never being swapped out by competing structures like transfer functions or image buffers [S2, unofficial][S3, unofficial]. Unlike sibling methods that may rely on hardware-specific instruction sets or GPU parallelism, UltraVis focuses on CPU cache architecture constraints, using bricked decomposition and ray flocking to ensure data locality [S3, unofficial].

## How it works
**Inputs and Outputs:**
- Input: Volumetric data, transfer function tables, camera parameters, and image resolution [G].
- Output: Rendered 2D image [G].

**Data Structures and Layout:**
1.  **Cache Partitioning:** The CPU cache is divided into four distinct, non-overlapping blocks:
    -   Volume bricks
    -   Transfer function tables
    -   Image blocks
    -   Temporary buffers [S2, unofficial][S3, unofficial]
2.  **Volume Memory Layout:**
    -   Volume data is mapped exclusively into the "Volume bricks" cache block [S3, unofficial].
    -   To prevent mapping conflicts with other data structures, volume data is placed in the first quarter of each memory page [S1].
    -   This results in a main memory footprint four times higher than standard contiguous storage because no volume data may be stored in an address space that would map outside the volume's private cache slots [S3, unofficial].
3.  **Interleaved Addressing:**
    -   Voxel values are reordered using a cubic interleaved address function [S1].
    -   This function assumes $n$ is a power of two and $n^3$ is smaller than the cache size [S1].
    -   Under these conditions, a cube of $n^3$ voxels occupies exactly $n \times n \times n$ cache locations [S1].

**Algorithm Steps:**
1.  **Brick Decomposition:** Divide the volume into bricks [S3, unofficial].
2.  **Ray Flocking:** Group rays into a "flock" that traverses the same brick simultaneously [S3, unofficial].
3.  **Cache Loading:** Load the entire brick data into the private volume cache block once [S3, unofficial].
4.  **Traversal:** Trace all rays in the flock through the loaded brick. Because the rays are close together within the brick, the interleaved memory layout ensures that neighboring rays access cached voxel values, increasing the cache hit ratio [S1].
5.  **Discard:** Once all rays in the flock have finished traversing the brick, discard the brick data from the cache [S3, unofficial].
6.  **Shading:** Apply transfer functions and compositing. The transfer function tables reside in their own private cache block, ensuring they are not evicted by volume data [S2, unofficial][S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def ultravis_render(volume_data, transfer_function, camera, image_width, image_height):
    # 1. Setup Cache Blocks
    # The system implicitly manages four cache blocks:
    # - volume_block: for volume bricks
    # - tf_block: for transfer function tables
    # - image_block: for output image tiles
    # - temp_block: for temporary buffers
    # [G] Implementation note: On actual hardware, this requires specific
    # memory allocation strategies (e.g., page alignment) to enforce
    # the partitioning described in [S1][S3, unofficial].

    # 2. Prepare Volume Data
    # Reorder voxel data using cubic interleaved address function.
    # Let n be the brick dimension (power of 2).
    # Ensure n^3 < cache_size.
    # Map data to first quarter of memory pages.
    interleaved_volume = apply_cubic_interleaving(volume_data)
    
    # 3. Prepare Transfer Function
    # Load transfer function into its dedicated cache block.
    tf_table = transfer_function
    
    # 4. Ray Flock Traversal
    # Generate rays from camera.
    rays = generate_rays(camera, image_width, image_height)
    
    # Group rays into flocks that share bricks.
    # This assumes a bricked volume decomposition.
    brick_flocks = group_rays_into_bricks(rays, interleaved_volume.brick_size)
    
    for flock in brick_flocks:
        brick_id = flock.brick_id
        
        # Load brick into private volume cache block.
        # [G] Ensure this load does not evict data from tf_block,
        # image_block, or temp_block.
        current_brick = load_brick(interleaved_volume, brick_id)
        
        # Trace all rays in the flock through the loaded brick.
        # The interleaved layout ensures high cache reuse for neighboring rays.
        for ray in flock.rays:
            color = trace_ray_through_brick(ray, current_brick, tf_table)
            composite_into_image(ray, color, flock.output_coords)
            
        # Discard brick data from cache after flock traversal.
        unload_brick(current_brick)

def apply_cubic_interleaving(volume_data):
    """
    Reorders volume data to minimize cache thrashing.
    Assumes n is power of 2 and n^3 < cache_size.
    [G] Standard stride-based permutation to map n^3 voxels to n x n x n cache lines.
    """
    n = volume_data.brick_size
    interleaved = empty_array_like(volume_data)
    for z in range(n):
        for y in range(n):
            for x in range(n):
                # Bit-reversal/stride permutation to ensure spatial locality in cache lines
                new_x = bit_reverse(x, log2(n))
                new_y = bit_reverse(y, log2(n))
                new_z = bit_reverse(z, log2(n))
                interleaved[new_z, new_y, new_x] = volume_data[z, y, x]
    return interleaved

def trace_ray_through_brick(ray, brick, tf_table):
    """
    Standard front-to-back volume rendering integration within a brick.
    Uses the interleaved memory layout for efficient access.
    """
    out_color = [0.0, 0.0, 0.0, 0.0]
    transmittance = 1.0
    t = ray.origin
    step = ray.step_size
    epsilon = 1e-4
    
    while t < ray.end and transmittance > epsilon:
        voxel_coords = ray.origin + t * ray.direction
        voxel_value = sample_volume(brick, voxel_coords)
        rgba = tf_table[voxel_value]
        
        # Standard front-to-back compositing
        alpha = 1.0 - exp(-voxel_value.density * step)
        out_color[0] += transmittance * rgba.r * alpha
        out_color[1] += transmittance * rgba.g * alpha
        out_color[2] += transmittance * rgba.b * alpha
        out_color[3] = min(1.0, out_color[3] + transmittance * rgba.a * alpha)
        transmittance *= (1.0 - alpha)
        t += step
        
    return out_color

def generate_rays(camera, image_width, image_height):
    rays = []
    for y in range(image_height):
        for x in range(image_width):
            direction = normalize(camera.project_pixel(x, y))
            rays.append(Ray(camera.origin, direction, camera.step_size))
    return rays

def group_rays_into_bricks(rays, brick_size):
    flocks = {}
    for ray in rays:
        brick_id = compute_brick_id(ray.origin, brick_size)
        if brick_id not in flocks:
            flocks[brick_id] = []
        flocks[brick_id].append(ray)
    return flocks

def load_brick(volume, brick_id):
    return volume.extract_brick(brick_id)

def unload_brick(brick):
    del brick

def composite_into_image(ray, color, coords):
    image[coords.y][coords.x] = color
```

**Parameters:**
-   **Brick Size ($n$):** Must be a power of two. $n^3$ must be smaller than the L1 cache size to ensure the entire brick fits in the private cache block [S1].
-   **Performance:** Renders a $256^3$ volume at 10 frames/s [S2, unofficial][S3, unofficial]. Renders UNC MRI-head ($256 \times 256 \times 110$) at 1.7 to 7.0 fps on a Pentium III 500MHz with $256 \times 256$ image resolution [S1].

**Edge Cases and Pitfalls:**
-   **Cache Thrashing:** If the volume data is not properly interleaved or if the brick size exceeds the cache capacity, cache thrashing occurs, severely degrading performance [S1].
-   **Memory Footprint:** The main memory footprint is four times higher than standard storage because no volume data may be stored in an address space that would map outside the volume's private cache slots [S3, unofficial].
-   **Hardware Specificity:** The optimization relies heavily on the specific cache replacement strategies and page mapping of the Intel Pentium III processor [S1].

## Operating characteristics
-   **Time Complexity:** Not explicitly defined in terms of Big-O, but performance is measured in frames per second (fps) [S1][S2, unofficial][S3, unofficial].
-   **Memory Complexity:** Main memory footprint is four times that of standard contiguous volume storage due to the private cache block requirement [S3, unofficial].
-   **Measured Performance:**
    -   $256^3$ volume at 10 fps [S2, unofficial][S3, unofficial].
    -   $256 \times 256 \times 110$ volume at 1.7–7.0 fps on Pentium III 500MHz [S1].
-   **Scaling:** Performance is limited by CPU cache size and memory bandwidth. The method is designed for single-PC execution [S1].
-   **Strengths:** High cache hit ratio for semi-transparent volume rendering on CPUs; prevents cache thrashing through rigid partitioning [S1][S3, unofficial].
-   **Limitations:** High memory overhead (4x); hardware-specific (Pentium III); not suitable for very large datasets that exceed cache capacity without out-of-core techniques (though bricking helps) [S1][S3, unofficial].

## Background
UltraVis was introduced by Knittel in 2000 [S1][S3, unofficial], though one source lists 2011 [S5, unofficial]. It was designed as a highly optimized ray casting approach for semi-transparent volume rendering, specifically targeting the Intel Pentium III processor [S1]. The system addresses the major obstacle of limited memory bandwidth and cache thrashing in ray tracing-based volume rendering [S1]. It was one of the early software-based systems to demonstrate that good cache management is key to achieving fast volume rendering on commodity hardware [S2, unofficial][S3, unofficial].

## Sources
[S1] Interactive Volume Rendering with Ray Tracing, https://doi.org/10.2312/egst.20061055, fetched 2026-10-24
[S2] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S3] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (unofficial)
[S4] GPU-Based Large-Scale Scientific Visualization, https://johanna-b.github.io/LargeSciVis2018/slides/LargeScaleVisualization_Architectures.pdf, fetched 2026-10-29 (book)
