# Fast Shear-Warp with Runlength Encoding

Also called: CompositeIntermediateLinearRLE, CompositeIntermediateNearestRLE, Fast shear-warp with RLE, Lacroute and Levoy method, Parallel Projection Shear-Warp Rendering, Shear-warp factorization for parallel projection, Volpack

## Overview
Fast Shear-Warp with Runlength Encoding is a scanline-based volume rendering algorithm optimized for parallel projections that uses run-length encoding (RLE) of voxel scanlines and intermediate image pixels to skip transparent voxels and occluded pixels, assuming a fixed opacity transfer function [S1]. It builds on [CPU-based Shear-Warp](1161-cpu-based-shear-warp.md) by explicitly leveraging the coherence of both the volume data and the resulting image to achieve high performance, rivaling raycasting and splatting at unity magnification while maintaining sub-second rendering times for moderately-sized datasets [S2]. Unlike general ray-casting, it achieves early ray termination via opaque pixel links with low overhead, as offsets are updated only when a pixel first becomes opaque [S1].

## How it works
The algorithm relies on the shear-warp factorization, which decomposes the viewing transformation into three steps: shearing the volume, projecting the sheared volume, and warping the resulting image [S1]. This method assumes parallel projection and a fixed opacity transfer function [S1].

### Inputs and Outputs
*   **Input:** A 3D volume dataset $V$ with dimensions $N_x \times N_y \times N_z$. A fixed opacity transfer function $T(v)$ that maps voxel values to opacity $\alpha \in [0, 1]$. A viewing direction defined by the shear-warp factorization parameters.
*   **Output:** A 2D intermediate image $I_{int}$ (composited in volume-parallel space) and a final screen image $I_{screen}$ (warped to screen space).

### Data Structures

#### Voxel RLE
To skip transparent voxels, the volume is preprocessed into a Run-Length Encoded (RLE) structure. This structure is view-independent and must be constructed for each of the three principal viewing directions (X, Y, Z axes) to avoid transposing the volume during rendering [S1, S2].

*   **Layout:** The RLE consists of a list of runs. Each run is defined by:
    *   `start_index`: The starting index of the run in the scanline.
    *   `length`: The number of voxels in the run.
    *   `type`: Enum indicating whether the run is `TRANSPARENT` or `NON_TRANSPARENT`.
*   **Definition:** A voxel is classified as `TRANSPARENT` if its opacity $\alpha < \alpha_{thresh}$, where $\alpha_{thresh}$ is a user-specified threshold [S1]. Otherwise, it is `NON_TRANSPARENT` [S1].
*   **Storage:** Transparent runs are stored to allow skipping, but the actual voxel values are not stored for these runs to save space [S1].

#### Image RLE
To skip occluded pixels, an RLE is maintained for the intermediate image $I_{int}$ during rendering.

*   **Layout:** For each pixel in the intermediate image scanline, if the pixel is opaque, an offset is stored pointing to the next non-opaque pixel in the same scanline [S1].
*   **Definition:** An intermediate image pixel is defined as `OPAQUE` if its accumulated opacity exceeds a user-specified threshold [S1].
*   **Computation:** This structure is computed on-the-fly during rendering [S1]. The cost is low because a pixel's offset is updated only when the pixel *first* becomes opaque [S1].

### Parameters
*   **Opacity Threshold ($\alpha_{thresh}$):** A user-specified threshold defining transparent vs. non-transparent voxel runs in the volume RLE and opaque vs. non-opaque pixel runs in the image RLE [S1].
*   **Magnification:** Rendering quality is optimal when object magnification is about unity [S2].

### Steps

1.  **Preprocessing (View-Independent):**
    *   Construct three separate RLE-encoded volumes, one for each principal viewing direction (X, Y, Z) [S1, S2].
    *   Classification is based on the fixed opacity transfer function [S2].
    *   *Note:* If the classification function changes, the RLE must be recomputed [S3].

2.  **Select Encoding:**
    *   During rendering, determine the permutation matrix $P$ from the shear-warp factorization of the current view [S1].
    *   Select the corresponding precomputed RLE encoding based on $P$ [S1].

3.  **Shear:**
    *   Shear the selected encoded volume such that the viewing rays are perpendicular to the volume slices [S2].
    *   This transformation aligns voxel scanlines with pixel scanlines in the intermediate image (Property 1 of shear-warp factorization) [S1].
    *   Since the transformation applied to each slice is only a translation (no scaling or rotation), the resampling weights are identical for every voxel in a slice [S1].

4.  **Traverse and Composit (Scanline Loop):**
    *   Traverse the volume and the intermediate image simultaneously in scanline order [S1].
    *   Use the Voxel RLE to skip runs of transparent voxels [S1].
    *   Use the Image RLE to skip runs of opaque (occluded) pixels [S1].
    *   For non-skipped voxel runs, execute a tightly-coded loop performing shading, resampling, and compositing [S1].

    **Resampling and Shading Details:**
    *   Use bilinear interpolation with a gather-type convolution (backward projection) [S1].
    *   Two voxel scanlines are traversed simultaneously to compute a single intermediate image scanline [S1].
    *   Sample values are obtained via bilinear interpolation within the traversed volume slices [S2].
    *   Shading is applied using a lookup-table (LUT) based system [S1].
    *   Voxel opacity is corrected for the current viewing angle using a LUT, as the apparent thickness of a slice depends on the viewing angle [S1].

5.  **Warp:**
    *   After compositing, warp the intermediate image $I_{int}$ into the final screen image $I_{screen}$ [S1].
    *   Use a general-purpose affine image warper with a bilinear filter [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def fast_shear_warp_rle(volume, view_matrix, alpha_thresh):
    """
    Renders a volume using Fast Shear-Warp with RLE.
    Assumes preprocessing has been done for the relevant principal axis.
    """
    # 1. Select Precomputed RLE based on view matrix permutation P
    # P determines which principal axis (X, Y, or Z) the volume is aligned with.
    # For this example, assume we are rendering along the Z-axis (standard case).
    # The volume is already sheared such that rays are perpendicular to XY slices.
    
    # Get the RLE structure for the current view direction
    # This structure contains runs of (start, length, type) for each scanline
    voxel_rle = get_precomputed_rle(view_matrix)
    
    # Initialize intermediate image and its RLE (offsets)
    # I_int is a 2D array of (color, opacity)
    width, height = get_screen_dimensions()
    I_int = create_empty_image(width, height)
    
    # Image RLE stores the offset to the next non-opaque pixel for opaque pixels
    # Initialize offsets to None or 0 (indicating no skip needed yet)
    image_offsets = create_empty_offsets(width, height)
    
    # 2. Traverse volume and image simultaneously
    # Iterate through each scanline of the intermediate image
    for y in range(height):
        for x in range(width):
            # Determine the corresponding voxel scanline in the sheared volume
            # Due to shearing, the voxel scanline for (x, y) might start at a different Z
            # or require interpolation from two Z-slices.
            
            # Get the voxel run for this (x, y) position from Voxel RLE
            run = voxel_rle.get_run(x, y)
            
            if run.type == TRANSPARENT:
                # Skip this run entirely
                continue
            
            # If the current image pixel is already opaque, check the offset
            if is_pixel_opaque(I_int[x, y], alpha_thresh):
                offset = image_offsets[x, y]
                if offset is not None:
                    # Skip to the next non-opaque pixel in this scanline
                    # Note: This optimization skips pixels that are fully occluded
                    # by previous voxels. The loop structure needs to handle
                    # jumping x indices.
                    x += offset
                    continue
            
            # 3. Resampling and Compositing
            # Since rays are perpendicular to slices, we sample from the current slice
            # and potentially the adjacent slice for bilinear interpolation in Z.
            
            # Get sample values via bilinear interpolation
            # Properties 2 and 3 allow simplified resampling: weights are constant per slice
            # Gather-type convolution: traverse two voxel scanlines to compute one image scanline
            sample_value = bilinear_interpolate_volume(volume, x, y, view_matrix)
            
            # Apply shading using LUT
            shaded_color = lookup_table_shading(sample_value, view_matrix)
            
            # Get voxel opacity (corrected for viewing angle via LUT)
            voxel_opacity = lookup_table_opacity_correction(sample_value, view_matrix)
            
            # Composite into intermediate image
            # Standard front-to-back alpha compositing
            c_new, a_new = composite_pixel(I_int[x, y], shaded_color, voxel_opacity)
            I_int[x, y] = (c_new, a_new)
            
            # Update Image RLE if pixel just became opaque
            if is_pixel_opaque(I_int[x, y], alpha_thresh) and not was_pixel_opaque(I_int[x, y], alpha_thresh):
                # Compute offset to next non-opaque pixel
                # This is a forward scan to find the next non-opaque pixel
                next_x = find_next_non_opaque_pixel(I_int, x, y, width)
                image_offsets[x, y] = next_x - x if next_x else 0

    # 4. Warp to Screen
    # Transform the intermediate image to the final screen coordinates
    final_image = affine_warp(I_int, view_matrix, filter='bilinear')
    
    return final_image

def find_next_non_opaque_pixel(I_int, start_x, y, width):
    """
    Helper to find the next non-opaque pixel in the same scanline.
    Used to populate Image RLE offsets.
    """
    for x in range(start_x + 1, width):
        if not is_pixel_opaque(I_int[x, y], alpha_thresh):
            return x
    return None # Or width, indicating end of scanline
```

### Edge Cases and Pitfalls
*   **Opaque Region Coherence:** The effectiveness of opaque pixel skipping depends on the coherence of opaque regions. If opaque regions are not coherent (e.g., scattered small objects), skipping is less effective [S1].
*   **Off-Axis View Quality:** Image quality degrades at off-axis views because the DVRI (Digital Volume Ray Interval) distance varies from 1.0 (axis-aligned) to 1.73 (corner-on), potentially violating the Nyquist theorem [S2]. This can be mitigated by using a volume with extra interpolated slices (Volpack's standard solution) [S2].
*   **Transfer Function Changes:** The algorithm requires an opacity-encoded volume, making interactive transfer function variation challenging because the RLE must be recomputed whenever the voxel classification function changes [S2, S3].
*   **Supersampling:** The DVRI distance cannot be varied to allow for supersampling along the ray [S2].

## Operating characteristics
*   **Time Complexity:** The algorithm is efficient due to skipping transparent voxels and occluded pixels. The cost of computing pixel offsets is low because an offset is updated only when a pixel first becomes opaque [S1].
*   **Memory Complexity:** Requires storage for three RLE-encoded volumes (one per principal axis) and the intermediate image RLE. Transparent voxels are not stored, so the encoded volume is typically much smaller than the original volume [S1].
*   **Performance:** Achieves sub-second rendering times for moderately-sized datasets [S2]. Quality rivals raycasting and splatting at unity magnification [S2].
*   **Strengths:** High performance for parallel projections; low overhead for early ray termination via image coherence; simple, high-quality resampling due to shear-warp factorization properties [S1].
*   **Limitations:** Fixed opacity transfer function required; quality degradation at off-axis views; challenging to support interactive transfer function changes [S2, S3].

## Background
The method was developed by Lacroute and Levoy [S1]. It is part of the "Volpack" distribution from Stanford [S2]. The algorithm is implemented in VTK as `CompositeIntermediateNearestRLE` and `CompositeIntermediateLinearRLE` [S3].

## Sources
[S1] Fast Volume Rendering Using a Shear-warp Factorization of The Viewing Transformation, http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
[S2] A Practical Evaluation of Popular Volume Rendering Algorithms, https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S3] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
