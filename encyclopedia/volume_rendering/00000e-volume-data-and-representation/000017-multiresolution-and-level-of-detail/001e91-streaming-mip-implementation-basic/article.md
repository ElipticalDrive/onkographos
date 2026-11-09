# Streaming MIP Implementation (Basic)

Also called: Streaming MIP

## Overview
Streaming MIP is a multiresolution volume rendering technique that constructs a progressive coefficient stream from a morphological pyramid to allow interactive frame rates on large datasets that do not fit in GPU memory [S1]. It solves the problem of discrete quality jumps in level-by-level rendering by resorting detail coefficients from all pyramid levels simultaneously based on decreasing magnitude of an error measure, providing near-continuous error control [S1]. It sets itself apart from basic texture-based MIP by storing only necessary voxels and from level-by-level MMIP by offering finer granularity in approximation error control [S1].

## How it works
**Inputs:** A morphological pyramid of detail coefficients $\{d_j(x, y, z)\}$ sorted by decreasing magnitude of an error measure (typically L1-error reduction) [S1].
**Outputs:** A rendered image approximating the Maximum Intensity Projection (MIP) of the volume.

**Data Structures:**
The implementation requires storing detail coefficients with specific attributes to support level-dependent dilation.
*   **Position Attribute:** 4 bytes. Contains the spatial coordinates $(x, y, z)$ and the originating pyramid level $j$ encoded in the lower two bits [S1].
*   **Intensity Attribute:** 2 bytes (short). Stores the voxel intensity value [S1].
*   **Total Size:** 6 bytes per voxel [S1].

**Algorithm Steps:**
1.  **Preprocessing:** Construct the morphological pyramid and sort all detail coefficients $\{d_j\}$ by decreasing error magnitude [S1].
2.  **Initialization:** Initialize the frame buffer (or output texture) with the minimum possible intensity value (e.g., 0 or background) [G].
3.  **Iteration:** Iterate through the sorted list of coefficients. For each coefficient $k$ with value $val$ and pyramid level $j$:
    a.  **Projection:** Project the coefficient to the 2D screen space based on the current view direction [S1].
    b.  **Local Dilation:** Determine the dilation neighborhood based on level $j$. For a square structuring element of size $2 \times 2$, the dilation size is $2^j$ (e.g., size 1 at level 0, size 2 at level 1, size 4 at level 2) [S1].
    c.  **Update:** For all pixels in the dilated neighborhood around the projection point:
        i.   Compare the coefficient value $val$ with the current frame buffer value $curval$ at that pixel.
        ii.  If $curval < val$, overwrite $curval$ with $val$ [S1].
    d.  **Termination:** Stop when a maximum number of coefficients is projected or a desired accuracy threshold is met [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def streaming_mip_basic(coefficients, frame_buffer, view_direction):
    """
    coefficients: List of tuples (level, intensity, x, y, z), sorted by error magnitude descending.
    frame_buffer: 2D array initialized to minimum value.
    view_direction: Vector defining the projection.
    """
    for level, intensity, x, y, z in coefficients:
        # Project 3D point to 2D screen coordinates
        sx, sy = project_to_screen(x, y, z, view_direction)
        
        # Determine dilation size based on pyramid level
        # Assuming square 2x2 structuring element
        dilation_size = 2 ** level
        
        # Define neighborhood bounds
        min_x = int(sx) - dilation_size
        max_x = int(sx) + dilation_size
        min_y = int(sy) - dilation_size
        max_y = int(sy) + dilation_size
        
        # Clip to frame buffer boundaries
        min_x = max(0, min_x)
        max_x = min(frame_buffer.width, max_x)
        min_y = max(0, min_y)
        max_y = min(frame_buffer.height, max_y)
        
        # Update frame buffer with maximum operator
        for px in range(min_x, max_x):
            for py in range(min_y, max_y):
                if frame_buffer[px, py] < intensity:
                    frame_buffer[px, py] = intensity
```

**Parameters:**
*   **Structuring Element:** Typically a square $2 \times 2$ element [S1].
*   **Point Size:** Mapped to hardware point size extension (PSIZE in Cg) to implement dilation [S1].
*   **Point Sprite:** Used if the projection of the structuring element is not square [S1].

**Edge Cases and Pitfalls:**
*   **Global Dilation:** Cannot be used because the scale index varies per coefficient, unlike level-by-level projection where the scale is constant per level [S1].
*   **Performance:** Voxels are no longer ordered by intensity, requiring writes to the frame buffer to use the maximum operator (blended primitives), causing some performance loss compared to ordered alpha blending [S1].
*   **Memory:** The 6-byte overhead per voxel increases memory usage compared to standard 4-byte representations [S1].

## Subtopics

### Optimized Streaming MIP
This variant avoids the drawbacks of the basic implementation (point sprites, frame buffer blending, 6-byte overhead) by using a histogram-based selection strategy [S1]. It uses only 4 bytes per voxel and avoids point sprites [S1]. The error-sorted list is used only to guide the error for speed/quality tradeoff, not strictly for rendering order [S1]. A second histogram-sorting step groups coefficients by (level, intensity) pairs, maintaining error ordering within bins [S1]. Rendering traverses the sorted list for a chosen percentage of coefficients, rendering voxels in low-to-high intensity order per level [S1].

### Advanced Algorithm with Mipmap Fallback
An approximation method where missing data at the needed resolution triggers a fallback to a higher mipmap level, resorting to iterative interrupt/resume only if no higher level is present or data is too coarse [S2, unofficial]. Fixing a budget for upload delivers a relatively stable frame rate [S2, unofficial].

### Center-Precomputed MIP Approximation
Reduces per-intersection cost compared to per-pixel Jacobian evaluation without visual loss in filtering quality [S3, unofficial]. Uses the formula $\rho_{\text{mip}}^2 = \Sigma_{uu}^{-1} u^2 + 2 \Sigma_{uv}^{-1} uv + \Sigma_{vv}^{-1} v^2$ and weight $w = n_f \cdot \exp(-0.5 \cdot \rho_{\text{mip}}^2)$ [S3, unofficial].

### Sparse Volume Texture Mip Mapping
Supports mip levels in Sparse Volume Texture Sample nodes, allowing for level-of-detail management similar to other texture types [S4]. Setting the MipLevel input pin attempts to sample at a specific mip level provided it has been streamed in, but streaming itself is not driven from there [S4]. If the requested mip level is not streamed in, sampling may fail or fall back [S4].

## Operating characteristics
*   **Time Complexity:** Depends on the number of coefficients projected. The method allows stopping early based on accuracy, providing variable rendering time [S1].
*   **Memory Complexity:** Requires storing the sorted list of detail coefficients. The basic implementation uses 6 bytes per voxel, while the optimized version uses 4 bytes [S1].
*   **Performance:** The basic implementation suffers from performance loss due to the need for the maximum operator in frame buffer writes and the use of point sprites [S1]. The optimized version improves performance by avoiding these drawbacks [S1].
*   **Strengths:** Provides near-continuous error control and interactive frame rates for large datasets that do not fit in GPU memory [S1].
*   **Limitations:** The basic implementation has higher memory overhead and performance costs due to blending and point sprites [S1].

## Background
Streaming MIP was introduced in the paper "Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware" [S1]. It builds on the principles of multiresolution and level of detail [Parent].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26
[S2] [PDF] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S3] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S4] Sparse Volume Textures in Unreal Engine | Unreal Engine 5.8 ... - Epic Dev, https://dev.epicgames.com/documentation/en-us/unreal-engine/sparse-volume-textures-in-unreal-engine, fetched 2026-10-28
