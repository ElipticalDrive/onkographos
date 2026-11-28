# Bipolar Scalar-to-Color Mapping with Level Set Support

Also called: PointAttributeGenerator

## Overview
This method maps scalar voxel values to RGB colors using a bipolar lookup table that treats positive and negative values independently, supporting both continuous interpolation and discrete level-set selection [S1, unofficial]. It differs from standard unipolar transfer functions by maintaining separate color ranges for positive and negative domains, allowing distinct visual encoding for opposing scalar signs [S1, unofficial].

## How it works
**Inputs:**
- `mPoints`: Array of point positions (world coordinates), stored as interleaved floats `[x0, y0, z0, x1, y1, z1, ...]` [S1, unofficial].
- `mGrid`: A scalar grid providing access to voxel values at integer indices [S1, unofficial].
- `mAccessor`: Grid accessor object to retrieve scalar values at voxel coordinates [S1, unofficial].
- `minValue`: The minimum scalar value in the grid, used for color mapping [S1, unofficial].
- `maxValue`: The maximum scalar value in the grid, used for color mapping [S1, unofficial].
- `mZeroValue`: The threshold separating positive and negative domains [S1, unofficial].
- `mColorMap`: Array of 4 `Vec3` colors used for interpolation: index 0 is positive minimum, index 1 is negative minimum, index 2 is positive maximum, index 3 is negative maximum [S1, unofficial].
- `isLevelSet`: Boolean flag indicating if the grid is a level set; if `true`, colors are selected directly from `mColorMap` based on sign; if `false`, colors are interpolated [S1, unofficial].
- `mNormals`: Optional output array for normals; if provided, normals are computed and stored [S1, unofficial].

**Outputs:**
- `mColors`: Array of RGB colors corresponding to each point in `mPoints` [S1, unofficial].
- `mNormals`: (Optional) Array of normal vectors corresponding to each point [S1, unofficial].

**Data Structures:**
- `mOffset[2]`: Array of two floats; `mOffset[0]` for positive values, `mOffset[1]` for negative values [S1, unofficial].
- `mScale[2]`: Array of two floats; `mScale[0]` for positive values, `mScale[1]` for negative values [S1, unofficial].
- `w`: Float weight for interpolation [S1, unofficial].

**Steps:**
1. **Initialize Scaling Parameters:**
   - Calculate `mOffset[0]` and `mScale[0]` for positive values:
     $$ mOffset[0] = \min(mZeroValue, minValue) $$
     $$ mScale[0] = \frac{1.0}{| \max(mZeroValue, maxValue) - mOffset[0] |} $$
   - Calculate `mOffset[1]` and `mScale[1]` for negative values:
     $$ mOffset[1] = \min(mZeroValue, minValue) $$
     $$ mScale[1] = \frac{1.0}{| \max(mZeroValue, maxValue) - mOffset[1] |} $$
2. **Iterate Over Points:**
   - Process points in parallel (e.g., using `tbb::blocked_range`) [S1, unofficial].
   - For each point index `n`:
     - Retrieve position `pos` from `mPoints` at indices `3*n`, `3*n+1`, `3*n+2` [S1, unofficial].
     - Convert `pos` to voxel index `ijk` using `mGrid.worldToIndex(pos)` and cast to integer coordinates [S1, unofficial].
     - Retrieve scalar `value` at `ijk` using `mAccessor.getValue(ijk)` [S1, unofficial].
3. **Compute Color:**
   - If `value < mZeroValue` (negative domain):
     - If `isLevelSet` is `true`:
       - Set `color = mColorMap[1]` (negative min color) [S1, unofficial].
     - Else:
       - Compute weight: $ w = (value - mOffset[1]) \times mScale[1] $ [S1, unofficial].
       - Interpolate between `mColorMap[1]` (negative min) and `mColorMap[0]` (positive min):
         $$ color = w \cdot mColorMap[0] + (1.0 - w) \cdot mColorMap[1] $$ [S1, unofficial].
   - If `value >= mZeroValue` (non-negative domain):
     - If `isLevelSet` is `true`:
       - Set `color = mColorMap[2]` (positive max color) [S1, unofficial].
     - Else:
       - Compute weight: $ w = (value - mOffset[0]) \times mScale[0] $ [S1, unofficial].
       - Interpolate between `mColorMap[3]` (negative max) and `mColorMap[2]` (positive max):
         $$ color = w \cdot mColorMap[2] + (1.0 - w) \cdot mColorMap[3] $$ [S1, unofficial].
4. **Store Color:**
   - Store `color` components into `mColors` at indices `3*n`, `3*n+1`, `3*n+2` [S1, unofficial].
5. **Compute Normals (Optional):**
   - If `mNormals` is provided:
     - Maintain a counter `voxelNum` (incremented for each point) [S1, unofficial].
     - If `voxelNum % 2 == 0`:
       - Compute normal `normal` at `ijk` using a `NormalGenerator` (e.g., finite differences on the grid) [S1, unofficial].
     - Store `normal` components into `mNormals` at indices `3*n`, `3*n+1`, `3*n+2` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

class Vec3:
    def __init__(self, x=0.0, y=0.0, z=0.0):
        self.x, self.y, self.z = x, y, z
    def __mul__(self, s):
        return Vec3(self.x * s, self.y * s, self.z * s)
    def __add__(self, other):
        return Vec3(self.x + other.x, self.y + other.y, self.z + other.z)
    def __getitem__(self, i):
        return (self.x, self.y, self.z)[i]

def normalize(v):
    mag = math.sqrt(v.x**2 + v.y**2 + v.z**2)
    if mag == 0: return Vec3(0.0, -1.0, 0.0)
    return Vec3(v.x / mag, v.y / mag, v.z / mag)

def bipolar_scalar_to_color_mapping(points, grid, accessor, min_value, max_value, zero_value, color_map, is_level_set, compute_normals=False):
    # Step 1: Initialize scaling parameters
    offset = [0.0, 0.0]
    scale = [0.0, 0.0]
    offset[0] = min(zero_value, min_value)
    scale[0] = 1.0 / abs(max(zero_value, max_value) - offset[0])
    offset[1] = min(zero_value, min_value)
    scale[1] = 1.0 / abs(max(zero_value, max_value) - offset[1])
    
    colors = []
    normals = [] if compute_normals else None
    voxel_num = 0
    normal_cache = Vec3(0.0, -1.0, 0.0)
    
    for n, pos in enumerate(points):
        # Step 2: Convert world position to voxel index
        voxel_pos = grid.world_to_index(pos)
        ijk = (int(voxel_pos.x), int(voxel_pos.y), int(voxel_pos.z))
        
        # Retrieve scalar value
        value = accessor.get_value(ijk)
        
        # Step 3: Compute color
        if value < zero_value:
            if is_level_set:
                color = color_map[1]
            else:
                w = (value - offset[1]) * scale[1]
                color = w * color_map[0] + (1.0 - w) * color_map[1]
        else:
            if is_level_set:
                color = color_map[2]
            else:
                w = (value - offset[0]) * scale[0]
                color = w * color_map[2] + (1.0 - w) * color_map[3]
        
        # Step 4: Store color
        colors.extend([color.x, color.y, color.z])
        
        # Step 5: Compute normals (optional)
        if compute_normals:
            if voxel_num % 2 == 0:
                # Standard central difference normal computation [G]
                try:
                    v_c = accessor.get_value(ijk)
                    v_xp = accessor.get_value((ijk[0]+1, ijk[1], ijk[2]))
                    v_xm = accessor.get_value((ijk[0]-1, ijk[1], ijk[2]))
                    v_yp = accessor.get_value((ijk[0], ijk[1]+1, ijk[2]))
                    v_ym = accessor.get_value((ijk[0], ijk[1]-1, ijk[2]))
                    v_zp = accessor.get_value((ijk[0], ijk[1], ijk[2]+1))
                    v_zm = accessor.get_value((ijk[0], ijk[1], ijk[2]-1))
                    normal_cache = normalize(Vec3(v_xp - v_xm, v_yp - v_ym, v_zp - v_zm))
                except Exception:
                    normal_cache = Vec3(0.0, -1.0, 0.0)
            normals.extend([normal_cache.x, normal_cache.y, normal_cache.z])
        
        voxel_num += 1
        
    return colors, normals
```

**Parameters:**
- `minValue`, `maxValue`: The minimum and maximum scalar values in the grid, used for color mapping [S1, unofficial].
- `mZeroValue`: The threshold separating positive and negative domains [S1, unofficial].
- `colorMap`: Array of 4 `Vec3` colors used for interpolation: index 0 is positive minimum, index 1 is negative minimum, index 2 is positive maximum, index 3 is negative maximum [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Normals Accuracy:** Normals are computed only for every other voxel (`voxelNum % 2 == 0`) and reused for the next point to save computation, which may lead to inaccurate normals if points are not aligned with voxel centers or if the grid has high-frequency details [S1, unofficial].
- **Color Index Mapping:** The mapping of `colorMap` indices to min/max values is implicit and depends on the caller's setup [S1, unofficial].
- **Precision:** Floating-point precision in weight calculation `w` should be considered for very small or large scalar ranges [G].

## Subtopics

### DEMIP with Color Sphere
An extension of DEMIP that uses a 3D spherical texture map to provide a color cue for depth perception based on the 3D position of the iso-surface determined by a second ray casting [S2]. It initializes a spherical texture map with distinct colors for the front and back regions (e.g., red and blue), interpolated across the sphere surface [S2]. For a given 3D point, it constructs a vector from the center of the bounding box to the 3D point, intersects the bounding sphere with a ray in that direction to sample the texture color (`colorsph`), and blends it with the DEMIP color using the formula:
$$ color = color_{DEMIP} \times (1.0 - sphw) + colorsph \times sphw $$
where `sphw` is the weight given to the sphere color, controlling the intensity of the color mapping effect [S2]. In a user study, DEMIP with Color Sphere achieved a mean Likert score of 4.99 (95% CI: 6.16) for stereo rendering depth perception, compared to 3.66 for DEMIP and 2.33 for MIP [S2]. Unlike pseudo-chromadepth which aims to perceive distances, the Color Sphere aims to reinforce overall spatial distribution perception and provides a manipulable visual reference [S2].

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of points, with parallelization using `tbb::blocked_range` improving performance on multi-core systems [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Strengths:** Efficient bipolar color mapping, support for level-set discrete coloring, and optional normal computation [S1, unofficial].
- **Limitations:** Normal approximation may be inaccurate due to skipping every other voxel, and implicit color index mapping requires careful setup [S1, unofficial].

## Background
This method is part of the OpenVDB library for point attribute generation [S1, unofficial]. It is used for visualizing scalar fields with distinct positive and negative values [S1, unofficial].

## Sources
[S1] AcademySoftwareFoundation/openvdb, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
[S2] Depth-Enhanced Maximum Intensity Projection, https://doi.org/10.2312/vg/vg10/093-100, fetched 2026-10-29
