# Compute Minimal Sample Distance Per Pixel

## Overview
This method computes the minimal sample distance per pixel for volume rendering by projecting each dataset cell axis onto the viewport, accounting for camera and volume transforms [S1, unofficial]. It serves as a geometric heuristic to determine the appropriate step size for ray marching, ensuring that samples are dense enough to resolve the smallest visible voxel features [S1, unofficial]. It differs from [Safe Raymarching Step Size](1462-safe-raymarching-step-size.md) by relying on explicit axis projection rather than conservative bounds, and from [Error Estimate](2768-error-estimate.md) by using geometric projection instead of field value derivatives.

## How it works
**Inputs:**
- `renderer`: A pointer to the rendering context, providing camera position, orientation, and viewport dimensions [S1, unofficial].
- `volume`: A pointer to the volume object, providing the dataset geometry (extent, spacing) and transformation matrix [S1, unofficial].

**Data Structures:**
- `CellFlag`: An integer indicating the scalar data type (e.g., point vs. cell scalars). If `CellFlag == 2` (field data), the method fails [S1, unofficial].
- `scalars`: The scalar data array associated with the volume. Must have 1 component or 4 components with `IndependentComponents` set to false [S1, unofficial].

**Steps:**
1. **Validation:** Check if `scalars` exist. If not, abort [S1, unofficial]. Check if `scalars` are field data (`CellFlag == 2`). If so, abort with error "Only point or cell scalar support" [S1, unofficial]. Verify component count is 1 or 4 (non-independent) [S1, unofficial].
2. **Transform Update:** Ensure the volume's transformed input data is up to date by calling `UpdateInformation()`, `SetUpdateExtentToWholeExtent()`, and `Update()` on the relevant input objects [S1, unofficial].
3. **Axis Projection:** For each of the three dataset cell axes (x, y, z):
   a. Determine the axis vector in world space using the volume's transformation matrix and spacing.
   b. Project this axis vector onto the viewport plane using the camera's projection matrix and viewport dimensions.
   c. Calculate the projected length of the axis in pixel units.
4. **Minimal Distance Calculation:** The minimal sample distance is derived from the minimum projected axis length. This value represents the smallest resolvable feature size in pixels for the current view [S1, unofficial].
5. **Degeneracy Check:** If any axis is degenerate (aligned with the view direction, resulting in near-zero projected length), set `IgnoreSampleDistancePerPixel` to true. This indicates that the geometric heuristic is unreliable for this view [S1, unofficial].

**Parameters:**
- `IgnoreSampleDistancePerPixel`: Boolean flag. Set to true if axes are degenerate [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_minimal_sample_distance_per_pixel(renderer, volume):
    # 1. Validation
    scalars = volume.get_scalars()
    if scalars is None:
        raise Error("No scalars found on input.")
    
    cell_flag = volume.get_cell_flag()
    if cell_flag == 2:
        raise Error("Only point or cell scalar support - found field scalars instead.")
    
    num_components = scalars.get_number_of_components()
    independent_components = volume.get_property().get_independent_components()
    
    if not (num_components == 1 or (num_components == 4 and independent_components == 0)):
        raise Error("Only one component scalars, or four component with non-independent components.")
    
    # 2. Transform Update
    # Ensure transformed input is updated
    volume.get_transformed_input().update_information()
    volume.get_transformed_input().set_update_extent_to_whole_extent()
    volume.get_transformed_input().update()
    
    # 3. Axis Projection
    # Get axis vectors in world space
    axes = volume.get_axes_in_world_space() # Returns list of 3 vectors
    
    min_projected_length = infinity
    
    for axis in axes:
        # Project axis onto viewport
        # This involves transforming the axis vector by the camera's projection matrix
        # and mapping to viewport coordinates
        projected_axis = project_axis_to_viewport(axis, renderer.get_camera(), renderer.get_viewport())
        
        # Calculate length in pixels
        length = projected_axis.get_length_in_pixels()
        
        if length < min_projected_length:
            min_projected_length = length
            
    # 4. Degeneracy Check
    ignore_sample_distance = False
    if min_projected_length < epsilon: # epsilon is a small threshold
        ignore_sample_distance = True
        
    return min_projected_length, ignore_sample_distance
```

## Subtopics

### [Projected Voxel Size / LOD Calculation](4189-projected-voxel-size-lod-calculation.md)
This method computes a scalar metric representing the projected size of a voxel relative to the camera, primarily used for Level of Detail (LOD) decisions and step size adaptation in distance-field ray marching. It evaluates the angular spread of the voxel as seen from the camera center to provide a normalized measure of voxel appearance in viewing space.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of axes (constant 3 for 3D data) plus the cost of matrix multiplications for projection.
- **Memory Complexity:** Constant, requiring storage for a few vectors and scalars.
- **Strengths:** Provides a direct geometric measure of resolvable detail. Simple to implement given camera and volume transforms.
- **Limitations:** Fails or becomes unreliable when axes are aligned with the view direction (degenerate). Does not account for the actual data values or transfer function, only geometry.
- **When to use:** When a quick geometric estimate of step size is needed and the view is not aligned with dataset axes.
- **When not to use:** When axes are degenerate (use fallback or ignore). When data-dependent step sizing is required.

## Background
This method is implemented in the `vtkMultiVolumeRayCaster` mapper within the VTK library ecosystem [S1, unofficial]. It is part of the broader effort to optimize volume rendering performance by adapting sample spacing to the viewer's perspective.

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
