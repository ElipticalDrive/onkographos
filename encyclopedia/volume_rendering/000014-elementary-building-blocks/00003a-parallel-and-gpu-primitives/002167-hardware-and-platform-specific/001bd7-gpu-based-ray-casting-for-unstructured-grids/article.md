# GPU-based Ray-casting for Unstructured Grids
Also called: Accurate and efficient GPU ray‐casting algorithm for volume rendering of unstructured grid data

## Overview
This method is a GPU-based ray-casting algorithm designed for the volume rendering of unstructured grid data, specifically guaranteeing accurate rendering results for non-convex meshes [S1, unofficial]. It solves the problem of efficiently mapping irregular unstructured grid geometry to GPU hardware while maintaining topological accuracy, which is often lost in simpler projection methods. It sets itself apart from siblings like [CUDA Volume Ray Casting](2048-cuda-volume-ray-casting.md) by explicitly handling non-convexity through per-pixel intersection lists, and differs from [VOGUE Architecture](3149-vogue-architecture.md) by relying on a ray-casting approach rather than a voxelization-based one. It builds on [Hardware and Platform Specific](2517-hardware-and-platform-specific.md) principles by optimizing work distribution for modern GPU architectures.

## How it works
**Inputs:** Unstructured grid dataset (mesh topology and cell data); rendering parameters (camera, transfer functions).
**Outputs:** Rendered image (pixel colors).

**Data Structures:**
1.  **Intersection Lists:** A flat array representing intersection lists for all faces. This structure replaces the traditional linked-list or complex tree structures to ensure coalesced memory access on the GPU. The construction algorithm maps face intersections to this array [S1, unofficial].
2.  **Thread Block Mapping:** Rays are grouped into thread blocks based on an extended image-tile-based work distribution method. This increases ray coherency within a block [S1, unofficial].

**Steps:**
1.  **Preprocessing (Intersection List Construction):**
    *   For each pixel in the image, determine the set of grid faces intersected by the viewing ray.
    *   Construct the intersection lists for all faces across the entire image.
    *   Store these lists in a flat array optimized for GPU memory access patterns [S1, unofficial].
2.  **Work Distribution:**
    *   Assign a GPU thread to handle each ray (one thread per pixel/ray).
    *   Group threads into blocks such that rays within a block correspond to spatially adjacent pixels (tiles), maximizing memory access efficiency [S1, unofficial].
3.  **Ray Casting (Per-Pixel):**
    *   Each thread retrieves the intersection list for its assigned pixel from the flat array.
    *   Sort the intersections to ensure correct visibility ordering (critical for non-convex meshes) [S1, unofficial].
    *   Perform ray-casting integration through the sorted intersections.
    *   **Calculation Strategy:** Use an on-demand calculation strategy. Do not use a per-thread local buffer to cache intermediate results, as this is inefficient on modern GPUs. Instead, allow duplicate computations if necessary to maintain throughput [S1, unofficial].
4.  **Compositing:**
    *   Composite the optical properties (color, opacity) along the ray path using the sorted intersection data.
    *   Write the final color to the output image buffer [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Kernel executed by one thread per pixel
def ray_cast_kernel(pixel_x, pixel_y, intersection_lists, grid_data, transfer_function):
    # 1. Retrieve intersection list for this pixel
    # The intersection_lists array is pre-constructed to map pixel indices to face intersection data
    ray = get_ray_from_pixel(pixel_x, pixel_y)
    intersections = intersection_lists[pixel_y * image_width + pixel_x]
    
    # 2. Sort intersections for correct visibility
    # Essential for non-convex meshes to handle occlusion correctly
    sort_intersections(intersections)
    
    # 3. On-demand integration
    # No local caching/buffering; compute values as needed
    accumulated_color = vec3(0.0)
    accumulated_opacity = 0.0
    
    for intersection in intersections:
        # Calculate optical properties on demand
        # This may involve redundant calculations but avoids memory latency of local buffers
        color, opacity = compute_cell_properties(intersection, grid_data, transfer_function)
        
        # Standard volume rendering compositing
        # C_out = C_in + (1 - C_in_opacity) * C_cell * O_cell
        accumulated_color = accumulated_color + (1.0 - accumulated_opacity) * color * opacity
        accumulated_opacity = accumulated_opacity + (1.0 - accumulated_opacity) * opacity
        
        if accumulated_opacity >= 1.0:
            break
            
    return accumulated_color
```

**Parameters:**
*   **Image Tile Size:** Determines the block size for thread grouping. Not given in the sources.
*   **Sorting Algorithm:** Used for visibility sorting. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Non-Convex Meshes:** Standard ray casting may fail to render correctly if faces are not sorted by depth. This method explicitly addresses this via the intersection list sorting [S1, unofficial].
*   **Per-Thread Local Buffers:** Using local buffers to reduce redundant computation is inappropriate for modern GPU architectures and should be avoided [S1, unofficial].
*   **Duplicate Computations:** The on-demand strategy allows duplicate computations. This is a deliberate trade-off for performance on modern hardware [S1, unofficial].

## Subtopics

### [Pixel Ray Image (PRI)](2467-pixel-ray-image-pri.md)
Replaces the irregular mesh with a pixelized projection plane and 1D linear splines. It pre-processes geometry into a texture-friendly format, differing from this method's explicit intersection list construction.

### [GPU-based Ray Traversal on Tetrahedral Meshes](3165-gpu-based-ray-traversal-on-tetrahedral-meshes.md)
Relies on explicit tetrahedral adjacency and neighbor pointers for traversal. It builds on this method by guaranteeing accurate traversal through non-convex meshes via imaginary cells.

## Operating characteristics
*   **Performance:** Achieves up to 36.5 times higher performance for the ray-casting process and 19.7 times higher performance for the whole volume rendering process compared with the Bunyk algorithm using a CPU core [S1, unofficial].
*   **Comparison to Cell Projection:** Shows up to 8.2 times higher performance than a GPU-based cell projection method while generating more accurate rendering results [S1, unofficial].
*   **Strengths:** Guarantees accurate results for non-convex meshes; efficient memory access via flat array intersection lists; high performance on modern GPUs due to on-demand calculation and tile-based work distribution [S1, unofficial].
*   **Limitations:** Allows duplicate computations due to the on-demand strategy; requires preprocessing to construct intersection lists [S1, unofficial].
*   **Scaling:** Performance scales with GPU hardware capabilities; the method was tested on three unstructured grid datasets with different characteristics [S1, unofficial].

## Background
The method was presented in a 2020 article in the *ETRI Journal* by Gibeom Gu and Duksu Kim [S1, unofficial]. It extends prior image-tile-based work distribution methods to fit modern GPU architectures. It addresses limitations in previous GPU-based approaches, such as the inefficiency of per-thread local buffers and the inaccuracy of cell projection methods for non-convex geometries. The work is part of a broader effort to optimize volume rendering on unstructured data using GPU shaders, CUDA, and ray tracing techniques [S1, unofficial].

## Sources
[S1] Accurate and efficient GPU ray‐casting algorithm for volume rendering of unstructured grid data, https://www.researchgate.net/publication/339158888_Accurate_and_efficient_GPU_ray_casting_algorithm_for_volume_rendering_of_unstructured_grid_data, fetched 2026-10-28 (unofficial)
