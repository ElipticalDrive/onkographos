# Compute Minimal Sample Distance Per Pixel

## Overview
This method computes the minimal sample distance for each pixel by projecting the dataset's cell axes onto the viewport, accounting for camera and volume transformations [S1, unofficial]. It serves as a geometric heuristic to determine the smallest viable step size for ray marching, ensuring that the sampling density respects the underlying voxel resolution [S1, unofficial]. Unlike adaptive methods that adjust step size based on local density values during rendering, this approach calculates a static, geometry-driven minimum bound per pixel based on the current view [S1, unofficial].

## How it works
**Inputs:**
- `renderer`: A pointer to the rendering context, providing camera position, orientation, and viewport dimensions [S1, unofficial].
- `volume`: A pointer to the volume object, providing dataset geometry (extent, spacing) and transformation matrices [S1, unofficial].

**Outputs:**
- A scalar value representing the minimal sample distance for the current pixel/camera configuration [S1, unofficial].

**Data Structures:**
The method relies on the internal state of the `vtkRenderer` and `vtkVolume` objects. Specifically, it accesses:
- Camera transformation matrix (model-view-projection) [S1, unofficial].
- Volume transformation matrix [S1, unofficial].
- Dataset extent and spacing (voxel dimensions) [S1, unofficial].

**Steps:**
1. Retrieve the camera and viewport state from the `renderer` [S1, unofficial].
2. Retrieve the dataset geometry and transformation from the `volume` [S1, unofficial].
3. Project each axis of the dataset cell (voxel) onto the viewport plane [S1, unofficial].
4. Calculate the projected length of these axes in screen space [S1, unofficial].
5. Determine the minimal sample distance based on the smallest projected axis dimension [S1, unofficial].
6. **Degeneracy Check:** If any dataset axis is degenerate (i.e., aligned with the view direction, resulting in zero projected length), set the flag `IgnoreSampleDistancePerPixel` to `true` [S1, unofficial]. This indicates that the geometric projection method cannot reliably determine a step size for this configuration [S1, unofficial].

**Parameters:**
- `IgnoreSampleDistancePerPixel`: A boolean flag. When `true`, the renderer should fall back to a default or fixed step size, ignoring the computed minimal distance [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_minimal_sample_distance(renderer, volume):
    # Get camera and viewport state
    camera = renderer.get_camera()
    viewport = renderer.get_viewport()
    
    # Get volume geometry and transformation
    volume_transform = volume.get_transformation_matrix()
    dataset_extent = volume.get_extent()
    dataset_spacing = volume.get_spacing()
    
    # Project cell axes onto viewport
    # Axes are typically defined by the dataset spacing and orientation
    axes = [
        dataset_spacing[0] * volume_transform.get_column(0),
        dataset_spacing[1] * volume_transform.get_column(1),
        dataset_spacing[2] * volume_transform.get_column(2)
    ]
    
    projected_lengths = []
    for axis in axes:
        # Project axis vector onto the view plane
        # This involves transforming the axis by the view-projection matrix
        # and measuring its length in screen space
        projected_axis = project_to_viewport(axis, camera, viewport)
        projected_lengths.append(length(projected_axis))
        
    # Find the minimal projected length
    if not projected_lengths:
        return 0.0
        
    min_distance = min(projected_lengths)
    
    # Check for degeneracy (axis aligned with view direction)
    # If any projected length is effectively zero, the axis is parallel to view
    is_degenerate = any(length(axis) < epsilon for axis in axes) or \
                    any(proj_len < epsilon for proj_len in projected_lengths)
                    
    if is_degenerate:
        # Set flag to ignore this calculation
        renderer.set_ignore_sample_distance_per_pixel(True)
        # Return a default or fallback value, or 0 to indicate invalidity
        return 0.0
        
    return min_distance
```

**Edge Cases and Pitfalls:**
- **Degenerate Axes:** If a dataset axis is aligned with the view direction, its projection onto the viewport is zero or near-zero. The method explicitly handles this by setting `IgnoreSampleDistancePerPixel` to `true` [S1, unofficial]. Failure to handle this may result in division by zero or invalid step sizes [S1, unofficial].
- **Precision:** Floating-point precision issues may arise when calculating projections for very small or very large datasets. [G] Use a small epsilon threshold for degeneracy checks.

## Subtopics

### Projected Voxel Size / LOD Calculation
This variant computes a scalar metric representing the projected size of a voxel relative to the camera, primarily used for Level of Detail (LOD) decisions and step size adaptation [S2, unofficial]. It differs from the primary method by using a dot-product-based metric rather than direct axis projection lengths [S2, unofficial].

**Definition:** Computes a scalar $S$ representing the projected size of a voxel [S2, unofficial].

**Steps:**
1. Compute vectors from the camera position to the lower-left ($LL$) and upper-right ($UR$) corners of the voxel's half-size box [S2, unofficial].
2. Normalize these vectors to obtain $\mathbf{u}_{LL}$ and $\mathbf{u}_{UR}$ [S2, unofficial].
3. Calculate the metric $S = 1 - \mathbf{u}_{LL} \cdot \mathbf{u}_{UR}$ [S2, unofficial].

**Formula:**
$$ S = 1 - \mathbf{u}_{LL} \cdot \mathbf{u}_{UR} $$
where $\mathbf{u}_{LL}$ and $\mathbf{u}_{UR}$ are normalized vectors from the camera to the voxel corners [S2, unofficial].

**Pitfalls:**
- The metric returns 0 when the camera is at the voxel center (dot product equals 1) [S2, unofficial].
- It increases as the viewing angle widens [S2, unofficial].
- Lacks explicit clamping or fallback for degenerate near-plane cases, as noted in the source implementation [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
- Time: $O(1)$ per pixel, as it involves a fixed number of matrix-vector multiplications and projections [S1, unofficial].
- Memory: $O(1)$ auxiliary memory, relying on existing renderer and volume data structures [S1, unofficial].

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Scales linearly with the number of pixels, as the calculation is performed per pixel [S1, unofficial].

**Strengths and Limitations:**
- **Strengths:** Provides a geometrically accurate lower bound for step size based on voxel resolution [S1, unofficial]. Simple to implement given access to camera and volume transforms [S1, unofficial].
- **Limitations:** Does not account for local density variations or opacity transfer functions [S1, unofficial]. Fails for degenerate views (axes aligned with view direction) [S1, unofficial].

**When to Use:**
- Use when a static, geometry-based minimum step size is required for ray marching [S1, unofficial].
- Use when LOD or voxel visibility is determined by screen-space projection [S2, unofficial].

**When Not to Use:**
- Do not use when adaptive sampling based on density is required [S1, unofficial].
- Do not use if the view direction may align with dataset axes without handling degeneracy [S1, unofficial].

## Background
This method originates from the VTK (Visualization Toolkit) ecosystem, specifically within the `vtkMultiVolumeRayCastMapper` implementation [S1, unofficial]. It is part of the volume rendering pipeline's step size optimization logic, ensuring that ray marching respects the underlying data resolution [S1, unofficial]. The projected voxel size metric is also discussed in the context of voxel visualization and LOD techniques [S2, unofficial].

## Sources
[S1] bozorgi/VTKMultiVolumeRayCaster, https://github.com/bozorgi/VTKMultiVolumeRayCaster, fetched 2026-10-26 (unofficial)
[S2] apecoraro/VoxVizThesis, https://github.com/apecoraro/VoxVizThesis, fetched 2026-10-27 (unofficial)
