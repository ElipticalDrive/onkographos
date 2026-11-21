# Bounding Volume Empty-Space Culling
Also called: Bounding volume strategy
## Overview
Bounding volume empty-space culling restricts volume reconstruction and rendering computation to the "effective" area where voxel values are valid, bypassing empty space outside this region [S1, unofficial]. It differs from its sibling [Parallel Ray Casting with Segment Lists](2565-parallel-ray-casting-with-segment-lists.md) by using a geometric bounding volume as a depth buffer to guide the pipeline, rather than explicit segment lists, and from [Visibility Culling](6226-visibility-culling.md) by focusing on the validity of the reconstruction domain itself rather than fragment-level occlusion [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Projection data (sinogram), viewing direction, and volume grid dimensions.
- Output: A depth buffer defining the valid reconstruction bounds for the current view, used to cull invalid voxels during reconstruction and rendering.

**Data Structures:**
- **Bounding Volume:** A geometric primitive (truncated circle for 2D, truncated cylinder for 3D) representing the effective reconstruction area [S1, unofficial].
- **Depth Buffer:** A buffer storing the slice boundaries of the bounding volume, used to restrict computation [S1, unofficial].

**Steps:**
1. **Calculate Initial Bounding Volume:** Determine the effective reconstruction area based on the assumption of parallel beam reconstruction. For a 2D square volume slice, this is a truncated circle; for a 3D cubic volume, it is a truncated cylinder [S1, unofficial].
2. **Slice Bounding Volume:** Slice the calculated bounding volume according to the current viewing direction [S1, unofficial].
3. **Generate Depth Buffer:** Use the resulting bounding volume slices to create a depth buffer [S1, unofficial].
4. **Restrict Computation:** Use the depth buffer to guide the reconstruction pipeline, ensuring that computation is restricted to the effective area throughout the entire process [S1, unofficial].

**Formulas:**
The effective area is defined by the geometry of the projection system. For parallel beam reconstruction, the valid voxel region is bounded by the projection of the object's extent. No explicit formula is provided in the sources for the truncation, but it is described as a truncated circle (2D) or truncated cylinder (3D) [S1, unofficial].

**Parameters:**
- Viewing direction: Determines the slicing of the bounding volume [S1, unofficial].
- Reconstruction grid: Defines the resolution of the volume [S1, unofficial].

**Edge Cases and Pitfalls:**
- The method assumes parallel beam reconstruction; deviations may affect accuracy [S1, unofficial].
- The bounding volume is an approximation; it may include some empty space or exclude valid voxels if the object shape is highly irregular [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_bounding_volume(grid_dims, projection_type):
    """
    Calculate the initial bounding volume based on parallel beam assumption.
    Returns a geometric primitive (truncated circle/cylinder).
    """
    if projection_type == "2D":
        # Truncated circle for 2D square volume slice
        radius = max(grid_dims.x, grid_dims.y) / 2.0
        return TruncatedCircle(radius)
    elif projection_type == "3D":
        # Truncated cylinder for 3D cubic volume
        radius = max(grid_dims.x, grid_dims.y) / 2.0
        height = grid_dims.z
        return TruncatedCylinder(radius, height)
    else:
        raise ValueError("Unsupported projection type")

def slice_bounding_volume(bounding_volume, viewing_direction):
    """
    Slice the bounding volume according to the viewing direction.
    Returns a list of slice boundaries.
    """
    # Implementation depends on the specific geometry and viewing direction
    # This step generates the depth buffer slices
    return slice_boundaries

def restrict_reconstruction(depth_buffer, reconstruction_pipeline):
    """
    Use the depth buffer to restrict computation within the effective area.
    """
    for slice in reconstruction_pipeline:
        if slice is within depth_buffer:
            process(slice)
        else:
            skip(slice)
```
## Subtopics

### Occupancy Mask Culling
A sophisticated preprocessing scheme that reconstructs a volume mask labeling all voxels in the shadow of non-zero projection data, which is then sliced per frame to drive reconstruction more accurately than a bounding sphere [S1, unofficial]. It is equivalent to empty space skipping, with the bounding volume method being a good approximation for many cases, while the occupancy mask is more accurate [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Reduces the number of voxels processed by culling those outside the effective area [S1, unofficial].
- **Memory Complexity:** Requires storage for the bounding volume and depth buffer [S1, unofficial].
- **Strengths:** Simple to implement; effective for regular object shapes [S1, unofficial].
- **Limitations:** Approximation may not be accurate for complex or irregular object shapes [S1, unofficial].
## Background
Originated in the context of GPU-accelerated Direct Volume Rendering (D2VR) [S1, unofficial]. The method is described in the paper "GPU-Accelerated D2VR" [S1, unofficial].
## Sources
[S1] GPU-Accelerated D2VR, https://doi.org/10.2312/vg/vg06/023-030, fetched 2026-10-26 (unofficial)
