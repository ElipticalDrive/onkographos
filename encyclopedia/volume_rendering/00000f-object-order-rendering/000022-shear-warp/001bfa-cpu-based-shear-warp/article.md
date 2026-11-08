# CPU-based Shear-Warp

Also called: CPU Shear-Warp

## Overview
CPU-based Shear-Warp is a volume rendering algorithm implemented on general-purpose processors that accelerates the standard Shear-warp method [S1, unofficial][S2, unofficial][S3, unofficial]. It solves the problem of efficient ray-casting by factorizing the viewing transformation and using Run-Length Encoding (RLE) to skip transparent voxels and occluded pixels [S1, unofficial][S2, unofficial]. Unlike GPU-based variants which leverage parallel hardware, this method relies on CPU cache coherence and branch prediction, typically achieving performance suitable for moderate datasets but significantly slower than GPU implementations [S2, unofficial]. It distinguishes itself from the original Shear-warp proposal by explicitly avoiding 3D data structures like octrees in favor of pre-computed RLE lists for each principal axis [S1, unofficial][S2, unofficial][S3, unofficial].

## How it works
The method builds on the [Shear-warp](24-shear-warp.md) factorization $M_{view} = M_{warp2D} \cdot S \cdot P$ [S2, unofficial][S4][S7, unofficial][unofficial]. The CPU implementation optimizes the Shear and Resampling phases using RLE compression.

### Inputs and Outputs
*   **Input:** A 3D volume dataset (e.g., DICOM CT data), a view direction vector $v_o$, a transfer function (typically linear with lower/upper bounds on the Hounsfield scale), and a sampling rate $s$ [S2, unofficial][S3, unofficial].
*   **Output:** A 2D image buffer containing the composited volume rendering [S2, unofficial].

### Data Structures
The algorithm pre-compresses the volume data into three RLE structures, one for each principal coordinate axis (X, Y, Z), corresponding to the three possible principal view directions [S1, unofficial][S2, unofficial][S3, unofficial].

Each structure consists of a set of lists of runs. Each list corresponds to one scan line of one slice [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Run Definition:** A run is a contiguous sequence of voxels classified as either:
    1.  **Completely transparent:** These are skipped during rendering [S1, unofficial][S2, unofficial][S3, unofficial].
    2.  **Not completely transparent:** These are processed [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Classification:** Voxels are classified using the Hounsfield scale and a linear transfer function with defined lower and upper bounds to convert input scalar values into opacities [S2, unofficial][S3, unofficial].

### Parameters
*   **Sampling Rate ($s$):** The number of sample points along a viewing ray. The sources indicate that for the tested CPU implementation, $s$ is limited to a maximum of 5 due to speed constraints [S2, unofficial].
*   **Lighting Model:** Phong shading is used for direct lighting [S2, unofficial].
*   **Transfer Function:** Linear curve with lower and upper bounds [S2, unofficial][S3, unofficial].

### Algorithm Steps

1.  **Preprocessing (RLE Construction):**
    For each of the three principal axes:
    a. Iterate through each slice perpendicular to the axis.
    b. For each scan line in the slice:
       i. Classify each voxel using the Hounsfield scale and transfer function to determine opacity.
       ii. Compress the scan line into a list of runs: one type for completely transparent voxels, one for non-transparent voxels [S1, unofficial][S2, unofficial][S3, unofficial].

2.  **Rendering (Shear and Warp):**
    For the current view direction, select the corresponding principal axis and its RLE structure [S1, unofficial][S2, unofficial][S3, unofficial].
    a. **Shear:** Calculate shear coefficients $s_x, s_y$ based on the view direction $v_o = [v_{o,x}, v_{o,y}, v_{o,z}]^T$:
       $$ s_x = -\frac{v_{o,x}}{v_{o,z}}, \quad s_y = -\frac{v_{o,y}}{v_{o,z}} $$
       [S1, unofficial][S2, unofficial]
    b. **Traverse and Compositing:**
       Iterate through the RLE structure slice by slice and line by line [S1, unofficial][S2, unofficial][S3, unofficial].
       i. **Skip Transparent Runs:** If a run is completely transparent, skip it entirely [S1, unofficial][S2, unofficial][S3, unofficial].
       ii. **Resample:** For non-transparent runs, resample the sheared slice into the intermediate image buffer using bilinear interpolation [S2, unofficial][unofficial].
       iii. **Composite:** Apply front-to-back compositing using the Porter-Duff "over" operator:
       $$ C_{new} = C_{slice} \cdot \alpha_{slice} + C_{accum} \cdot (1 - \alpha_{slice}) $$
       $$ \alpha_{new} = \alpha_{slice} + \alpha_{accum} \cdot (1 - \alpha_{slice}) $$
       [S2, unofficial][unofficial]
       iv. **Early Ray Termination:** If the accumulated opacity $\alpha_{accum}$ exceeds a threshold (e.g., 0.95), skip further slices for those pixels [S2, unofficial][unofficial].
    c. **Warp:** Apply the 2D warp matrix $M_{warp2D}$ to the intermediate image to correct distortion and project to the final image space [S2, unofficial][unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_shear_warp(volume, view_direction, transfer_function, sampling_rate):
    # 1. Preprocessing: Build RLE structures for all 3 axes
    # Each structure is a dict: { axis_index: { slice_index: [run_list] } }
    # run_list contains tuples: (start_voxel_idx, end_voxel_idx, is_transparent)
    rle_structures = {}
    for axis in [0, 1, 2]:
        rle_structures[axis] = build_rle_for_axis(volume, axis, transfer_function)
    
    # 2. Determine principal axis based on view_direction
    principal_axis = get_principal_axis(view_direction)
    rle_data = rle_structures[principal_axis]
    
    # 3. Calculate Shear Coefficients
    # Assuming view_direction is normalized and v_z != 0
    vx, vy, vz = view_direction
    sx = -vx / vz
    sy = -vy / vz
    
    # 4. Initialize Intermediate Image Buffer
    # Width and Height based on output resolution
    intermediate_image = create_empty_buffer(width, height)
    
    # 5. Traverse Slices and Lines
    for slice_idx in range(num_slices):
        run_list = rle_data[slice_idx]
        
        for run in run_list:
            start_idx, end_idx, is_transparent = run
            
            # Skip completely transparent runs
            if is_transparent:
                continue
            
            # Process non-transparent run
            # Resample sheared slice data into intermediate image
            # This involves calculating the 2D coordinates in the intermediate buffer
            # corresponding to the voxel coordinates in the sheared space
            resample_run_into_buffer(
                volume, 
                run, 
                slice_idx, 
                principal_axis, 
                sx, 
                sy, 
                intermediate_image,
                transfer_function
            )
            
            # Check for early ray termination in affected pixels
            # If accumulated opacity > threshold, mark pixels to skip future slices
            check_early_termination(intermediate_image)

    # 6. Warp Intermediate Image to Final Image
    final_image = apply_warp_matrix(intermediate_image, view_direction)
    
    return final_image

def build_rle_for_axis(volume, axis, transfer_function):
    rle_data = {}
    # Iterate over slices perpendicular to the axis
    for slice_idx in range(num_slices):
        run_list = []
        # Iterate over scan lines in the slice
        for line_idx in range(num_lines):
            # Get voxels for this scan line
            voxels = get_scan_line(volume, axis, slice_idx, line_idx)
            
            # Classify and compress
            current_run_start = 0
            current_transparent = is_transparent(voxels[0], transfer_function)
            
            for i in range(1, len(voxels)):
                is_trans = is_transparent(voxels[i], transfer_function)
                if is_trans != current_transparent:
                    run_list.append((current_run_start, i, current_transparent))
                    current_run_start = i
                    current_transparent = is_trans
            run_list.append((current_run_start, len(voxels), current_transparent))
        
        rle_data[slice_idx] = run_list
    return rle_data

def is_transparent(value, transfer_function):
    opacity = transfer_function(value)
    return opacity == 0.0
```

### Edge Cases and Pitfalls
*   **View Direction Alignment:** The shear coefficients require $v_{o,z} \neq 0$. If the view is exactly perpendicular to the principal axis, the shear becomes undefined. The algorithm typically handles this by selecting a different principal axis or handling the singularity separately [S1, unofficial][S2, unofficial].
*   **Sampling Rate Limit:** Due to CPU speed limitations, the sampling rate $s$ is often capped (e.g., at 5 in the cited experiments) to maintain reasonable frame times, which may reduce image quality compared to GPU implementations [S2, unofficial].
*   **Transfer Function Changes:** If the transfer function changes, the RLE structures must be recomputed, as the classification of voxels into transparent/non-transparent runs changes [S4].

## Subtopics

### [Fast Shear-Warp with Runlength Encoding](6737-fast-shear-warp-with-runlength-encoding.md)
Fast Shear-Warp with Runlength Encoding is a variant that explicitly leverages the coherence of volume data and image pixels using RLE to skip transparent voxels and occluded pixels, assuming a fixed opacity transfer function. It builds on [CPU-based Shear-Warp](1161-cpu-based-shear-warp.md) by achieving high performance, rivaling raycasting and splatting at unity magnification while maintaining sub-second rendering times for moderately-sized datasets. Unlike general ray-casting, it achieves early ray termination via opaque pixel links with low overhead, as offsets are updated only when a pixel first becomes opaque.

### Simple Shear-Warp Algorithm
The Simple Shear-Warp Algorithm is a variant that does not use acceleration data structures (except for the runlength-encoded intermediate image for early ray termination), resulting in a low memory footprint [S4]. It has specific optimized versions for 'nearest neighbour' and 'linear' interpolation, implemented as templated functions `CompositeIntermediateNearestSimple` and `CompositeIntermediateLinearSimple` [S4].

## Operating characteristics
*   **Time Complexity:** The rendering time depends on the number of non-transparent voxels and the sampling rate. The RLE compression reduces the number of samples processed by skipping transparent runs [S1, unofficial][S2, unofficial].
*   **Memory Complexity:** The algorithm requires storing three RLE structures, one for each principal axis. The size of these structures depends on the volume dimensions and the sparsity of the data [S1, unofficial][S2, unofficial][S3, unofficial].
*   **Measured Performance:** On a 3.8GHz Pentium 4 with 2GB RAM, rendering times for a Skull model (320x320x46) ranged from 22s to 107s for sampling rates $s=1$ to $s=5$, respectively. For a Knee model (512x512x73), times ranged from 110s to 370s for the same sampling rates [S2, unofficial].
*   **Strengths:** Low memory footprint compared to methods using 3D structures like octrees; efficient skipping of transparent regions; suitable for interactive classification if RLE is recomputed [S1, unofficial][S2, unofficial][S4].
*   **Limitations:** Significantly slower than GPU-based methods; limited sampling rate due to CPU speed; requires recomputation of RLE if transfer function changes [S2, unofficial][S4].
*   **When to Use:** When GPU hardware is unavailable or when memory constraints prevent the use of more complex acceleration structures, and the dataset is moderate in size [S2, unofficial].

## Background
The CPU-based Shear-Warp algorithm is an implementation of the Shear-warp method [S2, unofficial][S3, unofficial]. The original Shear-warp paper suggested using 3D data structures like octrees, but the CPU implementation described in the sources explicitly avoids these, relying instead on RLE compression for each principal axis [S1, unofficial][S2, unofficial][S3, unofficial]. The method was used as a baseline for comparison with GPU-based volume rendering techniques [S2, unofficial][S3, unofficial]. The specific implementation details, including the use of Hounsfield scale classification and linear transfer functions, are derived from medical imaging contexts [S2, unofficial][S3, unofficial].

## Sources
[S1] GPU-Based Volume Rendering for Medical Imagery, https://inria.hal.science/inria-00461484v1, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Rendering for Medical Imagery, https://scholarly.org/pdf/display/gpu-based-volume-rendering-for-medical-imagery, fetched 2026-10-28 (unofficial)
[S3] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
[S4] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
