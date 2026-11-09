# Fast Shear-Warp with Runlength Encoding

Also called: Fast shear-warp with RLE, Lacroute and Levoy method

## Overview
Fast Shear-Warp with Runlength Encoding is a scanline-based volume rendering algorithm optimized for parallel projections that uses run-length encoding (RLE) of voxel scanlines and intermediate image pixels to skip transparent voxels and occluded pixels, assuming a fixed opacity transfer function [S1]. It achieves sub-second rendering times for moderately-sized datasets, with quality rivaling raycasting and splatting at unity magnification [S2]. Unlike the Simple Shear-Warp Algorithm, which traverses every voxel, this method leverages object coherence (transparent voxel runs) and image coherence (occluded pixel runs) to significantly reduce the number of shading and compositing operations [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   A 3D volume dataset with dimensions $D_x \times D_y \times D_z$ [S4, unofficial].
*   A scalar opacity transfer function (SOTF) or an isovalue threshold [S4, unofficial].
*   A view matrix, decomposed into shear, warp, and permutation components [S1].
*   An opacity threshold (for compositing) or isovalue (for isosurface/MIP) [S4, unofficial].

**Outputs:**
*   A final 2D screen image [S1].
*   Precomputed RLE structures for three principal viewing axes [S1].

### Data Structures
The method relies on three primary data structures, implemented as templated classes in VTK [S4, unofficial]:

1.  **`vtkShearWarpVoxelData<T>`**: Stores metadata for each opaque voxel.
    *   `T Value`: The scalar value of the voxel.
    *   `unsigned short EncodedNormal`: Pre-computed normal for shading.
    *   `unsigned char GradientMagnitude`: Pre-computed gradient magnitude.
2.  **`vtkShearWarpRLERun<T>`**: Represents a contiguous run of voxels or pixels.
    *   `unsigned char Length`: The number of elements in the run. **Note:** This is capped at 254 to prevent overflow during increment operations; runs longer than 254 are split into multiple entries [S4, unofficial].
    *   `vtkShearWarpVoxelData<T> *VoxelData`: A pointer to the first voxel in the run. For transparent runs, this is `NULL`.
3.  **`vtkShearWarpRLESlice<T>`**: Represents a 2D slice of the volume encoded along a specific axis.
    *   `vtkShearWarpRLERun<T> *LineRuns[jCount]`: An array of pointers to the first run of each scanline (row) in the slice.
    *   `vtkShearWarpRLERun<T> SliceRuns[currentRun]`: A flat array containing all runs in the slice.
    *   `vtkShearWarpVoxelData<T> VoxelData[voxelIndex]`: A flat array containing the data for all opaque voxels in the slice.

**Image RLE:**
*   For the intermediate image, each opaque pixel stores an `Offset` field pointing to the next non-opaque pixel in the same scanline [S1]. This is computed on-the-fly during rendering [S1].

### Preprocessing: RLE Construction
Three view-independent RLE encodings are constructed, one for each principal viewing direction (X, Y, Z), to avoid transposing the volume during rendering [S1]. The encoding is performed in two passes per slice [S4, unofficial].

**Parameters:**
*   `axis`: `VTK_X_AXIS` (0), `VTK_Y_AXIS` (1), or `VTK_Z_AXIS` (2) [S4, unofficial].
*   `opacityThreshold`: Float threshold. Voxels with opacity $> threshold$ are opaque [S4, unofficial].
*   `isoValue`: Float threshold. Voxels with scalar $\ge isoValue$ are opaque [S4, unofficial].

**Step 1: Determine Strides and Bounds**
Based on the `axis` and volume dimensions $D_x, D_y, D_z$:
*   Let $plane = D_x \times D_y$.
*   **X-axis:** $iCount = D_y, jCount = D_z, vk = k, iIncrement = D_x, jIncrement = plane$.
*   **Y-axis:** $iCount = D_z, jCount = D_x, vk = k \times D_x, iIncrement = plane, jIncrement = 1$.
*   **Z-axis:** $iCount = D_x, jCount = D_y, vk = k \times plane, iIncrement = 1, jIncrement = D_x$.

**Step 2: Pass 1 (Count Runs)**
Iterate through the slice to count the number of runs (`currentRun`) and opaque voxels (`voxelIndex`).
Initialize: `currentIndex = 0`, `currentRun = 0`, `voxelIndex = 0`, `transparentRun = false`.

For each line $j$ from $0$ to $jCount-1$:
  For each voxel $i$ from $0$ to $iCount-1$:
    1.  Compute linear location: $location = vi + vj + vk$.
    2.  Fetch scalar: $scalar = dptr[location]$.
    3.  Determine opacity:
        *   If using opacity mode: $opacity = SOTF[scalar]$.
        *   If using scalar/isosurface mode: $opacity = (scalar \ge isoValue)$.
    4.  If opaque:
        *   If `!transparentRun` and $i > 0$ and $currentIndex < 254$:
            *   $currentIndex++$.
        *   Else:
            *   $currentIndex = 0$.
            *   $currentRun++$.
            *   $transparentRun = false$.
        *   $voxelIndex++$.
    5.  Else (transparent):
        *   If `transparentRun` and $i > 0$ and $currentIndex < 254$:
            *   $currentIndex++$.
        *   Else:
            *   $currentIndex = 0$.
            *   $currentRun++$.
            *   $transparentRun = true$.

**Step 3: Allocate Memory**
*   `LineRuns = new vtkShearWarpRLERun<T>*[jCount]`
*   `SliceRuns = new vtkShearWarpRLERun<T>[currentRun]`
*   `VoxelData = new vtkShearWarpVoxelData<T>[voxelIndex]`
*   `activeRun = &SliceRuns[0]`
*   Reset `voxelIndex = 0`, `currentRun = 0`.

**Step 4: Pass 2 (Encode Runs)**
Iterate through the slice again to populate the structures.
For each line $j$ from $0$ to $jCount-1$:
  1.  `LineRuns[j] = activeRun`
  2.  `currentIndex = 0`
  3.  For each voxel $i$ from $0$ to $iCount-1$:
      1.  Compute $location = vi + vj + vk$.
      2.  Fetch $scalar = dptr[location]$.
      3.  Determine opacity as in Step 2.
      4.  If opaque:
          *   If `!transparentRun` and $i > 0$ and $currentIndex < 254$:
              *   $currentIndex++$.
          *   Else:
              *   If $i > 0$:
                  *   `activeRun->Length = currentIndex + 1`
                  *   `activeRun++`
                  *   `currentRun++`
                  *   $currentIndex = 0$
              *   `activeRun->VoxelData = &VoxelData[voxelIndex]`
              *   $transparentRun = false$
          *   `VoxelData[voxelIndex].Value = scalar`
          *   `VoxelData[voxelIndex].EncodedNormal = nptr[location]`
          *   `VoxelData[voxelIndex].GradientMagnitude = gptr[location]`
          *   $voxelIndex++$
      5.  Else (transparent):
          *   If `transparentRun` and $i > 0$ and $currentIndex < 254$:
              *   $currentIndex++$.
          *   Else:
              *   If $i > 0$:
                  *   `activeRun->Length = currentIndex + 1`
                  *   `activeRun++`
                  *   `currentRun++`
                  *   $currentIndex = 0$
              *   `activeRun->VoxelData = NULL`
              *   $transparentRun = true$
  4.  End of row: `activeRun->Length = currentIndex + 1`, `activeRun++`, `currentRun++`.

*Note: The check $i > 0$ prevents creating a zero-length run for the first voxel. The final row end explicitly sets the length of the last run [S4, unofficial].*

### Rendering
1.  **Select Encoding:** Choose one of the three precomputed RLE encodings based on the permutation matrix $P$ in the shear-warp factorization [S1].
2.  **Shear:** Shear the selected encoded volume such that rays are perpendicular to the volume slices [S2].
3.  **Traverse and Composite:**
    *   March through the volume and the intermediate image simultaneously in scanline order [S1].
    *   Use the **Voxel RLE** to skip runs of transparent voxels.
    *   Use the **Image RLE** to skip runs of opaque (occluded) pixels. An image pixel is opaque if its accumulated opacity exceeds the threshold [S1].
    *   For non-skipped voxel runs, execute a tightly-coded loop:
        *   **Resampling:** Use bilinear interpolation with a gather-type convolution (backward projection). Two voxel scanlines are traversed to compute one intermediate image scanline [S1]. Since the shear transformation is a translation, resampling weights are constant for all voxels in a slice [S1].
        *   **Shading:** Apply shading using a lookup-table based system [S1].
        *   **Compositing:** Accumulate color and opacity. The opaque pixel links achieve early ray termination; an offset is stored with each opaque pixel to skip to the next non-opaque pixel [S1].
4.  **Warp:** Transform the intermediate image into the final screen image using a general-purpose affine image warper with a bilinear filter [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def composite_intermediate_linear_rle(volume_data, sotf, opacity_threshold, view_matrix, screen_width, screen_height):
    # 1. Decompose view matrix into shear, warp, permutation (P)
    shear, warp, P = decompose_view_matrix(view_matrix)
    
    # 2. Select precomputed RLE slice based on P
    #    P determines which axis the volume is sliced along
    rle_slice = get_precomputed_rle_slice(P)
    
    # 3. Initialize intermediate image and its RLE
    #    img_opaque[i] = True/False, img_offset[i] = index of next opaque pixel
    intermediate_image = zeros(screen_width * screen_height)
    img_opaque = [False] * (screen_width * screen_height)
    img_offset = [0] * (screen_width * screen_height)
    
    # 4. Iterate through RLE slice scanlines
    for line_idx in range(rle_slice.jCount):
        line_runs = rle_slice.LineRuns[line_idx]
        
        # Get current image scanline range
        # Map sheared volume coordinates to intermediate image coordinates
        img_y = map_to_intermediate_y(line_idx, shear)
        
        # Find start and end in intermediate image for this scanline
        img_start, img_end = get_intermediate_scanline_range(img_y, screen_width)
        
        # Initialize image RLE state for this scanline
        current_img_idx = img_start
        while current_img_idx < img_end:
            if not img_opaque[current_img_idx]:
                break
            current_img_idx = img_offset[current_img_idx]
        
        if current_img_idx >= img_end:
            continue # Entire scanline is occluded
            
        # 5. Traverse voxel runs
        for run in line_runs:
            if run.VoxelData is None:
                continue # Skip transparent run
            
            # Get voxel range for this run
            # Map sheared volume coordinates to intermediate image coordinates
            # Since weights are constant per slice, we can compute bounds
            voxel_start_idx = 0
            voxel_end_idx = run.Length
            
            # Compute corresponding image pixel range for this voxel run
            # This requires inverse mapping from intermediate image to sheared volume
            img_run_start, img_run_end = map_voxel_run_to_image(
                run, line_idx, shear, screen_width
            )
            
            # Check for occlusion using image RLE
            # Skip if the entire voxel run maps to already opaque pixels
            if is_fully_occluded(img_run_start, img_run_end, img_opaque, img_offset, img_end):
                continue
                
            # 6. Process non-skipped voxels
            for k in range(voxel_start_idx, voxel_end_idx):
                voxel_data = run.VoxelData[k]
                scalar = voxel_data.Value
                
                # Compute opacity
                opacity = sotf[scalar]
                
                # Compute resampling weights (bilinear, constant for slice)
                # Gather from two adjacent voxel scanlines
                weights = compute_bilinear_weights(voxel_data, shear)
                
                # Map to intermediate image pixel
                img_x = map_voxel_to_intermediate_x(voxel_data, line_idx, shear, screen_width)
                
                # Check bounds
                if img_x < img_start or img_x >= img_end:
                    continue
                
                # Composite
                color, alpha = composite_pixel(
                    intermediate_image[img_x], 
                    scalar, 
                    opacity, 
                    weights
                )
                intermediate_image[img_x] = (color, alpha)
                
                # Update Image RLE if pixel becomes opaque
                if alpha > opacity_threshold and not img_opaque[img_x]:
                    img_opaque[img_x] = True
                    # Set offset to next non-opaque pixel (computed lazily or updated)
                    # In practice, offset is set when the pixel first becomes opaque
                    # to point to the next pixel that is not yet opaque
                    img_offset[img_x] = find_next_non_opaque(img_x, img_end, img_opaque)

    # 7. Warp intermediate image to screen
    final_image = warp_image(intermediate_image, warp, screen_width, screen_height)
    return final_image
```

## Operating characteristics
*   **Time Complexity:** Proportional to the number of non-transparent, non-occluded voxels. The RLE skips transparent voxels and occluded pixels, reducing the effective work significantly for sparse volumes [S1].
*   **Memory Complexity:** Requires storing three RLE encodings (one per axis). The encoded volume is typically much smaller than the original volume because transparent voxels are not stored [S1].
*   **Performance:** Achieves sub-second rendering times for moderately-sized datasets [S2].
*   **Strengths:** High performance for parallel projections with fixed transfer functions. Early ray termination via image RLE offsets [S1].
*   **Limitations:**
    *   Requires a fixed opacity transfer function; recomputing RLE is necessary if the classification function changes [S3].
    *   Image quality degrades at off-axis views due to the fixed interpolation distance (DVRI) varying from 1.0 (axis-aligned) to 1.73 (corner-on), potentially violating the Nyquist theorem [S2].
    *   Effectiveness of opaque pixel skipping depends on the coherence of opaque regions [S1].
    *   Interactive transfer function variation is challenging due to the opacity-encoded volume requirement [S2].

## Background
The algorithm was developed by Lacroute and Levoy as part of the Shear-Warp factorization framework [S1]. It is a core component of the VTK (Visualization Toolkit) volume rendering module, specifically `vtkVolumeShearWarpMapper` [S3]. The method is designed to leverage the coherence of volume data and image space to accelerate rendering on CPUs [S1].

## Sources
[S1] Fast Volume Rendering Using a Shear-warp Factorization of The Viewing Transformation, http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
[S2] A Practical Evaluation of Popular Volume Rendering Algorithms, https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S3] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
[S4] VTK_SHEAR_WARP_COMPOSITE_FUNCTION, https://davis.lbl.gov/Manuals/VTK-4.5/vtkVolumeShearWarpDataStructure_8h-source.html, fetched 2026-10-29 (unofficial)
