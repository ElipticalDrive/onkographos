# Simple Shear-Warp Algorithm

Also called: CompositeIntermediateLinearSimple, CompositeIntermediateNearestSimple, Shear-Warp Volume Rendering

## Overview
The Simple Shear-Warp Algorithm is a CPU-based volume rendering method that traverses image and object space simultaneously by shearing the volume geometry so that orthographic rays become perpendicular to the base plane [S2, book]. It is distinguished from its siblings by its lack of complex acceleration structures (such as octrees or summed area tables), relying instead on a low-memory footprint and run-length encoding (RLE) of the intermediate image for early ray termination [S1]. It serves as the baseline implementation for the Shear-Warp family, optimized specifically for nearest-neighbor and linear interpolation variants [S1].

## How it works
The method transforms the volume rendering problem into an object-order traversal on a sheared grid, followed by a final warping step to Cartesian coordinates.

### Inputs and Outputs
*   **Input:** A 3D volume dataset $V$ with dimensions $D_x \times D_y \times D_z$. A view direction defined by an orthographic projection. A classification function mapping voxel values to opacity and color.
*   **Output:** A 2D image $I$ with dimensions $W \times H$.

### Data Structures
The algorithm uses a compressed volume representation. While the "Simple" variant does not use acceleration structures for the volume traversal itself (unlike the "Fast" variants), it utilizes RLE for the intermediate image to support early termination [S1]. The volume data is stored in three copies to support all view directions [S2, book].

**Volume Copy Layout:**
The volume is stored in three identical compressed copies, corresponding to the three principal axes (X, Y, Z). Each copy is run-length encoded to remove empty voxels [S2, book].
*   **Fields:**
    *   `voxel_value`: The scalar value of the voxel.
    *   `run_length`: The number of consecutive empty voxels preceding or following the current voxel (implementation detail of RLE).
*   **Size:** The size depends on the data classification; RLE can achieve a >10:1 reduction in voxel count [S2, book].

### Steps

1.  **Shear:**
    Shear the volume such that orthographic rays originating from the base plane become perpendicular to that plane. This transformation aligns the ray direction with one of the volume's principal axes (e.g., the Z-axis) [S2, book].

2.  **Precompute Weights:**
    Because the shear ensures that rays intersect voxels at consistent locations within each 2D plane, precompute the interpolation weights for bilinear interpolation once per plane [S2, book].
    *   For **Nearest Neighbor**: Weights are trivial (select the closest voxel).
    *   For **Linear**: Calculate the fractional offsets $(dx, dy)$ of the ray intersection within the voxel grid on the sheared plane. These offsets are constant for all rays traversing the same plane row [S2, book].

3.  **Traverse (Object-Order):**
    Perform an object-order traversal visiting voxels along each row in each plane in front-to-back order.
    *   There is a one-to-one correspondence between samples in a sheared plane and pixels on the image plane [S2, book].
    *   Iterate through the compressed volume copies. If the current view direction aligns with the Z-axis, use the Z-aligned copy.
    *   For each voxel $v$ with opacity $\alpha_v$ and color $C_v$:
        1.  Apply the precomputed interpolation weights to determine the effective sample value $S$ from the 2D plane neighbors.
        2.  Map $S$ to color $C$ and opacity $\alpha$ using the transfer function.
        3.  Composite $C$ and $\alpha$ into the intermediate image $I_{shear}$ using front-to-back alpha compositing:
            $$ C_{out} = C_{in} + C \cdot \alpha \cdot (1 - \alpha_{in}) $$
            $$ \alpha_{out} = \alpha_{in} + \alpha \cdot (1 - \alpha_{in}) $$
    *   **Early Termination:** If $\alpha_{out}$ reaches 1.0 (full opacity), skip the remaining samples for that pixel [S2, book]. The RLE of the intermediate image allows skipping pixels that are already fully opaque [S1].

4.  **Compress (RLE of Volume):**
    The volume is pre-processed or traversed using Run-Length Encoding (RLE) to skip empty voxels. The RLE structure indicates how many voxels are skipped, which corresponds to the number of pixels to skip in the image plane during the traversal [S2, book].

5.  **Warp:**
    Perform a final resampling (warping) operation to transform the image from the sheared space on the base plane to Cartesian space on the final image plane [S2, book]. This step corrects the geometric distortion introduced by the shear transformation.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def simple_shear_warp(volume, view_direction, width, height):
    """
    Implements the Simple Shear-Warp algorithm.
    volume: 3D array of voxel values.
    view_direction: Vector (dx, dy, dz) defining the orthographic view.
    width, height: Dimensions of the output image.
    """
    # 1. Determine which of the 3 principal axis copies to use.
    # The view direction is approximated to the nearest principal axis.
    # For orthographic, we pick the axis with the largest absolute component.
    abs_dx, abs_dy, abs_dz = abs(view_direction.x), abs(view_direction.y), abs(view_direction.z)
    
    if abs_dz >= abs_dx and abs_dz >= abs_dy:
        axis = 'Z'
    elif abs_dy >= abs_dx:
        axis = 'Y'
    else:
        axis = 'X'
        
    # 2. Shear the volume coordinates.
    # The shear matrix S transforms the volume such that rays are parallel to the chosen axis.
    # For a view direction (dx, dy, dz), the shear factors are:
    # sx = -dx/dz, sy = -dy/dz (assuming dz != 0)
    # This maps the volume into a sheared coordinate system.
    # [G] In practice, this is handled by adjusting the iteration order and interpolation offsets.
    
    # 3. Precompute interpolation weights.
    # Since rays are perpendicular to the base plane in sheared space,
    # the fractional part of the intersection within a voxel is constant for a given row.
    # Precompute these weights for each plane.
    weights = precompute_weights(volume, axis)
    
    # 4. Initialize intermediate image in sheared space.
    # Dimensions match the sheared base plane.
    intermediate_image = zeros(width, height)
    opacity_image = zeros(width, height)
    
    # 5. Traverse volume in object order.
    # Iterate through planes perpendicular to the view axis.
    for plane in volume.planes(axis):
        # Iterate through rows in the plane.
        for row in plane.rows:
            # Iterate through voxels in the row (front-to-back).
            for voxel in row.voxels:
                # Skip empty voxels if RLE is used.
                if voxel.is_empty:
                    continue
                
                # Get interpolated sample value using precomputed weights.
                sample_value = interpolate(voxel, weights)
                
                # Map to color and opacity.
                color, alpha = transfer_function(sample_value)
                
                # Alpha compositing.
                # C_out = C_in + C * alpha * (1 - alpha_in)
                # A_out = A_in + alpha * (1 - alpha_in)
                
                # [G] Map voxel coordinates to image coordinates.
                # In sheared space, there is a 1-to-1 mapping.
                u, v = voxel_to_image_coords(voxel, axis)
                
                if 0 <= u < width and 0 <= v < height:
                    current_alpha = opacity_image[v][u]
                    current_color = intermediate_image[v][u]
                    
                    # Front-to-back compositing
                    new_alpha = current_alpha + alpha * (1 - current_alpha)
                    new_color = current_color + color * alpha * (1 - current_alpha)
                    
                    intermediate_image[v][u] = new_color
                    opacity_image[v][u] = new_alpha
                    
                    # Early termination check
                    if new_alpha >= 1.0:
                        # Skip remaining samples for this pixel.
                        # [G] In a full implementation, the RLE of the intermediate image
                        # would be used to skip pixels, but here we just break the inner loop.
                        break

    # 6. Warp to Cartesian space.
    # Transform the intermediate image from sheared coordinates to final image coordinates.
    final_image = warp(intermediate_image, view_direction, width, height)
    
    return final_image
```

### Parameters
*   **Interpolation Type:** The algorithm has specific optimized versions for 'nearest neighbour' and 'linear' interpolation [S1].
*   **Number of Volume Copies:** Requires three copies of the compressed volume to allow front-to-back traversal from all view directions [S2, book].

### Edge Cases and Pitfalls
*   **Perspective Viewing:** If a perspective viewing transformation is used, the simple shear may not align rays perfectly perpendicular to the base plane for all pixels. This may require traversing all three compressed copies of the volume to achieve the correct traversal order [S2, book].
*   **Degenerate View Directions:** If the view direction is parallel to one of the principal axes, the shear factors become undefined (division by zero). The algorithm must handle these cases by directly using the axis-aligned copy without shear [G].
*   **Precision:** The precomputation of interpolation weights relies on the consistency of ray intersections. Floating-point precision errors in the shear transformation can lead to artifacts if not handled carefully [G].

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of non-empty voxels, assuming RLE compression is effective. The warping step is $O(W \times H)$.
*   **Memory Complexity:** Low memory footprint. Requires three copies of the volume data (compressed) and an intermediate image [S1]. No octrees or summed area tables are used [S1].
*   **Performance:** RLE can achieve a >10:1 reduction in voxels depending on data classification [S2, book]. This makes it efficient for datasets with large empty regions.
*   **Strengths:** Simple implementation, low memory usage, efficient for orthographic projections with empty space.
*   **Limitations:** Not suitable for perspective views without significant modifications (traversing all copies). Performance degrades if the volume has little empty space (RLE compression is ineffective).
*   **When to use:** Orthographic volume rendering with a need for low memory usage and simple implementation.
*   **When not to use:** Perspective rendering, or datasets with no empty space.

## Background
The shear-warp method was introduced by Lacroute and Levoy in 1994 [S2, book]. It is an efficient variant of ray casting that leverages the correspondence between samples and pixels in a sheared coordinate system. The Simple Shear-Warp Algorithm is a specific implementation within the VTK (Visualization Toolkit) framework, implemented as templated functions `CompositeIntermediateNearestSimple` and `CompositeIntermediateLinearSimple` [S1]. It is part of the Shear-Warp family, which also includes faster variants using RLE on the volume itself or classification-based acceleration structures [S1].

## Sources
[S1] Shear Warp, https://www.cg.tuwien.ac.at/courses/Visualisierung/2002-2003/Beispiel2/2-BrucknerSt-SeemannR/index2.htm, fetched 2026-10-29
[S2] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
