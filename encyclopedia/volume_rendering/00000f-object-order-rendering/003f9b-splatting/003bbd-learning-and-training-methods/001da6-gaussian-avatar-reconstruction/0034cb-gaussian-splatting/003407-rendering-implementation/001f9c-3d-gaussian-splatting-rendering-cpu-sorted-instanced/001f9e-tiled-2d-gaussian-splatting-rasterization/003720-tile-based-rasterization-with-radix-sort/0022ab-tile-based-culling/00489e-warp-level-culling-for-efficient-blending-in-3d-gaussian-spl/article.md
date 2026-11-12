# Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting

Also called: Warp-level sub-tile culling, Faster-GS optimization

## Overview
An optimization for the 3D Gaussian Splatting (3DGS) alpha blending stage that uses warp-level collaboration to subdivide image tiles and cull splats with no geometric overlap, avoiding expensive alpha evaluation [S1]. It improves rendering speed without changing the rendered output, implemented in the existing CUDA differentiable pipeline [S1]. Unlike PUP 3D-GS, Sort-Free Gaussians, or VKRayGS which sacrifice image quality or differentiability, this method maintains identical visual fidelity and stays in the CUDA differentiable framework [S1].

## How it works
The method optimizes the inference pipeline of 3DGS, which consists of four stages: (1) Preprocessing via `preprocess_cu` to select and project Gaussians; (2) Depth Sorting via CUB Device Radix Sort; (3) Instancing to divide the image into tiles and duplicate splats; (4) Blend via `blend_cu` for alpha-over compositing [S1]. The optimization targets the fourth stage, the `blend_cu` kernel, which is identified as one of the longest-running stages in the inference pipeline [S1].

**Inputs and Outputs:**
*   **Input:** A set of 2D Gaussian splats assigned to a tile, sorted by depth, with attributes including position, covariance, opacity, and spherical harmonics coefficients.
*   **Output:** The final color and opacity for each pixel in the tile.

**Data Structures:**
*   **Tile:** A rectangular region of the image, typically $16 \times 16$ pixels by default [S1].
*   **Splat Batch:** A group of splats processed together, typically in batches of 256 splats [S1].
*   **Warp:** A group of 32 threads in a CUDA thread block, mapped to a sub-tile or pixel group for collaborative processing [S1].

**Parameters:**
*   **Tile Size:** $16 \times 16$ pixels [S1].
*   **Batch Size:** 256 splats [S1].
*   **Contribution Threshold:** 1256 (minimum 8-bit RGB color value) used during instancing to cull splats with opacity below this level [S1].

**Algorithm Steps:**
1.  **Tile Subdivision:** The image tile is subdivided into smaller regions corresponding to warp-level parallelism. Each warp is responsible for a subset of pixels within the tile [S1].
2.  **Geometric Culling:** For each warp, splats are checked for geometric overlap with the pixels assigned to that warp. Splats with no geometric overlap are culled [S1].
3.  **Alpha Evaluation Avoidance:** By culling splats with no overlap, expensive alpha evaluation and compositing operations are avoided for those splats [S1].
4.  **Alpha-Over Compositing:** The remaining splats are composited using the alpha-over equation for each pixel in the warp's assigned region [S1].
5.  **Shared Memory Rearrangement:** Splat data is rearranged into optimized structures to facilitate efficient movement through shared memory, reducing memory transfer bottlenecks [S1].
6.  **Batch Processing:** The kernel processes splats in batches of 256, using synchronization fences between data fetch and render stages [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the optimized blend_cu kernel
# Assumes a CUDA-like execution model where threads are organized in warps
# and blocks map to tiles.

def warp_level_culling_blend(tile_splats, tile_pixels, tile_size=16):
    """
    Perform alpha-over compositing with warp-level culling.
    
    Args:
        tile_splats: List of splats assigned to this tile, sorted by depth.
        tile_pixels: List of pixel coordinates in the tile.
        tile_size: Size of the tile in pixels (default 16x16).
    
    Returns:
        Final color and opacity for each pixel in the tile.
    """
    # Get thread and warp identifiers
    warp_id = get_warp_id()
    lane_id = get_lane_id()
    
    # Subdivide tile pixels among warps
    # Each warp handles a subset of pixels
    warp_pixels = get_warp_pixels(tile_pixels, warp_id, tile_size)
    
    # Initialize output buffers for this warp
    out_color = zeros(len(warp_pixels), dtype=float32)
    out_opacity = zeros(len(warp_pixels), dtype=float32)
    
    # Rearrange splat data for shared memory efficiency [S1]
    # This step optimizes memory access patterns
    shared_splat_data = rearrange_for_shared_memory(tile_splats)
    
    # Process splats in batches
    batch_size = 256
    for i in range(0, len(tile_splats), batch_size):
        batch = tile_splats[i : i + batch_size]
        
        # Warp-level geometric culling
        # Cull splats that do not overlap with any pixel in this warp's subset
        active_splats = []
        for splat in batch:
            if has_geometric_overlap(splat, warp_pixels):
                active_splats.append(splat)
        
        # Alpha-over compositing for active splats
        for j, pixel in enumerate(warp_pixels):
            alpha_accum = 0.0
            color_accum = vec3(0.0)
            
            for splat in active_splats:
                # Project splat to pixel space
                proj_splat = project_splat(splat, pixel)
                
                # Evaluate Gaussian at pixel
                alpha = evaluate_opacity(proj_splat)
                color = evaluate_color(proj_splat)
                
                # Alpha-over compositing
                # C_out = C_s * alpha + C_b * (1 - alpha)
                # A_out = alpha + A_b * (1 - alpha)
                weight = alpha * (1.0 - alpha_accum)
                color_accum += color * weight
                alpha_accum += weight
                
                if alpha_accum >= 1.0:
                    break
            
            out_color[j] = color_accum
            out_opacity[j] = alpha_accum
    
    return out_color, out_opacity

def has_geometric_overlap(splat, pixels):
    """
    Check if a splat has any geometric overlap with a set of pixels.
    This is a simplified check; in practice, it involves bounding box or
    ellipse intersection tests.
    """
    # Check if the splat's bounding box intersects with the bounding box
    # of the pixels assigned to the warp
    splat_bbox = get_splat_bounding_box(splat)
    pixel_bbox = get_pixel_bounding_box(pixels)
    return intersect(splat_bbox, pixel_bbox)
```

**Edge Cases and Pitfalls:**
*   **Precision:** Standard floating-point precision is used for alpha-over compositing. No specific precision issues are noted, but standard IEEE 754 rules apply [S1].
*   **Boundaries:** The method assumes tiles are fully contained within the image. Edge tiles may require special handling if the image dimensions are not multiples of the tile size, though this is not explicitly detailed [S1].
*   **Degenerate Input:** Splats with zero opacity or zero area are effectively culled by the geometric overlap check or the contribution threshold [S1].

## Operating characteristics
*   **Time Complexity:** The method reduces the number of splats processed per pixel by culling those with no geometric overlap, thereby reducing the computational load of the alpha blending kernel [S1]. The baseline alpha-blending GPU kernel performs two to three times slower than the optimistic theoretical bound, and this optimization aims to close that gap [S1].
*   **Memory Complexity:** Memory usage is similar to the baseline, with optimizations focused on shared memory rearrangement to improve throughput [S1].
*   **Measured Performance:** On NVIDIA RTX 4090, the method is up to 19% faster than Faster-GS. On NVIDIA RTX 2070 Super, it is up to 32% faster. Across all tested GPUs (RTX 4090, RTX 3090, RTX 2070 Super), the changes reduce frame time over Faster-GS by 17% (+20% FPS) on average (geometric mean). On the RTX 2070 Super specifically, the frame time reduction is up to 24% (+32% FPS) [S1].
*   **Strengths:** Maintains identical visual fidelity and differentiability. Easy to integrate into existing CUDA pipelines. Significant speedup on various GPU architectures.
*   **Limitations:** Requires careful implementation of warp-level synchronization and shared memory management. Performance gains may vary depending on scene complexity and Gaussian distribution.
*   **When to Use:** When rendering speed is critical and identical output to standard 3DGS is required. Suitable for real-time applications and training pipelines where differentiability is needed.
*   **When Not to Use:** If maximum possible speed is required at the cost of image quality or differentiability, other methods like PUP 3D-GS or Sort-Free Gaussians might be considered, though they sacrifice fidelity [S1].

## Background
3D Gaussian Splatting (3DGS) [Kerbl et al. 2023] has rapidly become a dominant technique for novel view synthesis, with literally thousands of follow-up methods that use and extend the standard CUDA pipeline for rendering [S1]. Despite this vast body of work, there is still room for improvement in rendering speed [S1]. This method was presented as an optimization to the Faster-GS inference pipeline, focusing on the alpha blending stage [S1]. The code implementation can be found in the supplemental materials and can be used in the ICG Neural Radiance Fields and Novel View Synthesis Framework (NeRFICG) [Kappel et al. 2026], like Faster-GS [S1].

## Sources
[S1] Warp-Level Culling for Efficient Blending in 3D Gaussian Splatting, https://doi.org/10.1145/3820019, fetched 2026-11-01
