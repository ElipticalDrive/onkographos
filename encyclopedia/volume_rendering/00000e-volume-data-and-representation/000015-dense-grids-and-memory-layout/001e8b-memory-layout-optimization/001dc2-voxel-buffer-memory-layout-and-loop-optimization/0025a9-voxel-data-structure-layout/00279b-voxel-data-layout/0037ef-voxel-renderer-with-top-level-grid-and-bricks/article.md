# Voxel Renderer with Top-Level Grid and Bricks

Also called: Brick-based voxel rendering with dynamic synchronization, Two-level grid structure with bricks

## Overview
This method renders voxel data using a two-level grid structure (top-level grid containing 8x8x8 voxel bricks) to optimize memory transfer and cache locality. It solves the bottleneck of transferring large voxel datasets to the GPU by synchronizing only changed bricks and leveraging high internal GPU bandwidth for staging. It differs from flat dense grid renderers by introducing spatial tiling for cache efficiency and dynamic update budgets.

## How it works
**Inputs:**
- Top-level grid data (8MB).
- List of changed bricks (max 8192 bricks/frame).
- Camera parameters for ray generation.

**Outputs:**
- Rendered image buffer.

**Data Structures:**
1.  **Top-Level Grid:** Stored as a 3D image buffer on the GPU.
2.  **Brick:** 8x8x8 voxels, 512 bytes total. Fits in a handful of cache lines.
3.  **Staging Buffer:** GPU memory buffer used for intermediate data transfer.
4.  **Destination Buffer:** GPU 3D image buffer holding the final grid layout.

**Steps:**

1.  **Data Synchronization:**
    Identify bricks that have changed since the last frame. Send these bricks to a staging buffer on the GPU. The budget is capped at 8192 bricks (512KB) per frame [S1, unofficial].

2.  **Commit Kernel:**
    Execute a GPU kernel to copy data from the staging buffer to the final destination 3D image buffer. This utilizes `clEnqueueCopyBufferToImage` (or equivalent) which reformats data during the copy [S1, unofficial].

3.  **Ray Traversal:**
    For each pixel, trace a primary ray through the screen plane. Use Amanatides and Woo’s 3D-DDA traversal algorithm to step through the top-level grid [S1, unofficial].
    -   The traversal function `TraceRay` returns the hit voxel index or 0 if the ray leaves the scene.
    -   It also returns the intersection normal.

4.  **Shading:**
    Shade the hit voxel using precalculated values for 6 unique hemispheres (corresponding to the 6 unique normals of axis-aligned voxels), integrated over a skydome [S1, unofficial].

5.  **Optional Diffuse Bounce:**
    Perform a single diffuse bounce by distributing rays over the hemisphere using blue noise with limited length [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Data Layout Constants
BRICK_SIZE = 8
BRICK_VOXELS = BRICK_SIZE * BRICK_SIZE * BRICK_SIZE # 512
BRICK_BYTES = 512
MAX_BRICKS_PER_FRAME = 8192

def synchronize_bricks(changed_bricks, staging_buffer):
    """
    Copies changed bricks to the GPU staging buffer.
    Args:
        changed_bricks: List of brick indices or data chunks.
        staging_buffer: GPU buffer handle.
    """
    # Implementation details depend on API (e.g., OpenCL clEnqueueWriteBuffer)
    # Ensure total size <= MAX_BRICKS_PER_FRAME * BRICK_BYTES
    pass

def commit_to_gpu(staging_buffer, destination_3d_image):
    """
    Copies and reformats data from staging buffer to 3D image.
    """
    # Equivalent to clEnqueueCopyBufferToImage
    pass

def trace_ray(ray_origin, ray_direction, grid_3d_image):
    """
    Traces a ray through the voxel grid using 3D-DDA.
    Returns:
        voxel_index: Index of the hit voxel, or 0 if no hit.
        normal: Intersection normal vector.
    """
    # Amanatides and Woo 3D-DDA algorithm
    # Step through the top-level grid
    # Check brick boundaries and voxel contents
    pass

def shade_voxel(voxel_index, normal):
    """
    Shades a voxel based on precalculated hemisphere integrations.
    """
    # Map normal to one of 6 unique hemispheres
    # Return precalculated color value
    pass

def render_frame(camera, grid_data, changed_bricks):
    # 1. Sync changed bricks
    synchronize_bricks(changed_bricks, staging_buffer)
    
    # 2. Commit to GPU
    commit_to_gpu(staging_buffer, destination_3d_image)
    
    # 3. Trace rays
    for y in range(height):
        for x in range(width):
            ray_origin, ray_direction = camera.generate_ray(x, y)
            voxel_index, normal = trace_ray(ray_origin, ray_direction, destination_3d_image)
            
            if voxel_index != 0:
                color = shade_voxel(voxel_index, normal)
                # Optional: Add diffuse bounce here
            else:
                color = background_color
            
            image_buffer.set_pixel(x, y, color)
```

**Parameters:**
-   Max bricks synchronized per frame: 8192 [S1, unofficial].
-   Brick size: 8x8x8 voxels [S1, unofficial].
-   Brick size in bytes: 512 bytes [S1, unofficial].

**Edge Cases:**
-   **PCIe Bandwidth:** The method relies on the fact that GPU internal copy bandwidth (~320GB/s) is significantly higher than PCIe bus bandwidth (~16GB/s for PCI 3.0 x16), making the commit kernel fast [S1, unofficial].
-   **Cache Locality:** The 8x8x8 brick size is chosen to fit in a handful of cache lines, optimizing for spatial locality regardless of ray direction [S1, unofficial].

## Operating characteristics
-   **Performance:** Up to 1G Ray/s, producing over 60fps at 1600x900 resolution on a 2070 laptop GPU [S1, unofficial].
-   **Memory Transfer:** Only changed bricks are sent to the GPU, capped at 512KB per frame (8192 bricks) [S1, unofficial].
-   **Strengths:** High cache locality due to brick tiling; efficient use of GPU internal bandwidth for data updates; simple shading model using precalculated hemispheres.
-   **Limitations:** Limited to axis-aligned voxel normals for the basic shading model; dynamic synchronization requires tracking changed bricks on the CPU [S1, unofficial].

## Background
This method was described by Jacco Bikker in a blog post titled "A Voxel Renderer for Learning C/C++" [S1, unofficial]. It is part of a series of optimizations for a voxel renderer, focusing on memory layout and data transfer efficiency.

## Sources
[S1] A Voxel Renderer for Learning C/C++ – Jacco's Blog, https://jacco.ompf2.com/2021/02/01/a-voxel-renderer-for-learning-c-c/, fetched 2026-10-24 (unofficial)
