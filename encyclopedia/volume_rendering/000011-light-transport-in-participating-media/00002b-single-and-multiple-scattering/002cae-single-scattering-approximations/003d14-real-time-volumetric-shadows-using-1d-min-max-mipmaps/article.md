# Real-Time Volumetric Shadows using 1D Min-Max Mipmaps

Also called: 1D Min-Max Mipmap Volumetric Shadows

## Overview
This method computes real-time volumetric shadows for single-scattering participating media on the GPU by treating the shadow map as a 2D height field and reducing the integration problem to 1D via epipolar rectification [S1]. It differs from incremental integration methods [S1] by allowing parallel processing of all camera rays using only DirectX 9 pixel shader features, and from shadow volume methods [S1] by looping over pixels rather than volume elements, thereby supporting high-resolution shadow maps and textured lights without GPGPU APIs [S1].

## How it works
The algorithm evaluates the single-scattering integral by finding lit segments along camera rays intersecting a shadow map. It relies on the single-scattering approximation defined in [Single Scattering Approximations](5340-single-scattering-approximations.md).

### Inputs and Outputs
*   **Inputs:** Camera position and orientation; Light position and orientation; Volumetric density field $\sigma_t(\mathbf{x})$ (typically homogeneous or simple heterogeneous); Shadow map from the light; Depth map from the camera.
*   **Outputs:** Final color buffer with volumetric shadows.

### Data Structures
1.  **Shadow Map:** A 2D texture storing depth from the light's perspective.
2.  **Rectified Shadow Map:** A 2D texture where each row corresponds to an epipolar slice.
3.  **1D Min-Max Mipmap:** For each row of the rectified shadow map, a mipmap structure storing the minimum and maximum depth values for blocks of texels. This allows parallel traversal to find lit segments.

### Steps
1.  **Render Depth and Shadow Maps:** Render the scene depth from the camera and the shadow map from the light [S1].
2.  **Epipolar Rectification:** Transform the shadow map such that each camera ray intersects only a single row of the rectified shadow map (an epipolar slice) [S1]. This reduces the 2D intersection problem to a 1D problem per row [S1].
3.  **Low-Rank Approximation:** Compute a low-rank approximation for the non-visibility terms in the scattering integral (e.g., phase function, geometric attenuation) to simplify the per-segment accumulation [S1].
4.  **Build 1D Min-Max Mipmaps:** For each row of the rectified shadow map, construct a 1D min-max mipmap. Each level of the mipmap stores the min and max depth values for blocks of texels from the previous level [S1].
5.  **Traverse and Accumulate:** For each camera ray (pixel):
    *   Traverse the 1D min-max mipmap of the corresponding epipolar row to identify lit segments (intervals where the ray is not occluded by the shadow map geometry) [S1].
    *   Accumulate the scattering integral over these lit segments [S1].
6.  **Epipole Handling:** If the epipole (the projection of the light position onto the image plane) is on or near the screen, compute scattering near the epipole using brute-force ray marching to avoid artifacts from the rectification or mipmap approximation [S1].

### Pseudocode
The following pseudocode outlines the core traversal and accumulation logic for a single camera ray, assuming the rectified shadow map and its mipmap are available.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_volumetric_shadow(ray_origin, ray_direction, shadow_map_rectified, mipmap_levels, sigma_t, phase_func, light_pos):
    """
    Computes the scattered radiance for a single ray using 1D min-max mipmap traversal.
    
    Args:
        ray_origin: 3D vector, camera origin.
        ray_direction: 3D vector, normalized ray direction.
        shadow_map_rectified: 2D array, depth values in rectified coordinates.
        mipmap_levels: List of 2D arrays, each level stores (min_depth, max_depth) for blocks.
        sigma_t: float, extinction coefficient of the medium.
        phase_func: function, phase function p(wi, wo).
        light_pos: 3D vector, position of the light source.
        
    Returns:
        float, accumulated scattered radiance.
    """
    # 1. Intersect ray with the rectified shadow map plane to get the 1D interval [t_start, t_end]
    #    The rectified shadow map is defined in a coordinate system where the epipolar slice is a line.
    #    Let the intersection with the shadow map plane be at distance t_plane.
    #    The shadow map covers a range of u coordinates in the rectified space.
    #    For simplicity, assume we have already mapped the ray to the rectified 1D domain [u_min, u_max]
    #    and the corresponding distances along the ray [t_min, t_max].
    
    t_min, t_max = get_ray_interval_in_rectified_space(ray_origin, ray_direction)
    
    if t_min > t_max:
        return 0.0

    # 2. Traverse the min-max mipmap to find lit segments
    lit_segments = []
    current_t = t_min
    
    # Traverse from coarsest to finest level to find exact lit intervals
    # This is a simplified traversal; actual implementation may vary
    for level in reversed(mipmap_levels):
        # Determine the block index in the current level corresponding to current_t
        block_idx = get_block_index(current_t, level)
        min_depth, max_depth = level[block_idx]
        
        # Check if the ray segment through this block is potentially lit
        # If the min_depth of the block is greater than the ray's current t, 
        # the block is occluded. If max_depth is less than the ray's t, it's behind.
        # However, since we are traversing along the ray, we check visibility against the shadow map depth.
        
        # If the block's max_depth is less than the ray's entry point into the block, 
        # the entire block is behind the geometry (occluded).
        if max_depth < current_t:
            continue # Skip this block, it's occluded
        
        # If the block's min_depth is greater than the ray's exit point from the block,
        # the entire block is in front (lit, but we need to check finer levels for occlusion)
        # Actually, min-max mipmap is used to find *occluded* regions.
        # If min_depth > current_t, the geometry is in front, so the ray is occluded?
        # No, shadow map stores depth from light. 
        # Let's assume standard shadow mapping logic: 
        # A point is lit if its depth from light < shadow_map_depth.
        # Here, we are finding segments where the ray is NOT occluded.
        
        # Simplified logic: 
        # If the block contains any lit portion, we drill down.
        # If the block is fully occluded, we skip it.
        # If the block is fully lit, we add it.
        
        # For this pseudocode, we assume a helper that returns lit segments
        segments = find_lit_segments_in_block(current_t, t_max, min_depth, max_depth, shadow_map_rectified, level)
        lit_segments.extend(segments)
        
        # Update current_t to skip processed segments
        if segments:
            current_t = segments[-1].t_end
        else:
            current_t = t_max # No more lit segments in this direction
            
        if current_t >= t_max:
            break

    # 3. Accumulate scattering over lit segments
    scattered_radiance = 0.0
    for segment in lit_segments:
        t_start, t_end = segment.t_start, segment.t_end
        
        # Compute transmittance from camera to segment start
        T_cam = exp(-sigma_t * (t_start - t_min))
        
        # Compute transmittance from light to segment midpoint (approximation)
        # Or integrate along the segment if density varies
        mid_t = (t_start + t_end) / 2.0
        mid_point = ray_origin + mid_t * ray_direction
        dist_to_light = length(mid_point - light_pos)
        T_light = exp(-sigma_t * dist_to_light) # Simplified
        
        # Incident radiance from light (assuming point light with intensity I)
        L_in = I / (dist_to_light ** 2) * phase_func(ray_direction, normalize(light_pos - mid_point))
        
        # Scattering contribution
        # Integral of T_cam(x) * sigma_s * L_in(x) dx
        # Assuming homogeneous medium and constant L_in over small segment
        segment_length = t_end - t_start
        T_seg = exp(-sigma_t * segment_length) # Transmittance through segment
        
        # Approximate integral: L_in * sigma_s * T_cam * (1 - T_seg) / sigma_t
        # sigma_s is assumed equal to sigma_t for simplicity, or provided
        sigma_s = sigma_t # [G] Assuming single scattering coefficient equals extinction for homogeneous medium
        contribution = L_in * sigma_s * T_cam * (1.0 - T_seg) / sigma_t
        
        scattered_radiance += contribution

    return scattered_radiance
```

### Parameters and Edge Cases
*   **Mipmap Levels:** The number of levels depends on the shadow map resolution. More levels provide better precision but increase memory and traversal time [S1].
*   **Epipole:** When the epipole is near the screen, the rectification causes severe distortion or infinite depth values. Brute-force ray marching is used in this region [S1].
*   **Precision:** Floating-point precision in depth comparisons can cause z-fighting or gaps in lit segments. [G] A small epsilon should be added to depth comparisons.
*   **Aliasing:** High-resolution shadow maps are supported, but aliasing may occur if the mipmap levels are not fine enough. [S1]

## Operating characteristics
*   **Time Complexity:** $O(N \cdot \log M)$ per pixel, where $N$ is the number of pixels and $M$ is the shadow map resolution, due to mipmap traversal [S1].
*   **Memory Complexity:** $O(M)$ for the shadow map and mipmap structures [S1].
*   **Performance:** Achieves real-time performance on GPUs supporting DirectX 9 pixel shaders [S1]. Outperforms incremental integration methods [S1] and shadow volume methods [S1] in complex scenes like forests [S1].
*   **Strengths:** Parallel processing of all rays; no GPGPU required; supports textured lights; handles high-resolution shadow maps [S1].
*   **Limitations:** Sacrifices worst-case guarantees compared to incremental integration [S1]; requires epipolar rectification; brute-force marching needed near epipole [S1].

## Background
The method was introduced by [S1] as a real-time algorithm for volumetric shadows. It builds on the use of min-max mipmaps for soft shadows [Guennebaud et al. 2006] and global illumination [Nichols and Wyman 2009], adapting them for single-scattering volumetric shadows [S1]. It addresses the limitations of previous methods like Baran et al. [2010] and Billeter et al. [2010] by enabling parallel pixel-based processing on standard GPU shaders [S1].

## Sources
[S1] Real-Time Volumetric Shadows using 1D Min-Max Mipmaps, https://groups.csail.mit.edu/graphics/mmvs/mmvs.pdf, fetched 2026-10-28
