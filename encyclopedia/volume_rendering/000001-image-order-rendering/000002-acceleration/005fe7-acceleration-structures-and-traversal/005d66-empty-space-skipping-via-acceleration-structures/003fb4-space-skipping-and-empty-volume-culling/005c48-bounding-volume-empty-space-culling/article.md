# Bounding Volume Empty-Space Culling
Also called: Bounding volume strategy
## Overview
Bounding volume empty-space culling restricts volume reconstruction and rendering computation to the "effective" area where voxel values are valid, bypassing empty space outside this region [S1, unofficial]. It builds on [Space Skipping and Empty Volume Culling](10063-space-skipping-and-empty-volume-culling.md) by using a geometric bounding volume as a depth buffer to guide the pipeline [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: Projection data, viewing direction, and volume grid dimensions [S1, unofficial].
- Output: A depth buffer defining the valid reconstruction bounds for the current view, used to cull invalid voxels during reconstruction and rendering [S1, unofficial].

**Data Structures:**
- Bounding Volume: A geometric primitive representing the effective reconstruction area (a truncated circle for 2D square volume slices, or a truncated cylinder for 3D cubic volumes) [S1, unofficial].
- Depth Buffer: A buffer storing the slice boundaries of the bounding volume, used to restrict computation [S1, unofficial].

**Steps:**
1. Calculate Initial Bounding Volume: Determine the effective reconstruction area based on the assumption of parallel beam reconstruction [S1, unofficial].
2. Slice Bounding Volume: Slice the calculated bounding volume according to the current viewing direction [S1, unofficial].
3. Generate Depth Buffer: Use the resulting bounding volume slices to create a depth buffer [S1, unofficial].
4. Restrict Computation: Use the depth buffer to guide the reconstruction pipeline, ensuring that computation is restricted to the effective area throughout the entire process [S1, unofficial].

**Formulas:**
The sources do not provide explicit mathematical formulas for the truncation boundaries [S1, unofficial]. The effective area is geometrically defined as a truncated circle for 2D square slices or a truncated cylinder for 3D cubic volumes under parallel beam assumptions [S1, unofficial].

**Parameters:**
- Viewing direction: Determines the slicing of the bounding volume [S1, unofficial].
- Not given in the sources.

**Edge Cases and Pitfalls:**
- The method assumes parallel beam reconstruction; deviations may affect accuracy [S1, unofficial].
- The bounding volume is an approximation; it may include some empty space or exclude valid voxels if the object shape is highly irregular [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_bounding_volume(grid_dims, projection_type):
    """
    Calculate the initial bounding volume based on parallel beam assumption.
    Returns a dictionary representing the geometric primitive.
    """
    if projection_type == "2D":
        radius = max(grid_dims["x"], grid_dims["y"]) / 2.0
        return {"type": "truncated_circle", "radius": radius}
    elif projection_type == "3D":
        radius = max(grid_dims["x"], grid_dims["y"]) / 2.0
        height = grid_dims["z"]
        return {"type": "truncated_cylinder", "radius": radius, "height": height}
    else:
        raise ValueError("Unsupported projection type")

def slice_bounding_volume(bounding_volume, viewing_direction):
    """
    Slice the bounding volume according to the viewing direction.
    Returns a list of slice boundaries (depth buffer).
    """
    # Implementation depends on the specific geometry and viewing direction
    # This step generates the depth buffer slices
    # Not given in the sources.
    return []

def restrict_reconstruction(depth_buffer, reconstruction_pipeline):
    """
    Use the depth buffer to restrict computation within the effective area.
    """
    processed_slices = []
    for current_slice in reconstruction_pipeline:
        if current_slice["z"] in depth_buffer:
            processed_slices.append(current_slice)
    return processed_slices
```
## Subtopics

### Occupancy Mask Culling
A sophisticated preprocessing scheme that reconstructs a volume mask labeling all voxels in the shadow of non-zero projection data, which is then sliced per frame to drive reconstruction more accurately than a bounding sphere [S1, unofficial]. It is equivalent to empty space skipping, with the bounding volume method being a good approximation for many cases, while the occupancy mask is more accurate [S1, unofficial].

## Operating characteristics
- Time Complexity: Reduces the number of voxels processed by culling those outside the effective area [S1, unofficial].
- Memory Complexity: Requires storage for the bounding volume and depth buffer [S1, unofficial].
- Strengths: Simple to implement; effective for regular object shapes [S1, unofficial].
- Limitations: Approximation may not be accurate for complex or irregular object shapes [S1, unofficial].
## Background
The method is described in the paper "GPU-Accelerated D2VR" [S1, unofficial].
## Sources
[S1] GPU-Accelerated D2VR, https://doi.org/10.2312/vg/vg06/023-030, fetched 2026-10-26 (unofficial)
