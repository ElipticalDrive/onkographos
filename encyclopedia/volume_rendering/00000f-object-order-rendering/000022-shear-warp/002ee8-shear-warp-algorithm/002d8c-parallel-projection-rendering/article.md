# Parallel Projection Rendering

Also called: Parallel Projection Rendering 1

## Overview
Parallel Projection Rendering is an object-order volume rendering algorithm optimized for parallel viewing transformations, utilizing the shear-warp factorization to align voxel scanlines with pixel scanlines [S2]. It solves the performance bottleneck of traditional volume rendering by exploiting spatial coherence in both the volume data and the intermediate image through run-length encoding, allowing it to skip transparent voxels and occluded pixels efficiently [S2]. Unlike general shear-warp implementations that may suffer performance degradation when the major viewing axis changes, this method maintains consistent performance provided the encoded data fits in memory [S2].

## How it works
The algorithm assumes a static opacity transfer function and a parallel projection viewing transformation [S2]. It decomposes the rendering into a shear, a parallel projection, and a warp, but focuses specifically on the compositing step within the intermediate image space where geometric properties simplify the process [S2].

### Inputs and Outputs
*   **Input:** A 3D volume dataset (scalar field), a parallel viewing transformation, and a user-specified opacity threshold for defining transparent vs. non-transparent runs [S2].
*   **Output:** A 2D rendered image [S2].

### Data Structures
The algorithm relies on run-length encoding (RLE) for both the volume and the intermediate image to exploit spatial coherence [S2].

1.  **Volume Data Structure:** The sheared, resampled volume is stored as run-length encoded voxel scanlines [S1, unofficial]. Each scanline consists of two types of runs:
    *   **Transparent runs:** Voxels with opacity below the user-specified threshold.
    *   **Non-transparent runs:** Voxels with opacity at or above the threshold [S1, unofficial].
    *   **Layout:** Each copy of the run-length encoded volume contains four bytes per non-transparent voxel and one byte per run [S2]. Three copies are typically maintained to cover the traversal range [S2].

2.  **Intermediate Image Data Structure:** The intermediate image scanlines store run information to skip occluded pixels [S1, unofficial].
    *   **Layout:** For each opaque pixel (based on the user threshold), the structure stores a pointer (or offset) to the next non-opaque pixel in the scanline [S1, unofficial]. This allows the algorithm to skip over runs of opaque pixels that have already been filled by previous voxels [S1, unofficial].

### Mathematical Properties
The shear-warp factorization provides specific geometric properties for parallel projections [S2]:
*   **Property 1:** Scanlines of pixels in the intermediate image are parallel to scanlines of voxels in the volume [S2].
*   **Property 2:** All voxels in a given voxel slice are scaled by the same factor [S2].
*   **Property 3 (Parallel Projections Only):** Every voxel slice has the same scale factor, which can be chosen arbitrarily [S2]. The algorithm chooses a **unity scale factor** ($s=1$) [S2]. This ensures a one-to-one mapping between voxels in a voxel scanline and pixels in the corresponding intermediate-image scanline [S2]. Consequently, resampling uses the same weights for all voxels in a slice [S1, unofficial].

### Algorithm Steps
1.  **Preprocessing:**
    *   Compute the shear-warp factorization for the parallel projection [S2].
    *   Generate three run-length encoded copies of the volume data [S2].
    *   Initialize the intermediate image data structures with the opaque pixel pointers [S2].

2.  **Traversal and Compositing:**
    *   Traverse the volume and the intermediate image in synchrony, processing slice by slice [S2].
    *   For each volume scanline in the current slice:
        *   Walk through the volume scanline and the corresponding intermediate image scanline simultaneously [S1, unofficial].
        *   Use the volume RLE to skip transparent voxel runs [S1, unofficial].
        *   Use the intermediate image RLE (pointers) to skip occluded (already opaque) pixel runs [S1, unofficial].
        *   For the remaining (unskipped) voxel runs:
            *   Resample the voxel values to the intermediate image pixel centers. Since the scale factor is unity and all voxels in the slice are scaled equally, the resampling weights are constant for the slice [S1, unofficial].
            *   Composite the resampled values into the intermediate image using front-to-back or back-to-front accumulation [S2].
            *   Update the intermediate image RLE if new opaque pixels are created [S1, unofficial].

3.  **Warp:**
    *   Warp the intermediate image into the final screen space [S2].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_projection_render(volume, view_transform, opacity_threshold):
    # 1. Preprocessing
    # Compute shear-warp matrices for parallel projection
    shear_matrix, projection_matrix, warp_matrix = compute_shear_warp_factors(view_transform)
    
    # Create 3 RLE copies of the volume
    # Each copy stores runs of (is_transparent, length, data_pointer)
    rle_copies = create_rle_copies(volume, opacity_threshold)
    
    # Initialize intermediate image RLE
    # Each pixel has a pointer to the next non-opaque pixel
    img_rle = initialize_image_rle()
    
    # 2. Traversal
    for slice_index in range(num_slices):
        # Get current slice data from RLE copies
        slice_runs = get_slice_runs(rle_copies, slice_index)
        
        # Get corresponding intermediate image scanline
        img_scanline = get_img_scanline(img_rle, slice_index)
        
        # Traverse volume scanlines and image scanlines in sync
        # Due to Property 1, voxel scanlines align with pixel scanlines
        for voxel_run in slice_runs:
            if voxel_run.is_transparent:
                continue # Skip transparent runs
            
            # Map voxel run to image pixel range
            # Due to Property 3 (unity scale), 1 voxel = 1 pixel
            start_pixel = voxel_run.start_voxel_index
            end_pixel = voxel_run.end_voxel_index
            
            # Skip occluded pixels using image RLE
            current_pixel = start_pixel
            while current_pixel < end_pixel:
                next_non_opaque = img_rle.get_next_non_opaque(current_pixel)
                
                if next_non_opaque >= end_pixel:
                    break # All remaining pixels in this run are occluded
                
                # Process the valid pixel range
                valid_start = current_pixel
                valid_end = min(next_non_opaque, end_pixel)
                
                # Resample and Composite
                # Weights are constant for this slice
                for p in range(valid_start, valid_end):
                    voxel_value = resample_voxel(volume, voxel_run, p)
                    opacity = apply_transfer_function(voxel_value)
                    
                    # Front-to-back compositing
                    # C_out = C_in + (1 - C_in) * alpha * opacity
                    composite_pixel(img_scanline, p, voxel_value, opacity)
                
                current_pixel = next_non_opaque

    # 3. Warp
    final_image = apply_warp(img_rle, warp_matrix)
    return final_image
```

### Parameters and Edge Cases
*   **Opacity Threshold:** A user-specified value defining the boundary between transparent and non-transparent runs [S1, unofficial].
*   **Scale Factor:** Must be set to unity for the one-to-one mapping [S2].
*   **Memory Constraints:** The algorithm requires three run-length encoded copies of the volume to fit in memory simultaneously [S2]. If they do not fit, performance may degrade or the algorithm may fail [S2].
*   **Viewing Angle:** Performance is stable regardless of the major viewing axis, provided the memory constraint is met [S2].

## Operating characteristics
*   **Time Complexity:** The algorithm traverses the volume and image in synchrony, skipping transparent and occluded regions. The time complexity is proportional to the number of non-transparent, non-occluded voxels [S2].
*   **Memory Complexity:** Requires $O(N)$ memory for the volume RLE (3 copies) and $O(P)$ for the image RLE, where $N$ is the number of non-transparent voxels and $P$ is the number of opaque pixels [S2].
*   **Measured Performance:**
    *   On an SGI Indigo R4000, the algorithm renders a $128 \times 128 \times 109$ ('brainsmall') dataset in an average of 0.4 seconds [S2].
    *   On an SGI Indigo R4000, the algorithm renders a $128 \times 128 \times 113$ ('headsmall') dataset in an average of 0.4 seconds [S2].
    *   On an SGI Indigo workstation, the algorithm renders a $256^3$ voxel medical dataset in one second [S2].
    *   This is at least five times faster than previous algorithms on comparable hardware [S2].
*   **Scaling:** Color renderings take roughly twice as long as grayscale renderings [S2].
*   **Strengths:**
    *   High performance due to spatial coherence exploitation [S2].
    *   No performance degradation when the major viewing axis changes, unlike some other methods [S2].
    *   Minimal loss of image quality compared to subsampling methods [S2].
*   **Limitations:**
    *   Assumes a static opacity transfer function [S2].
    *   Requires sufficient memory for three RLE copies of the volume [S2].
    *   Effectiveness of coherence optimizations varies with viewpoint [S2].

## Background
The Parallel Projection Rendering algorithm was developed by Lacroute and Levoy as part of their work on fast volume rendering using shear-warp factorization [S2]. It was published in 1995 as part of the Stanford Computer Science Laboratory Technical Report CSL-TR-95-678 [S2]. The algorithm extends the shear-warp factorization introduced by Levoy [S2] to specifically handle parallel projections with optimized coherence data structures [S2]. It is used in interactive volume visualization tools where performance is critical [S2].

## Sources
[S1] Shear-Warp Volume Rendering, https://www.cs.unh.edu/~cs880/lectureNotes/ShearWarp.color.pdf, fetched 2026-10-27 (unofficial)
[S2] Fast Volume Rendering Using a Shear-warp Factorization of The Viewing Transformation, http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
