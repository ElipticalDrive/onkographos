# Parallel Projection Rendering

Also called: Parallel Projection Rendering 1

## Overview
Parallel Projection Rendering is an object-order volume rendering algorithm optimized for parallel viewing transformations, utilizing run-length encoded (RLE) data structures to exploit spatial coherence in both the volume and the intermediate image [S2]. It builds on the Shear-warp factorization [Parent](24-shear-warp.md) but specifically targets parallel projections where all voxel slices share a uniform scale factor, allowing a one-to-one mapping between sheared voxel scanlines and intermediate image pixels [S2]. Unlike general shear-warp implementations that may require multiple resampling passes, this method minimizes resampling to two steps (shear and final 2D warp) and achieves significant performance gains by skipping transparent voxels and occluded pixels via RLE traversal [S2].

## How it works
The algorithm factorizes the viewing transformation into a permutation, a shear, and a 2D warp [S2]. It assumes a static opacity transfer function and parallel projection geometry [S2].

**Inputs and Outputs:**
*   **Input:** 3D volume data $V$, viewing transformation matrix $M_{view}$, transfer function $f$ (mapping density to color/opacity), image dimensions $(W, H)$, and an opacity threshold $\tau$ for RLE classification [S2].
*   **Output:** 2D image buffer $I(u,v)$ containing color and opacity values [S2].

**Data Structures:**
*   **Volume RLE:** Three copies of the volume are pre-computed as RLE structures, one for each principal axis (X, Y, Z) [S2]. Each copy stores runs of transparent and non-transparent voxels based on threshold $\tau$ [S1, unofficial]. Each non-transparent voxel entry consumes 4 bytes, and each run header consumes 1 byte [S2].
*   **Intermediate Image RLE:** A 2D buffer of size $(W, H)$ stores accumulated color and opacity. To support early ray termination and occlusion skipping, each opaque pixel stores a pointer (offset) to the next non-opaque pixel in the same scanline [S2]. This allows the compositing loop to skip over fully occluded regions [S1, unofficial].

**Numbered Steps:**

1.  **Permutation (Axis Selection):** Determine the principal viewing axis by identifying the coordinate axis (X, Y, or Z) most aligned with the view direction vector $v_o$ [S12, unofficial]. Construct a permutation matrix $P$ to align this axis with the Z-axis of the intermediate coordinate system [S2].
    *   If $v_o$ is closest to Z, $P = I$ (identity).
    *   If $v_o$ is closest to X, swap X and Z axes.
    *   If $v_o$ is closest to Y, swap Y and Z axes.
    *   *Pitfall:* If the view direction is near 45 degrees to any axis, artifacts increase, and the intermediate image size may expand, requiring more compositing operations [S9, unofficial][S15, book].

2.  **Shear and Resampling:** Transform the volume slices perpendicular to the principal axis into the intermediate image space [S2].
    *   Calculate shear coefficients $s_x, s_y$ from the view direction $v_o = [v_{o,x}, v_{o,y}, v_{o,z}]^T$ in the permuted space:
        $$ s_x = -\frac{v_{o,x}}{v_{o,z}}, \quad s_y = -\frac{v_{o,y}}{v_{o,z}} $$
    *   The shear matrix $S$ for parallel projection is:
        $$ S = \begin{pmatrix} 1 & 0 & s_x & 0 \\ 0 & 1 & s_y & 0 \\ 0 & 0 & 1 & 0 \\ 0 & 0 & 0 & 1 \end{pmatrix} $$
    *   Due to Property 3 of the shear-warp factorization for parallel projections, every voxel slice has the same scale factor, which can be chosen as unity [S2]. This ensures a one-to-one mapping between voxels in a scanline and pixels in the intermediate image scanline [S2].
    *   Resample each volume slice using bilinear interpolation into the intermediate image. Because the scale is unity and consistent per slice, resampling weights are fixed for the frame [S1, unofficial].

3.  **Composite:** Traverse the volume RLE and the intermediate image RLE in synchrony along scanlines [S2].
    *   For each slice in front-to-back order, iterate through the volume scanline and the corresponding intermediate image scanline [S1, unofficial].
    *   Use the volume RLE to skip runs of transparent voxels (where opacity $< \tau$) [S1, unofficial].
    *   Use the intermediate image RLE to skip runs of occluded pixels (where accumulated opacity is high) [S1, unofficial].
    *   For unskipped voxel runs, apply the transfer function and composite into the intermediate image using the Porter-Duff "over" operator [S2]:
        $$ C_{new} = C_{slice} \cdot \alpha_{slice} + C_{accum} \cdot (1 - \alpha_{slice}) $$
        $$ \alpha_{new} = \alpha_{slice} + \alpha_{accum} \cdot (1 - \alpha_{slice}) $$
    *   *Early Ray Termination:* If $\alpha_{accum}$ exceeds a threshold (e.g., 0.95), stop processing further slices for that pixel [S2][S11, abstract only, unofficial].

4.  **Warp:** Transform the intermediate image to the final screen space using the 2D warp matrix $M_{warp2D}$ [S2].
    *   Compute $M_{warp2D}$ as:
        $$ M_{warp2D} = M_{view} \cdot S^{-1} \cdot P^{-1} $$
        (Note: Some implementations define $M_{warp2D} = M_{view} \cdot M_{shear}^{-1}$ assuming $P$ is handled implicitly [S7, S21, unofficial]).
    *   Resample the intermediate image using bilinear interpolation to produce the final image [S2]. This step corrects the distortion introduced by the shear and applies the correct projection geometry [S22, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_projection_render(volume, M_view, transfer_function, width, height, opacity_threshold):
    # 1. Permutation
    view_dir = get_view_direction(M_view)
    principal_axis = argmax(abs(view_dir)) # 0:x, 1:y, 2:z
    
    # Construct permutation matrix P to align principal axis with Z
    P = get_permutation_matrix(principal_axis)
    
    # 2. Shear
    # Calculate shear coefficients from view direction
    vo = compute_view_direction_from_matrix(M_view)
    sx = -vo[0] / vo[2]
    sy = -vo[1] / vo[2]
    
    # Shear matrix for parallel projection
    S = [
        [1, 0, sx, 0],
        [0, 1, sy, 0],
        [0, 0, 1, 0],
        [0, 0, 0, 1]
    ]
    
    # Pre-compute RLE structures for the volume along the principal axis
    # Three copies are maintained for X, Y, Z directions to avoid axis-switching penalties
    rle_data = build_rle(volume, transfer_function, principal_axis, opacity_threshold)
    
    # Initialize intermediate image with opaque pixel offsets for skipping occluded regions
    intermediate_image = init_intermediate_image(width, height)
    
    # 3. Composite
    # Iterate through slices in front-to-back order
    for slice_idx in range(num_slices):
        # Get slice data from RLE
        slice_rle = get_slice_rle(rle_data, slice_idx)
        
        # Traverse volume scanline and intermediate image scanline in synchrony
        # Use RLE to skip transparent voxels and occluded pixels
        composite_scanline(slice_rle, intermediate_image, width, height, opacity_threshold)
        
        # Early ray termination check can be integrated here if needed
        if check_early_termination(intermediate_image):
            break
            
    # 4. Warp
    # Compute warp matrix
    S_inv = invert_matrix(S)
    P_inv = invert_matrix(P)
    M_warp2D = multiply_matrices(M_view, S_inv, P_inv)
    
    # Warp intermediate image to final screen using bilinear interpolation
    final_image = warp_image(intermediate_image, M_warp2D, width, height)
    
    return final_image

def composite_scanline(slice_rle, intermediate_image, width, height, threshold):
    # Iterate through runs in the volume RLE and the intermediate image RLE
    # Skip transparent runs in volume
    # Skip occluded runs in intermediate image (using stored offsets)
    # Composite remaining voxel runs into intermediate image
    pass
```

## Subtopics

### Perspective Projection Rendering
Extends the shear-warp factorization to perspective views by introducing slice-dependent scaling factors and a perspective shear component. Choose when the camera is not at infinity. Link: [Perspective Projection Rendering](5558-perspective-projection-rendering.md).

### Fast Classification Algorithm
An extension of the parallel projection algorithm that supports unclassified (scalar) volumes using a classification-independent rain-max octree, avoiding expensive preprocessing when transfer functions change [S2]. Choose when dynamic transfer functions are required. Link: [Affine-Transformations Method](2038-affine-transformations-method.md) (related coherence optimizations).

## Operating characteristics
*   **Time Complexity:** The algorithm traverses the volume once. Performance is heavily dependent on the efficiency of the RLE traversal and the sparsity of the volume. On an SGI Indigo R4000, it renders a $128 \times 128 \times 109$ dataset in an average of 0.4 seconds [S2]. Color renderings take roughly twice as long as grayscale renderings [S2].
*   **Memory Complexity:** Requires storing three RLE copies of the volume. For a $256 \times 256 \times 226$ dataset, the three RLEs total 9.8 Mbytes (4 bytes per non-transparent voxel + 1 byte per run) [S2]. The intermediate image also requires storage for offsets.
*   **Strengths:** At least five times faster than previous algorithms on comparable hardware [S2]. Does not suffer from performance degradation when the major viewing axis changes, provided the three RLE copies fit in memory [S2]. Minimal loss of image quality compared to subsampling methods [S2].
*   **Limitations:** Requires pre-computation of RLE structures, which adds preprocessing time [S2]. Performance degrades as the rotation angle approaches 45 degrees due to increased intermediate image size and compositing operations [S2]. Assumes a static opacity transfer function for optimal performance [S2].
*   **When to use:** Ideal for parallel projection views of large, sparse medical or scientific datasets where interactive frame rates are required and the transfer function is static.
*   **When not to use:** Unsuitable for perspective views without modification, or when dynamic transfer functions are needed without the overhead of the fast classification extension.

## Background
The Parallel Projection Rendering algorithm was introduced by Lacroute and Levoy in 1995 as part of their work on fast volume rendering using shear-warp factorization [S2]. It was designed to address the performance bottlenecks of earlier volume rendering methods, which often required multiple resampling steps or suffered from quality degradation due to subsampling [S2]. The method leverages the geometric properties of the shear-warp factorization, specifically the alignment of voxel and pixel scanlines, to enable efficient scanline-based traversal using run-length encoding [S2]. It was implemented and tested on SGI Indigo workstations, demonstrating significant speedups over existing techniques [S2].

## Sources
[S1] Shear-Warp Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/ShearWarp.color.pdf, fetched 2026-10-27 (unofficial)
[S2] Fast Volume Rendering Using a Shear-warp Factorization of The Viewing Transformation, http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
