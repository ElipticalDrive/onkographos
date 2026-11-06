# Fast Shear-Warp with Runlength Encoding

Also called: CompositeIntermediateLinearRLE, CompositeIntermediateNearestRLE, Fast shear-warp with RLE, Lacroute and Levoy method, Shear-Warp, Volpack

## Overview
Fast Shear-Warp with Runlength Encoding is a high-performance volume rendering method that achieves sub-second rendering times for moderately-sized datasets by skipping transparent voxels using runlength-encoded (RLE) data structures [S1, S2]. It rivals the image quality of raycasting and splatting at unity magnification but is distinguished by its reliance on pre-classified opacity runs and a fixed view-dependent sampling interval that precludes adaptive supersampling [S1].

## How it works
The method renders a volume by shearing the data such that rays are perpendicular to volume slices, traversing RLE-encoded runs to skip empty space, and warping the result to screen space [S1].

**Inputs and Outputs**
*   **Input:** A 3D volume dataset $V$ of dimensions $D_x \times D_y \times D_z$, a view matrix defining the camera orientation, and a scalar opacity transfer function $T(s)$ mapping scalar values $s$ to opacity $\alpha \in [0, 1]$ [S1, S2].
*   **Output:** A 2D screen image (framebuffer) with color and opacity values [S1].

**Data Structures**
The core acceleration structure is the RLE-encoded volume. For each of the three principal axes (X, Y, Z), a separate encoded volume is constructed [S1].
*   **RLE Volume Layout:** The volume is traversed along the viewing direction. Voxels are classified using $T(s)$. Consecutive voxels with the same classification (transparent or opaque) are grouped into runs.
*   **Run Entry:** Each run is stored as a tuple $(start\_index, length, opacity\_class)$, where $start\_index$ is the linear index in the volume array, $length$ is the number of voxels, and $opacity\_class$ indicates if the run is transparent (skip) or opaque (process) [S1, S2].
*   **Note:** The RLE structure must be recomputed whenever the voxel classification function (transfer function) changes [S2].

**Parameters**
*   **Magnification:** Rendering quality is optimal when object magnification is approximately unity [S1].
*   **Interpolation:** Bilinear interpolation is used to obtain sample values within traversed volume slices [S1].

**Algorithm Steps**
1.  **Pre-processing:** Construct three separate RLE-encoded volumes, one for each principal axis (X, Y, Z). This involves classifying every voxel using the current transfer function and grouping consecutive voxels with identical opacity status into runs [S1].
2.  **Shear:** Select the encoded volume corresponding to the principal axis most aligned with the view direction. Apply a shear transformation to the volume data such that the viewing rays become perpendicular to the volume slices [S1].
3.  **Cast Rays:** Traverse the sheared volume along the viewing rays.
    *   Use the RLE structure to skip transparent runs.
    *   For opaque runs, perform bilinear interpolation within the sheared volume slices to obtain sample values [S1].
    *   The distance between samples (DVRI, $\Delta s$) is view-dependent:
        *   $\Delta s = 1.0$ for axis-aligned views.
        *   $\Delta s = \sqrt{2} \approx 1.41$ for edge-on views.
        *   $\Delta s = \sqrt{3} \approx 1.73$ for corner-on views [S1].
    *   Note: $\Delta s$ cannot be varied to allow for supersampling along the ray [S1].
4.  **Warp:** Transform the volume-parallel baseplane image (the image formed by the sheared slices) into the final screen image using a warping transformation [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_fast_shear_warp_rle(volume, view_matrix, transfer_function):
    """
    Renders a volume using Fast Shear-Warp with RLE.
    volume: 3D array of scalar values.
    view_matrix: 4x4 matrix defining camera orientation.
    transfer_function: function mapping scalar -> (color, opacity).
    """
    # 1. Pre-processing: Build RLE structures for all 3 axes
    # This step is expensive and must be repeated if transfer_function changes.
    rle_x = build_rle(volume, axis='x', transfer_function)
    rle_y = build_rle(volume, axis='y', transfer_function)
    rle_z = build_rle(volume, axis='z', transfer_function)

    # 2. Shear: Determine best axis and shear the volume
    # Select axis with normal closest to view direction
    best_axis = select_best_axis(view_matrix, [rle_x, rle_y, rle_z])
    sheared_volume = shear_volume(volume, best_axis)
    rle = get_rle_for_axis(best_axis, [rle_x, rle_y, rle_z])

    # 3. Cast Rays: Traverse sheared volume and composite
    # Initialize framebuffer
    framebuffer = initialize_framebuffer()

    # Determine sample spacing based on view angle
    # DVRI varies from 1.0 (axis-aligned) to 1.73 (corner-on)
    dvri = calculate_dvri(view_matrix, best_axis)

    for screen_pixel in framebuffer:
        # Map screen pixel to ray in sheared volume space
        ray_origin, ray_direction = screen_to_ray(screen_pixel, view_matrix, best_axis)
        
        # Traverse RLE runs along the ray
        current_depth = 0
        accumulated_color = (0, 0, 0)
        accumulated_opacity = 0
        
        # Iterate through RLE runs
        for run in rle:
            if run.is_transparent:
                continue # Skip transparent runs
            
            # Calculate intersection of ray with this run
            run_start, run_length = run.start_index, run.length
            
            # Sample the volume within the run using bilinear interpolation
            # Note: Interpolation occurs in sheared slices
            sample_value = interpolate_bilinear(sheared_volume, ray_origin, ray_direction, dvri)
            
            # Apply transfer function
            color, opacity = transfer_function(sample_value)
            
            # Alpha compositing (front-to-back)
            # C_out = C_in + C_obj * alpha_obj * (1 - alpha_in)
            # alpha_out = alpha_in + alpha_obj * (1 - alpha_in)
            accumulated_color = accumulated_color + color * opacity * (1 - accumulated_opacity)
            accumulated_opacity = accumulated_opacity + opacity * (1 - accumulated_opacity)
            
            if accumulated_opacity >= 1.0:
                break # Fully opaque, stop processing

        framebuffer[screen_pixel] = accumulated_color

    # 4. Warp: Transform baseplane image to screen image
    # The warping step maps the sheared coordinate system to the screen
    final_image = warp_image(framebuffer, view_matrix, best_axis)
    return final_image

def build_rle(volume, axis, transfer_function):
    """
    Constructs RLE structure for a specific axis.
    """
    runs = []
    # Traverse volume along the specified axis
    # Group consecutive voxels with same opacity classification
    current_run_start = 0
    current_opacity_class = None
    
    for voxel in volume_traversed_along_axis(volume, axis):
        _, opacity = transfer_function(voxel)
        opacity_class = 'opaque' if opacity > 0 else 'transparent'
        
        if opacity_class != current_opacity_class:
            if current_opacity_class is not None:
                runs.append(Run(current_run_start, current_index - current_run_start, current_opacity_class))
            current_run_start = current_index
            current_opacity_class = opacity_class
            
    # Append final run
    runs.append(Run(current_run_start, current_index - current_run_start, current_opacity_class))
    return runs
```

## Operating characteristics
*   **Time Complexity:** Rendering time is sub-second for moderately-sized datasets [S1]. The cost is dominated by the pre-processing step (RLE construction) which is view-independent but transfer-function-dependent, and the ray traversal which skips transparent regions [S1, S2].
*   **Memory Complexity:** Requires storing three RLE structures (one per axis) plus the original volume [S1]. The RLE structures have a low memory footprint compared to octrees but require recomputation on classification changes [S2].
*   **Performance:** Quality rivals raycasting and splatting at unity magnification [S1].
*   **Strengths:** High performance for static transfer functions; effective skipping of empty space [S1, S2].
*   **Limitations:**
    *   Image quality degrades at off-axis views due to the fixed DVRI potentially violating the Nyquist theorem (DVRI ranges from 1.0 to 1.73) [S1].
    *   Supersampling along the ray is not supported because DVRI is view-dependent and fixed per view [S1].
    *   Interactive transfer function variation is challenging because the RLE structure must be fully recomputed when the classification function changes [S1, S2].
    *   Requires a framebuffer color/RGBA resolution of at least 16 bits for good quality [S1].
*   **Mitigation:** Image quality at off-axis views can be improved by using a volume with extra interpolated slices (Volpack's standard solution) [S1].

## Background
The method is based on the Shear-Warp algorithm, originally proposed by Lacroute and Levoy [S1, S2]. It is implemented in the Volpack distribution from Stanford [S1]. The RLE acceleration variant is implemented in VTK as `CompositeIntermediateNearestRLE` and `CompositeIntermediateLinearRLE` [S2].

## Sources
[S1] A Practical Evaluation of Popular Volume Rendering ..., https://courses.cs.duke.edu/spring03/cps296.8/papers/PracticalEvaluationofVolumeRenderingAlgorithms.pdf, fetched 2026-10-29
[S2] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
