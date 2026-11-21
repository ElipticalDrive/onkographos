# Dynamic Warp Marching

Also called: 3D Warp Marching, Dynamic approach

## Overview
Dynamic Warp Marching is a GPU sampling strategy for direct volume ray casting that optimizes texture cache performance by dynamically aligning the sampling direction of a thread warp with the volume's X-dimension, which has the minimum memory stride [S1]. It integrates the Warp Marching concept with a dynamic viewing-direction adjustment, overcoming the limitation of the original Dynamic approach which could only sample along horizontal or vertical directions [S1].

## How it works
**Inputs:**
*   Volume data texture [S1].
*   Viewing direction (defined by the camera orientation relative to the volume axes) [S1].
*   Ray origin and direction for each pixel [S1].

**Outputs:**
*   Sampled voxel values along the ray path [S1].
*   Composited color and opacity for the pixel [S1].

**Data Structures:**
*   **Warp Shape:** $32 \times 1$. A single warp (32 threads) is responsible for one ray, with all 32 threads taking samples along the X-dimension of the volume [S1].
*   **Thread Block Shape:** Not given in the sources for the Dynamic method [S1].

**Steps:**
1.  **Determine Viewing Direction:** Identify the current viewing direction relative to the volume axes (e.g., XY, YZ, ZX, etc.) [S1].
2.  **Adjust Sampling Direction:** Unlike the Static method which aligns the major sampling direction with the horizontal direction of the view, the Dynamic method adjusts the major direction of the warp to always be parallel to the X-dimension of the volume [S1].
3.  **Sample Along X:** All 32 threads in the warp take samples along the X-dimension of the volume [S1]. This is done because the X-dimension has the minimum memory stride, ensuring best texture cache performance [S1].
4.  **Integrate Warp Marching:** The method integrates Warp Marching to allow sampling along any direction, overcoming the limitation of Standard sampling which only supports horizontal or vertical directions [S1].

**Formulas:**
*   Number of voxels covered along a dimension = (distance between samples along that dimension) × (number of samples taken along that dimension) [S1].
*   Let $V_d$ be the voxels covered, $D_s$ be the sampling distance, and $N_s$ be the number of samples [S1]. Then $V_d = D_s \times N_s$ [S1].

**Parameters:**
*   Warp shape: $32 \times 1$ [S1].
*   Block shape: Not given in the sources.
*   Sampling distance: 0.5 voxel length between samples along rays [S1].

**Edge Cases and Pitfalls:**
*   **Viewing Direction XY or XZ:** When the view is facing XY or XZ, the major warp sampling direction of the Static method is already aligned with the X-dimension of the volume. Therefore, the performance of the Static and Dynamic approaches is similar [S1].
*   **Viewing Direction YX:** The Dynamic approach is better than the Static approach when facing YX [S1].
*   **Ray Distance/Volume Image Ratio:** The performance difference between Dynamic and Static approaches is notable in datasets with larger ray distances (e.g., `vhm_body`) but marginal in datasets with smaller ray distances (e.g., `vhf_head`) [S1]. This is due to different volume image ratios [S1].
*   **Zero X-direction component:** If the ray direction's X-component is zero, stepping along X requires a fallback to another axis or a small epsilon to avoid division by zero [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for Dynamic Warp Marching
# Assumes a warp of 32 threads, each thread handles one sample along the X-dimension
# The ray direction and origin are shared among the warp threads

def dynamic_warp_marching(ray_origin, ray_direction, volume_texture, volume_dims):
    # Initialize compositing state [G]
    color = [0.0, 0.0, 0.0]
    opacity = 0.0
    alpha_contribution = 0.0

    # Step size along X dimension (0.5 voxel length per evidence) [S1]
    step_x = 0.5
    dx = ray_direction[0]
    dy = ray_direction[1]
    dz = ray_direction[2]

    # Handle zero X-component to avoid division by zero [G]
    if abs(dx) < 1e-6:
        dx = 1e-6

    # Calculate step in t and corresponding Y/Z increments
    step_t = step_x / dx
    step_y = dy * step_t
    step_z = dz * step_t

    # Current position
    pos_x = ray_origin[0]
    pos_y = ray_origin[1]
    pos_z = ray_origin[2]

    # Iterate through 32 samples (warp size) [S1]
    for i in range(32):
        # Check volume boundaries
        if not (0 <= pos_x < volume_dims[0] and 0 <= pos_y < volume_dims[1] and 0 <= pos_z < volume_dims[2]):
            break

        # Sample voxel value (standard texture fetch operation) [G]
        voxel = volume_texture.sample(pos_x, pos_y, pos_z)

        # Standard front-to-back compositing [G]
        alpha_contribution = voxel * (1.0 - opacity)
        color[0] += voxel[0] * alpha_contribution
        color[1] += voxel[1] * alpha_contribution
        color[2] += voxel[2] * alpha_contribution
        opacity += alpha_contribution

        # Early ray termination [S1]
        if opacity >= 0.95:
            break

        # Advance position along X dimension
        pos_x += step_x
        pos_y += step_y
        pos_z += step_z

    return color, opacity
```

## Operating characteristics
*   **Performance:** Significant performance improvements when the view is facing YZ, ZX, or ZY [S1]. When facing XY or XZ, performance is similar to the Static method [S1]. When facing YX, the Dynamic approach is better than the Static approach [S1].
*   **Scaling:** Performance difference is notable in datasets with larger ray distances (e.g., `vhm_body`) but marginal in datasets with smaller ray distances (e.g., `vhf_head`) [S1].
*   **Strengths:** Optimizes texture cache performance by aligning sampling with the X-dimension, which has the minimum memory stride [S1]. Allows sampling along any direction by integrating Warp Marching [S1].
*   **Limitations:** Performance gain is dependent on the viewing direction and dataset characteristics (ray distance/volume image ratio) [S1].

## Background
The Dynamic approach was introduced by Sugimoto et al. (2014) to adjust sampling direction according to viewing direction for best texture cache performance [S1]. The method integrates Warp Marching to overcome the limitation of the original Dynamic approach, which could only sample along horizontal or vertical directions due to the limitation of Standard sampling [S1].

## Sources
[S1] A cache-friendly sampling strategy for texture-based volume rendering on GPU, https://doi.org/10.1016/j.visinf.2017.08.001, fetched 2026-10-26

## Subtopics

### Static Warp Marching
Uses Standard sampling with a fixed thread block shape ($16 \times 16$) and aligns the major sampling direction with the horizontal direction of the view. Choose when the viewing direction is already aligned with the X-dimension (XY or XZ views) for similar performance to Dynamic. [Static Warp Marching](4299-static-warp-marching.md)

### Standard Sampling
Only supports horizontal or vertical sampling directions. Choose when dynamic adjustment is not needed or when the viewing direction is fixed and favorable. [Standard Sampling](4297-standard-sampling.md)


