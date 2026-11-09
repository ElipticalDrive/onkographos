# Sample Opacities from Mask

Also called: TensorBase Alpha Computation, Volume Alpha

## Overview
This method samples opacity values from a precomputed 3D alpha mask volume for a set of 3D point positions, mapping the raw mask values to a specified opacity range [S1, unofficial]. It enables volume rendering to incorporate spatially varying opacity constraints derived from external supervision or geometry masks [S2, unofficial].

## How it works
**Inputs:**
- `mask`: A 3D volume tensor representing the alpha mask.
- `points`: A tensor of 3D coordinates with shape $[N, 3]$, where $N$ is the number of points.
- `min_opacity`: Scalar, minimum opacity value (default 0.0) [S1, unofficial].
- `max_opacity`: Scalar, maximum opacity value (default 1.0) [S1, unofficial].

**Outputs:**
- `opacities`: A tensor of shape $[N]$ containing the mapped opacity values for each point.

**Data Structures:**
- The `mask` is stored as a dense 3D grid supporting trilinear interpolation [G].
- `points` are normalized or scaled to match the coordinate system of the mask volume [G].

**Steps:**
1. Sample alpha values from the mask at the given point locations [S2, unofficial].
2. Map the raw sampled values to the target opacity range using the linear transformation: $o_i = \text{min\_opacity} + v_i \times (\text{max\_opacity} - \text{min\_opacity})$, where $v_i$ is the raw value for point $i$ and $o_i$ is the final opacity [S1, unofficial].
3. Call `sample_intensities_from_volume` with the mask, enabling footprint pooling [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sample_opacities_from_mask(mask, points, min_opacity=0.0, max_opacity=1.0):
    # Step 1: Sample alpha values from the mask at the given locations
    # Source does not provide implementation for sample_intensities_from_volume; standard trilinear interpolation is used [G].
    raw_opacities = trilinear_interpolate(mask, points)
    
    # Step 2: Map raw mask values to opacity
    opacities = min_opacity + raw_opacities * (max_opacity - min_opacity)
    
    return opacities

def trilinear_interpolate(volume, points):
    # Standard trilinear interpolation implementation [G]
    # Returns raw values v_i for each point i
    pass
```

**Parameters:**
- `min_opacity`: Minimum opacity value to ensure visibility (default 0.0) [S1, unofficial].
- `max_opacity`: Maximum opacity value to prevent complete occlusion (default 1.0) [S1, unofficial].

**Edge Cases and Pitfalls:**
- Out-of-bounds points may return undefined or clamped values depending on the interpolation implementation [G].
- Coordinate system mismatches between `points` and the mask volume will cause incorrect sampling [G].

## Subtopics

### Update Opacities from Mask
This variant updates opacity values for points based on their current positions using a mask volume [S1, unofficial]. It ensures points are in $[N, 3]$ format, computes `mask_min` and `mask_max` from the mask tensor, and calls `sample_opacities_from_mask` to retrieve the updated opacities [S1, unofficial]. It is chosen when point positions change dynamically and require re-evaluation of opacity based on the mask [S1, unofficial].

## Operating characteristics
Not given in the sources.

## Background
Not given in the sources.

## Sources
[S1] ValentinKraft/vgs, https://github.com/ValentinKraft/vgs, fetched 2026-10-27 (unofficial)
[S2] https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, https://github.com/ZhenyangLiu/Collaborative-Neural-Radiance-Fields-for-Novel-View-Synthesis, fetched 2026-10-29 (unofficial)
