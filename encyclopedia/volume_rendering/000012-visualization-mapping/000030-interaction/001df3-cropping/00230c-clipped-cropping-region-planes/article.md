# Clipped Cropping Region Planes

Also called: ClipCroppingRegionPlanes, Cropping Region Plane Clipping, Volume Bounding Box Clipping

## Overview
Clipped Cropping Region Planes is a preprocessing step for volume rendering that constrains user-defined cropping planes to the physical bounds of the input volume dataset [S2, unofficial]. It solves the problem of invalid or out-of-bounds clipping regions by computing the intersection of user-defined cropping planes with the volume's bounding box to ensure valid clipping regions before rendering [S1, unofficial]. Unlike its sibling [Cropping](1648-cropping.md), which defines the logical clipping region, this method ensures the region is geometrically valid and contained within the data before rendering proceeds [S1, S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** The input volume dataset (typically `vtkImageData`), the user-defined cropping planes array `CroppingRegionPlanes[6]`, and the volume's bounding box bounds `volBounds[6]` [S1, S2, unofficial].
*   **Output:** The clipped cropping planes array `ClippedCroppingRegionPlanes[6]` [S1, S2, unofficial].

**Data Structures:**
*   `volBounds[6]`: An array of 6 doubles representing the axis-aligned bounding box of the volume. Indices 0-1 correspond to the minimum and maximum X coordinates, 2-3 to Y, and 4-5 to Z [S1, S2, unofficial].
*   `CroppingRegionPlanes[6]`: An array of 6 doubles representing the user-defined clipping planes. Indices 0-1 define the min/max X planes, 2-3 for Y, and 4-5 for Z [S1, S2, unofficial].
*   `ClippedCroppingRegionPlanes[6]`: An array of 6 doubles storing the result [S1, S2, unofficial].

**Steps:**
1.  **Validate Input:** Assert that the input volume exists (`this->GetInput(0) != 0`) [S1, S2, unofficial].
2.  **Retrieve Bounds:** Obtain the volume's bounding box into `volBounds[6]` using the dataset's `GetBounds` method (`this->GetInput(0)->GetBounds(volBounds)`) [S1, S2, unofficial].
3.  **Validate Cropping Order:** Assert that cropping is enabled and the planes are ordered correctly: `CroppingRegionPlanes[0] < CroppingRegionPlanes[1]`, `CroppingRegionPlanes[2] < CroppingRegionPlanes[3]`, and `CroppingRegionPlanes[4] < CroppingRegionPlanes[5]` [S1, S2, unofficial].
4.  **Clip Min Planes:** For each axis $i \in \{0, 2, 4\}$ (representing X, Y, Z minimums): if `CroppingRegionPlanes[i] < volBounds[i]`, set `ClippedCroppingRegionPlanes[i] = volBounds[i]`; otherwise, set `ClippedCroppingRegionPlanes[i] = CroppingRegionPlanes[i]` [S1, S2, unofficial].
5.  **Clip Max Planes:** For each axis $i \in \{1, 3, 5\}$ (representing X, Y, Z maximums): if `CroppingRegionPlanes[i] > volBounds[i]`, set `ClippedCroppingRegionPlanes[i] = volBounds[i]`; otherwise, set `ClippedCroppingRegionPlanes[i] = CroppingRegionPlanes[i]` [S1, S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clip_cropping_region_planes(cropping_region_planes, vol_bounds):
    """
    Clamps user-defined cropping planes to the volume's bounding box.
    
    Args:
        cropping_region_planes (list[float]): 6-element list [min_x, max_x, min_y, max_y, min_z, max_z].
        vol_bounds (list[float]): 6-element list [min_x, max_x, min_y, max_y, min_z, max_z].
        
    Returns:
        list[float]: Clipped cropping planes.
    """
    # Precondition checks [S1, S2, unofficial]
    assert len(cropping_region_planes) == 6
    assert len(vol_bounds) == 6
    
    # Assert ordering: min < max for each axis
    assert cropping_region_planes[0] < cropping_region_planes[1], "X planes not ordered"
    assert cropping_region_planes[2] < cropping_region_planes[3], "Y planes not ordered"
    assert cropping_region_planes[4] < cropping_region_planes[5], "Z planes not ordered"
    
    clipped_planes = [0.0] * 6
    
    # Clip minimum planes (indices 0, 2, 4)
    for i in [0, 2, 4]:
        if cropping_region_planes[i] < vol_bounds[i]:
            clipped_planes[i] = vol_bounds[i]
        else:
            clipped_planes[i] = cropping_region_planes[i]
            
    # Clip maximum planes (indices 1, 3, 5)
    for i in [1, 3, 5]:
        if cropping_region_planes[i] > vol_bounds[i]:
            clipped_planes[i] = vol_bounds[i]
        else:
            clipped_planes[i] = cropping_region_planes[i]
            
    return clipped_planes
```

**Parameters:**
*   Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Invalid Ordering:** If the user-defined planes are not ordered (min > max), the method fails the precondition check. The parent `Render()` method may silently stop if this check is not enforced by the caller [S2, unofficial].
*   **Out-of-Bounds Input:** If a user plane is outside the volume bounds, it is clamped to the nearest bound [S1, S2, unofficial].
*   **Empty Volume:** If the volume bounds are degenerate (e.g., min == max), the clipping logic still applies, potentially resulting in a zero-thickness slab [S1, unofficial].

## Subtopics

### [Polygon-based Cropping](11787-polygon-based-cropping.md)
Supports arbitrary convex or non-convex polygonal shapes in screen space, unlike the axis-aligned planar cuts of this method. Choose when free-form region selection is required.

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Not given in the sources.
*   **Limitations:** Not given in the sources.
*   **When to use:** Not given in the sources.
*   **When not to use:** Not given in the sources.

## Background
This method originates from the Visualization Toolkit (VTK) library, specifically within volume rendering mappers such as `vtkGPUMultiVolumeRayCastMapper` [S1, S2, unofficial]. It is part of the VTK volume rendering pipeline, ensuring that cropping operations are geometrically valid before being passed to the rendering engine [S1, unofficial]. The implementation is found in VTK source code repositories and related documentation [S1, S2, unofficial].

## Sources
[S1] karlkrissian/vtkMultiVolRen, https://github.com/karlkrissian/vtkMultiVolRen, fetched 2026-10-26 (unofficial)
[S2] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
