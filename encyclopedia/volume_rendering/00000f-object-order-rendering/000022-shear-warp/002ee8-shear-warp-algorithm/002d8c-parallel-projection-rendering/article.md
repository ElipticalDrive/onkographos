# Parallel Projection Rendering

Also called: Parallel Projection Rendering 1

## Overview
Parallel Projection Rendering is an object-order volume rendering algorithm optimized for parallel projections, utilizing the Shear-Warp factorization to align voxel scanlines with pixel scanlines [S2]. It solves the performance bottleneck of traditional ray-casting by exploiting spatial coherence in both the volume data and the intermediate image through run-length encoding (RLE), allowing it to skip transparent voxels and occluded pixels efficiently [S2]. Unlike general shear-warp variants, it assumes a static opacity transfer function and leverages the specific geometric property that all voxel slices share the same scale factor, enabling a one-to-one mapping between voxels and intermediate-image pixels [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (scalar field), a static opacity transfer function (or a user-specified threshold for transparency), and a parallel projection viewing transformation.
*   **Output:** A 2D rendered image.

**Data Structures:**
The algorithm relies on two primary run-length encoded structures to exploit spatial coherence [S2]:

1.  **Volume RLE:** The sheared and resampled volume is stored as run-length encoded voxel scanlines. Each scanline consists of two types of runs:
    *   **Transparent runs:** Voxels with opacity below a user-specified threshold.
    *   **Non-transparent runs:** Voxels with opacity at or above the threshold.
    *   **Layout:** Each non-transparent voxel entry consumes 4 bytes. Each run header consumes 1 byte [S2].

2.  **Intermediate Image RLE:** The intermediate image (before the final 2D warp) stores run information to skip occluded regions.
    *   **Layout:** For each opaque pixel in the scanline, the structure stores an offset (or pointer) to the next non-opaque pixel in that scanline [S1, unofficial][S2]. This allows the compositing loop to jump over regions where the image is already fully opaque, skipping further compositing operations for those pixels [S2].

**Algorithm Steps:**

1.  **Shear and Resample:** Apply the shear transformation to the volume data. Resample the volume into an intermediate image space. For parallel projections, Property 3 of the shear-warp factorization states that every voxel slice has the same scale factor, which can be chosen arbitrarily [S2]. A unity scale factor is typically chosen so that there is a one-to-one mapping between voxels in a scanline and pixels in the corresponding intermediate-image scanline [S2].

2.  **Construct Volume RLE:** Traverse the sheared volume. For each voxel scanline, identify contiguous runs of transparent and non-transparent voxels based on the opacity threshold. Store these runs in the volume RLE structure [S1, unofficial][S2].

3.  **Initialize Image RLE:** Initialize the intermediate image RLE structure. Initially, the image is empty (transparent). As compositing proceeds, opaque pixels are marked, and their offsets to the next non-opaque pixel are recorded [S2].

4.  **Synchronized Traversal and Compositing:** Traverse the volume and the intermediate image in synchrony, processing one slice at a time [S2]. For each slice and each volume scanline:
    *   Iterate through the volume RLE runs.
    *   For each **transparent run**, skip processing entirely [S1, unofficial].
    *   For each **non-transparent run**:
        *   Check the corresponding region in the intermediate image RLE.
        *   If the image region is already fully opaque (occluded), skip the compositing for these voxels using the image RLE offsets [S1, unofficial][S2].
        *   If the image region is not fully opaque, process the voxels. Since all voxels in the slice are scaled by the same factor (unity), resampling uses the same weights for all voxels in that scanline [S1, unofficial].
        *   Composite the voxel contributions into the intermediate image.
        *   Update the intermediate image RLE: if a pixel becomes opaque, record its offset to the next non-opaque pixel [S2].

5.  **Warp:** Apply the 2D warp transformation to the intermediate image to produce the final view [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def parallel_projection_render(volume, opacity_threshold, view_transform):
    # 1. Shear and Resample Volume
    # Apply shear transformation to volume.
    # Resample into intermediate image space with unity scale factor.
    sheared_volume = shear_and_resample(volume, view_transform, scale_factor=1.0)
    
    # 2. Build Volume RLE
    # Run structure: (start_index, length, type)
    # type: 0 for transparent, 1 for non-transparent
    volume_rle = []
    for slice_idx in range(sheared_volume.num_slices):
        slice_runs = []
        current_run_start = 0
        current_run_type = is_transparent(sheared_volume[0, 0], opacity_threshold)
        current_run_length = 1
        
        for x in range(1, sheared_volume.width):
            voxel_type = is_transparent(sheared_volume[slice_idx, x], opacity_threshold)
            if voxel_type == current_run_type:
                current_run_length += 1
            else:
                slice_runs.append((current_run_start, current_run_length, current_run_type))
                current_run_start = x
                current_run_type = voxel_type
                current_run_length = 1
        slice_runs.append((current_run_start, current_run_length, current_run_type))
        volume_rle.append(slice_runs)

    # 3. Initialize Intermediate Image and Image RLE
    # Intermediate image initialized to 0 (transparent)
    intermediate_image = zeros(sheared_volume.width, sheared_volume.height)
    # Image RLE: map from opaque pixel index to next non-opaque pixel index
    # Initially, all pixels are transparent, so next_non_opaque[i] = i
    next_non_opaque = list(range(sheared_volume.width))
    
    # 4. Synchronized Traversal and Compositing
    for slice_idx in range(sheared_volume.num_slices):
        slice_runs = volume_rle[slice_idx]
        
        for run_start, run_length, run_type in slice_runs:
            if run_type == 0: # Transparent run, skip
                continue
            
            # Non-transparent run
            # Check if this region in the intermediate image is already occluded
            # We need to composite from run_start to run_start + run_length
            
            current_x = run_start
            end_x = run_start + run_length
            
            while current_x < end_x:
                # Jump to next non-opaque pixel using Image RLE
                next_x = next_non_opaque[current_x]
                
                if next_x >= end_x:
                    # The entire remaining run is already occluded
                    break
                
                # Composite voxels from current_x to next_x
                # Since scale factor is 1, voxel x maps directly to pixel x
                for x in range(current_x, next_x):
                    voxel_value = sheared_volume[slice_idx, x]
                    opacity = compute_opacity(voxel_value) # Using static transfer function
                    color = compute_color(voxel_value)
                    
                    # Front-to-back compositing
                    # Note: In shear-warp, the order depends on the shear direction.
                    # Assuming standard front-to-back for this example.
                    alpha_i = intermediate_image[x].alpha
                    color_i = intermediate_image[x].color
                    
                    # New contribution
                    new_alpha = opacity * (1 - alpha_i)
                    new_color = color * new_alpha
                    
                    # Update image
                    intermediate_image[x].alpha += new_alpha
                    intermediate_image[x].color += new_color
                    
                    # If pixel becomes opaque, update Image RLE
                    if intermediate_image[x].alpha >= 1.0:
                        # Find next non-opaque pixel for this position
                        # This is a simplified update; in practice, this requires
                        # maintaining a disjoint set or linked list structure for efficiency.
                        # Here we assume next_non_opaque is updated to point to the 
                        # next pixel that is not fully opaque.
                        next_non_opaque[x] = find_next_non_opaque(x, end_x, intermediate_image)
                
                current_x = next_x

    # 5. Warp
    final_image = warp_2d(intermediate_image, view_transform.warp_matrix)
    return final_image

def is_transparent(voxel_value, threshold):
    # Placeholder for opacity transfer function evaluation
    # Returns True if opacity < threshold
    return compute_opacity(voxel_value) < threshold

def compute_opacity(voxel_value):
    # Static opacity transfer function
    return voxel_value / 255.0 # Example normalization

def compute_color(voxel_value):
    # Static color transfer function
    return (voxel_value, voxel_value, voxel_value) # Grayscale example

def find_next_non_opaque(start_x, end_x, image):
    # Find the next pixel index >= start_x that is not fully opaque
    for x in range(start_x, end_x):
        if image[x].alpha < 1.0:
            return x
    return end_x
```

**Parameters:**
*   **Opacity Threshold:** A user-specified value defining the boundary between transparent and non-transparent voxels. Typical values depend on the data range (e.g., 0.1 to 0.5 for normalized data) [S1, unofficial][S2].
*   **Scale Factor:** For parallel projections, a unity scale factor (1.0) is used to ensure a one-to-one mapping between voxels and intermediate-image pixels [S2].

**Edge Cases and Pitfalls:**
*   **Memory Usage:** The algorithm requires storing three run-length encoded copies of the volume in memory for optimal performance. For a 256x256x226 dataset, this totals 9.8 Mbytes [S2]. If memory is insufficient, performance may degrade [S2].
*   **Viewing Angle:** The effectiveness of coherence optimizations varies with the viewpoint. Rendering time may increase as the rotation angle approaches 45 degrees because the intermediate image size increases, requiring more compositing operations [S2].
*   **Static Transfer Function:** The algorithm assumes a static opacity transfer function. Changing the transfer function requires rebuilding the volume RLE, which incurs a preprocessing cost [S2].

## Operating characteristics
**Time Complexity:**
The algorithm is generally faster than previous algorithms by a factor of at least five on comparable hardware [S2]. The warp step typically accounts for 10-20% of the total rendering time [S2].

**Memory Complexity:**
Memory usage is proportional to the volume size and the number of non-transparent voxels. Each copy of the volume RLE uses 4 bytes per non-transparent voxel and 1 byte per run [S2].

**Measured Performance:**
*   On an SGI Indigo R4000 workstation (no hardware graphics accelerators):
    *   **brainsmall** (128x128x109): Average 0.4 seconds (Min/Max: 0.37-0.48 s) [S2].
    *   **headsmall** (128x128x113): Average 0.4 seconds (Min/Max: 0.35-0.43 s) [S2].
    *   **256^3 medical dataset:** 1 second [S2].
*   Color renderings take roughly twice as long as grayscale renderings [S2].

**Strengths and Limitations:**
*   **Strengths:** High performance for parallel projections; no performance degradation when the major viewing axis changes, provided the three RLE copies fit in memory [S2]; minimal loss of image quality compared to subsampling methods [S2].
*   **Limitations:** Requires static opacity transfer function; memory intensive for large volumes if three copies cannot fit in RAM; performance varies with viewing angle [S2].

## Background
The Parallel Projection Rendering algorithm was introduced by Lacroute and Levoy as part of their work on fast volume rendering using the shear-warp factorization [S2]. It was presented in the technical report CSL-TR-95-678 from Stanford University [S2]. The method extends the shear-warp factorization by introducing coherence optimizations specifically for parallel projections, utilizing run-length encoding to skip transparent voxels and occluded pixels [S2].

## Sources
[S1] Shear-Warp Volume Rendering - Computer Science, https://www.cs.unh.edu/~cs880/lectureNotes/ShearWarp.color.pdf, fetched 2026-10-27 (unofficial)
[S2] Fast Volume Rendering Using a Shear-warp Factorization of The Viewing ..., http://i.stanford.edu/pub/cstr/reports/csl/tr/95/678/CSL-TR-95-678.pdf, fetched 2026-11-04
