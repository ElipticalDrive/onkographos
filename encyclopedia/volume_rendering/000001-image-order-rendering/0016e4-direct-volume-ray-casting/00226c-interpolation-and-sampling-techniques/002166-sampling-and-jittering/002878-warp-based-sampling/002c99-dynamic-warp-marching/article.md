# Dynamic Warp Marching

Also called: Dynamic approach

## Overview
Dynamic Warp Marching is a GPU sampling strategy for direct volume ray casting that optimizes texture cache performance by dynamically aligning the sampling direction of a thread warp with the volume's X-dimension, which has the minimum memory stride [S1]. It integrates the Warp Marching concept with a dynamic viewing-direction adjustment, overcoming the limitation of the original Dynamic approach which could only sample along horizontal or vertical directions [S1].

## How it works
**Inputs:**
*   Volume data texture.
*   Viewing direction (defined by the camera orientation relative to the volume axes).
*   Ray origin and direction for each pixel.

**Outputs:**
*   Sampled voxel values along the ray path.
*   Composited color and opacity for the pixel.

**Data Structures:**
*   **Warp Shape:** $32 \times 1$. A single warp (32 threads) is responsible for one ray, with all 32 threads taking samples along the X-dimension of the volume [S1].
*   **Thread Block Shape:** Not explicitly defined for the Dynamic method in the sources, but the Static method uses $16 \times 16$ [S1]. The Dynamic method's warp shape is fixed at $32 \times 1$ [S1].

**Steps:**
1.  **Determine Viewing Direction:** Identify the current viewing direction relative to the volume axes (e.g., XY, YZ, ZX, etc.). There are six unique viewing directions considered [S1].
2.  **Adjust Sampling Direction:** Unlike the Static method which aligns the major sampling direction with the horizontal direction of the view, the Dynamic method adjusts the major direction of the warp to always be parallel to the X-dimension of the volume [S1].
3.  **Sample Along X:** All 32 threads in the warp take samples along the X-dimension of the volume [S1]. This is done because the X-dimension has the minimum memory stride, ensuring best texture cache performance [S1].
4.  **Integrate Warp Marching:** The method integrates Warp Marching to allow sampling along any direction, overcoming the limitation of Standard sampling which only supports horizontal or vertical directions [S1]. This implies that while the *warp's* internal sampling is along X, the *ray's* path through the volume is determined by the viewing direction, and the warp's threads are mapped to cover the ray's path efficiently [S1].

**Parameters:**
*   Warp shape: $32 \times 1$ [S1].
*   Block shape: Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Viewing Direction XY or XZ:** When the view is facing XY or XZ, the major warp sampling direction of the Static method is already aligned with the X-dimension of the volume. Therefore, the performance of the Static and Dynamic approaches is similar [S1].
*   **Viewing Direction YX:** The Dynamic approach is better than the Static approach when facing YX [S1].
*   **Ray Distance/Volume Image Ratio:** The performance difference between Dynamic and Static approaches is notable in datasets with larger ray distances (e.g., `vhm_body`) but marginal in datasets with smaller ray distances (e.g., `vhf_head`) [S1]. This is due to different volume image ratios [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Dynamic Warp Marching
# Assumes a warp of 32 threads, each thread handles one sample along the X-dimension
# The ray direction and origin are shared among the warp threads

def dynamic_warp_marching(ray_origin, ray_direction, volume_texture):
    # Determine the viewing direction relative to volume axes
    viewing_direction = get_viewing_direction(ray_direction)
    
    # Adjust the sampling direction to be parallel to the X-dimension
    # This is the key difference from Static method
    sampling_direction = Vector(1, 0, 0) # X-dimension
    
    # Initialize sample position
    sample_pos = ray_origin
    
    # Iterate through 32 samples (warp size)
    for i in range(32):
        # Sample along the X-dimension
        # Note: The actual ray path is determined by the viewing direction,
        # but the warp's threads are mapped to cover the ray's path efficiently
        # by sampling along X. This implies a transformation or mapping
        # of the ray's path to the X-dimension for cache efficiency.
        # The exact mapping is not detailed in the sources, but the
        # integration of Warp Marching allows sampling along any direction.
        
        # Get voxel value at sample_pos
        voxel_value = sample_volume(volume_texture, sample_pos)
        
        # Composite the voxel value
        # (Standard volume rendering compositing step)
        color, opacity = composite(voxel_value, color, opacity)
        
        # Move to next sample along the ray direction
        # The step size is adjusted to cover the ray path efficiently
        step_size = get_step_size(ray_direction, viewing_direction)
        sample_pos = sample_pos + ray_direction * step_size
        
        # Check for exit condition (e.g., end of volume or sufficient opacity)
        if is_exit_condition(sample_pos, opacity):
            break
            
    return color, opacity
```

## Subtopics

### Static Warp Marching
Uses Standard sampling with a fixed thread block shape ($16 \times 16$) and aligns the major sampling direction with the horizontal direction of the view. Choose when the viewing direction is already aligned with the X-dimension (XY or XZ views) for similar performance to Dynamic. [Static Warp Marching](4299-static-warp-marching.md)

### Standard Sampling
Only supports horizontal or vertical sampling directions. Choose when dynamic adjustment is not needed or when the viewing direction is fixed and favorable. [Standard Sampling](4297-standard-sampling.md)

## Operating characteristics
*   **Performance:** Significant performance improvements when the view is facing YZ, ZX, or ZY [S1]. When facing XY or XZ, performance is similar to the Static method [S1]. When facing YX, the Dynamic approach is better than the Static approach [S1].
*   **Scaling:** Performance difference is notable in datasets with larger ray distances (e.g., `vhm_body`) but marginal in datasets with smaller ray distances (e.g., `vhf_head`) [S1].
*   **Strengths:** Optimizes texture cache performance by aligning sampling with the X-dimension, which has the minimum memory stride [S1]. Allows sampling along any direction by integrating Warp Marching [S1].
*   **Limitations:** Performance gain is dependent on the viewing direction and dataset characteristics (ray distance/volume image ratio) [S1].

## Background
The Dynamic approach was introduced by Sugimoto et al. (2014) to adjust sampling direction according to viewing direction for best texture cache performance [S1]. The method integrates Warp Marching to overcome the limitation of the original Dynamic approach, which could only sample along horizontal or vertical directions due to the limitation of Standard sampling [S1].

## Sources
[S1] A cache-friendly sampling strategy for texture-based volume rendering on GPU, https://doi.org/10.1016/j.visinf.2017.08.001, fetched 2026-10-26
