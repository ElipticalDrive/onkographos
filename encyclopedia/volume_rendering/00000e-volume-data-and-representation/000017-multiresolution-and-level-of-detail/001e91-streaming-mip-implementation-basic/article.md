# Streaming MIP Implementation (Basic)

Also called: Streaming MIP

## Overview
Streaming MIP is a rendering technique that approximates the Maximum Intensity Projection (MIP) of volumetric data by projecting pre-sorted detail coefficients from a morphological pyramid directly to the frame buffer [S1, unofficial]. It solves the problem of rendering large datasets with limited memory bandwidth by streaming coefficients in order of their contribution to the L1-error, allowing for progressive refinement or early termination based on a quality threshold [S1, unofficial]. Unlike level-by-level MIP rendering, which processes entire resolution levels sequentially, Streaming MIP projects individual coefficients regardless of their pyramid level, requiring a local dilation operation to correctly propagate the maximum value to neighboring pixels [S1, unofficial].

## How it works
**Inputs:** A morphological pyramid of detail coefficients $\{d_j(x, y, z)\}$, sorted by error contribution (L1-error reduction) [S1, unofficial].
**Outputs:** A 2D frame buffer image representing the MIP.

**Data Structures:**
The implementation requires storing detail coefficients with specific attributes to support the projection logic. Each voxel (coefficient) is stored with:
1.  **Position:** 4 bytes. The pyramid level $j$ is encoded into two bits of the position attribute [S1, unofficial].
2.  **Intensity:** 2 bytes (short), storing the coefficient value [S1, unofficial].
Total memory footprint: 6 bytes per voxel [S1, unofficial].

**Numbered Steps:**
1.  **Initialization:** Initialize the frame buffer with the minimum possible intensity value (e.g., 0 or background) [G].
2.  **Iteration:** Iterate through the sorted list of detail coefficients $\{d_j\}$ [S1, unofficial].
3.  **Projection Condition:** For each coefficient $k$ with value $val$ and pyramid level $j$, determine the projection point $(x, y)$ [S1, unofficial].
4.  **Local Dilation:** Define a neighborhood around the projection point based on the level $j$. The dilation size is $j \cdot \tilde{A}$, where $\tilde{A}$ is the structuring element used in the pyramid synthesis [S1, unofficial]. For a square $2 \times 2$ structuring element, the dilation radius/size corresponds to point sizes of 1, 2, and 4 for levels 0, 1, and 2 respectively [S1, unofficial].
5.  **Update:** For all pixels in the dilated neighborhood:
    *   Let $curval$ be the current value in the frame buffer at that pixel.
    *   If $curval < val$, overwrite $curval$ with $val$ [S1, unofficial].
6.  **Termination:** Stop when a pre-defined maximum number of coefficients has been projected or the desired accuracy threshold is met [S1, unofficial].

**Formulas:**
The update rule for a pixel $p$ in the neighborhood of coefficient $k$ is:
$$ \text{buffer}[p] = \max(\text{buffer}[p], val) $$
where $val$ is the intensity of the current coefficient [S1, unofficial].

**Parameters:**
*   **Structuring Element $\tilde{A}$:** Typically a square $2 \times 2$ block [S1, unofficial].
*   **Point Size:** Mapped to the hardware vertex program's `PSIZE` semantic. For level $j$, the point size scales with $j$ [S1, unofficial].
*   **Sorting Criterion:** Coefficients are sorted by the reduction in L1-error between the partial reconstruction and the original data [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Global Dilation Invalid:** Global dilation cannot be used because the scale index (pyramid level) varies per coefficient, unlike level-by-level projection where the scale is constant for all coefficients in a level [S1, unofficial].
*   **Ordering Loss:** Because coefficients are projected by error rather than intensity, voxels are no longer ordered by intensity in the rendering stream. This necessitates using the maximum operator for frame buffer writes, which may cause performance loss compared to alpha-blending or simple overwriting, although the impact is mitigated by the small size of the blended primitives [S1, unofficial].
*   **Holes:** For non-axis-aligned parallel projections, the voxel-based method may yield pixel-sized holes in the result [S1, unofficial]. This is typically addressed in a post-processing step using morphological closing with the structuring element $\tilde{A}$ [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def streaming_mip_basic(coefficients, frame_buffer, structuring_element_size=2):
    """
    coefficients: List of tuples (level, x, y, z, value), sorted by error contribution.
    frame_buffer: 2D array initialized to min_value.
    structuring_element_size: Size of the base structuring element (e.g., 2 for 2x2).
    """
    for coeff in coefficients:
        level, x, y, z, val = coeff
        
        # Determine dilation radius based on level and structuring element
        # For a 2x2 element, level 0 -> size 1, level 1 -> size 2, level 2 -> size 4
        # The "size" here refers to the extent of the neighborhood to check/update.
        # A common interpretation for dilation size j with base size S is radius proportional to j*S.
        # Based on S1: "point size will be 1, 2 and 4 at level 0, 1 and 2" for 2x2 element.
        # This implies the dilation radius or diameter scales linearly.
        # Let's assume the neighborhood is a square of side length `dilation_size`.
        
        dilation_size = (level + 1) * (structuring_element_size // 2) 
        # Note: S1 says point size 1, 2, 4 for levels 0, 1, 2. 
        # This looks like 2^level * base? No, 1, 2, 4 is 2^0, 2^1, 2^2.
        # But it also says "scaled neighborhood j * A~". 
        # If A~ is 2x2, j=1 -> 2x2, j=2 -> 4x4. 
        # Let's stick to the explicit example: size = 2^level * (base_size/2)? 
        # Actually, S1 says: "point size will be 1, 2 and 4 at level 0, 1 and 2".
        # This suggests the radius or half-width might be 2^level? Or the diameter?
        # If point size is the diameter, radius is 0.5, 1, 2.
        # Let's implement the neighborhood as a square centered at (x,y) with side `dilation_size`.
        
        dilation_size = 2 ** level * (structuring_element_size // 2)
        if dilation_size < 1:
            dilation_size = 1
            
        half_size = dilation_size // 2
        
        # Iterate over the dilated neighborhood
        for dy in range(-half_size, half_size + 1):
            for dx in range(-half_size, half_size + 1):
                px = x + dx
                py = y + dy
                
                # Check bounds
                if 0 <= px < frame_buffer.width and 0 <= py < frame_buffer.height:
                    if frame_buffer[py, px] < val:
                        frame_buffer[py, px] = val
```

## Subtopics

### Optimized Streaming MIP
This variant avoids the performance drawbacks of the basic implementation (point sprites, frame buffer blending, 6-byte overhead) by using only 4 bytes per voxel and no point sprites [S1, unofficial]. It retains the error-sorted list but uses a second histogram-sorting step with one bin per (level, intensity) pair to guide rendering [S1, unofficial]. The algorithm selects a percentage of coefficients to render, then traverses the sorted list to render voxels for each pyramid level in low-to-high intensity order, guided by the histogram [S1, unofficial]. This allows for efficient rendering without the need for maximum operator blending during the write phase, as the intensity ordering ensures correct MIP approximation [S1, unofficial].

### Advanced Algorithm with Mipmap Fallback
This method handles missing data at the needed resolution by triggering a fallback to a higher mipmap level [S2, unofficial]. If no higher level is present or the data is too coarse, it resorts to iterative interrupt/resume methods [S2, unofficial]. This approach provides a stable frame rate by fixing a budget for data upload, leveraging the fact that users perceive fewer details during strong motion when missing data is most likely [S2, unofficial].

### Center-Precomputed MIP Approximation
This technique reduces per-intersection cost by precomputing MIP-filtered weights using the inverse covariance matrix of the surfel's tangent frame [S3, unofficial]. The weight $w$ is calculated as $w = n_f \cdot \exp(-0.5 \cdot \rho_{\text{mip}}^2)$, where $\rho_{\text{mip}}^2 = \Sigma_{uu}^{-1} u^2 + 2 \Sigma_{uv}^{-1} uv + \Sigma_{vv}^{-1} v^2$ [S3, unofficial]. This avoids per-pixel Jacobian evaluation without visual loss in filtering quality [S3, unofficial].

### Block Inspection (Streamed)
This approach involves opening a second streamed volume over the same store, cropped to the picked brick's common-grid box, allowing the octree to pull the crop toward level 0 [S4, unofficial]. It fixes issues where `extractSubVolume` returns null for streamed volumes by falling back to a specific sampler backed by the coarse floor, ensuring correct picking behavior [S4, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of coefficients projected. In the worst case, all coefficients are projected, but early termination allows for $O(k)$ where $k$ is the number of coefficients needed for the desired accuracy [S1, unofficial].
*   **Memory Complexity:** 6 bytes per voxel for the basic implementation (4 bytes position + 2 bytes intensity) [S1, unofficial]. The optimized variant uses 4 bytes per voxel [S1, unofficial].
*   **Performance:** The basic implementation suffers from some performance loss due to the need for maximum operator writes to the frame buffer, as voxels are not ordered by intensity [S1, unofficial]. However, the impact is limited because the blended primitives (dilated points) are small [S1, unofficial]. The optimized variant eliminates this bottleneck [S1, unofficial].
*   **Strengths:** Allows for progressive refinement; efficient use of memory bandwidth by streaming only necessary coefficients; hardware-accelerated via point rendering extensions [S1, unofficial].
*   **Limitations:** Requires preprocessing to build the morphological pyramid and sort coefficients; basic implementation has higher memory overhead and blending overhead; non-axis-aligned projections may require post-processing to fill holes [S1, unofficial].

## Background
Streaming MIP was introduced in the context of multiresolution MIP rendering of large volumetric data accelerated on graphics hardware [S1, unofficial]. It builds on the concept of morphological pyramids, where detail coefficients are computed to minimize L1-error [S1, unofficial]. The method was developed to address the limitations of level-by-level rendering, which cannot easily support progressive refinement or adaptive quality control [S1, unofficial]. The basic implementation leverages graphics hardware features such as point size extensions and point sprites to implement level-dependent local dilation [S1, unofficial].

## Sources
[S1] Multiresolution MIP Rendering of Large Volumetric Data Accelerated on Graphics Hardware, https://doi.org/10.2312/vissym/eurovis07/243-250, fetched 2026-10-26 (unofficial)
[S2] Ray-Guided Streaming for Efficient and Detailed Voxel Rendering, https://www-sop.inria.fr/reves/Basilic/2009/CNLE09/CNLE09.pdf, fetched 2026-10-26 (unofficial)
[S3] 3DSS: 3D Surface Splatting for Inverse Rendering, https://doi.org/10.48550/arxiv.2605.05876, fetched 2026-10-26 (unofficial)
[S4] feat(niivue): streamed multi-LOD rendering fixes, LOD compensation, budget plans, and block inspection, https://github.com/niivue/mono/pull/138, fetched 2026-10-27 (unofficial)
