# Bipolar Scalar-to-Color Mapping with Level Set Support

Also called: PointAttributeGenerator

## Overview
This method maps scalar voxel values to RGB colors using a bipolar lookup table that treats positive and negative values independently, supporting both continuous interpolation and discrete level-set selection. It differs from standard unipolar transfer functions by maintaining separate color ranges for positive and negative domains, allowing distinct visual encoding for opposing scalar signs [S1, unofficial].

## How it works
**Inputs:**
- `mPoints`: Array of point positions (world coordinates), stored as interleaved floats `[x0, y0, z0, x1, y1, z1, ...]`.
- `mGrid`: A scalar grid (e.g., OpenVDB grid) providing access to voxel values at integer indices.
- `mAccessor`: Grid accessor object to retrieve scalar values at voxel coordinates.
- `minValue`: Minimum scalar value in the grid domain.
- `maxValue`: Maximum scalar value in the grid domain.
- `mZeroValue`: The threshold separating positive and negative values (typically 0.0).
- `mColorMap`: Array of 4 colors (`Vec3s`):
  - Index 0: Color for positive minimum.
  - Index 1: Color for negative minimum.
  - Index 2: Color for positive maximum.
  - Index 3: Color for negative maximum.
- `isLevelSet`: Boolean flag. If `true`, colors are selected directly from `mColorMap` based on sign. If `false`, colors are interpolated.
- `mNormals`: Optional output array for normals. If provided, normals are computed and stored.

**Outputs:**
- `mColors`: Array of RGB colors corresponding to each point in `mPoints`.
- `mNormals`: (Optional) Array of normal vectors corresponding to each point.

**Data Structures:**
- `mOffset[2]`: Array of two floats. `mOffset[0]` for positive values, `mOffset[1]` for negative values.
- `mScale[2]`: Array of two floats. `mScale[0]` for positive values, `mScale[1]` for negative values.
- `w`: Float weight for interpolation.

**Steps:**
1. **Initialize Scaling Parameters:**
   - Calculate `mOffset[0]` and `mScale[0]` for positive values:
     $$ mOffset[0] = \min(mZeroValue, minValue) $$
     $$ mScale[0] = \frac{1.0}{| \max(mZeroValue, maxValue) - mOffset[0] |} $$
   - Calculate `mOffset[1]` and `mScale[1]` for negative values:
     $$ mOffset[1] = \min(mZeroValue, minValue) $$
     $$ mScale[1] = \frac{1.0}{| \max(mZeroValue, maxValue) - mOffset[1] |} $$

2. **Iterate Over Points:**
   - Process points in parallel (e.g., using `tbb::blocked_range`).
   - For each point index `n`:
     - Retrieve position `pos` from `mPoints` at indices `3*n`, `3*n+1`, `3*n+2`.
     - Convert `pos` to voxel index `ijk` using `mGrid.worldToIndex(pos)`. Cast to integer coordinates.
     - Retrieve scalar `value` at `ijk` using `mAccessor.getValue(ijk)`.

3. **Compute Color:**
   - If `value < mZeroValue` (negative domain):
     - If `isLevelSet` is `true`:
       - Set `color = mColorMap[1]` (negative min color).
     - Else:
       - Compute weight: $ w = (value - mOffset[1]) \times mScale[1] $
       - Interpolate between `mColorMap[1]` (negative min) and `mColorMap[0]` (positive min):
         $$ color = w \cdot mColorMap[0] + (1.0 - w) \cdot mColorMap[1] $$
   - If `value >= mZeroValue` (non-negative domain):
     - If `isLevelSet` is `true`:
       - Set `color = mColorMap[2]` (positive max color).
     - Else:
       - Compute weight: $ w = (value - mOffset[0]) \times mScale[0] $
       - Interpolate between `mColorMap[3]` (negative max) and `mColorMap[2]` (positive max):
         $$ color = w \cdot mColorMap[2] + (1.0 - w) \cdot mColorMap[3] $$

4. **Store Color:**
   - Store `color` components into `mColors` at indices `3*n`, `3*n+1`, `3*n+2`.

5. **Compute Normals (Optional):**
   - If `mNormals` is provided:
     - Maintain a counter `voxelNum` (incremented for each point).
     - If `voxelNum % 2 == 0`:
       - Compute normal `normal` at `ijk` using a `NormalGenerator` (e.g., finite differences on the grid).
     - Store `normal` components into `mNormals` at indices `3*n`, `3*n+1`, `3*n+2`.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def bipolar_scalar_to_color_mapping(points, grid, accessor, min_value, max_value, zero_value, color_map, is_level_set, compute_normals=False):
    """
    Maps scalar values to colors using bipolar lookup table.
    
    Args:
        points: List of [x, y, z] coordinates.
        grid: Grid object with worldToIndex method.
        accessor: Grid accessor with getValue method.
        min_value: Minimum scalar value in grid.
        max_value: Maximum scalar value in grid.
        zero_value: Threshold for positive/negative split.
        color_map: List of 4 Vec3 colors [pos_min, neg_min, pos_max, neg_max].
        is_level_set: Boolean, if True use discrete colors, else interpolate.
        compute_normals: Boolean, if True compute and store normals.
        
    Returns:
        colors: List of Vec3 colors.
        normals: List of Vec3 normals (if compute_normals is True).
    """
    # Step 1: Initialize scaling parameters
    offset = [0.0, 0.0]
    scale = [0.0, 0.0]
    
    # Positive values (index 0)
    offset[0] = min(zero_value, min_value)
    scale[0] = 1.0 / abs(max(zero_value, max_value) - offset[0])
    
    # Negative values (index 1)
    offset[1] = min(zero_value, min_value)
    scale[1] = 1.0 / abs(max(zero_value, max_value) - offset[1])
    
    colors = []
    normals = [] if compute_normals else None
    
    voxel_num = 0
    normal_cache = None
    
    for n, pos in enumerate(points):
        # Step 2: Convert world position to voxel index
        # Assuming grid.worldToIndex returns a Vec3-like object or tuple
        voxel_pos = grid.world_to_index(pos)
        ijk = (int(voxel_pos[0]), int(voxel_pos[1]), int(voxel_pos[2]))
        
        # Retrieve scalar value
        value = accessor.get_value(ijk)
        
        # Step 3: Compute color
        if value < zero_value:
            if is_level_set:
                color = color_map[1] # Negative min
            else:
                w = (value - offset[1]) * scale[1]
                # Interpolate between negative min (index 1) and positive min (index 0)
                color = w * color_map[0] + (1.0 - w) * color_map[1]
        else:
            if is_level_set:
                color = color_map[2] # Positive max
            else:
                w = (value - offset[0]) * scale[0]
                # Interpolate between negative max (index 3) and positive max (index 2)
                color = w * color_map[2] + (1.0 - w) * color_map[3]
        
        colors.append(color)
        
        # Step 5: Compute normals (optional)
        if compute_normals:
            if voxel_num % 2 == 0:
                # Compute normal using finite differences or similar
                # Placeholder for NormalGenerator logic
                normal_cache = compute_normal_at(ijk, accessor)
            normals.append(normal_cache)
        
        voxel_num += 1
        
    return colors, normals

def compute_normal_at(ijk, accessor):
    """
    Placeholder for normal computation.
    Typically uses finite differences on adjacent voxels.
    """
    # Example: central differences
    # This is a simplified example; actual implementation may vary
    try:
        v_center = accessor.get_value(ijk)
        v_x_plus = accessor.get_value((ijk[0]+1, ijk[1], ijk[2]))
        v_x_minus = accessor.get_value((ijk[0]-1, ijk[1], ijk[2]))
        v_y_plus = accessor.get_value((ijk[0], ijk[1]+1, ijk[2]))
        v_y_minus = accessor.get_value((ijk[0], ijk[1]-1, ijk[2]))
        v_z_plus = accessor.get_value((ijk[0], ijk[1], ijk[2]+1))
        v_z_minus = accessor.get_value((ijk[0], ijk[1], ijk[2]-1))
        
        grad_x = v_x_plus - v_x_minus
        grad_y = v_y_plus - v_y_minus
        grad_z = v_z_plus - v_z_minus
        
        normal = normalize(Vec3(grad_x, grad_y, grad_z))
        return normal
    except:
        return Vec3(0.0, -1.0, 0.0) # Default normal if computation fails
```

**Parameters:**
- `minValue`, `maxValue`: Typically the full range of the scalar field [S1, unofficial].
- `mZeroValue`: Typically 0.0, separating positive and negative domains [S1, unofficial].
- `colorMap`: 4 colors defining the bipolar range [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Normals Accuracy:** Normals are computed only for every other voxel (`voxelNum % 2 == 0`) and reused for the next point to save computation. This can lead to inaccurate normals if points are not aligned with voxel centers or if the grid has high-frequency details [S1, unofficial].
- **Color Index Mapping:** The mapping of `colorMap` indices to min/max values is implicit and depends on the caller's setup. Index 0 is positive min, 1 is negative min, 2 is positive max, 3 is negative max [S1, unofficial].
- **Precision:** Floating-point precision in weight calculation `w` should be considered for very small or large scalar ranges.

## Operating characteristics
- **Time Complexity:** $O(N)$ where $N$ is the number of points. Parallelization using `tbb::blocked_range` can improve performance on multi-core systems [S1, unofficial].
- **Memory Complexity:** $O(N)$ for storing colors and optionally normals [S1, unofficial].
- **Strengths:** Efficient bipolar color mapping, support for level-set discrete coloring, optional normal computation [S1, unofficial].
- **Limitations:** Normal approximation may be inaccurate due to skipping every other voxel [S1, unofficial]. Implicit color index mapping requires careful setup [S1, unofficial].

## Background
This method is part of the OpenVDB library, specifically in the `tools` module for point attribute generation [S1, unofficial]. It is used for visualizing scalar fields with distinct positive and negative values, such as density and void, or positive and negative charge distributions [S1, unofficial].

## Sources
[S1] OpenVDB PointAttributeGenerator, https://github.com/AcademySoftwareFoundation/openvdb, fetched 2026-10-27 (unofficial)
