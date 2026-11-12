# Clipped Cropping Region Planes

Also called: ClipCroppingRegionPlanes

## Overview
Clipped Cropping Region Planes is a preprocessing step that clamps user-defined axis-aligned cropping planes to the volume's bounding box, ensuring the resulting clipping region is valid and contained within the data bounds. It sets apart from its parent method, [Cropping](1648-cropping.md), by enforcing geometric validity before rendering, preventing undefined behavior when planes fall outside the dataset.

## How it works
**Inputs and Outputs:**
- Input: `CroppingRegionPlanes` (array of 6 doubles), volume bounds.
- Output: `ClippedCroppingRegionPlanes` (array of 6 doubles).

**Data Structures:**
- `volBounds[6]`: Array of 6 doubles representing the volume bounding box. Indices 0-1 are min/max X, 2-3 are min/max Y, 4-5 are min/max Z.
- `CroppingRegionPlanes[6]`: User-defined planes. Indices 0-1 are min/max X, 2-3 are min/max Y, 4-5 are min/max Z.
- `ClippedCroppingRegionPlanes[6]`: Resulting clipped planes.

**Steps:**
1. Retrieve volume bounds into `volBounds[6]` using `this->GetInput(0)->GetBounds(volBounds)` [S1, unofficial].
2. For each axis $i \in \{0, 1, 2\}$:
   - **Min Plane:** If `CroppingRegionPlanes[2*i] < volBounds[2*i]`, set `ClippedCroppingRegionPlanes[2*i] = volBounds[2*i]`; else set `ClippedCroppingRegionPlanes[2*i] = CroppingRegionPlanes[2*i]`.
   - **Max Plane:** If `CroppingRegionPlanes[2*i+1] > volBounds[2*i+1]`, set `ClippedCroppingRegionPlanes[2*i+1] = volBounds[2*i+1]`; else set `ClippedCroppingRegionPlanes[2*i+1] = CroppingRegionPlanes[2*i+1]`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def clip_cropping_region_planes(cropping_planes, vol_bounds):
    """
    Clamps cropping planes to volume bounds.
    
    Args:
        cropping_planes: list of 6 floats [min_x, max_x, min_y, max_y, min_z, max_z]
        vol_bounds: list of 6 floats [min_x, max_x, min_y, max_y, min_z, max_z]
        
    Returns:
        list of 6 floats: clipped planes
    """
    clipped = [0.0] * 6
    for i in range(3):
        # Min plane for axis i
        min_idx = 2 * i
        max_idx = 2 * i + 1
        
        # Clamp min plane to volume min bound
        if cropping_planes[min_idx] < vol_bounds[min_idx]:
            clipped[min_idx] = vol_bounds[min_idx]
        else:
            clipped[min_idx] = cropping_planes[min_idx]
            
        # Clamp max plane to volume max bound
        if cropping_planes[max_idx] > vol_bounds[max_idx]:
            clipped[max_idx] = vol_bounds[max_idx]
        else:
            clipped[max_idx] = cropping_planes[max_idx]
            
    return clipped
```

**Parameters:**
- None.

**Edge Cases and Pitfalls:**
- Precondition: Cropping must be enabled and planes must be ordered ($0<1, 2<3, 4<5$). If not, the parent `Render()` method silently stops [S1, unofficial].
- If the user-defined region is entirely outside the volume bounds, the clipped region will be invalid (min > max) unless additional logic handles empty regions. The sources do not specify behavior for this case [G].

## Subtopics

## Operating characteristics
- Time complexity: $O(1)$, as it performs a fixed number of comparisons.
- Memory complexity: $O(1)$, requiring only a small array for the clipped planes.
- Strengths: Ensures robustness by preventing invalid clipping regions.
- Limitations: Does not handle non-axis-aligned clipping or complex shapes.

## Background
Originated in VTK's `vtkGPUVolumeRayCastMapper` as a method to clamp user-defined cropping planes to the volume's bounding box. It was intended to be moved to `vtkVolumeMapper` after further testing [S1, unofficial].

## Sources
[S1] midas-journal/midas-journal-856, https://github.com/midas-journal/midas-journal-856, fetched 2026-10-26 (unofficial)
