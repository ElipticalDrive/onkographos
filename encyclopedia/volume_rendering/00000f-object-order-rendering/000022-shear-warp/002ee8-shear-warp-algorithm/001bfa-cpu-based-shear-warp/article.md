# CPU-based Shear-Warp

Also called: CPU Shear-Warp

## Overview
CPU-based Shear-Warp is a volume rendering method that implements the Shear-Warp algorithm on a central processing unit, utilizing Run-Length Encoding (RLE) to accelerate traversal by skipping empty voxels [S1, S2, S3, unofficial]. It distinguishes itself from the original Shear-Warp proposal by relying on 1D RLE compression along principal axes rather than 3D data structures like octrees, and from GPU-based variants by its sequential, CPU-bound execution model [S1, S2, S3, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset (typically medical imaging data in DICOM format) and a viewing direction defined by the view vector $\vec{v_0}$.
*   **Output:** A 2D rendered image.

**Data Structures:**
The method constructs three separate data structures, one for each principal coordinate axis (X, Y, Z) [S1, S2, S3, unofficial]. Each data structure is a set of lists of runs [S1, S2, S3, unofficial].
*   **Run List:** Corresponds to one scan line of one slice [S1, S2, S3, unofficial].
*   **Run:** A contiguous sequence of voxels classified as either:
    1.  **Completely transparent:** Considered subsets of empty voxels [S1, S2, S3, unofficial].
    2.  **Not completely transparent:** Contains at least one non-transparent voxel [S1, S2, S3, unofficial].

**Preprocessing (Classification and Compression):**
1.  **Classification:** Voxels are classified using the Hounsfield scale (a quantitative scale for describing radiodensity) [S1, S2, S3, unofficial].
2.  **Transfer Function:** A linear transfer function with defined lower and upper bounds converts the input data values into opacities [S1, S2, S3, unofficial].
3.  **Compression:** For each coordinate axis and for each slice, each scan line of voxels is compressed into a list of runs based on the computed opacities [S1, S2, S3, unofficial].

**Rendering Steps:**
1.  **Shear Calculation:** Compute the shear factors $s_x$ and $s_y$ based on the view vector $\vec{v_0} = (v_{0.x}, v_{0.y}, v_{0.z})$ [S1, S2, unofficial]:
    $$ s_x = -\frac{v_{0.x}}{v_{0.z}} $$
    $$ s_y = -\frac{v_{0.y}}{v_{0.z}} $$
2.  **Traversal:** Traverse the RLE data structures slice by slice and line by line [S1, S2, S3, unofficial].
3.  **Skipping:** Ignore completely transparent runs (empty voxels) during traversal to skip empty space [S1, S2, S3, unofficial].
4.  **Compositing:** Perform volume compositing for the non-transparent runs.
5.  **Lighting:** Apply a Phong model for direct lighting [S2, unofficial].

**Parameters:**
*   **Sampling Rate ($s$):** The number of sample points along a viewing ray. The CPU implementation is limited to a maximum sampling rate of 5 due to performance constraints [S2, unofficial].
*   **Transfer Function Bounds:** Lower and upper bounds for the linear curve [S1, S2, S3, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def cpu_shear_warp(volume_data, view_vector, lower_bound, upper_bound, sampling_rate):
    # 1. Preprocessing: Classify and Compress
    # Construct 3 data structures, one for each axis (X, Y, Z)
    # Each structure is a list of lists of runs
    # runs[axis][slice][line] = [ (start_index, length, is_transparent), ... ]
    
    axis_structures = []
    for axis in [0, 1, 2]: # X, Y, Z
        axis_runs = []
        # Iterate over slices perpendicular to the axis
        for slice_idx in range(volume_size[axis]):
            slice_runs = []
            # Iterate over lines in the slice
            for line_idx in range(volume_size[(axis + 1) % 3]):
                # Extract scan line voxels
                scan_line = get_scan_line(volume_data, axis, slice_idx, line_idx)
                
                # Classify voxels using Hounsfield scale and linear transfer function
                opacities = []
                for voxel in scan_line:
                    value = voxel.value
                    if value < lower_bound:
                        opacity = 0.0
                    elif value > upper_bound:
                        opacity = 1.0
                    else:
                        # Linear interpolation between 0 and 1
                        opacity = (value - lower_bound) / (upper_bound - lower_bound)
                    opacities.append(opacity)
                
                # Compress into runs
                runs = []
                current_run_start = 0
                current_run_transparent = (opacities[0] == 0.0)
                
                for i in range(1, len(opacities)):
                    is_transparent = (opacities[i] == 0.0)
                    if is_transparent != current_run_transparent:
                        runs.append((current_run_start, i - current_run_start, current_run_transparent))
                        current_run_start = i
                        current_run_transparent = is_transparent
                # Append final run
                runs.append((current_run_start, len(opacities) - current_run_start, current_run_transparent))
                
                slice_runs.append(runs)
            axis_runs.append(slice_runs)
        axis_structures.append(axis_runs)

    # 2. Shear Calculation
    vx, vy, vz = view_vector
    sx = -vx / vz
    sy = -vy / vz

    # 3. Rendering
    image = create_empty_image(width, height)
    
    # Traverse the primary axis structure (typically Z for perspective projection)
    # Note: The specific traversal order depends on the projection matrix decomposition
    # implied by the Shear-Warp algorithm [Parent]
    for slice_idx in range(volume_size[Z]):
        for line_idx in range(volume_size[X]): # Assuming X is the other horizontal axis
            runs = axis_structures[Z][slice_idx][line_idx]
            
            for start, length, is_transparent in runs:
                if is_transparent:
                    continue # Skip empty voxels
                
                # Perform compositing for this run
                # Apply shear and warp transformations to map to image coordinates
                # Sample along the ray with 'sampling_rate' steps
                # Apply Phong lighting model
                composite_run(image, start, length, sx, sy, slice_idx, line_idx, sampling_rate)
                
    return image
```

## Subtopics

### [Fast Shear-Warp with Runlength Encoding](6737-fast-shear-warp-with-runlength-encoding.md)
Fast Shear-Warp with Runlength Encoding is a high-performance volume rendering method that achieves sub-second rendering times for moderately-sized datasets by skipping transparent voxels using runlength-encoded (RLE) data structures. It rivals the image quality of raycasting and splatting at unity magnification but is distinguished by its reliance on pre-classified opacity runs and a fixed view-dependent sampling interval that precludes adaptive supersampling.

### Simple Shear-Warp Algorithm
A shear-warp volume rendering method that does not use acceleration data structures (except for the runlength-encoded intermediate image for early ray termination), resulting in a low memory footprint [S4]. It has specific optimized versions for 'nearest neighbour' and 'linear' interpolation, implemented as templated functions `CompositeIntermediateNearestSimple` and `CompositeIntermediateLinearSimple` [S4].

## Operating characteristics
*   **Performance:** The method is significantly slower than GPU-based methods [S2, unofficial]. On a 3.8Ghz Pentium 4 with 2GB RAM, rendering times for a Skull model (320x320x46 voxels) ranged from 22s to 107s for sampling rates $s=1$ to $s=5$ [S2, unofficial]. For a Knee model (512x512x73 voxels), times ranged from 110s to 370s for the same sampling rates [S2, unofficial].
*   **Sampling Limit:** Due to speed constraints, the CPU implementation is limited to a maximum sampling rate of 5 [S2, unofficial].
*   **Memory:** Uses 3 data structures (one per axis) of run lists. Does not use 3D data structures like octrees [S1, S2, S3, unofficial].
*   **Strengths:** Efficiently skips empty voxels via RLE, reducing computation for sparse volumes [S1, S2, S3, unofficial]. Low memory footprint compared to octree-based approaches [S4].
*   **Limitations:** Sequential CPU execution limits frame rates compared to GPU parallelism [S2, unofficial]. Limited sampling rate restricts image quality at higher resolutions or for detailed features [S2, unofficial].

## Background
The Shear-Warp algorithm was originally proposed for interactive volume rendering on the CPU [S3, unofficial]. The CPU-based implementation described here was developed to provide a baseline for comparison with GPU-based variants (GPU Shear-Warp and GPU Ray Marching) [S2, unofficial]. It was tested on medical imaging data (TDM scanner, DICOM format) [S2, unofficial]. The original Shear-Warp paper suggested using 3D data structures like octrees, but this CPU implementation opted for 1D RLE compression along principal axes instead [S1, S2, S3, unofficial].

## Sources
[S1] GPU-Based Volume Rendering for Medical Imagery, https://inria.hal.science/inria-00461484v1, fetched 2026-10-26 (unofficial)
[S2] GPU-Based Volume Rendering for Medical Imagery, https://scholarly.org/pdf/display/gpu-based-volume-rendering-for-medical-imagery, fetched 2026-10-28 (unofficial)
[S3] (PDF) GPU-Based Volume Rendering for Medical Imagery - ResearchGate, https://www.researchgate.net/publication/228912978_GPU-Based_Volume_Rendering_for_Medical_Imagery, fetched 2026-10-28 (unofficial)
[S4] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
